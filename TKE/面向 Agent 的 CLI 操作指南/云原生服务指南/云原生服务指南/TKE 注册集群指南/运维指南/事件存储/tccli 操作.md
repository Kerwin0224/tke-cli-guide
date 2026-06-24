# 事件存储（tccli）

> 对照官方：[事件存储](https://cloud.tencent.com/document/product/457/72150) · page_id `72150`

## 概述

将注册集群中的 **Kubernetes Event** 持久化存储至腾讯云日志服务（CLS）。Kubernetes 事件（如 Pod 调度、镜像拉取、OOMKill、节点驱逐等）默认仅保留 1 小时，配置事件存储后可长期留存并检索，便于运维回溯、告警联动与故障排查。

事件存储通过 TKE 控制台的「运维功能管理」入口开启，底层集成 CLS 日志主题。开启后可在 CLS 控制台/事件仪表盘中检索与可视化。

## 前置条件

- 注册集群已创建并处于 `Running` 状态。
- CLS 日志服务已开通，目标地域与集群一致。
- 已创建日志集（Logset）与日志主题（Topic）用于事件存储（参见 [日志采集](../日志采集/tccli%20操作.md) 步骤 1）。
- 单个日志集最多 500 个日志/指标主题，请合理规划。
- 事件存储依赖 `event-collector` 组件，该组件由 TKE 控制台在开启时自动部署。

## 控制台与 CLI 参数映射

| 控制台 | tccli / kubectl | 幂等 |
|--------|-----------------|------|
| 创建 CLS 日志集 | `tccli cls CreateLogset` | 否 |
| 创建 CLS 日志主题 | `tccli cls CreateTopic` | 否 |
| 查询日志集列表 | `tccli cls DescribeLogsets` | 是 |
| 查询日志主题列表 | `tccli cls DescribeTopics` | 是 |
| 开启事件存储 | 控制台操作（TKE 运维功能管理） | 否 |
| 更新日志集/主题 | 控制台修改配置 | 否 |
| 关闭事件存储 | 控制台关闭 | 否 |
| 检索事件 | `tccli cls SearchLog` | 是 |
| 查看 event-collector | `kubectl get pod -n kube-system` | 是 |

## 操作步骤

### 1. 创建 CLS 资源

```bash
# 创建事件专用日志集
tccli cls CreateLogset \
  --LogsetName "tke-registered-cluster-events" \
  --region ap-guangzhou \
  --output json
```

```json
{
  "LogsetId": "logset-tke-events-gz",
  "RequestId": "a1b2c3d4-5678-90ab-cdef-1234567890ab"
}
```

```bash
# 创建事件专用日志主题
tccli cls CreateTopic \
  --LogsetId logset-tke-events-gz \
  --TopicName "registry-cluster-events" \
  --StorageType hot \
  --Period 30 \
  --region ap-guangzhou \
  --output json
```

```json
{
  "TopicId": "topic-events-ghi789",
  "RequestId": "b2c3d4e5-6789-0abc-def1-234567890abc"
}
```

| 参数 | 建议值 | 说明 |
|------|--------|------|
| `StorageType` | `hot` | 热存储（高频检索）；低频可选 `cold` 降低成本 |
| `Period` | `30` | 日志保存天数，范围 1-366 |

### 2. 开启事件存储

此核心步骤需在控制台操作（当前无直接 tccli tke API 暴露事件存储开关）：

1. 进入 TKE 控制台 → **注册集群** → 选择目标集群
2. 点击 **运维功能管理**
3. 在**事件存储**卡片中点击 **设置**
4. 配置参数：
   - **事件日志集**：选择 `tke-registered-cluster-events`（或步骤 1 创建的其他 logset）
   - **事件日志主题**：选择 `registry-cluster-events`
   - **投递方式**：
     - **公网**：`event-collector` 通过公共互联网上报事件（节点需有公网访问）
     - **内网**：通过 CLS 内网上报（节点需与 CLS 内网互通）
5. 点击**确认**开启

开启后，TKE 控制台自动在集群中部署 `event-collector`（DaemonSet/Deployment），该组件负责收集集群事件并投递至 CLS。

### 3. 验证 event-collector 已部署

```bash
kubectl get pod -n kube-system | grep event
```

```
NAME                               READY   STATUS    RESTARTS   AGE
event-collector-6d4f8c9b-abc12    1/1     Running   0          2m
event-collector-6d4f8c9b-def34    1/1     Running   0          2m
event-collector-6d4f8c9b-ghi56    1/1     Running   0          2m
```

每个节点一个 `event-collector` Pod，负责收集该节点上产生的 Kubernetes Event。

### 4. 通过 tccli 检索存储的事件

```bash
# 检索最近 1 小时内所有事件
NOW=$(date +%s)000
ONE_HOUR_AGO=$((NOW - 3600000))

tccli cls SearchLog \
  --TopicId topic-events-ghi789 \
  --From $ONE_HOUR_AGO \
  --To $NOW \
  --Query "*" \
  --Limit 20 \
  --region ap-guangzhou \
  --output json
```

```json
{
  "Context": "",
  "ListOver": true,
  "Results": [
    {
      "Time": 1715901234567,
      "TopicId": "topic-events-ghi789",
      "LogJson": "{\"kind\":\"Event\",\"apiVersion\":\"v1\",\"metadata\":{\"name\":\"nginx-app.17a1b2c3d4e5f678\",\"namespace\":\"default\",\"uid\":\"abc123-def456\"},\"involvedObject\":{\"kind\":\"Pod\",\"namespace\":\"default\",\"name\":\"nginx-app-7d9f8c5b4d-abc12\"},\"reason\":\"Pulled\",\"message\":\"Successfully pulled image \\\"nginx:1.25\\\" in 2.345s\",\"source\":{\"component\":\"kubelet\",\"host\":\"node-uswest1\"},\"firstTimestamp\":\"2025-01-15T08:00:01Z\",\"lastTimestamp\":\"2025-01-15T08:00:01Z\",\"count\":1,\"type\":\"Normal\"}"
    },
    {
      "Time": 1715901235555,
      "LogJson": "{\"kind\":\"Event\",\"apiVersion\":\"v1\",\"metadata\":{\"name\":\"nginx-app.17a1b2c3d4e5f789\",\"namespace\":\"default\",\"uid\":\"def456-ghi789\"},\"involvedObject\":{\"kind\":\"Pod\",\"namespace\":\"default\",\"name\":\"nginx-app-7d9f8c5b4d-def34\"},\"reason\":\"OOMKilling\",\"message\":\"Container nginx exceeded memory limit. Memory used: 512Mi, limit: 256Mi\",\"source\":{\"component\":\"kubelet\",\"host\":\"node-uswest2\"},\"firstTimestamp\":\"2025-01-15T08:05:00Z\",\"lastTimestamp\":\"2025-01-15T08:05:00Z\",\"count\":1,\"type\":\"Warning\"}"
    }
  ],
  "RequestId": "c3d4e5f6-7890-abcd-ef12-34567890abcd"
}
```

**常用 CLS 检索条件**（针对事件的 `reason` 和 `type` 过滤）：

```bash
# 仅查询 Warning 级别事件
--Query "type:Warning"

# 查询 Pod 相关的 OOMKill 事件
--Query 'reason:OOMKilling AND involvedObject.kind:Pod'

# 查询特定命名空间的 FailedScheduling 事件
--Query 'namespace:default AND reason:FailedScheduling'

# 查询最近 24 小时节点驱逐事件
--Query 'reason:NodeNotReady AND type:Warning'

# 查询特定节点上的所有事件
--Query 'source.host:node-uswest1'
```

### 5. 更新事件存储配置

若需更换日志集或日志主题，进入控制台 **运维功能管理 → 事件存储 → 修改配置**，选择新的日志集/日志主题后确认。

更新操作不影响已存储的历史事件数据（旧 Topic 中的数据仍可检索）。

### 6. 关闭事件存储

1. 进入控制台 **运维功能管理 → 事件存储**
2. 点击 **关闭** 并在弹窗中确认

关闭后：
- `event-collector` 组件从集群中移除
- 不再有新事件投递至 CLS
- 已存储的 CLS 数据不受影响（仍可通过 `tccli cls SearchLog` 检索）

## 验证

### 检查 event-collector 运行状态

```bash
kubectl get deploy,ds -n kube-system | grep event
```

```
deployment.apps/event-collector   3/3     3            3           10m
```

```bash
# 查看 event-collector 日志确认事件正在上报
kubectl logs -n kube-system deploy/event-collector --tail=20
```

```
2025-01-15T08:00:00.000Z INFO event-collector started, watching events...
2025-01-15T08:00:01.000Z INFO new event: nginx-app.17a1b2c... reason=Pulled type=Normal
2025-01-15T08:00:01.234Z INFO event uploaded to CLS topic topic-events-ghi789
2025-01-15T08:00:05.000Z INFO new event: nginx-app.17a1b2c... reason=OOMKilling type=Warning
```

### 触发测试事件并验证存储

```bash
# 创建一个短暂容器触发事件
kubectl run test-event --image=nginx:alpine --restart=Never --rm -it -- echo "event test"

# 等待 30 秒后检索该事件
sleep 30

tccli cls SearchLog \
  --TopicId topic-events-ghi789 \
  --From $ONE_HOUR_AGO \
  --To $NOW \
  --Query "involvedObject.name:test-event" \
  --Limit 5 \
  --region ap-guangzhou \
  --output json
```

### 查看 CLS 事件仪表盘

TKE 控制台提供内置 **事件仪表盘**，可查看事件趋势、类型分布、Top 命名空间等图表。通过控制台访问（无 CLI 等价物）。

## 清理

### 关闭事件存储（停止新事件投递）

控制台操作（见 Procedure 步骤 6）。

### 清理 CLS 事件 Topic 与 Logset

```bash
# 注意：删除 Topic 会永久移除已存储的历史事件数据
tccli cls DeleteTopic --TopicId topic-events-ghi789 --region ap-guangzhou
tccli cls DeleteLogset --LogsetId logset-tke-events-gz --region ap-guangzhou
```

建议在删除前导出重要事件数据：

```bash
# 导出事件到本地 JSON 文件
tccli cls SearchLog \
  --TopicId topic-events-ghi789 \
  --From <start-timestamp> \
  --To <end-timestamp> \
  --Query "*" \
  --Limit 1000 \
  --region ap-guangzhou \
  --output json > events-backup.json
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 事件存储开启后 CLS 中无数据 | `kubectl get pods -n kube-system \| grep event-collector` 检查 Pod 状态 | event-collector 未 Running 或投递网络不通，新开启有 2-3 分钟延迟 | 等待 2-3 分钟；确认投递方式（公网/内网）网络连通性 |
| event-collector CrashLoopBackOff | `kubectl logs -n kube-system <event-collector-pod>` 查看日志 | CLS Topic ID 无效或凭证/网络/权限配置错误 | 检查 CLS Topic ID 有效性；根据日志具体错误修复 |
| event-collector OOMKilled | `kubectl describe pod -n kube-system <event-collector-pod>` 查看事件 | 高负载集群事件量大，默认 resource limits 不足 | 通过控制台调整 Pod resource limits |
| CLS Topic 数量超限 | `tccli cls DescribeTopics --Filters '[{"Key":"logsetId","Values":["<LogsetId>"]}]'` 查数 | 单 Logset 最多 500 个 Topic（此为环境限制） | 合并事件到已有 Topic 或创建新 Logset |
| 事件存储占用存储空间过大 | `tccli cls DescribeTopics --TopicId <TopicId>` 查保留天数 | `Period` 过长，未过滤低优先级事件 | 缩短 `Period`；过滤 Normal 级别等低优事件 |
| 控制台「运维功能管理」不可用 | `tccli tke DescribeClusters --ClusterIds '["<ClusterId>"]'` 查状态 | 注册集群状态非 Running 或处于解除注册 | 确认集群 Running 且未解除注册 |

## 下一步

- [日志采集](../日志采集/tccli%20操作.md) — 配置容器标准输出与文件日志采集
- [集群审计](../../运维指南/集群审计/tccli%20操作.md) — 配置 API Server 审计日志
- CLS 事件仪表盘 — 通过 TKE 控制台查看事件可视化图表

## 控制台替代

[控制台 → 注册集群 → 运维功能管理 → 事件存储](https://console.cloud.tencent.com/tke2/cluster?rid=1) — 控制台提供一键开启/关闭、选择 CLS 目标、查看事件仪表盘。
