# 日志采集配置（tccli）

> 对照官方：[通过 YAML 配置日志采集](https://cloud.tencent.com/document/product/457/56750) · page_id `56750`

## 概述

通过 LogConfig CRD 的 YAML 声明式配置，将 TKE Serverless 集群内 Pod 日志采集到 CLS（日志服务）。LogConfig 是 `cls.cloud.tencent.com/v1` API 组提供的自定义资源，命名空间级别，定义采集源（容器标准输出或容器文件路径）和投递目标（CLS 日志主题）。

与 [使用日志采集](../使用日志采集/tccli%20操作.md)（page_id `56751`）不同，本页聚焦 CLS 投递的 `clsDetail` 字段，Kafka 投递使用 `kafkaDetail` 字段。

LogConfig CRD 使用 `kubectl` 管理。若 `kubectl 不可达（公网端点 CAM 拒绝 strategyId:240463971）`，需先通过 [连接集群](../../../TKE%20Serverless%20集群管理/连接集群/tccli%20操作.md) 解决连通性问题后再操作。

## 前置条件

- TKE Serverless 集群已创建且状态为 `Running`，参见 [创建集群](../../../TKE%20Serverless%20集群管理/创建集群/tccli%20操作.md)。
- [环境准备](../../../../环境准备.md)：`tccli` 与 `kubectl` 已安装配置，kubectl 已连接到目标集群。
- 已 [开通日志采集](../开通日志采集/tccli%20操作.md)（page_id `61092`）—— 集群级别日志采集功能已启用。
- 目标 CLS 日志集和日志主题已创建。可通过 tccli 创建：

```bash
# 确保 CLS 资源就绪
tccli cls DescribeLogsets --region <Region> --output json
tccli cls DescribeTopics \
    --Filters '[{"Key":"logsetId","Values":["<LogsetId>"]}]' \
    --region <Region> \
    --output json
```

## 控制台与 CLI 参数映射

| 控制台操作 | CLI（kubectl） | 幂等 |
|-----------|---------------|------|
| 创建 LogConfig | `kubectl apply -f logconfig.yaml` | 是 |
| 查看 LogConfig 列表 | `kubectl get logconfig -A` | 是 |
| 查看 LogConfig 详情 | `kubectl describe logconfig <name> -n <ns>` | 是 |
| 更新 LogConfig | `kubectl apply -f logconfig.yaml` | 是（全量替换） |
| 删除 LogConfig | `kubectl delete logconfig <name> -n <ns>` | 否（删除后不可恢复） |

## 操作步骤

### 1. LogConfig CRD 完整结构（CLS 投递）

```yaml
apiVersion: cls.cloud.tencent.com/v1
kind: LogConfig
metadata:
  name: <LogConfigName>          # 采集规则名称
  namespace: <Namespace>         # 目标命名空间
  labels:                        # 标签（可选）
    app: <AppName>
spec:
  clsDetail:                     # CLS 投递配置
    logsetName: <LogsetName>     # CLS 日志集名称
    logsetId: <LogsetId>         # CLS 日志集 ID（与 logsetName 二选一）
    topicName: <TopicName>       # CLS 日志主题名称
    topicId: <TopicId>           # CLS 日志主题 ID（与 topicName 二选一）
    logType: <LogType>           # 日志解析类型（见下表）
    extractRule:                 # 提取规则（根据 logType 配置）
      beginningRegex: <Regex>    # 多行日志首行正则（multiline_log / multiline_fullregex_log 时必填）
      keys: [<Key1>, <Key2>]     # 提取字段名数组（fullregex_log / delimiter_log / json_log 时使用）
      timeKey: <TimeKey>         # 时间字段名
      timeFormat: <TimeFormat>   # 时间格式，如 "%Y-%m-%dT%H:%M:%S.%f%z"
      delimiter: <Delimiter>     # 分隔符（delimiter_log 时使用）
      filterKeys: [<FilterKey>]  # 需要过滤的字段（可选）
      filterRegex: [<Regex>]     # 过滤正则（可选）
      unMatchUpload: <Boolean>   # 解析失败时是否上传原始日志，默认 true
      unMatchedKey: <Key>        # 解析失败的日志存储字段名，默认 "LogParseFailure"
      backtracking: <Integer>    # 回溯量（KB），默认 1，最大 5
    storageType: <StorageType>   # 存储类型：hot（热存储）| cold（冷存储）
  inputDetail:                   # 日志输入配置
    type: <InputType>            # container_stdout | container_file | host_file
    # 以下子字段根据 type 选择
    containerStdout:             # type = container_stdout
      allContainers: <Boolean>   # true = 匹配所有容器
      namespace: <Namespace>     # 可选，指定命名空间范围
      workload:                  # workload 选择器
        name: <WorkloadName>
        kind: <Kind>             # Deployment | StatefulSet | DaemonSet | Job | CronJob
        labels:                  # 标签选择器（与 name/kind 二选一）
          <LabelKey>: <LabelValue>
        container: <ContainerName> # 容器名（可选）
    containerFile:               # type = container_file
      namespace: <Namespace>
      workload:
        name: <WorkloadName>
        kind: <Kind>
      container: <ContainerName>
      logPath: <LogPath>         # 容器内日志文件路径，支持通配符 *
      filePattern: <Pattern>     # 日志文件名匹配模式
      customLabels:              # 追加自定义标签（可选）
        <Key>: <Value>
    hostFile:                    # type = host_file（TKE Serverless 不常用）
      logPath: <HostLogPath>
      filePattern: <Pattern>
      customLabels:
        <Key>: <Value>
```

### 2. logType 解析类型枚举

| logType | 说明 | extractRule 必填字段 |
|--------|------|-------------------|
| `minimalist_log` | 单行全文 | 无（默认） |
| `multiline_log` | 多行全文 | `beginningRegex` |
| `fullregex_log` | 单行-完全正则 | `keys`、`regex` |
| `multiline_fullregex_log` | 多行-完全正则 | `beginningRegex`、`keys`、`regex` |
| `json_log` | JSON 日志 | `keys`（可选，自动解析 JSON 字段） |
| `delimiter_log` | 分隔符 | `keys`、`delimiter` |

### 3. 示例一：容器标准输出 — 单行全文

所有容器的标准输出，不做结构化解析。

```yaml
apiVersion: cls.cloud.tencent.com/v1
kind: LogConfig
metadata:
  name: stdout-minimalist
  namespace: default
spec:
  clsDetail:
    logsetName: "tke-serverless-logs"
    topicName: "stdout-minimalist-topic"
    logType: minimalist_log
  inputDetail:
    type: container_stdout
    containerStdout:
      allContainers: true
      namespace: "default"
```

应用配置：

```bash
kubectl apply -f logconfig-stdout-minimalist.yaml
```

示例输出：

```text
logconfig.cls.cloud.tencent.com/stdout-minimalist created
```

### 4. 示例二：容器标准输出 — JSON 解析

日志为 JSON 格式，CLS 自动解析为 key-value 字段。

```yaml
apiVersion: cls.cloud.tencent.com/v1
kind: LogConfig
metadata:
  name: stdout-json
  namespace: production
spec:
  clsDetail:
    logsetId: "logset-example-001"
    topicId: "topic-example-001"
    logType: json_log
    extractRule:
      timeKey: "timestamp"
      timeFormat: "%Y-%m-%dT%H:%M:%S.%f%z"
      unMatchUpload: true
      unMatchedKey: "LogParseFailure"
  inputDetail:
    type: container_stdout
    containerStdout:
      allContainers: false
      namespace: "production"
      workload:
        name: "api-gateway"
        kind: "Deployment"
```

应用配置：

```bash
kubectl apply -f logconfig-stdout-json.yaml
```

示例输出：

```text
logconfig.cls.cloud.tencent.com/stdout-json created
```

### 5. 示例三：容器标准输出 — 分隔符解析

使用 `|` 分隔的日志，提取三个字段：`level`、`time`、`message`。

```yaml
apiVersion: cls.cloud.tencent.com/v1
kind: LogConfig
metadata:
  name: stdout-delimiter
  namespace: production
spec:
  clsDetail:
    logsetName: "production-logs"
    topicName: "delimiter-topic"
    logType: delimiter_log
    extractRule:
      keys:
        - "level"
        - "time"
        - "message"
      delimiter: "|"
      timeKey: "time"
      timeFormat: "%Y-%m-%dT%H:%M:%S.%f%z"
      filterKeys:
        - "level"
      filterRegex:
        - "ERROR|WARN"
  inputDetail:
    type: container_stdout
    containerStdout:
      allContainers: false
      namespace: "production"
      workload:
        labels:
          app: "backend"
        container: "app"
```

应用配置：

```bash
kubectl apply -f logconfig-stdout-delimiter.yaml
```

示例输出：

```text
logconfig.cls.cloud.tencent.com/stdout-delimiter created
```

### 6. 示例四：容器文件日志 — 多行全文

采集 Java 应用的异常栈日志，每一条完整日志以时间戳开头。

```yaml
apiVersion: cls.cloud.tencent.com/v1
kind: LogConfig
metadata:
  name: file-multiline
  namespace: production
spec:
  clsDetail:
    logsetName: "production-logs"
    topicName: "java-app-logs"
    logType: multiline_log
    extractRule:
      beginningRegex: "^\\d{4}-\\d{2}-\\d{2}\\s+\\d{2}:\\d{2}:\\d{2}\\.\\d{3}"
      unMatchUpload: true
      unMatchedKey: "LogParseFailure"
      backtracking: 5
    storageType: "hot"
  inputDetail:
    type: container_file
    containerFile:
      namespace: "production"
      workload:
        name: "java-backend"
        kind: "Deployment"
      container: "java-app"
      logPath: "/var/log/app/*.log"
```

应用配置：

```bash
kubectl apply -f logconfig-file-multiline.yaml
```

示例输出：

```text
logconfig.cls.cloud.tencent.com/file-multiline created
```

### 7. 示例五：容器文件日志 — 多行-完全正则

采集 Nginx 错误日志，提取时间、级别、信息字段。

```yaml
apiVersion: cls.cloud.tencent.com/v1
kind: LogConfig
metadata:
  name: file-multiline-regex
  namespace: production
spec:
  clsDetail:
    logsetName: "production-logs"
    topicName: "nginx-error-logs"
    logType: multiline_fullregex_log
    extractRule:
      beginningRegex: "^\\d{4}/\\d{2}/\\d{2}\\s+\\d{2}:\\d{2}:\\d{2}"
      keys:
        - "time"
        - "level"
        - "message"
      timeKey: "time"
      timeFormat: "%Y/%m/%d %H:%M:%S"
      regex: "^(\\d{4}/\\d{2}/\\d{2}\\s+\\d{2}:\\d{2}:\\d{2})\\s+\\[(\\w+)\\]\\s+(.*)"
      unMatchUpload: true
  inputDetail:
    type: container_file
    containerFile:
      namespace: "production"
      workload:
        name: "nginx"
        kind: "Deployment"
      container: "nginx"
      logPath: "/var/log/nginx/error.log"
      filePattern: "error*.log"
```

应用配置：

```bash
kubectl apply -f logconfig-file-multiline-regex.yaml
```

示例输出：

```text
logconfig.cls.cloud.tencent.com/file-multiline-regex created
```

### 8. LogConfig 元数据字段

CLS 日志采集自动附加的 Kubernetes 元数据：

| 元数据字段 | 类型 | 说明 | 示例值 |
|----------|------|------|--------|
| `cluster_id` | string | TKE 集群 ID | `cls-example` |
| `container_name` | string | 容器名称 | `nginx` |
| `image_name` | string | 容器镜像全名 | `nginx:1.25.3` |
| `namespace` | string | Pod 命名空间 | `production` |
| `pod_uid` | string | Pod UID `metadata.uid` | `abc123-def456-ghi789` |
| `pod_name` | string | Pod 名称 | `nginx-deployment-7b9f8c6d5-x2k9m` |
| `pod_ip` | string | Pod IP 地址 | `10.0.1.5` |
| `pod_label_{labelKey}` | string | Pod 标签（每个 label 生成一个字段，labelKey 中 `-` 替换为 `_`，`.` 替换为 `_`） | `pod_label_app="nginx"` |

### 9. 管理 LogConfig 资源

#### 9.1 查看所有 LogConfig

```bash
kubectl get logconfigs -A
```

示例输出：

```text
NAMESPACE    NAME                     LOG-TYPE                  STATUS    AGE
default      stdout-minimalist        minimalist_log            Running   5m
production   stdout-json              json_log                  Running   4m
production   stdout-delimiter         delimiter_log             Running   3m
production   file-multiline           multiline_log             Running   2m
production   file-multiline-regex     multiline_fullregex_log   Running   1m
```

#### 9.2 查看单个 LogConfig 详情

```bash
kubectl describe logconfig file-multiline -n production
```

示例输出（关键字段）：

```text
Name:         file-multiline
Namespace:    production
Labels:       <none>
Annotations:  <none>
API Version:  cls.cloud.tencent.com/v1
Kind:         LogConfig
Spec:
  Cls Detail:
    Extract Rule:
      Beginning Regex:  ^\d{4}-\d{2}-\d{2}\s+\d{2}:\d{2}:\d{2}\.\d{3}
      Backtracking:     5
      Un Match Upload:  true
      Un Matched Key:   LogParseFailure
    Logset Name:         production-logs
    Log Type:            multiline_log
    Storage Type:        hot
    Topic Name:          java-app-logs
  Input Detail:
    Container File:
      Container:         java-app
      Log Path:          /var/log/app/*.log
      Namespace:         production
      Workload:
        Kind:            Deployment
        Name:            java-backend
    Type:                container_file
Status:
  Conditions:
    Message:             LogConfig is running
    Status:              True
    Type:                Running
```

#### 9.3 更新 LogConfig

修改 YAML 文件后重新 apply：

```bash
kubectl apply -f logconfig-file-multiline.yaml
```

示例输出：

```text
logconfig.cls.cloud.tencent.com/file-multiline configured
```

## 验证

### Data plane (kubectl)

```bash
# 所有 LogConfig 状态均为 Running
kubectl get logconfig -A --no-headers | awk '$5 != "Running" {print $0}' | wc -l
# 期望输出: 0

# 确认采集目标 CLS 主题正确
kubectl get logconfig stdout-json -n production -o jsonpath='{.spec.clsDetail.topicName}'
```

```text
NAME  STATUS  AGE
...
```

### CLS 端验证

```bash
tccli cls SearchLog \
    --TopicId "topic-example-001" \
    --From $(date -v-1H +%s)000 \
    --To $(date +%s)000 \
    --Query "" \
    --Limit 1 \
    --region ap-guangzhou \
    --output json
```

示例输出（确认日志数据正确上报）：

```json
{
    "Results": [
        {
            "Time": 1705311000000,
            "TopicId": "topic-example-001",
            "LogJson": "{\"cluster_id\":\"cls-example\",\"container_name\":\"nginx\",\"namespace\":\"default\",\"pod_name\":\"nginx-deployment-7b9f8c6d5-x2k9m\",\"content\":\"2024-01-15T10:50:00.000Z INFO request processed\"}"
        }
    ],
    "TotalCount": 1
}
```

## 清理

```bash
# 删除所有 LogConfig
kubectl delete logconfig --all -n default
kubectl delete logconfig --all -n production

# 验证清空
kubectl get logconfig -A

# 可选：清理 CLS 日志主题
tccli cls DeleteTopic --TopicId "topic-example-001" --region ap-guangzhou --output json
```

```text
NAME  STATUS  AGE
...
```

> **注意**：删除 LogConfig 资源不会删除 CLS 日志主题中已采集的日志数据。若需清理日志数据，需额外删除 CLS 日志主题。

## 排障

| 现象 | 原因 | 处理 |
|------|------|------|
| `kubectl apply` 报 `no matches for kind "LogConfig"` | CRD 未安装（日志采集未开通） | 先 [开通日志采集](../开通日志采集/tccli%20操作.md) |
| LogConfig 状态长时间 `Pending` | 采集组件未就绪或目标 CLS 主题不存在 | `kubectl describe logconfig <name> -n <ns>` 查看 Conditions |
| JSON 解析非 JSON 日志报 `LogParseFailure` | 非 JSON 格式日志触发解析失败 | 设置 `unMatchUpload: true` 保留原始日志，检查 `unMatchedKey` 字段 |
| 多行日志被切割为多条 | `beginningRegex` 不匹配 | 检查正则是否正确匹配每条日志的首行；使用 CLS 在线 Regex 测试工具 |
| `multiline_fullregex_log` 提取字段为空 | `regex` 捕获组不匹配 | 验证正则包含与 `keys` 数量一致的捕获组 `(...)` |
| 文件日志无数据上报 | `logPath` 无匹配文件 | 确认容器内文件存在且路径正确；路径需以 `/` 开头 |
| kubectl 不可达 | 公网端点 CAM 拒绝（strategyId:240463971） | 先 [连接集群](../../../TKE%20Serverless%20集群管理/连接集群/tccli%20操作.md) 解决连通性 |

## 下一步

- [使用日志采集](../使用日志采集/tccli%20操作.md)（page_id `56751`） — 配置 LogConfig 投递到 Kafka
- [采集容器内日志](../采集容器内日志/tccli%20操作.md)（page_id `56320`） — 通过 CLS 控制台/CLI 配置采集规则
- [开启集群审计](../../审计管理/开启集群审计/tccli%20操作.md)（page_id `58242`） — 开启 Kubernetes 审计日志

## 控制台替代

[容器服务控制台 → 集群 → 运维中心 → 日志采集 → 新增日志采集规则](https://console.cloud.tencent.com/tke2/cluster) — 通过表单配置采集源、目标 CLS 日志主题及解析规则。控制台自动生成等效 LogConfig CRD 并应用到集群。
