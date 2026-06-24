# 日志采集（tccli）

> 对照官方：[日志采集](https://cloud.tencent.com/document/product/457/72148) · page_id `72148`

## 概述

为注册集群配置**日志采集**，将容器标准输出、容器文件日志、节点文件日志采集至腾讯云日志服务（CLS）。采集组件以 DaemonSet 形式运行 `loglistener`，支持多种日志提取模式与投递方式（公网/内网）。

## 前置条件

- 注册集群已创建并处于 `Running` 状态，可通过 kubectl 访问。
- CLS 日志服务已开通，目标地域与集群一致（如 `ap-guangzhou`）。
- 采集器 `loglistener` 作为 DaemonSet 部署，每个节点需预留资源：**CPU: 0.11 ~ 1.1 cores**，**Memory: 24 ~ 560 MB**。
- 单条日志行上限为 **512 KB**，超出会被截断。
- 投递方式选择：公网（需节点可通过公网访问 CLS）、内网（需集群所在网络与 CLS 内网互通）。

## 控制台与 CLI 参数映射

| 控制台 | tccli | 幂等 |
|--------|-------|------|
| 创建日志集 | `tccli cls CreateLogset` | 否 |
| 查询日志集 | `tccli cls DescribeLogsets` | 是 |
| 创建日志主题 | `tccli cls CreateTopic` | 否 |
| 查询日志主题 | `tccli cls DescribeTopics` | 是 |
| 开启日志采集 | 控制台操作（TKE 运维功能管理）；CLI 通过 TKE APIs 配置采集规则 | 否 |
| 更新采集规则 | 控制台修改规则 | 否 |
| 查看采集状态 | `kubectl get pod -n kube-system \| grep loglistener` | 是 |
| 检索日志 | `tccli cls SearchLog` | 是 |

## 操作步骤

### 1. 创建 CLS 日志集与日志主题

日志集（Logset）是日志主题（Topic）的容器，一个日志集最多包含 500 个主题。

```bash
# 创建日志集
tccli cls CreateLogset \
  --LogsetName "tke-registered-cluster-logs" \
  --region ap-guangzhou \
  --output json
```

```json
{
  "LogsetId": "logset-tke-registered-001",
  "RequestId": "1a2b3c4d-5678-90ab-cdef-1234567890ab"
}
```

```bash
# 为容器标准输出创建日志主题
tccli cls CreateTopic \
  --LogsetId logset-tke-registered-001 \
  --TopicName "registry-cluster-stdout" \
  --region ap-guangzhou \
  --output json
```

```json
{
  "TopicId": "topic-stdout-abc123",
  "RequestId": "2b3c4d5e-6789-0abc-def1-234567890abc"
}
```

```bash
# 为容器文件路径创建日志主题
tccli cls CreateTopic \
  --LogsetId logset-tke-registered-001 \
  --TopicName "registry-cluster-file" \
  --region ap-guangzhou \
  --output json
```

```json
{
  "TopicId": "topic-file-def456",
  "RequestId": "3c4d5e6f-7890-abcd-ef12-34567890abcd"
}
```

### 2. 开启日志采集（控制台 → CLI 适配）

注册集群的日志采集通过 TKE 控制台「运维功能管理」入口配置。在控制台操作后，底层实际通过 TKE API 或通过 `kubectl apply` 配置 `LogConfig` CRD 实现。以下展示 **CRD 方式**（直接 kubectl 操作）：

**采集类型一：容器标准输出**

创建采集规则，采集所有命名空间下的容器标准输出（stdout/stderr）：

```yaml
cat << 'EOF' | kubectl apply -f -
apiVersion: cls.cloud.tencent.com/v1
kind: LogConfig
metadata:
  name: tke-registered-stdout
  namespace: clusternet-system
spec:
  clsDetail:
    logsetId: logset-tke-registered-001
    topicId: topic-stdout-abc123
    region: ap-guangzhou
    logType: minimalist_log
    extractRule:
      beginningRegex: ".*"
      keys: ["content"]
      timeKey: ""
      timeFormat: ""
      unMatchUpload: "true"
      unMatchedKey: "LOG_PARSE_FAILURE"
      backtracking: "1"
  inputDetail:
    type: container_stdout
    containerStdout:
      allContainers: true
      namespace: ""
      workload:
        name: ""
        kind: ""
      metadataLabels:
        app: "*"
    metadata:
      container_id: true
      container_name: true
      image_name: true
      namespace: true
      pod_uid: true
      pod_ip: true
      pod_name: true
      pod_label_app: true
  scaleTargetRef:
    apiVersion: apps/v1
    kind: DaemonSet
    name: loglistener
EOF
```

**采集类型二：容器文件路径**

采集 `/var/log/nginx/access.log` 路径下的容器日志：

```yaml
cat << 'EOF' | kubectl apply -f -
apiVersion: cls.cloud.tencent.com/v1
kind: LogConfig
metadata:
  name: tke-registered-nginx-log
  namespace: default
spec:
  clsDetail:
    logsetId: logset-tke-registered-001
    topicId: topic-file-def456
    region: ap-guangzhou
    logType: json_log
    extractRule:
      keys: ["time", "level", "msg"]
      timeKey: "time"
      timeFormat: "%Y-%m-%dT%H:%M:%S"
  inputDetail:
    type: container_file
    containerFile:
      namespace: default
      workload:
        name: nginx-app
        kind: Deployment
      container: nginx
      logPath: /var/log/nginx
      filePattern: "access.log"
      customLabels: {}
EOF
```

**采集类型三：节点文件路径**

采集节点上 `/var/log/messages`（系统日志）：

```yaml
cat << 'EOF' | kubectl apply -f -
apiVersion: cls.cloud.tencent.com/v1
kind: LogConfig
metadata:
  name: tke-registered-node-syslog
spec:
  clsDetail:
    logsetId: logset-tke-registered-001
    topicId: topic-file-def456
    region: ap-guangzhou
    logType: multiline_log
    extractRule:
      beginningRegex: '^\w{3}\s+\d{1,2}\s+\d{2}:\d{2}:\d{2}'
      keys: ["timestamp", "content"]
      timeKey: "timestamp"
      timeFormat: "%b %d %H:%M:%S"
  inputDetail:
    type: host_file
    hostFile:
      labels:
        node-role: "worker"
      logPath: /var/log
      filePattern: "messages"
      customLabels:
        env: production
EOF
```

```text
...log output...
```

### 3. 支持的日志提取模式

| 模式 | 对应 `logType` | 说明 |
|------|---------------|------|
| 单行全文 | `minimalist_log` | 每行作为一条日志，`keys: ["content"]` |
| JSON 格式 | `json_log` | 自动解析 JSON 字段，需指定 `keys` |
| 分隔符 | `delimiter_log` | 按分隔符（如 `\|`）切分字段 |
| 多行全文 | `multiline_log` | 按正则首行匹配聚合多行，指定 `beginningRegex` |
| 完全正则 | `fullregex_log` | 用正则表达式提取字段 |

### 4. 更新日志采集规则

修改已存在的 LogConfig：

```bash
# 修改采集规则（如增加 metadata 标签）
kubectl edit logconfig tke-registered-stdout -n clusternet-system

# 或通过 patch
kubectl patch logconfig tke-registered-stdout -n clusternet-system \
  --type merge \
  -p '{"spec":{"inputDetail":{"containerStdout":{"metadataLabels":{"app":"myapp"}}}}}'
```

## 验证

### 检查 loglistener DaemonSet 运行状态

```bash
kubectl get pod -n kube-system | grep loglistener
```

```
NAME                READY   STATUS    RESTARTS   AGE
loglistener-8xq2k   1/1     Running   0          5m
loglistener-p4v7n   1/1     Running   0          5m
loglistener-z9y1x   1/1     Running   0          5m
```

每个节点应有一个 `loglistener` Pod 处于 `Running` 状态。

### 检查 LogConfig 状态

```bash
kubectl get logconfig -A
```

```
NAMESPACE           NAME                        AGE
clusternet-system   tke-registered-stdout       5m
default             tke-registered-nginx-log    5m
-                   tke-registered-node-syslog  5m
```

### 验证日志已上报到 CLS

```bash
tccli cls SearchLog \
  --TopicId topic-stdout-abc123 \
  --From 1715900000000 \
  --To 1715903600000 \
  --Query "*" \
  --Limit 5 \
  --region ap-guangzhou \
  --output json
```

```json
{
  "Context": "",
  "ListOver": true,
  "Analysis": false,
  "Results": [
    {
      "Time": 1715901234567,
      "TopicId": "topic-stdout-abc123",
      "TopicName": "registry-cluster-stdout",
      "Source": "10.0.1.10",
      "LogJson": "{\"content\":\"2025-01-15T08:00:01.234Z INFO Starting application\",\"namespace\":\"default\",\"pod_name\":\"nginx-app-7d9f8c5b4d-abc12\",\"container_name\":\"nginx\"}"
    }
  ],
  "RequestId": "d7e8f9a0-1234-5678-90ab-cdef01234567"
}
```

### 检查 loglistener 资源使用

```bash
kubectl top pod -n kube-system -l app=loglistener
```

```
NAME                CPU(cores)   MEMORY(bytes)
loglistener-8xq2k   50m          128Mi
loglistener-p4v7n   45m          115Mi
loglistener-z9y1x   60m          140Mi
```

## 清理

### 停止日志采集（删除 LogConfig）

```bash
kubectl delete logconfig tke-registered-stdout -n clusternet-system
kubectl delete logconfig tke-registered-nginx-log -n default
kubectl delete logconfig tke-registered-node-syslog
```

### 清理 CLS 资源

```bash
# 删除日志主题
tccli cls DeleteTopic --TopicId topic-stdout-abc123 --region ap-guangzhou
tccli cls DeleteTopic --TopicId topic-file-def456 --region ap-guangzhou

# 删除日志集（需先删除所有主题）
tccli cls DeleteLogset --LogsetId logset-tke-registered-001 --region ap-guangzhou
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `loglistener` Pod 未调度 | `kubectl describe pod -n kube-system <loglistener-pod>` 查看事件 | 节点资源不足（每节点需至少 0.11 core / 24 Mi）或有污点未容忍 | 增加节点资源或减少污点；检查 nodeSelector/affinity |
| `loglistener` CrashLoopBackOff | `kubectl logs -n kube-system <loglistener-pod>` 查看日志 | CLS Topic/Logset ID 无效或凭证配置错误 | 用 `tccli cls DescribeTopics` / `DescribeLogsets` 确认 ID 正确且存在于目标地域 |
| 日志未上报到 CLS | `kubectl logs -n kube-system <loglistener-pod>` 检查投递状态 | 网络不通：节点无法通过选择的投递方式连通 CLS | 检查安全组/防火墙放通 CLS 端口；内网投递确认路由/云联网 |
| 单行日志超过 512KB 被截断 | `wc -c <log_file>` 检查日志大小 | CLS 单行上限 512KB（产品限制，不可配置） | 调整应用日志输出大小或将大日志拆分为多行 |
| JSON 解析失败（`LOG_PARSE_FAILURE`） | `echo '<log_line>' \| python3 -m json.tool` 验证 JSON | `keys` 字段与日志 JSON 结构不匹配，或输出非合法 JSON | 修正 `keys` 匹配日志结构；确认应用输出合法 JSON |
| 多行日志聚合异常 | `kubectl logs -n kube-system <loglistener-pod>` 检查解析日志 | `beginningRegex` 未正确匹配每行起始特征 | 修正正则表达式使其匹配每行日志起始模式 |
| LogConfig status 为 Error | `kubectl get logconfig -n <Namespace> -o yaml` 查看 status | `logsetId`/`topicId` 在 CLS 中不存在或地域不一致 | 用 `tccli cls DescribeLogsets`/`DescribeTopics` 确认 ID 正确且地域一致 |

## 下一步

- [集群审计](../../运维指南/集群审计/tccli%20操作.md) — 配置 API Server 审计日志采集
- [事件存储](../../运维指南/事件存储/tccli%20操作.md) — 将集群事件持久化到 CLS
- CLS 控制台检索与仪表盘：使用 `tccli cls SearchLog` 查询采集到的日志

## 控制台替代

[控制台 → 注册集群 → 运维功能管理 → 日志采集 → 设置](https://console.cloud.tencent.com/tke2/cluster?rid=1) — 可视化选择采集类型（标准输出/容器文件/节点文件）、CLS 日志主题、提取模式、投递方式。
