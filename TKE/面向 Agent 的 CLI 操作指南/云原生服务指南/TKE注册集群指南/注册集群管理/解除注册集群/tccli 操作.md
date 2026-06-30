# 解除注册集群

> 对照官方：[解除注册集群](https://cloud.tencent.com/document/product/457/63219) · page_id `63219`

## 概述

通过 `tccli tke DeleteExternalCluster` 将已注册的外部集群从 TKE 管理面解除关联。此操作只移除 TKE 侧的管理记录，外部集群自身的控制面和所有工作负载**完全不受影响**，继续独立运行。若需彻底清理，还需在外部集群手动删除 TKE agent 相关资源。

> **重要**：解除注册后，TKE 控制台不再显示该集群，所有基于 TKE 的监控、日志、审计等功能随之失效。请确认不再需要 TKE 统一管理后再执行。

## 前置条件

- [环境准备](../../../../环境准备.md)
- 当前账户有目标注册集群的管理权限
- 确认目标集群非生产核心（解除后 TKE 管理功能立即失效）

### 环境检查

```bash
tccli --version
# expected: tccli version >= 1.0.0

tccli configure list
# expected: secretId, secretKey, region 均已配置

tccli tke DescribeExternalClusters --region <Region>
# expected: exit 0，返回注册集群列表
```

```json
{"RequestId": "..."}
```

### 资源检查

```bash
# 确认要解除的集群信息
tccli tke DescribeExternalClusters --region <Region> \
  --ClusterIds '["CLUSTER_ID"]' --output json \
  | jq '.ClusterSet[0] | {ClusterId, ClusterName, ClusterStatus}'
# expected: 确认是目标集群

# 统计关联资源（TKE 侧日志/审计/监控配置）
# 记录这些资源 ID，解除注册后可能需要单独清理
```

```json
{"RequestId": "..."}
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看注册集群列表 | `tccli tke DescribeExternalClusters --region <Region>` | 是（只读） |
| 查看指定集群 | `tccli tke DescribeExternalClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'` | 是（只读） |
| 解除注册 | `tccli tke DeleteExternalCluster --region <Region> --ClusterId CLUSTER_ID` | 是 |
| 清理外部集群 agent | `kubectl delete -f agent.yaml`（外部集群执行） | 是 |
| 确认已解除 | `tccli tke DescribeExternalClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'` | 是（只读） |

## 操作步骤

### 步骤1：解除前确认

```bash
# 列出所有注册集群，确认目标
tccli tke DescribeExternalClusters --region <Region> --output json \
  | jq '.ClusterSet[] | {ClusterId, ClusterName, ClusterStatus}'
# expected: 确认 CLUSTER_ID 对应的集群名称和状态

# 记录集群信息（用于回滚）
tccli tke DescribeExternalClusters --region <Region> \
  --ClusterIds '["CLUSTER_ID"]' --output json > /tmp/external-cluster-CLUSTER_ID-backup.json
# expected: 导出集群信息备份
```

```json
{"RequestId": "..."}
```

### 步骤2：解除 TKE 注册

```bash
tccli tke DeleteExternalCluster --region <Region> \
  --ClusterId CLUSTER_ID --output json
# expected: exit 0，返回 RequestId
```

预期输出：

```json
{
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

### 步骤3：验证解除成功

```bash
# 方式1：按 ID 查询（应返回 ResourceNotFound 或空）
tccli tke DescribeExternalClusters --region <Region> \
  --ClusterIds '["CLUSTER_ID"]'
# expected: ResourceNotFound 或 ClusterSet 为空

# 方式2：列出全部注册集群，确认 ID 不再出现
tccli tke DescribeExternalClusters --region <Region> --output json \
  | jq '[.ClusterSet[]?.ClusterId]'
# expected: CLUSTER_ID 不在列表中
```

```json
{"RequestId": "..."}
```

### 步骤4：清理外部集群 Agent 资源（可选，在外部集群执行）

> 以下命令在**外部集群 kubectl 管理节点**执行。此步骤解除注册后仍可选执行，不影响外部集群正常运行。

```bash
# 确认 TKE agent 命名空间
kubectl get ns | grep tke
# expected: 显示 tke-xxx 命名空间

# 删除 agent 资源
kubectl delete ns tke-xxx
# expected: namespace "tke-xxx" deleted

# 确认清理完毕
kubectl get ns | grep tke
# expected: 无输出
```

```text
NAME  STATUS  AGE
...
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `REGION` | 地域 | 如 `ap-guangzhou` | `tccli configure list` |
| `CLUSTER_ID` | 注册集群 ID | 格式 `cls-xxxxxxxx` | `tccli tke DescribeExternalClusters --region <Region>` |

## 验证

### TKE 管理面验证

```bash
# 集群已从 TKE 中移除
tccli tke DescribeExternalClusters --region <Region> \
  --ClusterIds '["CLUSTER_ID"]'
# expected: ResourceNotFound 或空列表
```

```json
{"RequestId": "..."}
```

### 外部集群独立性验证（在外部集群执行）

```bash
# 外部集群自身仍然正常运行
kubectl cluster-info
# expected: 外部集群控制面正常响应

kubectl get nodes
# expected: 节点列表正常

kubectl get pods --all-namespaces | grep -v tke
# expected: 原有工作负载不受影响
```

```text
NAME  STATUS  AGE
...
```

## 清理

> **警告**：`DeleteExternalCluster` 操作**不可逆**。解除注册后，只能通过 [创建注册集群](../创建注册集群/tccli%20操作.md) 重新注册。重新注册会生成新的 ClusterId，之前关联的日志/审计/监控配置均需重建。
> **计费提醒**：解除注册后 TKE 不再管理该集群，但若外部集群使用了 TKE 附加功能（如 CLS 日志、Prometheus 监控），对应云资源需单独在所属产品控制台清理，否则持续计费。

### 1. TKE 侧已清理（DeleteExternalCluster 完成即生效）

### 2. 清理外部集群 agent（如步骤4未执行）

```bash
# 在外部集群执行
kubectl delete ns tke-xxx
# expected: namespace deleted
```

### 3. 清理 TKE 关联资源（在对应产品控制台或 CLI）

```bash
# 清理 CLS 日志主题（如有配置日志采集）
# tccli cls DeleteTopic --region <Region> --TopicId <LogTopicId>

# 清理 Prometheus 关联（如有配置监控）
# tccli monitor DescribePrometheusInstances --region <Region> ...

# 清理本地备份文件
rm /tmp/external-cluster-CLUSTER_ID-backup.json
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DeleteExternalCluster` 返回 `ResourceNotFound` | `DescribeExternalClusters` 确认集群存在 | 集群 ID 错误或已解除 | 确认正确的 `ClusterId` 和 `--region`，若已解除则无需操作 |
| `DeleteExternalCluster` 返回 `AuthFailure` | `tccli configure list` 检查凭据 | CAM 权限不足 | 授予 `tke:DeleteExternalCluster` 权限 |
| `DeleteExternalCluster` 返回 `FailedOperation.ClusterState` | `DescribeExternalClusters` 查看状态 | 集群处于不可删除状态（如删除中） | 等待状态变更后重试 |
| `DeleteExternalCluster` 返回 `InvalidParameter.ClusterId` | 检查参数格式 | `ClusterId` 格式错误 | 使用标准格式 `cls-xxxxxxxx` |

### 操作成功但异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 解除注册后外部集群 kubectl 报 `Unable to connect` | `kubectl cluster-info` 测试连通性 | 外部集群自身控制面问题，与解除无关 | 检查外部集群 API Server 状态，该问题与 TKE 解除注册无关 |
| 解除注册后 TKE 控制台仍显示集群 | 等待 1-2 分钟后刷新控制台 | 控制台缓存延迟 | 正常延迟，使用 `DescribeExternalClusters` CLI 验证（API 为实时结果） |
| 外部集群 agent 无法删除（namespace Terminating 状态） | `kubectl get ns tke-xxx -o yaml` 查看 finalizers | agent 有 finalizer 阻塞删除 | `kubectl patch ns tke-xxx -p '{"metadata":{"finalizers":[]}}' --type=merge` |
| 重新注册后发现旧日志/监控配置丢失 | `tccli tke DescribeExternalClusters` 查看新 ClusterId | 重新注册生成新 ClusterId，旧配置不继承 | 为新 ClusterId 重新配置日志/审计/监控 |

## 下一步

- [创建注册集群](../创建注册集群/tccli%20操作.md) — 如需重新注册（page_id `63218`）
- [连接注册集群](../连接注册集群/tccli%20操作.md) — 获取 kubeconfig（page_id `63216`）

## 控制台替代

[容器服务控制台 → 集群列表](https://console.cloud.tencent.com/tke2/external) 选中目标注册集群，点击 **更多 → 删除**，按提示确认解除注册。
