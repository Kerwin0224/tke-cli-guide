# 审计日志

> 对照官方：[审计日志](https://cloud.tencent.com/document/product/457/58242) · page_id `58242`

## 概述

TKE Serverless（EKS）集群支持开启 **Kubernetes 审计日志（Audit Log）** 功能，将对 APIServer 的所有请求记录到 CLS 日志服务。审计日志记录了谁（User/ServiceAccount）、在什么时间、对什么资源执行了什么操作以及结果，是集群安全合规、操作审计和异常排查的重要依据。

审计日志包含以下字段：`user`（操作者）、`verb`（操作类型: create/update/delete/get/list/watch/patch）、`resource`（目标资源）、`namespace`、`responseStatus.code`（HTTP 状态码）、`requestObject`（请求体）、`sourceIPs`（来源 IP）。

采集链路：**APIServer 审计事件 -> 集群审计组件（平台管理）-> CLS 日志主题 -> 检索分析**。

TKE Serverless 集群审计功能由平台自动管理和配置，开启后无需维护审计组件。

## 前置条件

- [环境准备](../../../../../环境准备.md)
- 至少一个运行中的 TKE Serverless 集群
- 已开通 CLS（日志服务），且 CAM 角色具有 CLS 写权限
- 集群状态为 Running

### 环境检查

```bash
tccli --version
# expected: tccli version 3.0.x

tccli configure list
# expected: secretId, secretKey, region 均已配置

# 验证 CAM Action — CLS 写权限
tccli cls DescribeLogsets --region <Region> --Limit 1
# expected: (含有 Logsets 数组和 RequestId，无 UnauthorizedOperation)

# 验证 CAM Action — TKE 管理权限
tccli tke DescribeEKSClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
# expected:
# {
#     "Clusters": [
#         {
#             "ClusterId": "CLUSTER_ID",
#             "ClusterStatus": "Running",
#             ...
#         }
#     ],
#     "RequestId": "xxx"
# }

# 查看审计日志开关状态
tccli tke DescribeLogSwitches \
    --region <Region> \
    --ClusterId CLUSTER_ID
# expected: (返回 AuditEnabled 字段，true 或 false)
```

```json
{
  "TotalCount": "<TotalCount>",
  "Clusters": "<Clusters>",
  "ClusterId": "<ClusterId>",
  "ClusterName": "<ClusterName>",
  "VpcId": "<VpcId>",
  "SubnetIds": "<SubnetIds>"
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 开启集群审计 | `tccli tke EnableClusterAudit` | 是（重复开启无副作用） |
| 关闭集群审计 | `tccli tke DisableClusterAudit` | 是（重复关闭无副作用） |
| 查看审计开关状态 | `tccli tke DescribeLogSwitches` | 是 |
| 创建审计日志集（CLS） | `tccli cls CreateLogset` | 否（同名冲突） |
| 创建审计日志主题（CLS） | `tccli cls CreateTopic` | 否（同名冲突） |
| 创建日志索引 | `tccli cls CreateIndex` | 是（覆盖写） |
| 检索审计日志 | `tccli cls SearchLog` | 是 |

## 操作步骤

### 占位符说明

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|----------|
| `REGION` | 地域 | 如 ap-guangzhou | 固定值 |
| `CLUSTER_ID` | EKS 集群 ID | 格式 cls-xxxxxxxx | `DescribeEKSClusters` |
| `AUDIT_LOGSET_ID` | 审计日志集 ID | UUID 格式 | `CreateLogset` 或 `DescribeLogSwitches` |
| `AUDIT_TOPIC_ID` | 审计日志主题 ID | UUID 格式 | `CreateTopic` 或 `DescribeLogSwitches` |
| `NAMESPACE` | Kubernetes 命名空间 | 不超过 63 字符 | 自定义 |

### 步骤1: 创建 CLS 日志集和日志主题（如尚未创建）

```bash
# 创建审计专用日志集
tccli cls CreateLogset \
    --region <Region> \
    --LogsetName "eks-audit-logs" \
    --Period 180
# expected:
# {
#     "LogsetId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
#     "RequestId": "xxx"
# }
AUDIT_LOGSET_ID="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"

# 创建审计专用日志主题
tccli cls CreateTopic \
    --region <Region> \
    --LogsetId $AUDIT_LOGSET_ID \
    --TopicName "eks-audit-topic" \
    --Period 180 \
    --StorageType hot
# expected:
# {
#     "TopicId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
#     "RequestId": "xxx"
# }
AUDIT_TOPIC_ID="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"

# 创建全文索引
cat > audit-index.json <<'EOF'
{
    "TopicId": "AUDIT_TOPIC_ID",
    "Rule": {
        "FullText": {
            "CaseSensitive": false,
            "Tokenizer": " \t\n\r,;:.|[]{}()!@#$%^&*\"'<>/"
        }
    },
    "Status": true
}
EOF
sed -i '' "s/AUDIT_TOPIC_ID/$AUDIT_TOPIC_ID/g" audit-index.json

tccli cls CreateIndex \
    --region <Region> \
    --cli-input-json file://audit-index.json
# expected:
# {
#     "RequestId": "xxx"
# }
```

### 步骤2: 开启集群审计日志

```bash
cat > enable-audit.json <<'EOF'
{
    "ClusterId": "CLUSTER_ID",
    "LogsetId": "AUDIT_LOGSET_ID",
    "TopicId": "AUDIT_TOPIC_ID"
}
EOF
sed -i '' "s/CLUSTER_ID/$CLUSTER_ID/g" enable-audit.json
sed -i '' "s/AUDIT_LOGSET_ID/$AUDIT_LOGSET_ID/g" enable-audit.json
sed -i '' "s/AUDIT_TOPIC_ID/$AUDIT_TOPIC_ID/g" enable-audit.json

tccli tke EnableClusterAudit \
    --region <Region> \
    --cli-input-json file://enable-audit.json
# expected:
# {
#     "RequestId": "xxx"
# }
```

### 步骤3: 验证审计开关状态

```bash
tccli tke DescribeLogSwitches \
    --region <Region> \
    --ClusterId CLUSTER_ID
# expected:
# {
#     "AuditEnabled": true,
#     "LogsetId": "AUDIT_LOGSET_ID",
#     "TopicId": "AUDIT_TOPIC_ID",
#     "RequestId": "xxx"
# }
```

```json
{
  "SwitchSet": "<SwitchSet>",
  "Audit": "<Audit>",
  "Enable": "<Enable>",
  "ErrorMsg": "<ErrorMsg>",
  "LogsetId": "<LogsetId>",
  "Status": "<Status>"
}
```

### 步骤4: 检索审计日志

审计开启后，所有对 APIServer 的请求都会记录。以下示例检索指定 namespace 下近 1 小时的审计日志。

```bash
# 检索特定 namespace 的审计日志
tccli cls SearchLog \
    --region <Region> \
    --TopicId $AUDIT_TOPIC_ID \
    --From $(python3 -c "import time; print(int((time.time()-3600)*1000))") \
    --To $(python3 -c "import time; print(int(time.time()*1000))") \
    --Query "objectRef.namespace:NAMESPACE" \
    --Limit 20
# expected: Results 数组含审计条目，每项的 Content 包含 objectRef、user、verb 等字段

# 检索删除操作的审计日志（安全审计常用）
tccli cls SearchLog \
    --region <Region> \
    --TopicId $AUDIT_TOPIC_ID \
    --From $(python3 -c "import time; print(int((time.time()-3600)*1000))") \
    --To $(python3 -c "import time; print(int(time.time()*1000))") \
    --Query "verb:delete OR verb:deletecollection" \
    --Limit 20
# expected: 返回所有 delete 操作记录

# 检索失败操作的审计日志
tccli cls SearchLog \
    --region <Region> \
    --TopicId $AUDIT_TOPIC_ID \
    --From $(python3 -c "import time; print(int((time.time()-3600)*1000))") \
    --To $(python3 -c "import time; print(int(time.time()*1000))") \
    --Query "responseStatus.code:>=400" \
    --Limit 20
# expected: 返回所有响应码 >=400 的请求
```

### 步骤5: 通过 kubectl 查看实时审计规则（需 VPN/IOA）

```bash
# 查看 APIServer 审计策略（需集群管理员权限）
kubectl get --raw /apis/audit.k8s.io/v1 | jq .
# expected: 返回审计 API 组信息

# 查看审计日志来源——创建测试操作生成审计事件
kubectl create configmap audit-test --from-literal=key=value -n NAMESPACE
# expected: configmap/audit-test created

kubectl delete configmap audit-test -n NAMESPACE
# expected: configmap "audit-test" deleted

# 随后在 CLS 中检索 "audit-test" 验证审计日志已记录
tccli cls SearchLog \
    --region <Region> \
    --TopicId $AUDIT_TOPIC_ID \
    --From $(python3 -c "import time; print(int((time.time()-300)*1000))") \
    --To $(python3 -c "import time; print(int(time.time()*1000))") \
    --Query "audit-test"
# expected: Results 含 audit-test ConfigMap 的 create 和 delete 操作记录
```

```text
NAME  STATUS  AGE
...
```

## 验证

### 控制面（tccli）

```bash
# 1. 验证审计功能已开启
tccli tke DescribeLogSwitches \
    --region <Region> \
    --ClusterId CLUSTER_ID | jq '.AuditEnabled'
# expected: true

# 2. 验证审计日志已投递到 CLS
tccli cls SearchLog \
    --region <Region> \
    --TopicId $AUDIT_TOPIC_ID \
    --From $(python3 -c "import time; print(int((time.time()-600)*1000))") \
    --To $(python3 -c "import time; print(int(time.time()*1000))") \
    --Query "*" \
    --Limit 5 | jq '.Results | length'
# expected: >0（有审计日志条目）

# 3. 验证审计日志包含核心字段
tccli cls SearchLog \
    --region <Region> \
    --TopicId $AUDIT_TOPIC_ID \
    --From $(python3 -c "import time; print(int((time.time()-600)*1000))") \
    --To $(python3 -c "import time; print(int(time.time()*1000))") \
    --Query "*" \
    --Limit 1 | jq '.Results[0] | keys'
# expected: Content 包含 user.username, verb, objectRef.resource, responseStatus.code
```

```json
{
  "SwitchSet": "<SwitchSet>",
  "Audit": "<Audit>",
  "Enable": "<Enable>",
  "ErrorMsg": "<ErrorMsg>",
  "LogsetId": "<LogsetId>",
  "Status": "<Status>"
}
```

### 数据面（需 VPN/IOA）

```bash
# 4. 验证 kubectl 操作被审计记录
kubectl get pods -n NAMESPACE > /dev/null
# 等待 1-2 分钟后检索
tccli cls SearchLog \
    --region <Region> \
    --TopicId $AUDIT_TOPIC_ID \
    --From $(python3 -c "import time; print(int((time.time()-300)*1000))") \
    --To $(python3 -c "import time; print(int(time.time()*1000))") \
    --Query "verb:list AND objectRef.resource:pods"
# expected: Results 含 kubectl get pods 对应的 list 操作记录
```

```text
NAME  STATUS  AGE
...
```

## 清理

> **计费警告**：CLS 按日志写入量、存储量和索引流量计费。审计日志写入量较大（每次 APIServer 请求均记录），持续产生费用。建议根据合规需求设置合理的日志保留天数（如 30-90 天）。

```bash
# 1. 关闭集群审计（停止审计日志写入）
cat > disable-audit.json <<'EOF'
{
    "ClusterId": "CLUSTER_ID"
}
EOF
sed -i '' "s/CLUSTER_ID/$CLUSTER_ID/g" disable-audit.json

tccli tke DisableClusterAudit \
    --region <Region> \
    --cli-input-json file://disable-audit.json
# expected:
# {
#     "RequestId": "xxx"
# }

# 2. 验证审计已关闭
tccli tke DescribeLogSwitches \
    --region <Region> \
    --ClusterId CLUSTER_ID | jq '.AuditEnabled'
# expected: false

# 3. （可选）删除审计日志主题
tccli cls DeleteTopic \
    --region <Region> \
    --TopicId $AUDIT_TOPIC_ID
# expected:
# {
#     "RequestId": "xxx"
# }

# 4. （可选）删除审计日志集
tccli cls DeleteLogset \
    --region <Region> \
    --LogsetId $AUDIT_LOGSET_ID
# expected:
# {
#     "RequestId": "xxx"
# }
```

```json
{
  "SwitchSet": "<SwitchSet>",
  "Audit": "<Audit>",
  "Enable": "<Enable>",
  "ErrorMsg": "<ErrorMsg>",
  "LogsetId": "<LogsetId>",
  "Status": "<Status>"
}
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `EnableClusterAudit` 返回 `FailedOperation.RecordNotSupport` | 检查集群类型和状态 | 集群不是 EKS 类型或集群状态异常 | 确认集群为 TKE Serverless（EKS）类型且状态为 Running |
| `EnableClusterAudit` 返回 `UnauthorizedOperation` | `tccli cls DescribeLogsets --region <Region>` | CAM 缺少 CLS 创建权限 | 为主账号添加 `QcloudCLSFullAccess` 策略 |
| `DescribeLogSwitches` 返回 `ResourceNotFound` | 检查 ClusterId 拼写和格式 | 集群 ID 无效或集群不存在 | 通过 `tccli tke DescribeEKSClusters` 确认正确 ClusterId |
| `SearchLog` 返回 `FailedOperation.QueryError` | CLS 控制台检索验证语法 | CLS 查询语法不正确（字段名区分大小写） | 确认字段名使用正确大小写（如 `objectRef.namespace` 而非 `ObjectRef.Namespace`） |

### 创建成功但状态异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 开启审计后 CLS 无日志 | `tccli cls SearchLog --TopicId AUDIT_TOPIC_ID --Query "*" --From ...` | 审计日志投递延迟（首次可能需要 5-10 分钟） | 等待 10 分钟后重试，执行 kubectl 操作触发审计事件 |
| 审计日志缺失部分 kubectl 操作 | `tccli cls SearchLog --Query "verb:get" --TopicId AUDIT_TOPIC_ID` | APIServer 对高频 get/list 操作可能采样 | 审计日志对于高频 get/list 操作有采样机制，非安全审计需求时可忽略 |
| 审计日志量过大导致 CLS 费用高 | `tccli cls DescribeTopics --TopicId AUDIT_TOPIC_ID` 查看日志量 | 集群请求量大，每次 APIServer 请求均记录 | 缩短日志保留天数（Period 参数），或关闭非业务时段的审计 |
| 检索时字段过滤无效 | CLS 控制台中查看日志 JSON 结构 | 审计日志字段为嵌套 JSON，CLS 检索语法需使用点号导航 | 使用 `objectRef.resource:pods` 而非 `resource:pods` |

## 下一步

- [事件日志](../../事件管理/事件日志/tccli%20操作.md)：集群事件日志的查看和管理
- [TKE 集群审计最佳实践](https://cloud.tencent.com/document/product/457/50519)：审计日志的安全使用指南
- [CLS 检索语法](https://cloud.tencent.com/document/product/614/47044)：CLS 检索分析语法参考

## 控制台替代

控制台路径：**TKE 控制台 > Serverless 集群 > 集群 ID > 运维中心 > 审计**。在控制台中开启集群审计，选择 CLS 日志集和主题，随后在审计页面直接检索审计日志。控制台提供可视化检索界面和预设仪表盘（操作者分析、操作类型分布、错误率趋势），适合日常安全审计和合规检查。tccli 方式适合自动化审计日志采集和与第三方 SIEM 系统集成。
