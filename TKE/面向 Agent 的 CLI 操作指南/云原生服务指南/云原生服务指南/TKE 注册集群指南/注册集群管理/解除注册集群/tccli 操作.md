# 解除注册集群（tccli）

> 对照官方：[解除注册集群](https://cloud.tencent.com/document/product/457/63219) · page_id `63219`

## 概述

从 TKE 中移除已纳管的注册集群。解除注册后：
- TKE 控制台不再显示该集群，无法通过 TKE 进行管理操作。
- 外部集群中部署的代理组件（`clusternet-agent` 及相关 RBAC 资源）将被清除。
- **外部集群本身的业务 Pod、Service、Deployment 等工作负载不受影响**，集群正常运行。
- TKE 侧的集群记录与监控/日志配置同步删除（需单独清理 CLS 资源）。

此操作不可逆。操作前请确认你不再需要通过 TKE 平台纳管该集群。

## 前置条件

- 注册集群当前状态为 `Running`（通过 `DescribeClusters` 确认）。
- 已备份 TKE 控制台配置的日志采集规则、审计策略、事件存储配置（解除注册后不可恢复）。
- 确认具备删除集群的 IAM 权限（tke:DeleteCluster）。
- 如需保留 CLS 日志/事件数据，请勿在解除注册后立即删除 CLS logset/topic。

## 控制台与 CLI 参数映射

| 控制台 | tccli | 幂等 |
|--------|-------|------|
| 查看集群详情 | `DescribeClusters` + `ClusterIds` | 是 |
| 解除注册 | `DeleteCluster` + `--ClusterId` | 否（集群已删除后再次调用返回 NotFound） |
| 确认集群已删除 | `DescribeClusters` + `ClusterIds`（返回空） | 是 |

## 操作步骤

### 1. 确认目标集群

列出所有注册类型集群，确认待解除的集群 ID 与名称：

```bash
tccli tke DescribeClusters --region ap-guangzhou --output json \
  --filter "Clusters[?ClusterType=='REGISTERED_CLUSTER'].{ClusterId:ClusterId,ClusterName:ClusterName,ClusterStatus:ClusterStatus,CreatedTime:CreatedTime}"
```

```json
[
  {
    "ClusterId": "cls-registered-example",
    "ClusterName": "prod-uswest-registered",
    "ClusterStatus": "Running",
    "CreatedTime": "2024-11-15T08:00:00Z"
  },
  {
    "ClusterId": "cls-registered-dev",
    "ClusterName": "dev-euwest-registered",
    "ClusterStatus": "Running",
    "CreatedTime": "2024-10-01T14:30:00Z"
  }
]
```

### 2. 查看集群详细信息（解除前审计）

获取待删除集群的完整信息，确认无遗漏：

```bash
tccli tke DescribeClusters --region ap-guangzhou --output json \
  --ClusterIds '["cls-registered-example"]'
```

```json
{
  "TotalCount": 1,
  "Clusters": [
    {
      "ClusterId": "cls-registered-example",
      "ClusterName": "prod-uswest-registered",
      "ClusterStatus": "Running",
      "ClusterType": "REGISTERED_CLUSTER",
      "ClusterVersion": "1.23.6",
      "ClusterNetworkSettings": {
        "ClusterCIDR": "172.16.0.0/16",
        "ServiceCIDR": "10.96.0.0/12",
        "MaxNodePodNum": 64,
        "MaxClusterServiceNum": 4096
      },
      "TagSpecification": [
        {"Key": "env", "Value": "production"},
        {"Key": "location", "Value": "us-west-onprem"}
      ]
    }
  ],
  "RequestId": "a1b2c3d4-5678-90ab-cdef-1234567890ab"
}
```

### 3. 执行解除注册

```bash
tccli tke DeleteCluster \
  --ClusterId cls-registered-example \
  --InstanceDeleteMode terminate \
  --region ap-guangzhou \
  --output json
```

```json
{
  "Response": {
    "RequestId": "e04b8f1a-7d6c-49e5-b3f2-a8c9d0e1f2a3"
  }
}
```

`InstanceDeleteMode` 取值说明：
- `terminate` — 销毁 TKE 侧集群记录并清除外部集群中的 agent 组件（推荐，适用于注册集群）。
- `retain` — 保留外部集群中的某些资源（具体行为因产品设计而异；解除注册默认使用 `terminate`）。

## 验证

### 确认 TKE 侧已删除

```bash
tccli tke DescribeClusters --region ap-guangzhou --output json \
  --ClusterIds '["cls-registered-example"]'
```

```json
{
  "TotalCount": 0,
  "Clusters": [],
  "RequestId": "f1a2b3c4-5678-90ab-cdef-1234567890ab"
}
```

`TotalCount: 0` 且 `Clusters` 为空数组，表示该集群已在 TKE 侧成功删除。

### 确认外部集群中 agent 已清除

在外部集群中执行：

```bash
kubectl get ns clusternet-system
```

预期输出：

```
Error from server (NotFound): namespaces "clusternet-system" not found
```

若命名空间仍存在（agent 清除有 1-2 分钟延迟），可手动清理：

```bash
kubectl delete ns clusternet-system --force --grace-period=0
```

### 确认外部集群业务无影响

```bash
kubectl get deploy,pod,svc -n default
```

```
NAME                            READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx-app       3/3     3            3           14d

NAME                                 READY   STATUS    RESTARTS   AGE
pod/nginx-app-7d9f8c5b4d-abc12       1/1     Running   0          3d
pod/nginx-app-7d9f8c5b4d-def34       1/1     Running   0          3d
pod/nginx-app-7d9f8c5b4d-ghi56       1/1     Running   0          3d

NAME                 TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
service/nginx-svc    ClusterIP   10.96.100.50   <none>        80/TCP    14d
```

业务工作负载不受解除注册影响，持续运行。

## 清理

解除注册后，建议清理相关的 TKE 侧辅助资源：

### 清理 CLS 日志/事件资源

若之前为注册集群配置了日志采集、集群审计或事件存储，需手动清理 CLS 资源：

```bash
# 查看与集群关联的 Logset（示例过滤）
tccli cls DescribeLogsets --region ap-guangzhou --output json \
  --filter "Logsets[?contains(LogsetName, 'cls-registered-example')].{LogsetId:LogsetId,LogsetName:LogsetName}"

# 若不再需要，删除 Topic 和 Logset
tccli cls DeleteTopic --TopicId <LogTopicId> --region ap-guangzhou
tccli cls DeleteLogset --LogsetId <LogsetId> --region ap-guangzhou
```

### 清理 Hub 集群侧 Clusternet 相关资源（如同一个 Hub 下无其他注册集群）

```bash
kubectl --context hub-cls-example get managedcluster
# 确认对应 ManagedCluster 已被自动删除

# 如果 Hub 集群不再使用，删除 Hub 集群
tccli tke DeleteCluster \
  --ClusterId hub-cls-example \
  --InstanceDeleteMode terminate \
  --region ap-guangzhou
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DeleteCluster` 返回权限错误 `UnauthorizedOperation.CamNoAuth` | `tccli configure list` 检查当前凭据 | 子账号缺少 `tke:DeleteCluster` 权限（此为环境限制，非命令错误） | 联系主账号授予 `QcloudTKEFullAccess` 或最小权限 `tke:DeleteCluster` |

### 删除已提交但状态延迟

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 集群删除后 `DescribeClusters` 仍返回记录 | `tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'` 检查 | 删除为异步操作，状态同步有延迟 | 等待 30 秒后重试；若持续存在，检查 `--region` 是否为正确地域 |
| 外部集群中 `clusternet-system` 命名空间未被删除 | `kubectl get ns clusternet-system` 检查 | 命名空间中存在 finalizer 阻止删除 | 手动执行 `kubectl delete ns clusternet-system`；若卡在 Terminating，清除 finalizers：`kubectl patch ns clusternet-system -p '{"metadata":{"finalizers":[]}}'` |
| 解除注册后外部集群自动扩缩容失效 | `kubectl get deployments -n kube-system` 检查关键组件 | 解除注册不修改集群本身配置，误删了其他命名空间或服务 | 检查是否误删了 TKE 无关的组件；根据集群类型重新部署所需服务 |
| 想恢复已解除注册的集群 | — | 解除后需重新注册 | 重新执行 [创建注册集群](../创建注册集群/tccli%20操作.md) 流程，生成新的注册命令并在外部集群中执行 |

## 下一步

- [创建注册集群](../创建注册集群/tccli%20操作.md) — 如需重新纳管该集群
- [连接注册集群](../连接注册集群/tccli%20操作.md) — 直接通过 kubectl 操作外部集群（无需 TKE）
- 查看 [TKE 标准集群管理](../../../../../集群配置/集群管理/创建集群/tccli%20操作.md) — 若考虑迁移到 TKE 托管集群

## 控制台替代

[控制台 → 注册集群 → 更多 → 解除注册](https://console.cloud.tencent.com/tke2/cluster?rid=1) — 控制台提供二次确认弹窗：解除后删除 TKE 记录，外部集群不受影响。
