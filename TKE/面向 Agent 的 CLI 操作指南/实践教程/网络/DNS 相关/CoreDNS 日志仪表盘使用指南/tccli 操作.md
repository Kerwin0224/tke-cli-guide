# CoreDNS 日志仪表盘使用指南（tccli）

> 对照官方：[CoreDNS 日志仪表盘使用指南](https://cloud.tencent.com/document/product/457/109060) · page_id `109060`

## 概述

通过 TKE 的 CoreDNS 组件日志采集功能，将 CoreDNS Pod 的 DNS 查询日志投递到 CLS（日志服务），并在 CLS 仪表盘中可视化展示 DNS 查询量、响应码分布、慢解析、NXDOMAIN 突增等指标，帮助快速定位 DNS 解析异常。

核心操作路径：创建 CLS 日志集与日志主题 → 通过 `UpdateAddon` 更新 CoreDNS 组件的 `RawValues` 开启日志采集 → 在 CLS 中检索和可视化日志。

> **kubectl 数据面不可达**：CAM拒绝公网端点(strategyId:240463971)，内网需VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

## 前置条件

- [环境准备](../../../../环境准备.md)
- 集群类型为 MANAGED_CLUSTER，Kubernetes 版本 >= 1.30.0
- 已开通 CLS（日志服务）服务

### 环境检查

```bash
# 1. 检查 tccli 版本和当前凭据
tccli --version
# expected: tccli version >= 1.0.0

tccli configure list
# expected: secretId, secretKey, region 均已配置

# 2. 检查 CAM 权限 — 需要以下 Action 名
#    tke:DescribeClusters, tke:DescribeAddon, tke:UpdateAddon, tke:DescribeAddonValues
#    cls:CreateLogset, cls:CreateTopic, cls:DescribeTopics, cls:DeleteTopic, cls:DeleteLogset, cls:SearchLog
# 验证：
tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
# expected: TotalCount >= 1，ClusterStatus: "Running"
```

**预期输出**：

```json
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "cls-example",
            "ClusterStatus": "Running"
        }
    ]
}
```

### 资源检查

```bash
# 3. 确认 CoreDNS 组件已安装且运行正常
tccli tke DescribeAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName coredns
# expected: Phase: "running"，返回 AddonVersion 和 RawValues
```

**预期输出**：

```json
{
    "AddonName": "coredns",
    "AddonVersion": "1.30.0",
    "Phase": "running",
    "RawValues": "...",
    "RequestId": "..."
}
```

```bash
# 4. 检查 CLS 日志集列表（确认 CLS 服务已开通）
tccli cls DescribeLogsets --region <Region>
# expected: 返回日志集列表（可为空）
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli / kubectl 命令 | 幂等 |
|-----------|----------------------|:--:|
| 创建 CLS 日志集 | `tccli cls CreateLogset --cli-input-json file://cls-logset.json` | 否 |
| 创建 CLS 日志主题 | `tccli cls CreateTopic --cli-input-json file://cls-topic.json` | 否 |
| 查看 CoreDNS 组件配置 | `tccli tke DescribeAddon --ClusterId CLUSTER_ID --AddonName coredns` | 是 |
| 查看组件可用参数 | `tccli tke DescribeAddonValues --ClusterId CLUSTER_ID --AddonName coredns` | 是 |
| 更新 CoreDNS 日志配置 | `tccli tke UpdateAddon --cli-input-json file://update-coredns-log.json` | 否 |
| 查看 CLS 日志 | `tccli cls SearchLog --TopicId TOPIC_ID` | 是 |
| 查看 CoreDNS Corefile | `kubectl get configmap coredns -n kube-system -o yaml`（需 VPN/IOA） | 是 |
| 查看 CoreDNS Pod 日志 | `kubectl logs -n kube-system -l k8s-app=kube-dns`（需 VPN/IOA） | 是 |

## 关键字段说明

### UpdateAddon RawValues 日志参数

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `log.enabled` | Boolean | 是 | `true` 开启日志采集，`false` 关闭 | 不设或为 `false` → 日志不投递 |
| `log.cls.logsetId` | String | 是 | CLS 日志集 ID，`cls DescribeLogsets` 获取 | 日志集不存在 → 投递失败 |
| `log.cls.topicId` | String | 是 | CLS 日志主题 ID，`cls CreateTopic` 返回 | 日志主题不存在 → 投递失败 |
| `log.level` | String | 否 | 日志级别：`info`、`warning`、`error`、`debug`；默认 `info` | 级别过高 → 日志量过少，漏报问题 |

> **注意**：`RawValues` 是 JSON 格式的 base64 编码字符串。在 `--cli-input-json` 中传入时，需先将完整参数 JSON 做 base64 编码再赋值给 `RawValues` 字段。`UpdateStrategy` 默认为 `merge`，仅更新指定字段，保留其余组件配置不变。

## 操作步骤

### 步骤 1：创建 CLS 日志集和日志主题

#### 选择依据

- **日志集（Logset）**：按集群维度创建，一个集群一个日志集，便于按集群隔离和配额管理。
- **日志主题（Topic）**：按组件维度创建，CoreDNS 单独一个 Topic，避免与应用日志混在一起。
- **保存周期**：生产环境建议 30 天（默认），排查问题足够；如需长期留存审计，单独调整 Topic 存储周期。
- **为何不直接复用已有 Topic**：CoreDNS 日志量与业务日志差异大，混存会导致检索噪音和存储成本上升。

#### 创建日志集

`cls-logset.json`：

```json
{
    "LogsetName": "LOG_SET_NAME",
    "Period": 30
}
```

```bash
tccli cls CreateLogset --region <Region> \
    --cli-input-json file://cls-logset.json
# expected: exit 0，返回 LogsetId
```

**预期输出**：

```json
{
    "LogsetId": "logset-example",
    "RequestId": "..."
}
```

#### 创建日志主题

`cls-topic.json`：

```json
{
    "LogsetId": "LOG_SET_ID",
    "TopicName": "TOPIC_NAME"
}
```

```bash
tccli cls CreateTopic --region <Region> \
    --cli-input-json file://cls-topic.json
# expected: exit 0，返回 TopicId
```

**预期输出**：

```json
{
    "TopicId": "topic-example",
    "RequestId": "..."
}
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `LOG_SET_NAME` | CLS 日志集名称 | 全局唯一，1-255 字符 | 自定义 |
| `LOG_SET_ID` | CLS 日志集 ID | `CreateLogset` 返回 | `tccli cls DescribeLogsets --region <Region>` |
| `TOPIC_NAME` | CLS 日志主题名称 | 日志集内唯一 | 自定义 |
| `TOPIC_ID` | CLS 日志主题 ID | `CreateTopic` 返回 | `tccli cls DescribeTopics --region <Region>` |
| `REGION` | 地域 | 须与集群所在地域一致 | `tccli configure list` |
| `CLUSTER_ID` | 目标集群 ID | 格式 `cls-xxxxxxxx` | `tccli tke DescribeClusters --region <Region>` |

### 步骤 2：获取 CoreDNS 组件当前配置

#### 选择依据

- 更新前必须先获取当前 `RawValues`，避免覆盖已有配置（如 forward 上游、cache TTL 等）。
- `DescribeAddon` 返回的 `RawValues` 是 base64 编码的 JSON 字符串，解码后合并日志字段再重新编码。

```bash
tccli tke DescribeAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName coredns
# expected: 返回 AddonName、AddonVersion、Phase、RawValues
```

**预期输出**：

```json
{
    "AddonName": "coredns",
    "AddonVersion": "1.30.0",
    "Phase": "running",
    "RawValues": "eyJsb2ciOnsiZW5hYmxlZCI6ZmFsc2V9fQ==",
    "RequestId": "..."
}
```

```bash
# 解码当前 RawValues 查看（需 VPN/IOA 外可执行，纯本地解码）
echo "eyJsb2ciOnsiZW5hYmxlZCI6ZmFsc2V9fQ==" | base64 -d | jq .
# expected: 解码后为 JSON，包含 log 等字段
```

**解码后示例**：

```json
{
    "log": {
        "enabled": false
    }
}
```

### 步骤 3：开启 CoreDNS 日志采集

#### 选择依据

- 通过 `UpdateAddon` 更新 CoreDNS 组件的 `RawValues`，将 `log.enabled` 设为 `true` 并指定 CLS 日志集和主题。
- `log.level` 设为 `info` 以记录全部 DNS 查询；如日志量过大可调整为 `warning` 仅记录异常。
- `UpdateStrategy` 使用默认 `merge`，只更新日志相关字段，保留 CoreDNS 其他配置不变。

#### 构建 RawValues

目标参数 JSON：

```json
{
    "log": {
        "enabled": true,
        "cls": {
            "logsetId": "LOG_SET_ID",
            "topicId": "TOPIC_ID"
        },
        "level": "info"
    }
}
```

将上述 JSON 做 base64 编码后赋值给 `RawValues`。更新文件 `update-coredns-log.json`：

```json
{
    "ClusterId": "CLUSTER_ID",
    "AddonName": "coredns",
    "RawValues": "BASE64_ENCODED_RAW_VALUES",
    "UpdateStrategy": "merge"
}
```

> `BASE64_ENCODED_RAW_VALUES` 是步骤 3 中目标参数 JSON 的 base64 编码字符串。生成方式：
> `echo -n '{"log":{"enabled":true,"cls":{"logsetId":"LOG_SET_ID","topicId":"TOPIC_ID"},"level":"info"}}' | base64`

#### 执行更新

```bash
tccli tke UpdateAddon --region <Region> \
    --cli-input-json file://update-coredns-log.json
# expected: exit 0，返回 RequestId
```

**预期输出**：

```json
{
    "RequestId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
}
```

### 步骤 4：验证组件更新完成

```bash
# 轮询 CoreDNS 组件状态，等待 Phase 回到 running
tccli tke DescribeAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName coredns
# expected: Phase: "running"
```

**预期输出**：

```json
{
    "AddonName": "coredns",
    "AddonVersion": "1.30.0",
    "Phase": "running",
    "RawValues": "eyJsb2ciOnsiZW5hYmxlZCI6dHJ1ZSwiY2xzIjp7ImxvZ3NldElkIjoiTE9HX1NFVF9JRCIsInRvcGljSWQiOiJUT1BJQ19JRCJ9LCJsZXZlbCI6ImluZm8ifX0=",
    "RequestId": "..."
}
```

```bash
# 解码更新后的 RawValues 确认日志配置已写入
echo "eyJsb2ciOnsiZW5hYmxlZCI6dHJ1ZSwiY2xzIjp7ImxvZ3NldElkIjoiTE9HX1NFVF9JRCIsInRvcGljSWQiOiJUT1BJQ19JRCJ9LCJsZXZlbCI6ImluZm8ifX0=" | base64 -d | jq .
# expected: log.enabled=true，log.cls.logsetId 和 log.cls.topicId 已填充
```

## 验证

### 控制面（tccli）

```bash
# 维度 1：确认 CoreDNS 组件状态正常
tccli tke DescribeAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName coredns
# expected: Phase: "running"
```

```json
{
  "Addons": "<Addons>",
  "AddonName": "<AddonName>",
  "AddonVersion": "<AddonVersion>",
  "RawValues": "<RawValues>",
  "Phase": "<Phase>",
  "Reason": "<Reason>"
}
```

```bash
# 维度 2：确认日志主题存在且可写入
tccli cls DescribeTopics --region <Region> \
    --Filters '[{"Key":"topicId","Values":["TOPIC_ID"]}]'
# expected: TotalCount >= 1
```

```bash
# 维度 3：在 CLS 中检索 CoreDNS 日志（等待 30-60 秒后日志投递生效）
CURRENT_TIME=$(date +%s)
START_TIME=$((CURRENT_TIME - 3600))
tccli cls SearchLog --region <Region> \
    --TopicId TOPIC_ID \
    --From ${START_TIME} \
    --To ${CURRENT_TIME} \
    --Query ""
# expected: 返回日志记录，包含 DNS 查询日志
```

### 数据面（需 VPN/IOA）

```bash
# 维度 4：确认 CoreDNS Corefile 中包含 log 插件
kubectl get configmap coredns -n kube-system -o jsonpath='{.data.Corefile}'
# expected: Corefile 中含 .:53 { ... log ... } 配置
```

```text
NAME  STATUS  AGE
...
```

```bash
# 维度 5：查看 CoreDNS Pod 日志，确认有 DNS 查询输出
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=20
# expected: 含 DNS 查询记录，如 "[INFO] 10.0.0.1:53 - 1234 \"A IN kubernetes.default.svc.cluster.local. udp 54 false 512\""
```

| 验证维度 | 命令 | 预期 |
|---------|------|------|
| 组件状态 | `DescribeAddon --AddonName coredns` | `Phase: "running"` |
| 日志主题 | `cls DescribeTopics --Filters topicId` | `TotalCount >= 1` |
| CLS 日志投递 | `cls SearchLog --TopicId TOPIC_ID` | 返回 CoreDNS DNS 查询日志 |
| Corefile 配置 | `kubectl get cm coredns -n kube-system` | Corefile 含 `log` 插件 |
| Pod 日志 | `kubectl logs -n kube-system -l k8s-app=kube-dns` | 有 DNS 查询记录输出 |

## 清理

> **计费警告**：CLS 日志服务按写入量和存储量计费。CoreDNS 日志量在大型集群中可能很大（每秒数千条查询），未关闭日志采集会持续产生费用。清理时建议先关闭日志采集，再删除 CLS 日志主题。
> **副作用警告**：关闭 CoreDNS 日志采集会立即停止日志投递，但不影响 DNS 解析功能。已在 CLS 仪表盘中创建的图表不会自动删除。

### 控制面（tccli）

#### 1. 关闭 CoreDNS 日志采集

构建关闭 RawValues（`log.enabled=false`）：

```json
{
    "log": {
        "enabled": false
    }
}
```

更新文件 `disable-coredns-log.json`：

```json
{
    "ClusterId": "CLUSTER_ID",
    "AddonName": "coredns",
    "RawValues": "BASE64_DISABLE_RAW_VALUES",
    "UpdateStrategy": "merge"
}
```

> `BASE64_DISABLE_RAW_VALUES` = `echo -n '{"log":{"enabled":false}}' | base64`

```bash
tccli tke UpdateAddon --region <Region> \
    --cli-input-json file://disable-coredns-log.json
# expected: exit 0，返回 RequestId
```

```bash
# 验证日志采集已关闭
tccli tke DescribeAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName coredns
# expected: 解码 RawValues 后 log.enabled=false
```

```json
{
  "Addons": "<Addons>",
  "AddonName": "<AddonName>",
  "AddonVersion": "<AddonVersion>",
  "RawValues": "<RawValues>",
  "Phase": "<Phase>",
  "Reason": "<Reason>"
}
```

#### 2. 删除 CLS 日志主题

```bash
tccli cls DeleteTopic --region <Region> --TopicId TOPIC_ID
# expected: exit 0
```

```bash
# 验证日志主题已删除
tccli cls DescribeTopics --region <Region> \
    --Filters '[{"Key":"topicId","Values":["TOPIC_ID"]}]'
# expected: TotalCount: 0
```

#### 3. 删除 CLS 日志集（可选）

> **警告**：删除日志集将级联删除其下所有日志主题，不可恢复。

```bash
tccli cls DeleteLogset --region <Region> --LogsetId LOG_SET_ID
# expected: exit 0
```

```bash
tccli cls DescribeLogsets --region <Region> \
    --Filters '[{"Key":"logsetId","Values":["LOG_SET_ID"]}]'
# expected: TotalCount: 0
```

### 数据面（需 VPN/IOA）

> 数据面无需清理操作。CoreDNS ConfigMap 中的 `log` 插件由 `UpdateAddon` 自动管理，关闭日志采集后 CoreDNS 会自动 reload 移除 `log` 插件。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl` 命令超时或连接被拒绝 | `tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]'`（控制面返回 Running） | CAM 策略 strategyId:240463971 拒绝公网端点，数据面需内网访问 | 通过 VPN/IOA 接入 VPC 内网后执行 kubectl；或使用控制台替代 |
| `UpdateAddon` 返回 `InvalidParameter.RawValues` | 解码当前 RawValues 检查 JSON 格式 | `RawValues` 不是有效的 base64 编码字符串，或解码后不是合法 JSON | 重新构建 JSON → base64 编码 → 赋值给 `RawValues` 字段 |
| `UpdateAddon` 返回 `ResourceNotFound.Addon` | `tccli tke DescribeAddon --ClusterId CLUSTER_ID --AddonName coredns` | AddonName 大小写错误或 CoreDNS 组件未安装 | 使用小写 `coredns`；若未安装先通过控制台安装 CoreDNS 组件 |
| `cls CreateTopic` 返回 `InvalidParameter.LogsetNotExist` | `tccli cls DescribeLogsets --region <Region>` 确认日志集 | LogsetId 不存在或已被删除 | 用步骤 1 重新创建日志集，获取正确的 LogsetId |
| `cls SearchLog` 返回 `UnauthorizedOperation` | `tccli configure list` 检查当前凭据 | 子账号缺少 `cls:SearchLog` 权限（此为环境限制，非命令错误） | 联系主账号授予 CLS 只读权限（`QcloudCLSReadOnlyAccess`） |
| `DescribeAddon` 返回 `Phase: "failed"` | 查看 `Reason` 字段 | CLS 日志集/主题不存在，或 CoreDNS 版本不兼容 | 确认 LogsetId 和 TopicId 有效；检查 CoreDNS 版本 >= 1.8 |

### 配置成功但日志未投递

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| CLS 主题无日志 | `tccli cls SearchLog --TopicId TOPIC_ID` 检索最近 1 小时 | CoreDNS Corefile 中无 `log` 插件，日志采集未生效 | 确认 `UpdateAddon` 后 `Phase` 回到 `running`；检查解码后 `log.enabled=true` |
| CLS 日志延迟较大 | 等待 1-2 分钟后重新检索 | CLS 日志投递有 30-60 秒延迟 | 正常现象，等待后重新查询 |
| NXDOMAIN 突增告警 | CLS 检索 Query: `response_code:NXDOMAIN` | 集群中有大量不存在的域名查询（如配置错误的 service name） | 检查应用配置中的域名拼写；确认 CoreDNS `fallthrough` 配置正确 |
| 日志量过大导致 CLS 费用激增 | CLS 控制台查看存储用量 | `log.level` 设为 `info` 时记录全部查询，大集群日志量可达 GB/天 | 将 `log.level` 调整为 `warning` 或 `error`；缩短 Topic 存储周期 |

## 下一步

- [TKE DNS 最佳实践](../TKE%20DNS%20最佳实践/tccli%20操作.md) -- page_id `78005`
- [在 TKE 集群中使用 NodeLocal DNS Cache](../在%20TKE%20集群中使用%20NodeLocal%20DNS%20Cache/tccli%20操作.md) -- page_id `40613`
- [在 TKE 中实现自定义域名解析](../在%20TKE%20中实现自定义域名解析/tccli%20操作.md) -- page_id `50865`
- [CLS 产品文档](https://cloud.tencent.com/document/product/614) — 日志服务完整文档

## 控制台替代

登录 [容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) → 选择集群 → **组件管理** → **CoreDNS** → **更新配置** → 开启日志采集 → 选择 CLS 日志集和日志主题 → 保存。仪表盘在 [CLS 控制台](https://console.cloud.tencent.com/cls) → 仪表盘 中配置。
