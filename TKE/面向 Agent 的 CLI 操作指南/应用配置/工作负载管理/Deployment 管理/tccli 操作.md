# Deployment 管理（tccli）

> 对照官方：[Deployment 管理](https://cloud.tencent.com/document/product/457/31705) · page_id `31705`

## 概述

Deployment 是 Kubernetes 控制器，管理 Pod 的声明式配置。通过 kubectl 创建、扩缩、更新和回滚 Deployment，实现无状态应用的滚动升级和弹性伸缩。

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
| 创建 Deployment | `kubectl apply -f deployment.yaml` | 是 |
| 查看 Deployment 列表 | `kubectl get deployment` | 是 |
| 查看详情 | `kubectl describe deployment/<name>` | 是 |
| 更新镜像 | `kubectl set image deployment/<name> <container>=<image>` | 否 |
| 调整副本数 | `kubectl scale deployment/<name> --replicas=<N>` | 是 |
| 滚动更新状态 | `kubectl rollout status deployment/<name>` | 是 |
| 版本回滚 | `kubectl rollout undo deployment/<name>` | 否 |
| 删除 Deployment | `kubectl delete deployment/<name>` | 是 |

## 操作步骤

### 步骤 1：创建 Deployment

YAML 清单：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deployment-example
  namespace: default
  labels:
    app: deployment-example
spec:
  replicas: 2
  selector:
    matchLabels:
      app: deployment-example
  template:
    metadata:
      labels:
        app: deployment-example
    spec:
      containers:
        - name: nginx
          image: nginx:latest
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 256Mi
```

```bash
kubectl apply -f deployment.yaml
# expected: deployment.apps/deployment-example created
```

**预期输出**：

```text
deployment.apps/deployment-example created
```

### 步骤 2：查看 Deployment 状态

```bash
kubectl get deployment deployment-example
# expected: READY 列显示期望副本数
```

```text
NAME  STATUS  AGE
...
```

```bash
kubectl describe deployment/deployment-example
# expected: 显示 Deployment 详情和事件
```

```text
NAME  STATUS  AGE
...
```

### 步骤 3：滚动更新镜像

```bash
kubectl set image deployment/deployment-example nginx=nginx:1.25
# expected: deployment.apps/deployment-example image updated

kubectl rollout status deployment/deployment-example
# expected: deployment "deployment-example" successfully rolled out
```

### 步骤 4：版本回滚

```bash
kubectl rollout history deployment/deployment-example
# expected: REVISION 列表

kubectl rollout undo deployment/deployment-example
# expected: deployment.apps/deployment-example rolled back
```

### 步骤 5：扩缩容

```bash
kubectl scale deployment/deployment-example --replicas=3
# expected: deployment.apps/deployment-example scaled
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
kubectl get deployment deployment-example
# expected: READY 2/2 或 3/3

kubectl get pods -l app=deployment-example
# expected: 关联 Pod Running
```

```text
NAME  STATUS  AGE
...
```

## 清理

```bash
kubectl delete deployment deployment-example
# expected: deployment.apps "deployment-example" deleted
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `Unable to connect to the server` | `kubectl cluster-info` 检查连通性 | 公网端点被 CAM 策略 strategyId:240463971 以 `tke:clusterExtranetEndpoint=true` 条件拒绝 | 通过 IOA/VPN 连接内网，或使用同 VPC CVM 执行 kubectl 命令 |
| Pod `ImagePullBackOff` | `kubectl describe pod <pod>` 查看事件 | 镜像拉取失败 | 检查镜像地址和访问凭证 |
| Pod `Pending` | `kubectl describe pod <pod>` 检查 Events | 资源不足或调度约束 | 检查节点资源和调度规则 |

### 资源创建成功但状态异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Pod `CrashLoopBackOff` | `kubectl logs <pod>` 查看容器日志 | 容器启动崩溃 | 检查应用启动命令和配置 |
| Deployment READY 长时间不匹配 | `kubectl rollout status deployment/<name>` | 滚动更新卡住 | 检查探针配置和资源限制 |

## 下一步

- [StatefulSet 管理](../StatefulSet%20管理/tccli%20操作.md) — page_id `31707`
- [设置工作负载的资源限制](../设置工作负载的资源限制/tccli%20操作.md) — page_id `32813`

## 控制台替代

[容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) → 集群 → 工作负载 → Deployment → 新建。
