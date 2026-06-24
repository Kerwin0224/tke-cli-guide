# 工作负载平滑升级（tccli）

> 对照官方：[工作负载平滑升级](https://cloud.tencent.com/document/product/457/77969) · page_id `77969`
## 概述

使用 Deployment 的 RollingUpdate 策略实现工作负载平滑升级，配置 `maxSurge`、`maxUnavailable` 控制升级速率，支持金丝雀发布和蓝绿部署。

> **kubectl 数据面不可达**：CAM拒绝公网端点(strategyId:240463971)，内网需VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

## 前置条件

- [环境准备](../../../环境准备.md)：`tccli` 已配置，`kubectl` >= v1.28
- 已获取集群 kubeconfig：

```bash
tccli tke DescribeClusterKubeconfig --region <Region> --ClusterId <ClusterId> \
    --filter "Kubeconfig" > kubeconfig.yaml
export KUBECONFIG=$(pwd)/kubeconfig.yaml
# expected: 生成 kubeconfig.yaml
```

```json
{
  "Kubeconfig": "<Kubeconfig>",
  "RequestId": "<RequestId>"
}
```

- 需要 CAM 权限：`tke:DescribeClusters`、`tke:DescribeClusterKubeconfig`

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 获取集群凭证 | `tccli tke DescribeClusterKubeconfig` | 是 |
| 查看集群状态 | `tccli tke DescribeClusterStatus --region <Region> --ClusterIds '["<ClusterId>"]'` | 是 |
| 相关 kubectl 操作 | 见 Procedure | -- |

## 操作步骤

### 1. 控制面：获取集群凭证

```bash
tccli tke DescribeClusterKubeconfig --region <Region> --ClusterId <ClusterId> \
    --filter "Kubeconfig" > kubeconfig.yaml
export KUBECONFIG=$(pwd)/kubeconfig.yaml
# expected: kubeconfig 写入本地文件
```

### 2. 配置滚动更新策略（需 VPN/IOA）

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rolling-demo
spec:
  replicas: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2
      maxUnavailable: 1
  selector:
    matchLabels:
      app: rolling-demo
  template:
    metadata:
      labels:
        app: rolling-demo
    spec:
      containers:
        - name: app
          image: nginx:1.25
```

| 参数 | 说明 | 推荐值 |
|------|------|--------|
| `maxSurge` | 升级过程中最多超出 replicas 的 Pod 数 | 1-2 |
| `maxUnavailable` | 升级过程中最多不可用 Pod 数 | 1 |
| `minReadySeconds` | 新 Pod 就绪后等待时间 | 10-30 |

```bash
kubectl apply -f rolling-deployment.yaml
# expected: deployment.apps/rolling-demo created
```

### 3. 执行滚动升级

```bash
# 更新镜像触发滚动升级
kubectl set image deployment/rolling-demo app=nginx:1.27
# expected: deployment.apps/rolling-demo image updated

# 监控升级进度
kubectl rollout status deployment/rolling-demo
# expected: deployment "rolling-demo" successfully rolled out
```

### 4. 升级版本管理

```bash
# 查看版本历史
kubectl rollout history deployment/rolling-demo
# expected: REVISION 列表

# 回滚到上一版本
kubectl rollout undo deployment/rolling-demo
# expected: deployment.apps/rolling-demo rolled back

# 暂停/恢复滚动升级
kubectl rollout pause deployment/rolling-demo
kubectl rollout resume deployment/rolling-demo
```

## 验证

### 控制面

```bash
tccli tke DescribeClusterStatus --region <Region> --ClusterIds '["<ClusterId>"]'
# expected: ClusterState "Running"
```

```json
{
  "ClusterStatusSet": "<ClusterStatusSet>",
  "ClusterId": "<ClusterId>",
  "ClusterState": "<ClusterState>",
  "ClusterInstanceState": "<ClusterInstanceState>",
  "ClusterBMonitor": "<ClusterBMonitor>",
  "ClusterInitNodeNum": "<ClusterInitNodeNum>"
}
```

### 数据面（需 VPN/IOA）

```bash
kubectl get deployment rolling-demo
# expected: READY 5/5

kubectl rollout status deployment/rolling-demo
# expected: successfully rolled out
```

```text
NAME  STATUS  AGE
...
```

## 清理

```bash
kubectl delete deployment rolling-demo
# expected: deployment.apps "rolling-demo" deleted
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl` 命令超时或连接被拒绝 | `tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]'`（控制面返回 Running） | CAM 策略 strategyId:240463971 拒绝公网端点，数据面需内网访问 | 通过 VPN/IOA 接入 VPC 内网后执行 kubectl；或使用控制台替代 |
| 升级卡住不推进 | `kubectl describe deployment <name>` 查看 Conditions | 新 Pod 无法变成 Ready（readinessProbe 失败或镜像拉取失败） | 排查 readinessProbe；检查镜像与资源调度 |
| 升级后服务异常 | `kubectl rollout undo deployment <name>` | 新版镜像有 Bug | 快速回滚到上一版本 |
| maxSurge 过大 | `kubectl get pods` 观察调度失败事件 | 升级期间资源不足 | 降低 maxSurge 值 |

## 下一步

- [应用高可用部署](../../服务部署/应用高可用部署/tccli%20操作.md) -- page_id `40212`
- [Deployment 管理](../../../应用配置/工作负载管理/Deployment%20管理/tccli%20操作.md) -- page_id `31705`

## 控制台替代

[容器服务控制台](https://console.cloud.tencent.com/tke2/cluster)
