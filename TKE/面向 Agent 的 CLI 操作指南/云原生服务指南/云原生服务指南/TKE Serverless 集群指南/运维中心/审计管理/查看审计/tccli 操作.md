# 查看审计（tccli）

> 对照官方：[审计仪表盘](https://cloud.tencent.com/document/product/457/58243) · page_id `58243`

## 概述

通过 tccli 的 CLS `SearchLog` 命令查询 TKE Serverless 集群审计日志，并基于查询结果配置告警。TKE 控制台提供四个审计仪表盘，本页将每个仪表盘的核心查询逻辑适配为可执行的 CLS 检索语句，供 Agent 和开发者直接使用。

四个审计仪表盘及其功能：

| 仪表盘 | 功能 | 目的 |
|--------|------|------|
| 审计总览 | 审计事件总量、操作者分布、操作类型分布 | 宏观了解集群审计态势 |
| K8S 对象操作概览 | 按资源类型（Pod/Deployment/Service 等）统计操作 | 了解各类资源操作频率 |
| 聚合检索 | 按用户名、命名空间、资源类型等维度聚合分析 | 定位特定用户或资源的审计事件 |
| 全局检索 | 自由组合条件搜索所有审计事件 | 精确检索特定操作记录 |

## 前置条件

- TKE Serverless 集群已创建且状态为 `Running`，参见 [创建集群](../../../TKE%20Serverless%20集群管理/创建集群/tccli%20操作.md)。
- [环境准备](../../../../环境准备.md)：`tccli` 已安装并配置 `region`（本文以 `ap-guangzhou` 为例）。
- 集群审计已 [开启](../开启集群审计/tccli%20操作.md)（page_id `58242`），审计事件正在投递到 CLS 日志主题。
- 已获取审计日志目标 CLS 日志主题 ID（`TopicId`），可在控制台 **运维中心 → 审计管理** 页面查看。

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|------|
| 查看审计总览 | `cls SearchLog --Query ...` | 是 |
| 查看 K8S 对象操作概览 | `cls SearchLog --Query ...` | 是 |
| 聚合检索 | `cls SearchLog --Query ...` (带 GROUP BY / 聚合) | 是 |
| 全局检索 | `cls SearchLog --Query ...` (自由查询) | 是 |
| 创建告警策略 | `cls CreateAlarm` | 否 |
| 查看告警策略 | `cls DescribeAlarms` | 是 |
| 查看 CLS 日志主题 | `cls DescribeTopics` | 是 |

## 操作步骤

### 1. 审计总览

审计总览仪表盘提供集群审计事件的宏观统计视图。

#### 1.1 审计事件总量（指定时间范围）

```bash
# 查询最近 1 小时的审计事件总数
tccli cls SearchLog \
    --TopicId "topic-example-audit" \
    --From $(date -v-1H +%s)000 \
    --To $(date +%s)000 \
    --Query "* | SELECT COUNT(*) AS total_events" \
    --region ap-guangzhou \
    --output json
```

示例输出：

```json
{
    "AnalysisRecords": [
        {
            "total_events": 1523
        }
    ]
}
```

#### 1.2 按操作者（用户）统计

```bash
tccli cls SearchLog \
    --TopicId "topic-example-audit" \
    --From $(date -v-1H +%s)000 \
    --To $(date +%s)000 \
    --Query "* | SELECT \"user.username\" AS username, COUNT(*) AS count GROUP BY username ORDER BY count DESC LIMIT 10" \
    --region ap-guangzhou \
    --output json
```

示例输出：

```json
{
    "AnalysisRecords": [
        {"username": "kubernetes-admin", "count": 856},
        {"username": "system:serviceaccount:kube-system:deployment-controller", "count": 234},
        {"username": "system:serviceaccount:kube-system:replicaset-controller", "count": 189},
        {"username": "100012345678", "count": 67}
    ]
}
```

#### 1.3 按操作类型（verb）统计

```bash
tccli cls SearchLog \
    --TopicId "topic-example-audit" \
    --From $(date -v-1H +%s)000 \
    --To $(date +%s)000 \
    --Query "* | SELECT verb, COUNT(*) AS count GROUP BY verb ORDER BY count DESC" \
    --region ap-guangzhou \
    --output json
```

示例输出：

```json
{
    "AnalysisRecords": [
        {"verb": "get", "count": 612},
        {"verb": "list", "count": 423},
        {"verb": "watch", "count": 201},
        {"verb": "create", "count": 89},
        {"verb": "update", "count": 78},
        {"verb": "delete", "count": 45},
        {"verb": "patch", "count": 39}
    ]
}
```

#### 1.4 按响应状态码统计

```bash
tccli cls SearchLog \
    --TopicId "topic-example-audit" \
    --From $(date -v-1H +%s)000 \
    --To $(date +%s)000 \
    --Query "* | SELECT \"responseStatus.code\" AS status_code, COUNT(*) AS count GROUP BY status_code ORDER BY count DESC" \
    --region ap-guangzhou \
    --output json
```

示例输出：

```json
{
    "AnalysisRecords": [
        {"status_code": "200", "count": 1012},
        {"status_code": "201", "count": 89},
        {"status_code": "404", "count": 23},
        {"status_code": "403", "count": 5},
        {"status_code": "409", "count": 2}
    ]
}
```

### 2. K8S 对象操作概览

按 Kubernetes 资源类型统计操作频率。

#### 2.1 按资源类型统计

```bash
tccli cls SearchLog \
    --TopicId "topic-example-audit" \
    --From $(date -v-1H +%s)000 \
    --To $(date +%s)000 \
    --Query "* | SELECT \"objectRef.resource\" AS resource, COUNT(*) AS count GROUP BY resource ORDER BY count DESC LIMIT 10" \
    --region ap-guangzhou \
    --output json
```

示例输出：

```json
{
    "AnalysisRecords": [
        {"resource": "pods", "count": 342},
        {"resource": "endpoints", "count": 156},
        {"resource": "configmaps", "count": 89},
        {"resource": "deployments", "count": 67},
        {"resource": "services", "count": 54},
        {"resource": "secrets", "count": 34},
        {"resource": "replicasets", "count": 28},
        {"resource": "nodes", "count": 23}
    ]
}
```

#### 2.2 按资源类型 + 操作动词统计

```bash
tccli cls SearchLog \
    --TopicId "topic-example-audit" \
    --From $(date -v-1H +%s)000 \
    --To $(date +%s)000 \
    --Query "* | SELECT \"objectRef.resource\" AS resource, verb, COUNT(*) AS count GROUP BY resource, verb ORDER BY count DESC LIMIT 20" \
    --region ap-guangzhou \
    --output json
```

示例输出：

```json
{
    "AnalysisRecords": [
        {"resource": "pods", "verb": "get", "count": 156},
        {"resource": "pods", "verb": "list", "count": 89},
        {"resource": "pods", "verb": "create", "count": 34},
        {"resource": "deployments", "verb": "update", "count": 28}
    ]
}
```

### 3. 聚合检索

按多维度组合聚合分析审计事件，定位特定用户或资源的操作行为。

#### 3.1 按用户名 + 资源类型聚合

```bash
tccli cls SearchLog \
    --TopicId "topic-example-audit" \
    --From $(date -v-1H +%s)000 \
    --To $(date +%s)000 \
    --Query "* | SELECT \"user.username\" AS username, \"objectRef.resource\" AS resource, COUNT(*) AS count GROUP BY username, resource ORDER BY count DESC LIMIT 20" \
    --region ap-guangzhou \
    --output json
```

#### 3.2 按命名空间聚合

```bash
tccli cls SearchLog \
    --TopicId "topic-example-audit" \
    --From $(date -v-1H +%s)000 \
    --To $(date +%s)000 \
    --Query "* | SELECT \"objectRef.namespace\" AS namespace, COUNT(*) AS count GROUP BY namespace ORDER BY count DESC LIMIT 10" \
    --region ap-guangzhou \
    --output json
```

示例输出：

```json
{
    "AnalysisRecords": [
        {"namespace": "default", "count": 412},
        {"namespace": "kube-system", "count": 298},
        {"namespace": "production", "count": 156},
        {"namespace": "staging", "count": 67}
    ]
}
```

#### 3.3 按来源 IP 聚合

```bash
tccli cls SearchLog \
    --TopicId "topic-example-audit" \
    --From $(date -v-1H +%s)000 \
    --To $(date +%s)000 \
    --Query "* | SELECT \"sourceIPs\"[0] AS source_ip, COUNT(*) AS count GROUP BY source_ip ORDER BY count DESC LIMIT 10" \
    --region ap-guangzhou \
    --output json
```

### 4. 全局检索

自由组合条件精确搜索审计事件。以下为常用检索场景。

#### 4.1 检索指定用户的所有操作

```bash
tccli cls SearchLog \
    --TopicId "topic-example-audit" \
    --From $(date -v-1H +%s)000 \
    --To $(date +%s)000 \
    --Query "\"user.username\":\"kubernetes-admin\"" \
    --Limit 10 \
    --region ap-guangzhou \
    --output json
```

示例输出：

```json
{
    "Results": [
        {
            "Time": 1705311000000,
            "LogJson": "{\"auditID\":\"abc-001\",\"user\":{\"username\":\"kubernetes-admin\",\"groups\":[\"system:masters\"]},\"verb\":\"create\",\"objectRef\":{\"resource\":\"pods\",\"namespace\":\"default\",\"name\":\"nginx\"},\"responseStatus\":{\"code\":201},\"sourceIPs\":[\"10.0.1.100\"],\"stage\":\"ResponseComplete\",\"requestReceivedTimestamp\":\"2024-01-15T10:30:00Z\"}"
        }
    ],
    "TotalCount": 1
}
```

#### 4.2 检索指定命名空间的操作

```bash
tccli cls SearchLog \
    --TopicId "topic-example-audit" \
    --From $(date -v-1H +%s)000 \
    --To $(date +%s)000 \
    --Query "\"objectRef.namespace\":\"production\"" \
    --Limit 10 \
    --region ap-guangzhou \
    --output json
```

#### 4.3 检索指定资源类型的删除操作

```bash
tccli cls SearchLog \
    --TopicId "topic-example-audit" \
    --From $(date -v-24H +%s)000 \
    --To $(date +%s)000 \
    --Query "verb:delete AND \"objectRef.resource\":pods" \
    --Limit 10 \
    --region ap-guangzhou \
    --output json
```

#### 4.4 检索失败（HTTP 4xx/5xx）请求

```bash
tccli cls SearchLog \
    --TopicId "topic-example-audit" \
    --From $(date -v-1H +%s)000 \
    --To $(date +%s)000 \
    --Query "\"responseStatus.code\":403 OR \"responseStatus.code\":404 OR \"responseStatus.code\":409 OR \"responseStatus.code\":500" \
    --Limit 10 \
    --region ap-guangzhou \
    --output json
```

#### 4.5 检索 API Server panic 事件

```bash
tccli cls SearchLog \
    --TopicId "topic-example-audit" \
    --From $(date -v-24H +%s)000 \
    --To $(date +%s)000 \
    --Query "stage:Panic" \
    --Limit 10 \
    --region ap-guangzhou \
    --output json
```

#### 4.6 检索指定资源名的完整操作链

```bash
tccli cls SearchLog \
    --TopicId "topic-example-audit" \
    --From $(date -v-1H +%s)000 \
    --To $(date +%s)000 \
    --Query "\"objectRef.name\":\"nginx-deployment-7b9f8c6d5-x2k9m\"" \
    --Limit 20 \
    --region ap-guangzhou \
    --output json
```

### 5. 配置审计告警

基于审计仪表盘的查询结果，通过 `cls CreateAlarm` 创建告警策略，在异常操作发生时自动通知。

#### 5.1 创建告警策略：非法删除操作

监控非管理员用户的 Pod 删除操作（verb:delete, resource:pods），每 5 分钟检查一次，触发条件为出现 1 条及以上。

```bash
tccli cls CreateAlarm \
    --Name "audit-pod-delete-by-non-admin" \
    --AlarmTargets '[{"TopicId":"topic-example-audit","Query":"verb:delete AND \"objectRef.resource\":pods NOT (\"user.username\":kubernetes-admin OR \"user.username\":system:serviceaccount:* )","Number":1,"StartTimeOffset":-300,"EndTimeOffset":0,"LogsetId":"logset-example-audit"}]' \
    --MonitorTime '{"Type":"Period","Time":5}' \
    --TriggerCount 1 \
    --AlarmPeriod 15 \
    --AlarmNoticeIds '["notice-example-001"]' \
    --Status true \
    --region ap-guangzhou \
    --output json
```

示例输出：

```json
{
    "AlarmId": "alarm-example-001",
    "RequestId": "abc123-def456-ghi789"
}
```

#### 5.2 创建告警策略：403 权限拒绝

```bash
tccli cls CreateAlarm \
    --Name "audit-403-forbidden-requests" \
    --AlarmTargets '[{"TopicId":"topic-example-audit","Query":"\"responseStatus.code\":403","Number":5,"StartTimeOffset":-300,"EndTimeOffset":0,"LogsetId":"logset-example-audit"}]' \
    --MonitorTime '{"Type":"Period","Time":5}' \
    --TriggerCount 5 \
    --AlarmPeriod 15 \
    --AlarmNoticeIds '["notice-example-001"]' \
    --Status true \
    --region ap-guangzhou \
    --output json
```

#### 5.3 查看告警策略

```bash
tccli cls DescribeAlarms \
    --region ap-guangzhou \
    --output json
```

示例输出：

```json
{
    "Alarms": [
        {
            "AlarmId": "alarm-example-001",
            "Name": "audit-pod-delete-by-non-admin",
            "Status": true,
            "AlarmTargets": [
                {
                    "TopicId": "topic-example-audit",
                    "Query": "verb:delete AND \"objectRef.resource\":pods NOT (...)"
                }
            ],
            "MonitorTime": {"Type":"Period","Time":5},
            "TriggerCount": 1,
            "AlarmPeriod": 15,
            "CreateTime": "2024-01-15T11:00:00Z"
        }
    ],
    "TotalCount": 1
}
```

### 6. CLS SearchLog 常用语法速查

| 语法 | 说明 | 示例 |
|------|------|------|
| `*` | 返回所有日志 | `*` |
| `key:value` | 字段全文匹配 | `verb:delete` |
| `"key":"value"` | 精确匹配（含特殊字符） | `"responseStatus.code":"403"` |
| `AND` / `OR` / `NOT` | 布尔组合 | `verb:delete AND resource:pods` |
| `"nested.key"` | 嵌套字段（JSON key 带引号） | `"user.username"` |
| `\| SELECT ... GROUP BY` | SQL 聚合分析 | `\| SELECT verb, COUNT(*) GROUP BY verb` |
| `\| SELECT ... ORDER BY ... LIMIT N` | 排序限制 | `ORDER BY count DESC LIMIT 10` |

## 验证

### 验证审计检索功能正常

```bash
# 确认审计事件总数 > 0
tccli cls SearchLog \
    --TopicId "topic-example-audit" \
    --From $(date -v-1H +%s)000 \
    --To $(date +%s)000 \
    --Query "* | SELECT COUNT(*) AS cnt" \
    --region ap-guangzhou \
    --output json
```

### 验证告警策略已创建

```bash
tccli cls DescribeAlarms --region ap-guangzhou --output json | grep -A2 "audit-
```

## 清理

### 删除告警策略

```bash
tccli cls DeleteAlarm \
    --AlarmId "alarm-example-001" \
    --region ap-guangzhou \
    --output json
```

### 关闭审计

参见 [开启集群审计](../开启集群审计/tccli%20操作.md)（page_id `58242`）的 Cleanup 章节。

## 排障

| 现象 | 原因 | 处理 |
|------|------|------|
| `cls SearchLog` 无结果 | 时间范围无审计事件，或 `TopicId` 错误 | 扩大时间范围（如 `-24H`），确认 `TopicId` 与控制台审计页面一致 |
| `cls SearchLog` 报 `TopicNotExist` | 日志主题 ID 不存在或已删除 | `cls DescribeTopics` 确认当前可用日志主题 |
| 嵌套字段查询无结果 | JSON 字段路径中带点号需加引号 | `"user.username"` 而非 `user.username`（不带引号 CLS 会将 `.` 解释为分隔符） |
| SQL 聚合查询报语法错误 | CLS 检索语法与标准 SQL 有差异 | 确认使用 CLS 支持的 `SELECT ... GROUP BY` 语法；聚合查询需以 `*` 或条件开头再接 `\|` |
| `GROUP BY` 结果与预期不一致 | 审计事件 JSON 字段为嵌套结构 | 确认字段路径正确（如 `"objectRef.resource"`）；先用 `* | LIMIT 1` 确认字段存在 |
| `cls CreateAlarm` 报 `AlarmNoticeIds` 错误 | 告警通知组不存在 | 先在 [CLS 控制台](https://console.cloud.tencent.com/cls/alarm) 创建通知组 |

## 下一步

- [开启集群审计](../开启集群审计/tccli%20操作.md)（page_id `58242`） — 管理审计开关与配置
- [集群事件](../../事件管理/集群事件/tccli%20操作.md) — 查看集群事件（与审计日志互补）
- [监控和告警](../../监控和告警/查看监控/tccli%20操作.md) — 为集群配置更多监控与告警

## 控制台替代

[容器服务控制台 → 集群 → 运维中心 → 审计管理 → 查看审计](https://console.cloud.tencent.com/tke2/cluster) — 提供四个仪表盘（审计总览 / K8S 对象操作概览 / 聚合检索 / 全局检索），通过图形化界面查看和分析审计日志。
