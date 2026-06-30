# 使用日志采集（tccli）

> 对照官方：[使用 CRD 采集日志到 Kafka](https://cloud.tencent.com/document/product/457/56751) · page_id `56751`

## 概述

通过 LogConfig CRD（自定义资源）将 TKE Serverless 集群内 Pod 日志采集到外部 Kafka 集群。与 CLS 日志采集不同，Kafka 投递将日志以流式方式发送到指定的 Kafka broker，由下游消费端自行处理日志数据。LogConfig CRD 由 `cls.cloud.tencent.com/v1` API 组定义，资源范围为命名空间级别。

支持的输入类型：

- `container_stdout`：采集容器标准输出（stdout/stderr）日志
- `container_file`：采集容器内指定路径的文件日志

LogConfig CRD 使用 `kubectl` 管理。若 `kubectl 不可达（公网端点 CAM 拒绝 strategyId:240463971）`，不可在集群内执行 `kubectl` 命令，需先通过 [连接集群](../../../TKE%20Serverless%20集群管理/连接集群/tccli%20操作.md) 解决连通性问题后再操作。

## 前置条件

- TKE Serverless 集群已创建且状态为 `Running`，参见 [创建集群](../../../TKE%20Serverless%20集群管理/创建集群/tccli%20操作.md)。
- [环境准备](../../../../环境准备.md)：`tccli` 与 `kubectl` 已安装配置，kubectl 已连接到目标集群。
- 已 [开通日志采集](../开通日志采集/tccli%20操作.md)（page_id `61092`）—— 集群级别日志采集功能已启用。
- 目标 Kafka 集群可达，且已创建所需 topic。Kafka broker 地址格式为 `IP:Port` 或 `hostname:Port`。
- 如果 Kafka 在集群外，确保 VPC 网络可达（如同 VPC、对等连接、或云联网）。

## 控制台与 CLI 参数映射

| 控制台操作 | CLI（kubectl） | 幂等 |
|-----------|---------------|------|
| 创建日志采集规则 | `kubectl apply -f logconfig.yaml` | 是 |
| 查看日志采集规则 | `kubectl get logconfig -n <ns>` | 是 |
| 查看采集规则详情 | `kubectl describe logconfig <name> -n <ns>` | 是 |
| 更新日志采集规则 | `kubectl apply -f logconfig.yaml` | 是 |
| 删除日志采集规则 | `kubectl delete logconfig <name> -n <ns>` | 否（删除后不可恢复） |

## 操作步骤

### 1. LogConfig CRD 结构概览

LogConfig 是命名空间级别的自定义资源，定义了一条从容器日志到 Kafka 的采集管道。核心字段如下：

```yaml
apiVersion: cls.cloud.tencent.com/v1
kind: LogConfig
metadata:
  name: <LogConfigName>          # 采集规则名称
  namespace: <Namespace>         # 目标命名空间
spec:
  kafkaDetail:                   # Kafka 投递配置
    brokers:                     # Kafka broker 地址列表
    topic:                       # 目标 topic
    messageKey:                  # Kafka 消息 key 字段（可选）
    timestampKey:                # 时间戳字段名（可选）
    timestampFormat:             # 时间戳格式，如 "2006-01-02T15:04:05.000Z"（可选）
  inputDetail:                   # 日志输入配置
    type:                        # container_stdout | container_file
    containerStdout:             # 当 type=container_stdout 时
      allContainers:             # true = 匹配所有容器
      namespace:                 # 容器所属命名空间（可选）
      workload:                  # workload 选择器（可选）
      container:                 # 容器名（可选）
    containerFile:               # 当 type=container_file 时
      namespace:                 # 命名空间
      workload:                  # workload 选择器
      container:                 # 容器名
      logPath:                   # 容器内日志文件路径，如 /var/log/app/*.log
```

### 2. 采集容器标准输出到 Kafka

#### 2.1 采集指定命名空间内所有容器的标准输出

创建文件 `logconfig-kafka-stdout-all.yaml`：

```yaml
apiVersion: cls.cloud.tencent.com/v1
kind: LogConfig
metadata:
  name: stdout-to-kafka-all
  namespace: default
spec:
  kafkaDetail:
    brokers:
      - "10.0.1.10:9092"
      - "10.0.1.11:9092"
    topic: "tke-logs-stdout"
    messageKey: "pod_name"
    timestampKey: "@timestamp"
    timestampFormat: "2006-01-02T15:04:05.000Z"
  inputDetail:
    type: container_stdout
    containerStdout:
      allContainers: true
      namespace: "default"
```

应用配置：

```bash
kubectl apply -f logconfig-kafka-stdout-all.yaml
```

示例输出：

```text
logconfig.cls.cloud.tencent.com/stdout-to-kafka-all created
```

#### 2.2 采集指定 workload 的标准输出

创建文件 `logconfig-kafka-stdout-workload.yaml`：

```yaml
apiVersion: cls.cloud.tencent.com/v1
kind: LogConfig
metadata:
  name: stdout-to-kafka-nginx
  namespace: production
spec:
  kafkaDetail:
    brokers:
      - "10.0.2.20:9092"
    topic: "nginx-access-logs"
    messageKey: "pod_name"
    timestampKey: "@timestamp"
    timestampFormat: "2006-01-02T15:04:05.000Z"
  inputDetail:
    type: container_stdout
    containerStdout:
      allContainers: false
      namespace: "production"
      workload:
        name: "nginx-deployment"
        kind: "Deployment"
      container: "nginx"
```

应用配置：

```bash
kubectl apply -f logconfig-kafka-stdout-workload.yaml
```

示例输出：

```text
logconfig.cls.cloud.tencent.com/stdout-to-kafka-nginx created
```

### 3. 采集容器文件日志到 Kafka

#### 3.1 采集容器内指定路径的文件日志

创建文件 `logconfig-kafka-file.yaml`：

```yaml
apiVersion: cls.cloud.tencent.com/v1
kind: LogConfig
metadata:
  name: file-logs-to-kafka
  namespace: production
spec:
  kafkaDetail:
    brokers:
      - "10.0.3.30:9092"
      - "10.0.3.31:9092"
    topic: "app-file-logs"
    messageKey: "pod_name"
    timestampKey: "@timestamp"
    timestampFormat: "2006-01-02T15:04:05.000Z"
  inputDetail:
    type: container_file
    containerFile:
      namespace: "production"
      workload:
        name: "backend-api"
        kind: "Deployment"
      container: "api-server"
      logPath: "/var/log/app/*.log"
```

应用配置：

```bash
kubectl apply -f logconfig-kafka-file.yaml
```

示例输出：

```text
logconfig.cls.cloud.tencent.com/file-logs-to-kafka created
```

#### 3.2 使用标签选择器匹配 workload

```yaml
apiVersion: cls.cloud.tencent.com/v1
kind: LogConfig
metadata:
  name: file-logs-label-selector
  namespace: production
spec:
  kafkaDetail:
    brokers:
      - "10.0.4.40:9092"
    topic: "microservice-logs"
    messageKey: "pod_name"
    timestampKey: "@timestamp"
    timestampFormat: "2006-01-02T15:04:05.000Z"
  inputDetail:
    type: container_file
    containerFile:
      namespace: "production"
      workload:
        labels:
          app: "microservice"
      container: "app"
      logPath: "/var/log/app/error*.log"
```

应用配置：

```bash
kubectl apply -f logconfig-file-labels.yaml
```

示例输出：

```text
logconfig.cls.cloud.tencent.com/file-logs-label-selector created
```

### 4. kafkaDetail 字段详解

| 字段 | 必填 | 类型 | 说明 |
|------|------|------|------|
| `brokers` | 是 | `[]string` | Kafka broker 地址列表，格式 `IP:Port` |
| `topic` | 是 | `string` | Kafka 目标 topic，需在 Kafka 集群中提前创建 |
| `messageKey` | 否 | `string` | 每条消息的 key，支持 `pod_name`、`namespace`、`container_name` 等内置变量 |
| `timestampKey` | 否 | `string` | 消息体中时间戳字段名，默认为 `@timestamp` |
| `timestampFormat` | 否 | `string` | 时间戳格式化模板，遵循 Go `time.Format` 布局，如 `"2006-01-02T15:04:05.000Z"` |

### 5. 管理 LogConfig 资源

#### 5.1 查看所有 LogConfig

```bash
kubectl get logconfigs -A
```

示例输出：

```text
NAMESPACE    NAME                      STATUS    AGE
default      stdout-to-kafka-all       Running   5m
production   stdout-to-kafka-nginx     Running   3m
production   file-logs-to-kafka        Running   2m
production   file-logs-label-selector  Running   1m
```

#### 5.2 查看单个 LogConfig 详情

```bash
kubectl describe logconfig stdout-to-kafka-nginx -n production
```

示例输出（关键字段）：

```text
Name:         stdout-to-kafka-nginx
Namespace:    production
Labels:       <none>
Annotations:  <none>
API Version:  cls.cloud.tencent.com/v1
Kind:         LogConfig
Metadata:
  ...
Spec:
  Input Detail:
    Type:                  container_stdout
    Container Stdout:
      All Containers:      false
      Namespace:           production
      Container:           nginx
      Workload:
        Kind:              Deployment
        Name:              nginx-deployment
  Kafka Detail:
    Brokers:
      10.0.2.20:9092
    Message Key:           pod_name
    Timestamp Key:         @timestamp
    Timestamp Format:      2006-01-02T15:04:05.000Z
    Topic:                 nginx-access-logs
Status:
  Conditions:
    Message:               LogConfig is running
    Status:                True
    Type:                  Running
```

#### 5.3 删除 LogConfig

```bash
kubectl delete logconfig stdout-to-kafka-all -n default
```

示例输出：

```text
logconfig.cls.cloud.tencent.com "stdout-to-kafka-all" deleted
```

## 验证

### Data plane (kubectl)

```bash
# 确认 LogConfig 状态为 Running
kubectl get logconfig -A | grep -v "Running" || echo "All LogConfigs are Running"

# 确认具体规则
kubectl get logconfig stdout-to-kafka-nginx -n production -o jsonpath='{.status.conditions[0].type}: {.status.conditions[0].status}'
```

示例输出：

```text
NAME                       STATUS    AGE
stdout-to-kafka-nginx      Running   5m
file-logs-to-kafka         Running   5m
```

### Kafka 端验证

在 Kafka 消费者端确认消息到达：

```bash
kafka-console-consumer --bootstrap-server 10.0.2.20:9092 --topic nginx-access-logs --max-messages 3
```

示例输出（JSON 格式日志消息）：

```json
{"@timestamp":"2024-01-15T10:45:00.000Z","pod_name":"nginx-deployment-7b9f8c6d5-x2k9m","namespace":"production","container_name":"nginx","content":"10.0.1.5 - - [15/Jan/2024:10:44:59 +0000] \"GET / HTTP/1.1\" 200 612"}
```

## 清理

```bash
# 删除所有 LogConfig 资源
kubectl delete logconfig --all -n default
kubectl delete logconfig --all -n production

# 确认已清空
kubectl get logconfig -A
```

```text
NAME  STATUS  AGE
...
```

## 排障

| 现象 | 原因 | 处理 |
|------|------|------|
| LogConfig 状态长时间非 `Running` | 采集组件未就绪或 Kafka 不可达 | `kubectl describe logconfig <name> -n <ns>` 查看 Conditions 详细信息 |
| `kubectl apply` 报 `no matches for kind "LogConfig"` | CRD 未安装（集群日志采集未开通） | 先 [开通日志采集](../开通日志采集/tccli%20操作.md) |
| Kafka 消费者无消息 | broker 地址错误、topic 不存在、或网络不通 | 确认 Kafka 网络可达（同 VPC 或对等连接），topic 已创建 |
| `container_file` 无日志采集 | 日志路径 `logPath` 无匹配文件 | 确认容器内文件路径正确且文件存在；路径支持 `*` 通配符 |
| timestampFormat 解析异常 | 时间戳格式不匹配 Go `time.Format` 布局 | 参考 Go 文档：`2006-01-02T15:04:05.000Z` 为 ISO 8601 带毫秒 |
| kubectl 不可达 | 公网端点 CAM 拒绝（strategyId:240463971） | 先 [连接集群](../../../TKE%20Serverless%20集群管理/连接集群/tccli%20操作.md) 解决连通性 |

## 下一步

- [日志采集配置](../日志采集配置/tccli%20操作.md)（page_id `56750`） — 通过 YAML CRD 配置 CLS 日志采集
- [采集容器内日志](../采集容器内日志/tccli%20操作.md)（page_id `56320`） — 通过 CLS 控制台/CLI 配置采集规则
- [监控和告警](../../监控和告警/查看监控/tccli%20操作.md) — 为日志采集指标配置告警

## 控制台替代

控制台不支持直接配置 Kafka 投递的 LogConfig CRD；LogConfig 到 Kafka 为 YAML CRD 声明式管理，通过 `kubectl` 操作。
