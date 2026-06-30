# 集群审计

> 对照官方：[集群审计](https://cloud.tencent.com/document/product/457/72149) · page_id `72149`

## 概述

TKE 注册集群支持将 Kubernetes Audit 日志持久化到 CLS（日志服务），记录对 API Server 的所有请求（用户、时间、资源、操作、结果）。审计日志对安全合规和排障至关重要。

本文演示：**tccli** 侧在 TKE 管理面为注册集群开启审计日志并指定 CLS 日志主题，**kubectl** 侧在外部集群验证 audit 策略和日志输出。审计日志由外部集群 API Server 生成，通过 agent 转发到 CLS。

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
# 1. 查询 CLS 日志主题（用于接收审计日志）
tccli cls DescribeTopics --region <Region> \
  --Filters '[{"Key":"logsetId","Values":["LOGSET_ID"]}]'
# expected: 确认目标 TopicId 存在

# 2. 确认外部集群 agent 正常（在外部集群执行）
kubectl get pods -n tke-xxx
# expected: agent Pod STATUS 为 Running

# 3. 检查外部集群 API Server 审计配置（在外部集群执行）
kubectl get cm -n kube-system | grep audit
# expected: 若有 audit-policy ConfigMap 则已有审计策略，否则需创建
```

```text
NAME  STATUS  AGE
...
```

### 审计策略说明

- **审计日志级别**：`None`（不记录）、`Metadata`（记录请求元数据）、`Request`（记录请求体）、`RequestResponse`（记录请求和响应体）
- **默认策略**：TKE 开启审计后使用默认审计策略，记录所有请求的 Metadata 级别日志
- **高级策略**：可在外部集群通过 `--audit-policy-file` 自定义审计策略文件，筛选特定资源或操作的日志

## 控制台与 CLI 参数映射

| 控制台操作 | tccli / kubectl 命令 | 幂等 |
|-----------|-----------|:--:|
| 开启审计日志 | `tccli tke EnableExternalClusterAudit --region <Region> --ClusterId CLUSTER_ID --TopicId TOPIC_ID` | 是 |
| 查看审计配置 | `tccli tke DescribeExternalClusterAudit --region <Region> --ClusterId CLUSTER_ID` | 是（只读） |
| 关闭审计日志 | `tccli tke DisableExternalClusterAudit --region <Region> --ClusterId CLUSTER_ID` | 是 |
| 查看审计策略 | `kubectl get cm audit-policy -n kube-system -o yaml`（外部集群） | 是（只读） |
| 查询审计日志 | `tccli cls SearchLog --region <Region> --TopicId TOPIC_ID --From <ts> --To <ts> --Query "<query>"` | 是（只读） |

## 操作步骤

### 步骤1：在外部集群检查/配置审计策略（可选）

> 以下命令在**外部集群 kubectl 管理节点**执行。TKE 默认使用 metadata 级别审计，如需自定义策略才需此步骤。

```bash
# 查看当前审计策略
kubectl get cm audit-policy -n kube-system -o yaml 2>/dev/null
# expected: 若有则返回策略内容；NotFound 则使用 TKE 默认策略
```

若需自定义审计策略，创建 `audit-policy.yaml`：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: audit-policy
  namespace: kube-system
data:
  policy: |
    apiVersion: audit.k8s.io/v1
    kind: Policy
    rules:
    - level: Metadata
    - level: RequestResponse
      resources:
      - group: ""
        resources: ["secrets", "configmaps"]
```

```bash
kubectl apply -f audit-policy.yaml
# expected: configmap/audit-policy created 或 configured
```

### 步骤2：在 TKE 管理面开启审计日志

```bash
tccli tke EnableExternalClusterAudit --region <Region> \
  --ClusterId CLUSTER_ID \
  --TopicId AUDIT_TOPIC_ID \
  --output json
# expected: exit 0，返回 RequestId
```

预期输出：

```json
{
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

### 步骤3：验证 TKE 侧审计配置

```bash
tccli tke DescribeExternalClusterAudit --region <Region> \
  --ClusterId CLUSTER_ID --output json
# expected: AuditEnabled 为 true，TopicId 与配置一致
```

预期输出：

```json
{
    "AuditEnabled": true,
    "TopicId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

### 步骤4：在外部集群验证审计日志生成

> 以下命令在**外部集群 kubectl 管理节点**执行。

```bash
# 执行一些操作以生成审计日志
kubectl get pods --all-namespaces
kubectl get services --all-namespaces
kubectl get configmaps -n kube-system
# expected: 正常返回资源列表（同时生成对应审计日志）
```

```text
NAME  STATUS  AGE
...
```

### 步骤5：查询 CLS 审计日志

```bash
# 等待 2-3 分钟后查询 CLS（审计日志有少量延迟）
tccli cls SearchLog --region <Region> \
  --TopicId AUDIT_TOPIC_ID \
  --From $(date -v-10M +%s)000 \
  --To $(date +%s)000 \
  --Query "pods" \
  --Limit 5 \
  --output json
# expected: 返回审计日志条目（objectRef.resource: pods）
```

审计日志典型字段（在 CLS 中搜索）：

```json
{
    "auditID": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
    "requestURI": "/api/v1/namespaces/default/pods",
    "verb": "list",
    "user": {
        "username": "kubernetes-admin"
    },
    "sourceIPs": ["x.x.x.x"],
    "objectRef": {
        "resource": "pods",
        "namespace": "default"
    },
    "responseStatus": {
        "code": 200
    }
}
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `REGION` | 地域 | 如 `ap-guangzhou` | `tccli configure list` |
| `CLUSTER_ID` | 注册集群 ID | 格式 `cls-xxxxxxxx` | `tccli tke DescribeExternalClusters --region <Region>` |
| `AUDIT_TOPIC_ID` | CLS 审计日志主题 ID | UUID 格式，建议专用日志主题 | `tccli cls DescribeTopics --region <Region>` |

## 验证

### TKE 管理面验证

```bash
tccli tke DescribeExternalClusterAudit --region <Region> \
  --ClusterId CLUSTER_ID
# expected: AuditEnabled 为 true
```

```json
{"RequestId": "..."}
```

### CLS 审计日志验证

```bash
# 查询最近 15 分钟的审计日志
tccli cls SearchLog --region <Region> \
  --TopicId AUDIT_TOPIC_ID \
  --From $(date -v-15M +%s)000 \
  --To $(date +%s)000 \
  --Query "*" \
  --Limit 10 \
  --output json | jq '.Results | length'
# expected: > 0

# 按操作类型查询（列出最近 list 和 get 操作）
tccli cls SearchLog --region <Region> \
  --TopicId AUDIT_TOPIC_ID \
  --From $(date -v-15M +%s)000 \
  --To $(date +%s)000 \
  --Query "verb:list OR verb:get" \
  --Limit 5 \
  --output json | jq '.Results[] | {verb, resource: .objectRef.resource, user: .user.username}'
# expected: 返回操作记录列表
```

### 外部集群验证（kubectl）

```bash
# agent 正常转发审计日志
kubectl logs -n tke-xxx deployment/tke-agent --tail=20 | grep audit
# expected: 有 audit 相关日志，无 ERROR 级别
```

```text
NAME  STATUS  AGE
...
```

## 清理

> **警告**：关闭审计后，新的 API Server 请求不再记录到 CLS。已采集的历史审计日志不受影响，按 CLS 数据保留策略管理。
> **计费提醒**：关闭审计停止写入后，CLS 日志主题仍按存储量计费。如需完全停止计费，删除 CLS 日志主题。

### 1. 关闭 TKE 审计

```bash
tccli tke DisableExternalClusterAudit --region <Region> \
  --ClusterId CLUSTER_ID --output json
# expected: exit 0
```

### 2. 验证已关闭

```bash
tccli tke DescribeExternalClusterAudit --region <Region> \
  --ClusterId CLUSTER_ID
# expected: AuditEnabled 为 false
```

```json
{"RequestId": "..."}
```

### 3. 清理 CLS 审计日志主题（可选，不可逆）

```bash
# > 警告：删除日志主题会永久删除所有审计日志，不可恢复
tccli cls DeleteTopic --region <Region> --TopicId AUDIT_TOPIC_ID
# expected: exit 0
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `EnableExternalClusterAudit` 返回 `ResourceNotFound` | `DescribeExternalClusters` 确认集群存在 | 集群 ID 错误或 agent 未部署 | 确认 `ClusterId` 正确，检查 agent 状态 |
| `EnableExternalClusterAudit` 返回 `InvalidParameter.TopicId` | `tccli cls DescribeTopics` 验证 | TopicId 不存在或地域不匹配 | 确认审计日志主题在目标地域 |
| `EnableExternalClusterAudit` 返回 `AuthFailure` | `tccli configure list` 检查凭据 | CAM 缺少 TKE 或 CLS 权限 | 授予 `tke:EnableExternalClusterAudit` 和 `cls:*` 权限 |
| `EnableExternalClusterAudit` 返回 `FailedOperation.AuditAlreadyEnabled` | `DescribeExternalClusterAudit` 查看当前状态 | 审计已开启 | 如需更新配置，先 `DisableExternalClusterAudit` 再重新 Enable |

### 操作成功但异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| TKE 侧配置成功但 CLS 查不到审计日志 | 等待 5+ 分钟后重试 | 审计日志采集和上报有延迟（首次可能更长） | 继续等待；在外部集群执行 `kubectl get pods` 主动生成审计日志 |
| 审计日志只有 Metadata 没有 Request Body | `SearchLog` 检查 `requestObject` 字段 | 审计策略默认级别为 Metadata | 自定义 audit-policy ConfigMap，提升关键资源的审计级别为 RequestResponse |
| 审计日志量过大 | `SearchLog` 统计单位时间日志量 | 审计策略记录所有操作 | 自定义 audit-policy 添加 exclude 规则，过滤高频低风险操作（如 kubelet 心跳） |
| agent 日志中有 CLS 连接错误 | `kubectl logs -n tke-xxx deployment/tke-agent` | agent 无法访问 CLS endpoint | 确认外部集群节点能出公网访问 `REGION.cls.tencentyun.com`，检查防火墙/NAT |

## 下一步

- [日志采集](../日志采集/tccli%20操作.md) — 配置容器日志采集（page_id `72148`）
- [事件存储](../事件存储/tccli%20操作.md) — 持久化 Kubernetes Events（page_id `72150`）
- [CLS 控制台](https://console.cloud.tencent.com/cls) — 检索和分析审计日志

## 控制台替代

[容器服务控制台 → 集群列表](https://console.cloud.tencent.com/tke2/external) 选中注册集群 → **运维功能 → 集群审计** → 开启并选择 CLS 日志主题。
