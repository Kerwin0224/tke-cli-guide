# 事件存储

> 对照官方：[事件存储](https://cloud.tencent.com/document/product/457/72150) · page_id `72150`

## 概述

Kubernetes Events 记录了集群中资源的状态变更（Pod 调度失败、节点异常、OOMKilled 等），是排障的第一手信息。注册集群支持将 Events 持久化到 CLS（日志服务），避免 Events 因 TTL 过期而丢失，并支持长期检索和分析。

本文演示：**tccli** 侧在 TKE 管理面为注册集群开启事件存储并绑定 CLS 日志主题，**kubectl** 侧在外部集群验证 Events 正常生成和上报。Events 由外部集群 event-exporter 组件采集，通过 agent 转发到 CLS。

## 前置条件

- [环境准备](../../../../环境准备.md)
- 已完成 [创建注册集群](../../注册集群管理/创建注册集群/tccli%20操作.md)，外部集群 agent 已部署且状态正常
- 已创建 CLS 日志集和日志主题（参见 [日志采集](../日志采集/tccli%20操作.md) 中的 CLS 资源准备）

### 环境检查

```bash
tccli --version
# expected: tccli version >= 1.0.0

tccli configure list
# expected: secretId, secretKey, region 均已配置

tccli tke DescribeExternalClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
# expected: exit 0，集群状态正常
```

```json
{"RequestId": "..."}
```

### 资源检查

```bash
# 1. 查询 CLS 日志主题（用于接收 Events）
tccli cls DescribeTopics --region <Region> \
  --Filters '[{"Key":"logsetId","Values":["LOGSET_ID"]}]'
# expected: 确认目标 TopicId 存在

# 2. 确认外部集群 agent 正常（在外部集群执行）
kubectl get pods -n tke-xxx
# expected: agent Pod STATUS 为 Running

# 3. 确认外部集群当前 Events（在外部集群执行）
kubectl get events --all-namespaces --sort-by='.lastTimestamp' | tail -10
# expected: 返回最近的 Events 列表

# 4. 检查 event-exporter 组件（在外部集群执行）
kubectl get pods -n kube-system | grep event
# expected: event-exporter Pod 存在（若已安装），否则 Events 需要该组件上报
```

```text
NAME  STATUS  AGE
...
```

### Events 说明

- **Events 特性**：默认保留 1 小时（由 `--event-ttl` 控制），过期后自动删除
- **CLS 持久化**：开启后 Events 实时写入 CLS，不受 TTL 限制，支持长期检索
- **event-exporter**：部署在外部集群 kube-system 命名空间，负责将 Events 写入 CLS

## 控制台与 CLI 参数映射

| 控制台操作 | tccli / kubectl 命令 | 幂等 |
|-----------|-----------|:--:|
| 开启事件存储 | `tccli tke EnableExternalClusterEventPersistence --region <Region> --ClusterId CLUSTER_ID --TopicId TOPIC_ID` | 是 |
| 查看事件存储配置 | `tccli tke DescribeExternalClusterEventPersistence --region <Region> --ClusterId CLUSTER_ID` | 是（只读） |
| 关闭事件存储 | `tccli tke DisableExternalClusterEventPersistence --region <Region> --ClusterId CLUSTER_ID` | 是 |
| 查看 Kubernetes Events | `kubectl get events --all-namespaces`（外部集群） | 是（只读） |
| 查询 CLS 事件 | `tccli cls SearchLog --region <Region> --TopicId TOPIC_ID --From <ts> --To <ts> --Query "<query>"` | 是（只读） |

## 操作步骤

### 步骤1：在 TKE 管理面开启事件存储

```bash
tccli tke EnableExternalClusterEventPersistence --region <Region> \
  --ClusterId CLUSTER_ID \
  --TopicId EVENT_TOPIC_ID \
  --output json
# expected: exit 0，返回 RequestId
```

预期输出：

```json
{
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

### 步骤2：验证 TKE 侧事件存储配置

```bash
tccli tke DescribeExternalClusterEventPersistence --region <Region> \
  --ClusterId CLUSTER_ID --output json
# expected: EventPersistenceEnabled 为 true，TopicId 与配置一致
```

预期输出：

```json
{
    "EventPersistenceEnabled": true,
    "TopicId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

### 步骤3：在外部集群验证 event-exporter 组件

> 以下命令在**外部集群 kubectl 管理节点**执行。开启事件存储后，TKE 会通过 agent 在外部集群自动部署 event-exporter。

```bash
# 等待 event-exporter 部署完成（约 1-2 分钟）
kubectl get pods -n kube-system --watch | grep event
# expected: event-exporter-xxx Pod 出现并变为 Running

# 查看 event-exporter 日志确认正常上报
kubectl logs -n kube-system deployment/event-exporter --tail=20
# expected: 日志无 ERROR，显示 Events 写入成功
```

```text
NAME  STATUS  AGE
...
```

### 步骤4：生成测试 Events 以验证采集

> 在外部集群执行以下命令生成可观测的 Events。

```bash
# 创建一个测试 Pod（会生成 Normal 类型 Event）
kubectl run test-event --image=nginx:alpine --restart=Never -n default
# expected: pod/test-event created

# 等待 Pod Running
kubectl wait --for=condition=Ready pod/test-event -n default --timeout=60s
# expected: pod/test-event condition met

# 查看本地 Events
kubectl get events -n default --field-selector involvedObject.name=test-event
# expected: 返回 Created、Scheduled、Pulled、Started 等 Events

# 清理测试 Pod
kubectl delete pod test-event -n default
# expected: pod "test-event" deleted
```

```text
NAME  STATUS  AGE
...
```

### 步骤5：查询 CLS 事件日志

```bash
# 等待 2-3 分钟后查询 CLS（Events 采集有少量延迟）
tccli cls SearchLog --region <Region> \
  --TopicId EVENT_TOPIC_ID \
  --From $(date -v-10M +%s)000 \
  --To $(date +%s)000 \
  --Query "test-event" \
  --Limit 5 \
  --output json
# expected: 返回包含 test-event 的事件条目
```

查询特定类型 Events（Warning）：

```bash
# 查询 Warning 级别事件（排查异常）
tccli cls SearchLog --region <Region> \
  --TopicId EVENT_TOPIC_ID \
  --From $(date -v-30M +%s)000 \
  --To $(date +%s)000 \
  --Query "type:Warning" \
  --Limit 10 \
  --output json | jq '.Results[] | {reason, message, object: .involvedObject.name, timestamp: .lastTimestamp}'
# expected: 返回 Warning 事件列表
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `REGION` | 地域 | 如 `ap-guangzhou` | `tccli configure list` |
| `CLUSTER_ID` | 注册集群 ID | 格式 `cls-xxxxxxxx` | `tccli tke DescribeExternalClusters --region <Region>` |
| `EVENT_TOPIC_ID` | CLS 事件日志主题 ID | UUID 格式，建议专用日志主题 | `tccli cls DescribeTopics --region <Region>` |

## 验证

### TKE 管理面验证

```bash
tccli tke DescribeExternalClusterEventPersistence --region <Region> \
  --ClusterId CLUSTER_ID
# expected: EventPersistenceEnabled 为 true
```

```json
{"RequestId": "..."}
```

### 外部集群验证（kubectl）

```bash
# event-exporter 组件正常运行
kubectl get pods -n kube-system | grep event-exporter
# expected: 1/1 Running

# event-exporter 日志无错误
kubectl logs -n kube-system deployment/event-exporter --tail=20
# expected: 日志显示 Events 上报成功

# 本地 Events 与 CLS 侧一致
kubectl get events --all-namespaces --sort-by='.lastTimestamp' | wc -l
# expected: Events 数量 >= 0（正常集群始终有 Events）
```

```text
NAME  STATUS  AGE
...
```

### CLS 事件验证

```bash
# 查询最近 15 分钟的所有事件
tccli cls SearchLog --region <Region> \
  --TopicId EVENT_TOPIC_ID \
  --From $(date -v-15M +%s)000 \
  --To $(date +%s)000 \
  --Query "*" \
  --Limit 10 \
  --output json | jq '.Results | length'
# expected: > 0

# 按事件类型统计
tccli cls SearchLog --region <Region> \
  --TopicId EVENT_TOPIC_ID \
  --From $(date -v-15M +%s)000 \
  --To $(date +%s)000 \
  --Query "type:Normal OR type:Warning" \
  --Limit 20 \
  --output json | jq '[.Results[] | .type] | group_by(.) | map({type: .[0], count: length})'
# expected: 返回 Normal 和 Warning 事件统计
```

## 清理

> **警告**：关闭事件存储后，新的 Kubernetes Events 不再写入 CLS。已持久化的历史事件不受影响（按 CLS 数据保留策略管理）。
> **计费提醒**：关闭事件存储停止写入后，CLS 日志主题仍按存储量计费。如需完全停止计费，删除 CLS 日志主题。

### 1. 关闭 TKE 事件存储

```bash
tccli tke DisableExternalClusterEventPersistence --region <Region> \
  --ClusterId CLUSTER_ID --output json
# expected: exit 0
```

### 2. 验证已关闭

```bash
tccli tke DescribeExternalClusterEventPersistence --region <Region> \
  --ClusterId CLUSTER_ID
# expected: EventPersistenceEnabled 为 false
```

```json
{"RequestId": "..."}
```

### 3. 清理 event-exporter 组件（在外部集群执行）

```bash
kubectl delete deployment event-exporter -n kube-system
# expected: deployment "event-exporter" deleted

# 确认已删除
kubectl get pods -n kube-system | grep event-exporter
# expected: 无输出
```

```text
NAME  STATUS  AGE
...
```

### 4. 清理 CLS 事件主题（可选，不可逆）

```bash
# > 警告：删除日志主题会永久删除所有持久化事件，不可恢复
tccli cls DeleteTopic --region <Region> --TopicId EVENT_TOPIC_ID
# expected: exit 0
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `EnableExternalClusterEventPersistence` 返回 `ResourceNotFound` | `DescribeExternalClusters` 确认集群存在 | 集群 ID 错误或 agent 未部署 | 确认 `ClusterId` 正确，检查 agent 状态 |
| `EnableExternalClusterEventPersistence` 返回 `InvalidParameter.TopicId` | `tccli cls DescribeTopics` 验证 | TopicId 不存在或地域不匹配 | 确认事件主题在目标地域 |
| `EnableExternalClusterEventPersistence` 返回 `AuthFailure` | `tccli configure list` 检查凭据 | CAM 缺少 TKE 或 CLS 权限 | 授予 `tke:EnableExternalClusterEventPersistence` 和 `cls:*` 权限 |
| `EnableExternalClusterEventPersistence` 返回 `FailedOperation.AlreadyEnabled` | `DescribeExternalClusterEventPersistence` 查看状态 | 事件存储已开启 | 如需更新配置，先 Disable 再重新 Enable |

### 操作成功但异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| TKE 侧配置成功但 CLS 查不到 Events | 等待 3-5 分钟后重试 | event-exporter 部署有延迟，或 Events 采集周期未到 | 继续等待；在外部集群执行 `kubectl run test --image=nginx:alpine` 主动生成 Events |
| event-exporter Pod 未自动创建 | `kubectl get pods -n kube-system --watch` 持续观察 | agent 同步延迟或集群资源不足 | 等待 5 分钟，检查 agent 日志中的 event-exporter 部署状态 |
| event-exporter Pod CrashLoopBackOff | `kubectl describe pod -n kube-system <event-exporter-pod>` | 资源不足或 CLS endpoint 不可达 | 增加 Pod 资源配额，确认 CLS endpoint 可达 |
| CLS 中 Warning 事件缺失 | `kubectl get events --all-namespaces \| grep Warning` 对比本地 | event-exporter 过滤策略丢弃了部分事件 | 检查 event-exporter 配置，确认无事件过滤规则 |
| Events 有延迟（本地已出现但 CLS 未查到） | 对比本地 `kubectl get events` 与 CLS `SearchLog` 结果 | event-exporter 采集周期默认 30-60 秒 | 正常延迟，等待 1-2 分钟后重试；持续延迟超过 5 分钟则检查 event-exporter 日志 |

## 下一步

- [日志采集](../日志采集/tccli%20操作.md) — 配置容器日志采集（page_id `72148`）
- [集群审计](../集群审计/tccli%20操作.md) — 启用审计日志（page_id `72149`）
- [CLS 控制台](https://console.cloud.tencent.com/cls) — 检索和分析 Event 日志

## 控制台替代

[容器服务控制台 → 集群列表](https://console.cloud.tencent.com/tke2/external) 选中注册集群 → **运维功能 → 事件存储** → 开启并选择 CLS 日志主题。
