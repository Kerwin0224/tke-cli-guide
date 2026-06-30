# Request 智能推荐（tccli）

> 对照官方：[Request 智能推荐](https://cloud.tencent.com/document/product/457/78330) · page_id `78330`

## 概述

Request 智能推荐组件（Crane/Craned）通过对工作负载最长 14 天监控数据分析，为 Deployment、StatefulSet、DaemonSet 中的每个 Container 智能推荐 CPU/Memory Request 数值，减少因 Request 设置过大造成的资源浪费。仅适用于 TKE 标准集群（Kubernetes 1.10+）。一键更新后 Pod 将重新调度到原生节点。

## 前置条件

- `tccli` 已配置凭证
- kubectl 不可达：演示集群 cls-xxxxxxxx 为托管集群，公网端点可能受 CAM 策略限制（strategyId:240463971）；内网无 VPN 时不可达。以下 kubectl 命令作为文档参考。
- 集群为标准集群且 K8s >= 1.10；有原生节点资源

```bash
tccli tke DescribeClusters --ClusterIds '["cls-xxxxxxxx"]' --region ap-guangzhou
```

```json
{
  "RequestId": "e.g.-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "TotalCount": 1,
  "Clusters": [
    {
      "ClusterId": "cls-xxxxxxxx",
      "ClusterName": "my-test-cluster",
      "ClusterType": "MANAGED_CLUSTER",
      "ClusterStatus": "Running",
      "ClusterVersion": "1.30.0",
      "ClusterNodeNum": 2
    }
  ]
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 安装 Request 推荐组件 | `tccli tke InstallAddon --ClusterId cls-xxxxxxxx --AddonName craned --region ap-guangzhou` | 是 |
| 查看组件状态 | `tccli tke DescribeAddon --ClusterId cls-xxxxxxxx --AddonName craned --region ap-guangzhou` | 是 |
| 查看分析任务 | `kubectl get analytics -A` | 是 |
| 查看推荐结果 | `kubectl get recommendations -A` | 是 |
| 查看 Workload annotation 推荐值 | `kubectl get deploy <Name> -o jsonpath='{.metadata.annotations}'` | 是 |
| 一键更新 Request | 控制台操作（将添加 `nodeSelector` 调度到原生节点） | 是 |
| 卸载组件 | `tccli tke DeleteAddon --ClusterId cls-xxxxxxxx --AddonName craned --region ap-guangzhou` | 是 |

## 操作步骤

### 安装 Request 智能推荐组件

```bash
tccli tke InstallAddon --ClusterId cls-xxxxxxxx --AddonName craned --region ap-guangzhou
```

```json
{
  "RequestId": "e.g.-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

### 查看组件状态

```bash
tccli tke DescribeAddon --ClusterId cls-xxxxxxxx --AddonName craned --region ap-guangzhou
```

```json
{
  "RequestId": "e.g.-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "Addons": [
    {
      "AddonName": "craned",
      "AddonVersion": "1.0.0",
      "Status": "Running"
    }
  ]
}
```

### 查看 CRD 和资源对象

组件在 crane-system 命名空间下创建 Analytics CR 对象，生成 Recommendation CR 存储推荐数据：

```bash
kubectl get analytics -A
```

```text
NAMESPACE      NAME                 AGE
crane-system   housekeeper-default  3d
```

```bash
kubectl get recommendations -A
```

```text
NAMESPACE      NAME                                     AGE
crane-system   recommendation-deployment-default-nginx   2d
```

### 查看 Workload 推荐值

推荐数据写入工作负载 Annotation `analysis.crane.io/resource-recommendation`：

```bash
kubectl get deploy <DeploymentName> -n <Namespace> -o jsonpath='{.metadata.annotations.analysis\.crane\.io/resource-recommendation}'
```

```text
containers:
- containerName: nginx
  target:
    cpu: 125m
    memory: 125Mi
```

### 查看组件部署的资源对象

```bash
kubectl get all -n crane-system
```

```text
NAME                          READY   STATUS    RESTARTS   AGE
pod/craned-6d8cf9b84-xm9lk    1/1     Running   0          4d

NAME             TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/craned   ClusterIP   10.0.128.55     <none>        80/TCP    4d

NAME                     READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/craned   1/1     1            1           4d
```

## 验证

### Control plane (tccli)

```bash
tccli tke DescribeAddon --ClusterId cls-xxxxxxxx --AddonName craned --region ap-guangzhou
```

```json
{
  "RequestId": "e.g.-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "Addons": [
    {
      "AddonName": "craned",
      "AddonVersion": "1.0.0",
      "Status": "Running"
    }
  ]
}
```

确认 Status 为 `Running`。

### Data plane (kubectl)

```bash
kubectl get analytics -A && kubectl get recommendations -A
```

```text
NAMESPACE      NAME                 AGE
crane-system   housekeeper-default  3d
NAMESPACE      NAME                                     AGE
crane-system   recommendation-deployment-default-nginx   2d
```

## 清理

### Control plane (tccli)

```bash
tccli tke DeleteAddon --ClusterId cls-xxxxxxxx --AddonName craned --region ap-guangzhou
```

```json
{
  "RequestId": "e.g.-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

### Data plane (kubectl)

```bash
kubectl delete namespace crane-system
```

## 排障

| 现象 | 处理 |
|------|------|
| 无推荐数据 | 工作负载需运行至少 1 天才能产生推荐数据；新建 Workload 也需约 1 天 |
| 推荐最小阈值 | CPU 最小推荐值 125m，内存最小推荐值 125Mi |
| 一键更新后 Pod Pending | 一键更新自动添加 nodeSelector 调度到原生节点，原生节点资源不足时 Pending |
| 不支持 Job/CronJob | 仅支持 Deployment 和 StatefulSet |
| 组件不推荐 Limit | 组件仅推荐 Request；更新时维持 Request/Limit 原比例 |

## 下一步

- [Node Map](../../Node Map/tccli 操作.md) — 节点可视化热力图
- [Workload Map](../../Workload Map/tccli 操作.md) — 工作负载可视化热力图
- [成本洞察](../../成本洞察/tccli 操作.md) — 集群成本可视化

## 控制台替代

登录 [容器服务控制台](https://console.cloud.tencent.com/tke2) > TKE Insight > Node Map > 节点详情 > 开启 Request 推荐开关；或 Workload Map > 工作负载详情 > 使用推荐值更新。
