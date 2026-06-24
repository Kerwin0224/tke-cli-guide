# 可观测性集成（tccli）

> 对照官方：[可观测性集成](https://cloud.tencent.com/document/product/457/104861) · page_id `104861`

## 概述

为自建 Nginx Ingress Controller 集成可观测性能力：启用 Prometheus Metrics 暴露指标、接入 Grafana 可视化仪表盘、配置 CLS 日志服务收集 Nginx 访问和错误日志。

**可观测性矩阵**：

| 维度 | 工具 | 实现方式 |
|------|------|---------|
| 指标（Metrics） | Prometheus + Grafana | 启用 Nginx Ingress `metrics` 开关，Prometheus 自动发现采集 |
| 日志（Logs） | CLS（日志服务） | Sidecar 或 DaemonSet 采集 Nginx access/error 日志 |
| 链路追踪（Tracing） | 可选 | Nginx Ingress 支持 OpenTracing 插件 |

> **kubectl 数据面不可达**：CAM 拒绝公网端点（strategyId:240463971），内网需 VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

## 前置条件

- [环境准备](../../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId、secretKey、region 均已配置

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:DescribeClusters, tke:DescribeClusterKubeconfig
#    cls:CreateTopic, cls:CreateLogset, cls:DescribeTopics
#    cls:DescribeLogsets
# 验证
tccli tke DescribeClusters --region <Region>
# expected: exit 0

tccli cls DescribeTopics --region <Region>
# expected: exit 0

# 4. 检查 kubectl 和 Helm（须 VPN/IOA）
kubectl version --client
# expected: Client Version v1.30+
helm version --short
# expected: v3.x
```

```json
{
  "TotalCount": "<TotalCount>",
  "Clusters": "<Clusters>",
  "ClusterId": "<ClusterId>",
  "ClusterName": "<ClusterName>",
  "ClusterDescription": "<ClusterDescription>",
  "ClusterVersion": "<ClusterVersion>"
}
```

### 资源检查

```bash
# 5. 确认 Nginx Ingress 已安装
helm list -n ingress-nginx
# expected: nginx-ingress 状态 deployed

# 6. 确认 Prometheus 已部署（如使用 TKE 监控组件）
tccli tke DescribeAddon --region <Region> \
    --ClusterId CLUSTER_ID --AddonName prometheus
# expected: 组件状态为已安装或确认可用

# 7. 确认 CLS 日志集和主题可用
tccli cls DescribeLogsets --region <Region>
# expected: 返回日志集列表
```

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 查看集群组件 | `tccli tke DescribeAddon` | 是 |
| 创建 CLS 日志集 | `tccli cls CreateLogset` | 否 |
| 创建 CLS 日志主题 | `tccli cls CreateTopic` | 否 |
| 查看 CLS 日志主题 | `tccli cls DescribeTopics` | 是 |
| 启用 Nginx Metrics | `helm upgrade`（设置 `controller.metrics.enabled`） | 否 |
| 查看 ServiceMonitor | `kubectl get servicemonitor` | 是 |
| 部署 LogCollector | `kubectl apply -f` | 否 |

## 关键字段说明

### Helm Values

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `controller.metrics.enabled` | Boolean | 否 | `true` 启用 Prometheus 指标暴露 | 不启用 → Prometheus 无法采集 Nginx Ingress 指标 |
| `controller.metrics.serviceMonitor.enabled` | Boolean | 否 | `true` 自动创建 ServiceMonitor（需 Prometheus Operator） | 未创建 → Prometheus 不会自动发现采集目标 |
| `controller.metrics.serviceMonitor.namespace` | String | 否 | Prometheus 所在命名空间 | 填错命名空间 → Prometheus 找不到 ServiceMonitor |

### CLS API

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `LogsetName` | String | 是 | CLS 日志集名称 | 重名 → `InvalidParameter.DupLogsetName` |
| `TopicName` | String | 是 | 日志主题名称，如 `nginx-ingress-access` | 重名 → `InvalidParameter.DupTopicName` |
| `Period` | Integer | 是 | 日志保存天数，1-3600 | 设错 → 日志提前删除 |
| `PartitionCount` | Integer | 否 | 分区数，默认 1 | 分区数过少 → 写入性能受限 |

## 操作步骤

### 步骤 1：启用 Nginx Ingress Prometheus Metrics

#### 选择依据

- **Prometheus 采集方式**：推荐通过 ServiceMonitor（Prometheus Operator）自动发现，无需手动配置 Prometheus 采集目标。
- **TKE 环境**：如果集群已安装 TKE 监控组件（prometheus addon），ServiceMonitor 会被自动识别。

`nginx-ingress-metrics.yaml`：

```yaml
controller:
  metrics:
    enabled: true
    serviceMonitor:
      enabled: true
      namespace: "CLUSTER_ID"
```

```bash
helm upgrade nginx-ingress ingress-nginx/ingress-nginx \
    --namespace ingress-nginx \
    -f nginx-ingress-metrics.yaml \
    --reuse-values
# expected: STATUS: deployed
```

> **kubectl 数据面不可达**：CAM 拒绝公网端点（strategyId:240463971），内网需 VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

### 步骤 2：创建 CLS 日志集和日志主题

#### 选择依据

- **CLS vs 自建 ELK**：TKE 集群推荐使用 CLS，与腾讯云深度集成，免运维。
- **采集方式**：使用 CLS LogCollector（DaemonSet）自动采集标准输出或文件日志。

#### 创建日志集

`cls-logset.json`：

```json
{
  "LogsetName": "nginx-ingress-logs",
  "Period": 30
}
```

```bash
tccli cls CreateLogset --region <Region> --cli-input-json file://cls-logset.json
# expected: exit 0，返回 LogsetId
```

```json
{
  "LogsetId": "LOG_SET_ID"
}
```

#### 创建日志主题

`cls-topic.json`：

```json
{
  "LogsetId": "LOG_SET_ID",
  "TopicName": "nginx-ingress-access",
  "Period": 30,
  "PartitionCount": 1
}
```

```bash
tccli cls CreateTopic --region <Region> --cli-input-json file://cls-topic.json
# expected: exit 0，返回 TopicId
```

```json
{
  "TopicId": "TOPIC_ID"
}
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `LOG_SET_ID` | CLS 日志集 ID | 由上一步 `CreateLogset` 返回 | `tccli cls DescribeLogsets --region <Region>` |
| `TOPIC_ID` | CLS 日志主题 ID | 由 `CreateTopic` 返回 | `tccli cls DescribeTopics --region <Region>` |

### 步骤 3：配置 Nginx Ingress 日志输出格式

#### 配置 JSON 格式日志（便于 CLS 结构化查询）

`nginx-ingress-logging.yaml`：

```yaml
controller:
  config:
    log-format-upstream: '{"time":"$time_iso8601","remote_addr":"$remote_addr","host":"$host","method":"$request_method","uri":"$uri","status":$status,"request_time":$request_time,"upstream_response_time":$upstream_response_time,"upstream_addr":"$upstream_addr","http_user_agent":"$http_user_agent","http_referer":"$http_referer"}'
```

```bash
helm upgrade nginx-ingress ingress-nginx/ingress-nginx \
    --namespace ingress-nginx \
    -f nginx-ingress-logging.yaml \
    --reuse-values
# expected: STATUS: deployed
```

### 步骤 4：部署 CLS LogCollector（参考步骤）

CLS LogCollector 通过 TKE 集群组件安装，DaemonSet 形式运行在每个节点上：

```bash
# 查看 TKE 日志采集组件
tccli tke DescribeAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName cls
# expected: 组件状态，如未安装则安装
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

如果需要手动静音安装 CLS LogCollector（数据面操作）：

```bash
# 通过 CLS 控制台获取安装脚本（参考格式）
kubectl apply -f https://CLS_LOG_COLLECTOR_URL
# expected: daemonset created
```

## 验证

### 控制面（tccli）

```bash
# 验证 CLS 日志主题
tccli cls DescribeTopics --region <Region> \
    --Filters '[{"Key":"logsetId","Values":["LOG_SET_ID"]}]'
```

```json
{
  "Topics": [
    {
      "TopicId": "TOPIC_ID",
      "TopicName": "nginx-ingress-access",
      "Status": true
    }
  ]
}
```

```bash
# 验证 CLS LogCollector 组件状态
tccli tke DescribeAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName cls
# expected: 组件状态为已安装
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

### 数据面（需 VPN/IOA）

```bash
# 验证 Metrics 端点
kubectl -n ingress-nginx get svc nginx-ingress-ingress-nginx-controller-metrics
# expected: metrics Service 存在

kubectl -n ingress-nginx port-forward svc/nginx-ingress-ingress-nginx-controller-metrics 10254:10254 &
curl -s http://localhost:10254/metrics | head -20
# expected: 输出 Nginx Ingress Prometheus metrics（nginx_ingress_controller_requests 等）

# 验证 ServiceMonitor
kubectl get servicemonitor -n ingress-nginx
# expected: nginx-ingress-ingress-nginx-controller 存在

# 验证日志格式（查看 Controller 日志输出）
kubectl -n ingress-nginx logs -l app.kubernetes.io/name=ingress-nginx --tail=5
# expected: JSON 格式的访问日志

# 验证 LogCollector Pod 运行
kubectl -n kube-system get pods -l app=loglistener
# expected: 每节点一个 Pod Running
```

```text
NAME  STATUS  AGE
...
```

## 清理

### 数据面（需 VPN/IOA）

```bash
# 关闭 Metrics
helm upgrade nginx-ingress ingress-nginx/ingress-nginx \
    --namespace ingress-nginx \
    --set controller.metrics.enabled=false
# expected: STATUS: deployed
```

### 控制面（tccli）

> **⚠️ 警告**：删除 CLS 日志主题会同时删除其中的所有日志数据。确认无合规保留需求后再执行。

```bash
# 删除 CLS 日志主题
tccli cls DeleteTopic --region <Region> --TopicId TOPIC_ID
# expected: exit 0

# 删除 CLS 日志集（需先删除所有主题）
tccli cls DeleteLogset --region <Region> --LogsetId LOG_SET_ID
# expected: exit 0

# 验证已删除
tccli cls DescribeTopics --region <Region> \
    --Filters '[{"Key":"topicId","Values":["TOPIC_ID"]}]'
# expected: Topics 为空
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `CreateLogset` 返回 `InvalidParameter.DupLogsetName` | `tccli cls DescribeLogsets --region <Region>` 检查是否有同名日志集 | 日志集名称重复 | 更换 `LogsetName`，如 `nginx-ingress-logs-v2` |
| `CreateTopic` 返回 `InvalidParameter.DupTopicName` | `tccli cls DescribeTopics --region <Region>` 检查同名主题 | 日志主题名称重复 | 更换 `TopicName` |
| Prometheus 无 Nginx Ingress 指标 | `kubectl -n ingress-nginx get endpoints` 检查 metrics 端口 | ServiceMonitor namespace 配置错误或 Prometheus 不在同一命名空间 | 修改 `serviceMonitor.namespace` 为 Prometheus Operator 所在命名空间 |

### 配置生效后功能异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Prometheus 收不到指标 | `kubectl -n ingress-nginx port-forward svc/... 10254:10254 & curl localhost:10254/metrics` | Metrics Service 未创建或端口不通 | 确认 `controller.metrics.enabled: true`；检查 Service 是否存在 |
| CLS 无日志数据 | `tccli cls DescribeTopics --region <Region>` 检查主题状态 | LogCollector 未安装或采集路径配置错误 | 确认 `tke DescribeAddon --AddonName cls` 已安装；检查 LogCollector 日志：`kubectl -n kube-system logs -l app=loglistener` |
| Grafana 仪表盘无数据 | 检查 Prometheus 数据源 URL 和 Dashboard 变量 | Dashboard 数据源或 Job 名称不匹配 | 确认 DataSource 指向正确的 Prometheus URL；Dashboard 变量 `job` 为 `ingress-nginx` |
| JSON 日志格式解析失败 | `kubectl -n ingress-nginx logs -l app.kubernetes.io/name=ingress-nginx --tail=1 | jq` | JSON 格式有语法错误（单个引号或换行符） | 检查 `log-format-upstream` 的 JSON 字符串是否正确转义 |

## 下一步

- [高并发场景优化](https://cloud.tencent.com/document/product/457/104859) — Worker 进程、keepalive、sysctl 调优
- [高可用配置优化](https://cloud.tencent.com/document/product/457/104860) — PDB、拓扑约束、健康检查
- [接入腾讯云 WAF](https://cloud.tencent.com/document/product/457/104862) — 为 Ingress 接入 WAF 防护

## 控制台替代

通过 [TKE 控制台 - 监控](https://console.cloud.tencent.com/tke2/cluster/sub/list/monitor?rid=1) 查看集群监控，通过 [CLS 控制台](https://console.cloud.tencent.com/cls) 管理日志集和日志检索。
