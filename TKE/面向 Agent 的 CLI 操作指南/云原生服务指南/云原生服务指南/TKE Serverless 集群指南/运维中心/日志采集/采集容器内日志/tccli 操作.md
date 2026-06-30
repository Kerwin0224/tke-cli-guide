# 采集容器内日志（tccli）

> 对照官方：[通过控制台配置日志采集](https://cloud.tencent.com/document/product/457/56320) · page_id `56320`

## 概述

通过 tccli（CLS 产品和 TKE 产品命令）配置 TKE Serverless 集群的容器日志采集规则。日志采集支持两种类型：**容器标准输出**（container stdout/stderr）和**容器文件路径**（容器内指定路径的日志文件）。采集到的日志投递到 CLS（日志服务）日志主题，支持多种解析模式对日志内容进行结构化处理。

与 LogConfig CRD（YAML 声明式）不同，本页聚焦于通过 tccli 的 CLS 产品命令管理 CLS 日志集和日志主题，并结合 TKE 集群查询（`DescribeEKSClusters`）完成采集规则的全生命周期管理。日志的采集规则配置在控制台 **运维中心 → 日志采集 → 配置日志规则** 中完成。

## 前置条件

- TKE Serverless 集群已创建且状态为 `Running`，参见 [创建集群](../../../TKE%20Serverless%20集群管理/创建集群/tccli%20操作.md)。
- [环境准备](../../../../环境准备.md)：`tccli` 已安装并配置 `region`（本文以 `ap-guangzhou` 为例）。
- 已 [开通日志采集](../开通日志采集/tccli%20操作.md)（page_id `61092`）—— 集群级别日志采集功能已启用。
- 已开通 [CLS（日志服务）](https://console.cloud.tencent.com/cls)，目标地域有可用 CLS 服务。
- 若使用 `kubectl` 验证 Pod 日志输出，需确保 kubectl 可达；`kubectl 不可达（公网端点 CAM 拒绝 strategyId:240463971）` 时仅限 tccli 操作。

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|------|
| 查看 CLS 日志集 | `cls DescribeLogsets` | 是 |
| 创建 CLS 日志集 | `cls CreateLogset` | 否（同名可能冲突） |
| 查看 CLS 日志主题 | `cls DescribeTopics` | 是 |
| 创建 CLS 日志主题 | `cls CreateTopic` | 否（同名可能冲突） |
| 修改日志主题 | `cls ModifyTopic` | 否 |
| 创建采集配置 | `cls CreateConfig` | 否 |
| 查看采集配置 | `cls DescribeConfigs` | 是 |
| 修改采集配置 | `cls ModifyConfig` | 否 |
| 查询集群信息 | `tke DescribeEKSClusters` | 是 |
| 删除采集配置 | `cls DeleteConfig` | 否（删除后不可恢复） |

## 操作步骤

### 1. 理解采集类型与解析模式

#### 1.1 采集类型

| 采集类型 | 说明 | 适用场景 |
|---------|------|---------|
| 容器标准输出 | 采集 Pod 容器 stdout/stderr 输出 | 应用将日志打印到标准输出（如 nginx 的 access log 输出到 stdout） |
| 容器文件路径 | 采集容器内指定路径的日志文件 | 应用将日志写入文件（如 `/var/log/app/app.log`） |

#### 1.2 日志解析模式

CLS 日志主题在创建时需指定解析模式，决定日志内容如何被结构化处理：

| 解析模式 | 说明 | 日志示例 |
|---------|------|---------|
| 单行全文 | 每行一条日志，不做结构化 | `Error: connection timeout` |
| 多行全文 | 一条日志跨多行（如 Java 异常栈），用首行正则识别 | `2024-01-15 10:30:00 ERROR ...\n\tat ...` |
| 单行-完全正则 | 每行一条日志，用正则提取字段 | 正则 `(\S+) (\S+) (.*)` |
| 多行-完全正则 | 跨行日志，用首行正则识别 + 完全正则提取 | Java 异常栈 + 正则提取字段 |
| JSON | 日志本身为 JSON 格式，自动解析为 key-value | `{"level":"ERROR","msg":"timeout"}` |
| 分隔符 | 日志用固定分隔符分隔字段（如 `,` `\|`） | `ERROR,2024-01-15,timeout` |
| 组合解析 | 结合多种解析模式，如先 JSON 再正则 | 复杂非标准格式 |

#### 1.3 元数据字段

CLS 日志采集自动附加以下 Kubernetes 元数据：

| 字段 | 说明 | 示例值 |
|------|------|--------|
| `cluster_id` | TKE 集群 ID | `cls-example` |
| `container_name` | 容器名称 | `nginx` |
| `image_name` | 镜像名称 | `nginx:1.25.3` |
| `namespace` | Pod 所在命名空间 | `default` |
| `pod_uid` | Pod 唯一标识 | `abc123-def456-ghi789` |
| `pod_name` | Pod 名称 | `nginx-deployment-7b9f8c6d5-x2k9m` |
| `pod_ip` | Pod IP 地址 | `10.0.1.5` |
| `pod_label_{name}` | Pod 标签（每个 label 一个字段） | `pod_label_app: nginx` |

### 2. 准备 CLS 日志集和日志主题

#### 2.1 查看已有日志集和日志主题

```bash
# 查看日志集列表
tccli cls DescribeLogsets \
    --region ap-guangzhou \
    --output json
```

示例输出：

```json
{
    "TotalCount": 2,
    "Logsets": [
        {
            "LogsetId": "logset-example-001",
            "LogsetName": "tke-serverless-logs",
            "CreateTime": "2024-01-15T10:00:00Z",
            "TopicCount": 2
        },
        {
            "LogsetId": "logset-example-002",
            "LogsetName": "production-logs",
            "CreateTime": "2024-01-10T08:00:00Z",
            "TopicCount": 1
        }
    ]
}
```

查看指定日志集下的日志主题：

```bash
tccli cls DescribeTopics \
    --Filters '[{"Key":"logsetId","Values":["logset-example-001"]}]' \
    --region ap-guangzhou \
    --output json
```

示例输出：

```json
{
    "TotalCount": 1,
    "Topics": [
        {
            "TopicId": "topic-example-001",
            "TopicName": "tke-serverless-stdout",
            "LogsetId": "logset-example-001",
            "Period": 30,
            "Status": true,
            "CreateTime": "2024-01-15T10:30:00Z"
        }
    ]
}
```

#### 2.2 创建日志集

```bash
tccli cls CreateLogset \
    --LogsetName "tke-serverless-container-logs" \
    --region ap-guangzhou \
    --output json
```

示例输出：

```json
{
    "LogsetId": "logset-example-003",
    "RequestId": "abc123-def456-ghi789"
}
```

#### 2.3 创建日志主题（容器标准输出，JSON 解析）

```bash
tccli cls CreateTopic \
    --LogsetId "logset-example-003" \
    --TopicName "stdout-json-logs" \
    --Period 30 \
    --region ap-guangzhou \
    --output json
```

示例输出：

```json
{
    "TopicId": "topic-example-002",
    "RequestId": "abc123-def456-ghi789"
}
```

#### 2.4 创建日志主题（容器文件日志，多行-完全正则解析）

```bash
tccli cls CreateTopic \
    --LogsetId "logset-example-003" \
    --TopicName "file-multiline-logs" \
    --Period 30 \
    --region ap-guangzhou \
    --output json
```

> **注意**：日志主题的解析模式（单行全文、JSON、正则等）在控制台日志采集规则中配置，或通过 LogConfig CRD 的 `extractRule` 字段指定。`cls CreateTopic` 命令创建日志主题时不直接指定解析模式。

### 3. 配置采集规则：容器标准输出

在控制台 **运维中心 → 日志采集** 中创建采集规则，或通过 LogConfig CRD 声明式管理。

#### 3.1 控制台配置步骤

1. 进入 [TKE 控制台](https://console.cloud.tencent.com/tke2/cluster) 集群列表。
2. 点击目标 Serverless 集群 ID（`cls-example`）。
3. 左侧导航栏选 **运维中心 → 日志采集**。
4. 点击 **新增日志采集规则**。
5. **采集类型**选 **容器标准输出**。
6. 配置采集源：选择命名空间、工作量类型、工作负载名称，或选 **所有容器**。
7. **消费端**选择目标 CLS 日志集和日志主题。
8. 配置解析规则（单行全文 / JSON / 分隔符等）与过滤器（可选）。
9. 确认并完成。

#### 3.2 查询集群确认配置已生效

配置规则后，可通过集群信息确认日志采集状态：

```bash
tccli tke DescribeEKSClusters \
    --ClusterIds '["cls-example"]' \
    --region ap-guangzhou \
    --output json
```

```json
{
  "TotalCount": 0,
  "Clusters": [],
  "ClusterId": "<ClusterId>",
  "ClusterName": "<ClusterName>",
  "VpcId": "<VpcId>",
  "SubnetIds": [],
  "K8SVersion": "<K8SVersion>",
  "Status": "<Status>"
}
```

### 4. 配置采集规则：容器文件路径

容器文件路径采集适用于日志以文件形式写入容器内的情况。

#### 4.1 控制台配置步骤

1. 同上进入 **运维中心 → 日志采集**。
2. 点击 **新增日志采集规则**。
3. **采集类型**选 **容器文件路径**。
4. 配置采集源：
   - 选择命名空间、工作负载类型、工作负载名称、容器名。
   - 填写 **容器内日志路径**，如 `/var/log/app/*.log`（支持通配符 `*`）。
5. **消费端**选择目标 CLS 日志集和日志主题。
6. 配置解析规则（多行全文 / 多行-完全正则 等）与提取规则。
7. 确认并完成。

#### 4.2 日志路径通配符说明

| 路径模式 | 说明 |
|---------|------|
| `/var/log/app/app.log` | 精确匹配单个文件 |
| `/var/log/app/*.log` | 匹配 `/var/log/app/` 下所有 `.log` 文件 |
| `/var/log/app/access*` | 匹配以 `access` 开头的所有文件 |
| `/var/log/app/**/*.log` | 递归匹配所有子目录下的 `.log` 文件 |

### 5. 更新已有采集规则

在控制台 **运维中心 → 日志采集** 中，点击已有规则的 **编辑** 按钮，修改采集源、目标日志主题、解析规则或过滤器。修改后采集组件将自动应用新配置，无需重启 Pod。

### 6. 通过 CLS 命令管理采集配置

CLS 产品也提供了采集配置管理命令，用于将采集配置绑定到指定的机器组或集群：

```bash
# 查看采集配置列表
tccli cls DescribeConfigs \
    --region ap-guangzhou \
    --output json

# 创建采集配置（CLS 原生采集配置，适用于 CVM/自建集群场景）
tccli cls CreateConfig \
    --Name "tke-serverless-log-config" \
    --Output "logset-example-003" \
    --Path "/var/log/app/*.log" \
    --LogType "multiline_log" \
    --region ap-guangzhou \
    --output json
```

> **注意**：对于 TKE Serverless 集群，采集配置主要通过控制台或 LogConfig CRD 管理，而非 CLS 原生采集配置（CLS 原生采集配置面向 CVM/自建集群）。

## 验证

### 验证 CLS 资源

```bash
# 确认日志集已创建
tccli cls DescribeLogsets --region ap-guangzhou --output json

# 确认日志主题已创建
tccli cls DescribeTopics \
    --Filters '[{"Key":"logsetId","Values":["logset-example-003"]}]' \
    --region ap-guangzhou \
    --output json
```

### 验证日志数据到达

```bash
# 检索目标日志主题（最近 1 小时）
tccli cls SearchLog \
    --TopicId "topic-example-002" \
    --From $(date -v-1H +%s)000 \
    --To $(date +%s)000 \
    --Query "" \
    --Limit 5 \
    --region ap-guangzhou \
    --output json
```

示例输出（有日志数据时）：

```json
{
    "Results": [
        {
            "Time": 1705311000000,
            "TopicId": "topic-example-002",
            "LogJson": "{\"cluster_id\":\"cls-example\",\"container_name\":\"nginx\",\"namespace\":\"default\",\"pod_name\":\"nginx-deployment-7b9f8c6d5-x2k9m\",\"pod_ip\":\"10.0.1.5\",\"content\":\"10.0.1.6 - - [15/Jan/2024:10:50:00 +0000] \\\"GET / HTTP/1.1\\\" 200 612\"}"
        }
    ],
    "TotalCount": 1
}
```

### Data plane (kubectl) - 可选

若 kubectl 可达：

```bash
kubectl get logconfigs -A
```

```text
NAME  STATUS  AGE
...
```

`kubectl 不可达（公网端点 CAM 拒绝 strategyId:240463971）` 时，仅通过 CLS 检索验证。

## 清理

若不再需要采集规则和 CLS 资源：

```bash
# 删除日志主题
tccli cls DeleteTopic \
    --TopicId "topic-example-002" \
    --region ap-guangzhou \
    --output json

tccli cls DeleteTopic \
    --TopicId "topic-example-003" \
    --region ap-guangzhou \
    --output json

# 删除日志集（需先删除所有子日志主题）
tccli cls DeleteLogset \
    --LogsetId "logset-example-003" \
    --region ap-guangzhou \
    --output json
```

> **注意**：删除日志主题将永久删除其中的日志数据，请确认已备份或不再需要。控制台中的采集规则需在 **运维中心 → 日志采集** 中手动删除。

## 排障

| 现象 | 原因 | 处理 |
|------|------|------|
| `cls CreateLogset` 报 `LogsetConflict` | 同名日志集已存在 | 使用已有日志集或更名 |
| `cls CreateTopic` 报 `LogsetNotExist` | `LogsetId` 不存在或格式错误 | 确认 `LogsetId` 为 `describe` 查询到的真实 ID |
| `cls SearchLog` 无结果 | 日志尚未上报或采集规则未生效 | 等待 2-5 分钟重试；确认 Pod 有日志输出 |
| 控制台采集规则创建失败 | 目标日志主题不存在或集群日志采集未开通 | 先确认 CLS 日志主题已创建，再确认 [开通日志采集](../开通日志采集/tccli%20操作.md) |
| 容器文件路径日志未采集 | `logPath` 无匹配文件或权限不足 | 确认容器内文件路径正确，文件可读；确保通配符 `*` 能匹配到文件 |
| kubectl 不可达 | 公网端点 CAM 拒绝（strategyId:240463971） | 先 [连接集群](../../../TKE%20Serverless%20集群管理/连接集群/tccli%20操作.md) 解决连通性 |

## 下一步

- [日志采集配置](../日志采集配置/tccli%20操作.md)（page_id `56750`） — 通过 YAML CRD 配置 LogConfig 到 CLS
- [使用日志采集](../使用日志采集/tccli%20操作.md)（page_id `56751`） — 配置日志到 Kafka
- [开启集群审计](../../审计管理/开启集群审计/tccli%20操作.md)（page_id `58242`） — 开启 Kubernetes 审计日志采集

## 控制台替代

[容器服务控制台 → 集群 → 运维中心 → 日志采集 → 新增日志采集规则](https://console.cloud.tencent.com/tke2/cluster) — 通过表单配置采集类型、源、目标 CLS 日志主题及解析规则。
