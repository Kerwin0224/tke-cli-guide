# 设置工作负载的资源限制（tccli）

> 对照官方：[设置工作负载的资源限制](https://cloud.tencent.com/document/product/457/32813) · page_id `32813`

## 概述

为 Kubernetes 工作负载配置 CPU 和内存的 `requests` 与 `limits`，实现资源分配和限制。`requests` 用于调度决策（kube-scheduler 按此值选择节点），`limits` 用于运行时强制约束（cgroup 按此值限制上限）。

## 前置条件

- [环境准备](../../../环境准备.md)：`tccli` 已配置，`kubectl` >= v1.28
- 需要 CAM 权限：`tke:DescribeClusters`、`tke:DescribeClusterKubeconfig`

```bash
# 检查集群状态
tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'
# expected: ClusterStatus "Running"

# 获取 kubeconfig
tccli tke DescribeClusterKubeconfig --region <Region> --ClusterId <ClusterId> \
    | jq -r '.Kubeconfig' > kubeconfig.yaml
export KUBECONFIG=$(pwd)/kubeconfig.yaml
```

> **注意**：kubectl 数据面当前因公网端点被组织级 CAM 策略拒绝（strategyId:240463971）而不可达。外网端点被 `tke:clusterExtranetEndpoint=true` 条件拦截，内网端点需通过 IOA/VPN 或同 VPC 内 CVM 访问。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 获取集群凭证 | `tccli tke DescribeClusterKubeconfig --region <Region> --ClusterId <ClusterId>` | 是 |
| 创建带资源限制的工作负载 | `kubectl apply -f deployment-with-resources.yaml` | 是 |
| 查看资源使用 | `kubectl top pods` | 是 |
| 查看资源配额 | `kubectl describe resourcequota` | 是 |
| 修改资源限制 | `kubectl edit deployment/<name>` | 否 |
| 设置资源限制（命令行） | `kubectl set resources deployment/<name> --limits=... --requests=...` | 否 |

## 操作步骤

### 步骤 1：创建带资源限制的 Deployment

YAML 清单：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: resource-demo
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: resource-demo
  template:
    metadata:
      labels:
        app: resource-demo
    spec:
      containers:
        - name: nginx
          image: nginx:alpine
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: "250m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
```

| 参数 | 说明 | 推荐值 |
|------|------|--------|
| `resources.requests.cpu` | 调度保证 CPU | 100m–500m |
| `resources.requests.memory` | 调度保证内存 | 128Mi–512Mi |
| `resources.limits.cpu` | CPU 硬上限 | 500m–2000m |
| `resources.limits.memory` | 内存硬上限（超限 OOMKilled） | 256Mi–2Gi |

```bash
kubectl apply -f deployment-with-resources.yaml
# expected: deployment.apps/resource-demo created
```

**预期输出**：

```text
deployment.apps/resource-demo created
```

### 步骤 2：查看资源使用

```bash
kubectl top pods -l app=resource-demo
# expected: CPU/MEM 列显示实际使用量
```

**预期输出**（示例）：

```text
NAME                           CPU(cores)   MEMORY(bytes)
resource-demo-7d8c4f9b6c-abc   2m           4Mi
resource-demo-7d8c4f9b6c-def   3m           5Mi
```

### 步骤 3：通过命令行修改资源限制

```bash
kubectl set resources deployment/resource-demo \
    --containers=nginx \
    --requests=cpu=200m,memory=256Mi \
    --limits=cpu=1000m,memory=1Gi
# expected: deployment.apps/resource-demo resource requirements updated
```

## 验证

### 控制面（tccli）

```bash
tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'
# expected: ClusterStatus "Running"
```

```json
{
  "TotalCount": "<TotalCount>",
  "Clusters": "<Clusters>",
  "ClusterId": "<ClusterId>",
  "ClusterName": "<ClusterName>",
  "ClusterDescription": "<ClusterDescription>",
  "ClusterVersion": "<ClusterVersion>"
}
```

### 数据面（kubectl）

```bash
kubectl get deployment resource-demo
# expected: READY 2/2

kubectl describe deployment resource-demo | grep -A4 "Limits\|Requests"
# expected: 显示资源配置

kubectl top pods -l app=resource-demo
# expected: CPU/MEM 用量在 limits 范围内
```

```text
NAME  STATUS  AGE
...
```

## 清理

```bash
kubectl delete deployment resource-demo
# expected: deployment.apps "resource-demo" deleted
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `Unable to connect to the server` | `kubectl cluster-info` 检查连通性 | 公网端点被 CAM 策略 strategyId:240463971 以 `tke:clusterExtranetEndpoint=true` 条件拒绝 | 通过 IOA/VPN 连接内网，或使用同 VPC CVM 执行 kubectl 命令 |
| `error: Metrics API not available` | `kubectl top pods` 失败 | metrics-server 未安装或未就绪 | 确认 kube-system 下 metrics-server Pod Running |

### 资源创建成功但状态异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Pod `Pending` | `kubectl describe pod <pod>` 查看 Events | `requests` 超过节点可用资源 | 降低 requests 或扩容节点 |
| Pod `OOMKilled` | `kubectl describe pod <pod>` 查看 Events | 内存超过 `limits` 上限 | 增加 memory limits |
| CPU Throttling | `kubectl top pods` 观察 CPU 使用 | CPU 超过 `limits` 上限 | 增加 CPU limits |
| `ResourceQuota` 拒绝创建 | `kubectl describe resourcequota` | 命名空间配额不足 | 增加 ResourceQuota 或释放资源 |

## 下一步

- [设置工作负载的调度规则](../设置工作负载的调度规则/tccli%20操作.md) — page_id `32814`
- [设置工作负载的健康检查](../设置工作负载的健康检查/tccli%20操作.md) — page_id `32815`

## 控制台替代

[容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) → 集群 → 工作负载 → 编辑 YAML → 修改 `resources` 字段。
