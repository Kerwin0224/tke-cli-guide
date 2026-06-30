# 设置工作负载的健康检查（tccli）

> 对照官方：[设置工作负载的健康检查](https://cloud.tencent.com/document/product/457/32815) · page_id `32815`

## 概述

为容器配置 livenessProbe（存活探针）、readinessProbe（就绪探针）和 startupProbe（启动探针），实现 Pod 健康状态自动检测和故障自愈。

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
| 设置存活探针 | `kubectl apply -f deployment-with-liveness.yaml` | 是 |
| 设置就绪探针 | `kubectl apply -f deployment-with-readiness.yaml` | 是 |
| 查看探针状态 | `kubectl describe pod <pod>` | 是 |
| 修改探针 | `kubectl edit deployment/<name>` | 否 |

## 操作步骤

### 步骤 1：创建带健康检查的 Deployment

YAML 清单：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: health-check-demo
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: health-check-demo
  template:
    metadata:
      labels:
        app: health-check-demo
    spec:
      containers:
        - name: nginx
          image: nginx:alpine
          ports:
            - containerPort: 80
          livenessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 10
            periodSeconds: 10
            timeoutSeconds: 3
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 5
            timeoutSeconds: 2
            successThreshold: 1
            failureThreshold: 3
          startupProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 0
            periodSeconds: 5
            failureThreshold: 30
```

| 探针类型 | 用途 | 失败行为 |
|----------|------|----------|
| `livenessProbe` | 检测容器是否存活 | 失败后重启容器 |
| `readinessProbe` | 检测容器是否就绪接收流量 | 失败后从 Service 端点移除 |
| `startupProbe` | 检测容器是否启动完成 | 失败后重启；成功前禁用其他探针 |

| 参数 | 说明 | 推荐值 |
|------|------|--------|
| `initialDelaySeconds` | 首次探测前等待秒数 | 5–30 |
| `periodSeconds` | 探测间隔秒数 | 5–10 |
| `timeoutSeconds` | 探测超时秒数 | 1–3 |
| `failureThreshold` | 连续失败多少次判定失败 | 3 |
| `successThreshold` | 连续成功多少次判定就绪 | 1 |

```bash
kubectl apply -f deployment-health-check.yaml
# expected: deployment.apps/health-check-demo created
```

**预期输出**：

```text
deployment.apps/health-check-demo created
```

### 步骤 2：验证探针状态

```bash
kubectl describe pod -l app=health-check-demo | grep -A10 "Liveness\|Readiness\|Startup"
# expected: 显示探针配置和状态
```

```text
NAME  STATUS  AGE
...
```

```bash
kubectl get pods -l app=health-check-demo
# expected: READY 1/1, STATUS Running
```

```text
NAME  STATUS  AGE
...
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
kubectl get pods -l app=health-check-demo
# expected: 所有 Pod Running

kubectl describe pod -l app=health-check-demo | grep -E "Liveness|Readiness|Startup"
# expected: 探针均正常
```

```text
NAME  STATUS  AGE
...
```

## 清理

```bash
kubectl delete deployment health-check-demo
# expected: deployment.apps "health-check-demo" deleted
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `Unable to connect to the server` | `kubectl cluster-info` 检查连通性 | 公网端点被 CAM 策略 strategyId:240463971 以 `tke:clusterExtranetEndpoint=true` 条件拒绝 | 通过 IOA/VPN 连接内网，或使用同 VPC CVM 执行 kubectl 命令 |

### 资源创建成功但状态异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Pod 反复重启 | `kubectl logs <pod>` 查看原因 | livenessProbe 失败（应用启动慢或路径错误） | 调大 `initialDelaySeconds` 或增加 startupProbe |
| Pod READY 0/1 | `kubectl describe pod <pod>` 查看探针状态 | readinessProbe 失败 | 检查探针路径和端口，确认应用已启动 |
| Pod 启动慢被 kill | `kubectl describe pod <pod>` 查看 Events | startupProbe `failureThreshold` 太短 | 调大 `failureThreshold` 或 `periodSeconds` |

## 下一步

- [设置工作负载的资源限制](../设置工作负载的资源限制/tccli%20操作.md) — page_id `32813`
- [设置工作负载的调度规则](../设置工作负载的调度规则/tccli%20操作.md) — page_id `32814`

## 控制台替代

[容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) → 集群 → 工作负载 → 新建 → 高级设置 → 健康检查。
