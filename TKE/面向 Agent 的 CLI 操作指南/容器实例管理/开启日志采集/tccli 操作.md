# 开启日志采集

> 对照官方：[开启日志采集](https://cloud.tencent.com/document/product/457/57351) · page_id `57351`

## 概述

通过 `CreateEksLogConfig` 为容器实例配置日志采集规则，将容器标准输出或文件日志持久化采集到 CLS（日志服务）。每条规则指定日志源、解析格式、CLS 日志主题等参数。采集的日志可在 CLS 控制台检索和分析。

## 前置条件

- [环境准备](../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:CreateEksLogConfig, tke:DescribeLogConfigs, tke:DeleteLogConfigs
#    cls:CreateTopic, cls:CreateLogset, cls:DescribeTopics, cls:DescribeLogsets
# 验证：执行 DescribeLogConfigs 确认 TKE 日志权限
tccli tke DescribeLogConfigs --region <Region>
# expected: exit 0，返回日志配置列表（可为空）
```

```json
{
  "Total": "<Total>",
  "Message": "<Message>",
  "LogConfigs": "<LogConfigs>",
  "RequestId": "<RequestId>"
}
```

> **说明**：示例中 `<Region>` 替换为 `ap-guangzhou`。

### 资源检查

```bash
# 4. 确认 CLS 日志服务已开通
# 登录 https://console.cloud.tencent.com/cls 确认可访问，无"未开通"提示

# 5. 确认已有 CLS 日志集和日志主题（需预先创建）
# CLS 日志集和日志主题需与容器实例在同一地域
# 在 CLS 控制台创建：https://console.cloud.tencent.com/cls/logset

# 6. 确认容器实例存在
tccli tke DescribeEKSContainerInstances --region <Region> \
    --EksCiIds '["<EksCiId>"]'
# expected: TotalCount >= 1，Status: "Running"
```

## 关键字段说明

`CreateEksLogConfig` 的主要参数：

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `ClusterId` | String | 是 | 集群 ID，格式 `cls-xxxxxxxx`。EKS CI 会分配到一个虚拟集群 ID | 集群不存在 → `InvalidParameter.ClusterId` |
| `LogConfig` | String | 是 | 日志采集配置 JSON 字符串。需用 `kind: LogConfig`，指定 `spec.inputDetail.type`（`container_stdout` 或 `container_file`）和 `spec.clsDetail`（logsetId + topicId） | JSON 格式错误 → `InvalidParameter.LogConfig`；logsetId/topicId 格式错误 → `InvalidParameter.ClsLogsetId` / `InvalidParameter.ClsTopicId` |
| `LogsetId` | String | 否 | CLS 日志集 ID，格式 UUID | 与 LogConfig 内的 logsetId 不一致时以 LogConfig 内的为准 |


## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 |
|-----------|----------|
| 参见官方文档 | `tccli tke DescribeClusters` |

## 操作步骤

### 步骤 1：在 CLS 控制台创建日志集和日志主题

登录 [CLS 控制台](https://console.cloud.tencent.com/cls)，在与容器实例同地域下：
1. 创建日志集（如 `eksci-logset`），记录日志集 ID
2. 在日志集下创建日志主题（如 `eksci-nginx-log`），记录日志主题 ID

### 步骤 2：创建日志采集规则

#### 选择依据

- **采集类型**：`container_stdout`（容器标准输出）适用于大多数场景，采集容器的 stdout/stderr 输出。`container_file` 适用于将应用日志写入文件的场景。
- **日志集/主题**：需与容器实例在同一地域，否则日志无法写入。

`log-config.json`：

```json
{
    "ClusterId": "<ClusterId>",
    "LogConfig": "{\"apiVersion\":\"cls.cloud.tencent.com/v1\",\"kind\":\"LogConfig\",\"metadata\":{\"name\":\"eksci-nginx-log\"},\"spec\":{\"clsDetail\":{\"logsetId\":\"<LogsetId>\",\"topicId\":\"<TopicId>\"},\"inputDetail\":{\"type\":\"container_stdout\",\"containerStdout\":{\"allContainers\":false,\"namespace\":\"default\"}}}}",
    "LogsetId": "<LogsetId>"
}
```

```bash
tccli tke CreateEksLogConfig --region <Region> \
    --cli-input-json file://log-config.json
# expected: exit 0，返回 TopicId
```

**预期输出**：

```json
{
    "TopicId": "<TopicId>",
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

| 占位符 | 说明 | 获取方式 |
|--------|------|---------|
| `<ClusterId>` | 集群 ID | `tccli tke DescribeClusters --region <Region>` |
| `<LogsetId>` | CLS 日志集 ID | [CLS 控制台](https://console.cloud.tencent.com/cls/logset) 获取 |
| `<TopicId>` | CLS 日志主题 ID | [CLS 控制台](https://console.cloud.tencent.com/cls/topic) 获取 |
| `<Region>` | 地域 | `tccli configure list` |

> **说明**：示例中 `<ClusterId>` 替换为 `cls-xxxxxxxx`（v1.30.0，Running，MANAGED_CLUSTER，ap-guangzhou）。

#### LogConfig JSON 结构说明

`LogConfig` 字段是一个内嵌的 JSON 字符串，核心结构：

| 路径 | 值 | 说明 |
|------|-----|------|
| `apiVersion` | `cls.cloud.tencent.com/v1` | 固定值 |
| `kind` | `LogConfig` | 固定值 |
| `metadata.name` | 自定义名称 | 日志配置名称 |
| `spec.clsDetail.logsetId` | CLS 日志集 ID | 格式 UUID |
| `spec.clsDetail.topicId` | CLS 日志主题 ID | 格式 UUID |
| `spec.inputDetail.type` | `container_stdout` 或 `container_file` | 采集类型 |
| `spec.inputDetail.containerStdout.allContainers` | `true` 或 `false` | 是否采集全部容器 |
| `spec.inputDetail.containerStdout.namespace` | 如 `default` | 采集的命名空间 |

### 步骤 3：查看已有采集规则

```bash
tccli tke DescribeLogConfigs --region <Region>
# expected: 返回日志配置列表
```

**预期输出**：

```json
{
    "Total": 1,
    "LogConfigs": [
        {
            "Name": "eksci-nginx-log",
            "LogType": "container_stdout",
            "LogsetId": "<LogsetId>",
            "TopicId": "<TopicId>"
        }
    ],
    "RequestId": "..."
}
```

### 步骤 4：在 CLS 控制台验证日志

日志写入 CLS 后，到 [CLS 检索分析](https://console.cloud.tencent.com/cls/search) 页面对应日志主题检索确认。

## 验证

```bash
# 确认采集规则存在
tccli tke DescribeLogConfigs --region <Region>
# expected: LogConfigs 包含目标规则
```

```json
{
  "Total": "<Total>",
  "Message": "<Message>",
  "LogConfigs": "<LogConfigs>",
  "RequestId": "<RequestId>"
}
```

| 验证维度 | 命令 | 预期 |
|------|------|------|
| 规则存在 | `DescribeLogConfigs --region <Region>` | 列表中包含 `eksci-nginx-log` |
| CLS 日志到达 | 登录 [CLS 控制台](https://console.cloud.tencent.com/cls/search) 检索 | 可见容器日志内容 |

## 清理

> **警告**：`DeleteLogConfigs` 删除采集规则后，日志将停止写入 CLS。已写入 CLS 的日志保留策略由日志主题设置决定，不会随采集规则删除而删除。

### 1. 清理前状态检查

```bash
tccli tke DescribeLogConfigs --region <Region>
# 确认目标规则存在，记录规则名称
```

```json
{
  "Total": 0,
  "Message": "<Message>",
  "LogConfigs": "<LogConfigs>",
  "RequestId": "<RequestId>"
}
```

### 2. 删除日志采集规则

```bash
tccli tke DeleteLogConfigs --region <Region>
# expected: exit 0，返回 RequestId
```

### 3. 验证已删除

```bash
tccli tke DescribeLogConfigs --region <Region>
# expected: 目标规则已移除
```

```json
{
  "Total": "<Total>",
  "Message": "<Message>",
  "LogConfigs": "<LogConfigs>",
  "RequestId": "<RequestId>"
}
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `CreateEksLogConfig` 返回 `InvalidParameter.ClsLogsetId` | 检查日志集 ID 格式 | CLS 日志集 ID 格式错误（应为 UUID 格式） | 在 [CLS 控制台](https://console.cloud.tencent.com/cls/logset) 复制日志集 ID |
| `CreateEksLogConfig` 返回 `InvalidParameter.ClsTopicId` | 检查日志主题 ID 格式 | CLS 日志主题 ID 格式错误（应为 UUID 格式） | 在 [CLS 控制台](https://console.cloud.tencent.com/cls/topic) 复制日志主题 ID |
| `CreateEksLogConfig` 返回 `InvalidParameter.LogConfig` | 用 `python3 -m json.tool` 按行解析 `LogConfig` 字段中的内嵌 JSON | `LogConfig` 内嵌 JSON 字符串格式错误（缺少引号转义、嵌套错误等） | 将 `LogConfig` 内容单独提取后用 `jq .` 验证 JSON 合法性 |
| `CreateEksLogConfig` 返回 CLS 服务未开通错误 | 登录 [CLS 控制台](https://console.cloud.tencent.com/cls) 确认服务状态 | 未激活 CLS 日志服务（此为环境限制，非命令错误） | 在 CLS 控制台开通日志服务 |
| `CreateEksLogConfig` 返回 `UnauthorizedOperation.CamNoAuth` | `tccli configure list` 检查凭据 | 子账号缺少 `tke:CreateEksLogConfig` 或 `cls:*` 权限（此为环境限制，非命令错误） | 联系主账号授予 CLS 相关权限 |

### 操作成功但日志未出现在 CLS

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 采集规则创建成功但 CLS 无日志 | 确认 CLS 日志集/主题与容器实例在同一地域 | 日志集/主题与容器实例不在同一地域 | 删除规则后在同地域创建新的日志集/主题，重新执行 `CreateEksLogConfig` |
| CLS 有日志但内容异常 | 检查 LogConfig 中 `inputDetail.type` 配置 | 采集类型配置错误（选了 `container_file` 但应用日志输出到 stdout） | 修正 `inputDetail.type` 为 `container_stdout` 后重新创建规则 |
| 部分容器日志缺失 | 检查 `allContainers` 和 `namespace` 配置 | `allContainers: false` 时仅采集指定命名空间和容器的日志 | 如需采集所有容器的日志，设置 `allContainers: true` |

## 下一步

- [查看日志及事件](../查看日志及事件/tccli%20操作.md) — 如何检索容器日志与事件
- [CLS 日志服务文档](https://cloud.tencent.com/document/product/614) — CLS 完整功能文档
- [容器实例操作手册](../容器实例操作手册/tccli%20操作.md) — 完整 CRUD 操作
- [登录实例](../登录实例/tccli%20操作.md) — 通过日志和事件诊断容器

## 控制台替代

[TKE 控制台 → 弹性容器 → 日志管理 → 新建日志配置](https://console.cloud.tencent.com/tke2/eks/log)：可视化配置日志采集规则。
