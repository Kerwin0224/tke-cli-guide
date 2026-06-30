# 日志采集相关（tccli）
> 对照官方：[日志采集相关](https://cloud.tencent.com/document/product/457/60223) · page_id `60223`

## 概述

TKE Serverless（EKS）集群中深度学习任务的日志采集依赖 **CLS（日志服务）**。Pod stdout/stderr 日志通过 LogConfig CRD 投递至 CLS 日志主题，支持实时检索与分析。常见问题包括：Pod 销毁后日志丢失、CLS 投递延迟、日志格式解析失败、采集组件异常等。

本文档覆盖 EKS 日志采集的全链路诊断与排障流程。核心诊断链路：**CLS 日志集/主题状态 → EKS LogConfig CRD → 采集组件 Pod → CLS 检索验证**。

环境约定：`REGION=ap-guangzhou`，Kubernetes 版本 `1.30.0`，集群类型 `MANAGED_CLUSTER`（EKS）。

> **kubectl 数据面不可达**：CAM拒绝公网端点(strategyId:240463971)，内网需VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

## 前置条件

### 环境检查

```bash
tccli --version
# expected: 3.0.x

tccli configure list
# expected: secretId/secretKey/region 均已配置
{
    "secretId": "AKID******************************",
    "secretKey": "********************************",
    "region": "ap-guangzhou"
}
```

CAM 权限检查（执行以下 Describe* 命令，确认无 `UnauthorizedOperation` 错误）：

| 服务 | CAM Action | 用途 |
|------|-----------|------|
| tke | `DescribeEKSClusters` | 查询 EKS 集群状态 |
| tke | `DescribeEksLogSwitches` | 查询日志采集开关状态 |
| cls | `DescribeLogsets` | 查询日志集 |
| cls | `DescribeTopics` | 查询日志主题 |
| cls | `DescribeIndex` | 查询日志主题索引配置 |
| cls | `SearchLog` | 检索日志内容 |

### 资源检查

```bash
# 确认 EKS 集群存在
tccli tke DescribeEKSClusters \
    --region <Region> \
    --ClusterIds '["EKS_CLUSTER_ID"]'
# expected:
{
    "Response": {
        "Clusters": [
            {
                "ClusterId": "EKS_CLUSTER_ID",
                "ClusterName": "eks-deep-learning",
                "Status": "Running"
            }
        ],
        "TotalCount": 1,
        "RequestId": "..."
    }
}
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

```bash
# 确认 CLS 日志集存在
tccli cls DescribeLogsets \
    --region <Region>
# expected:
{
    "Response": {
        "Logsets": [
            {
                "LogsetId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
                "LogsetName": "eks-logset",
                "CreateTime": "2024-01-01 00:00:00"
            }
        ],
        "TotalCount": 1,
        "RequestId": "..."
    }
}
```

## 控制台与 CLI 参数映射

| 控制台字段 | CLI 参数/路径 | 类型 | 必填 | 说明 | 幂等性 |
|-----------|-------------|------|------|------|--------|
| 地域 | `--region` / `Region` | String | 是 | 地域标识 | 无影响 |
| 集群 ID | `--ClusterId` | String | 是 | EKS 集群 ID | 查询操作天然幂等 |
| 日志集 ID | `--LogsetId` / `LogsetId` | String | 是 | CLS 日志集 ID | 查询操作天然幂等 |
| 日志主题 ID | `--TopicId` / `TopicId` | String | 是 | CLS 日志主题 ID | 查询操作天然幂等 |
| 日志主题名称 | `--TopicName` / `TopicName` | String | 是 | CLS 主题名称 | 创建时非幂等（同名会报错） |
| 索引规则 | `--Rule` / `Rule` | Object | 否 | 日志索引配置 | 覆盖写幂等 |
| 检索语句 | `--Query` | String | 是 | CLS 检索语法 | 查询天然幂等 |
| 时间范围 | `--From` / `--To` | Integer | 是 | Unix 毫秒时间戳 | 查询天然幂等 |
| Pod 名称 | kubectl `POD_NAME` | String | 是 | 目标 Pod | 查询天然幂等 |

## 操作步骤

### 选择依据

日志采集问题按症状分为三类诊断路径：

1. **日志采集未启用 / 采集组件异常**：跳至「场景一：采集开关与组件诊断」。
2. **日志未投递到 CLS / 投递延迟**：跳至「场景二：CLS 主题与 LogConfig 诊断」。
3. **日志内容检索不到 / 解析失败**：跳至「场景三：CLS 索引与检索诊断」。

### 最小创建（诊断基础）

#### 场景一：采集开关与组件诊断

**步骤 1** — 确认日志采集开关已启用。

```bash
tccli tke DescribeEksLogSwitches \
    --region <Region> \
    --ClusterId EKS_CLUSTER_ID
# expected:
{
    "Response": {
        "SwitchSet": [
            {
                "LogType": "container_stdout",
                "Switch": true,
                "LogsetId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
                "TopicId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
            }
        ],
        "RequestId": "..."
    }
}
```

```json
{"RequestId": "..."}
```

**步骤 2** — 确认 LogConfig CRD 配置正确。

```bash
kubectl get logconfigs -n NAMESPACE
# expected:
# NAME         LOG TYPE       LOGSET ID                           TOPIC ID                            STATUS
# dl-logger    container_stdout  xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx  xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx  Running
```

```text
NAME  STATUS  AGE
...
```

```bash
kubectl describe logconfig dl-logger -n NAMESPACE
# expected: Status 字段显示 Running，Conditions 无错误
# Status:
#   Conditions:
#     Last Transition Time:  2024-01-01T00:00:00Z
#     Status:                True
#     Type:                  Running
```

```text
NAME  STATUS  AGE
...
```

#### 场景二：CLS 主题与投递诊断

**步骤 1** — 查询 CLS 日志主题详情。

```bash
tccli cls DescribeTopics \
    --region <Region> \
    --Filters '[{"Key":"logsetId","Values":["xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"]}]'
# expected:
{
    "Response": {
        "Topics": [
            {
                "TopicId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
                "TopicName": "eks-container-log",
                "Status": "1",
                "LogsetId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
                "Period": 30
            }
        ],
        "TotalCount": 1,
        "RequestId": "..."
    }
}
```

**步骤 2** — 创建 Pod 写入测试日志并检索。

```bash
# 创建测试 Pod 产生 stdout 日志
kubectl run log-test \
    --image=busybox:1.28 \
    --restart=Never \
    -n NAMESPACE \
    -- /bin/sh -c \
    'for i in $(seq 1 10); do echo "TEST_LOG_$(date +%s)_LINE_$i"; sleep 1; done'
# expected: pod/log-test created
```

```bash
# 等待 2-3 分钟让日志投递到 CLS，然后检索
tccli cls SearchLog \
    --region <Region> \
    --TopicId xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx \
    --From $(python3 -c "import time; print(int((time.time()-600)*1000))") \
    --To $(python3 -c "import time; print(int(time.time()*1000))") \
    --Query "TEST_LOG"
# expected:
{
    "Response": {
        "Results": [
            {
                "Time": 1704067200000,
                "Source": "10.0.1.100",
                "Content": "...TEST_LOG_1704067200_LINE_1..."
            }
        ],
        "RequestId": "..."
    }
}
```

### 增强配置

#### 场景三：CLS 索引与检索诊断

**步骤 1** — 确认日志主题已配置索引。

```bash
tccli cls DescribeIndex \
    --region <Region> \
    --TopicId xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
# expected:
{
    "Response": {
        "TopicId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
        "Rule": {
            "FullText": {
                "CaseSensitive": false,
                "Tokenizer": " \t\n\r,;"
            }
        },
        "Status": true,
        "RequestId": "..."
    }
}
```

**步骤 2** — 若日志内容解析失败（如深度学习框架特有格式），配置键值索引。

| 参数 | 值 | 说明 |
|------|----|------|
| `TopicId` | 日志主题 ID | CLS 目标主题 |
| `REGION` | 地域 | `ap-guangzhou` |
| `Rule.FullText.Tokenizer` | 分隔符 | 全文索引分词符 |
| `Rule.KeyValue.KeyValues` | 键值列表 | 字段级索引配置 |

```bash
cat > create-index.json <<'EOF'
{
    "TopicId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
    "Rule": {
        "FullText": {
            "CaseSensitive": false,
            "Tokenizer": " \t\n\r,;:|[]{}"
        },
        "KeyValue": {
            "CaseSensitive": false,
            "KeyValues": [
                {"Key": "level", "Value": {"Type": "text", "Tokenizer": " \t\n\r"}},
                {"Key": "epoch", "Value": {"Type": "long"}},
                {"Key": "loss", "Value": {"Type": "double"}},
                {"Key": "step", "Value": {"Type": "long"}},
                {"Key": "message", "Value": {"Type": "text", "Tokenizer": " \t\n\r,;"}}
            ]
        }
    }
}
EOF
tccli cls CreateIndex \
    --region <Region> \
    --cli-input-json file://create-index.json
# expected:
{
    "Response": {
        "RequestId": "..."
    }
}
```

```bash
# 按结构化字段检索（level=ERROR 的日志）
tccli cls SearchLog \
    --region <Region> \
    --TopicId xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx \
    --From $(python3 -c "import time; print(int((time.time()-3600)*1000))") \
    --To $(python3 -c "import time; print(int(time.time()*1000))") \
    --Query "level:ERROR"
# expected: 返回 level 字段值为 ERROR 的日志条目
```

## 验证

### 控制面（tccli）

```bash
# 验证日志采集开关
tccli tke DescribeEksLogSwitches \
    --region <Region> \
    --ClusterId EKS_CLUSTER_ID \
    | jq '.Response.SwitchSet[] | select(.LogType=="container_stdout") | .Switch'
# expected: true

# 验证 CLS 日志主题状态
tccli cls DescribeTopics \
    --region <Region> \
    --TopicId xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx \
    | jq '.Response.Topics[0].Status'
# expected: "1"（正常）

# 验证索引已启用
tccli cls DescribeIndex \
    --region <Region> \
    --TopicId xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx \
    | jq '.Response.Status'
# expected: true
```

```json
{"RequestId": "..."}
```

### 数据面（需 VPN/IOA）

```bash
# 验证 LogConfig CRD 状态
kubectl get logconfigs -n NAMESPACE
# expected: 目标 LogConfig STATUS 字段为 Running

# 验证采集组件 Pod 正常运行
kubectl get pods -n kube-system -l app=log-collector
# expected: 所有 Pod STATUS=Running, READY=1/1

# 验证 Pod 日志可正常输出
kubectl logs -n NAMESPACE deep-learning-trainer --tail=5
# expected: 最近 5 行 stdout 日志
```

```text
NAME  STATUS  AGE
...
```

## 清理

> **账单警告**：CLS 按日志写入量（写流量）、存储量（存储空间）和索引流量计费。闲置的日志主题仍产生存储费用（按保留天数计）。请及时清理不再需要的日志主题。

### 控制面（tccli）

**清理 CLS 日志主题（按顺序执行）：**

```bash
# 1. 先关闭 EKS 日志采集开关
cat > disable-log.json <<'EOF'
{
    "ClusterId": "EKS_CLUSTER_ID",
    "SwitchList": [
        {"LogType": "container_stdout", "Switch": false}
    ]
}
EOF
tccli tke EnableClusterAudit \
    --region <Region> \
    --cli-input-json file://disable-log.json
# expected:
{
    "Response": {
        "RequestId": "..."
    }
}

# 2. 确认采集开关已关闭
tccli tke DescribeEksLogSwitches \
    --region <Region> \
    --ClusterId EKS_CLUSTER_ID \
    | jq '.Response.SwitchSet[] | select(.LogType=="container_stdout") | .Switch'
# expected: false

# 3. 删除日志主题
tccli cls DeleteTopic \
    --region <Region> \
    --TopicId xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
# expected:
{
    "Response": {
        "RequestId": "..."
    }
}

# 4. 验证已删除
tccli cls DescribeTopics \
    --region <Region> \
    --TopicId xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
# expected: ResourceNotFound 或 TotalCount=0
```

```json
{"RequestId": "..."}
```

> **副作用警告**：删除日志主题后，该主题下的所有历史日志将被永久删除且不可恢复。建议在清理前导出重要日志。关闭采集开关后，Pod 新产生的 stdout/stderr 日志不会被采集。

### 数据面（需 VPN/IOA）

```bash
# 删除 LogConfig CRD（停止特定 namespace 的日志采集）
kubectl delete logconfig dl-logger -n NAMESPACE
# expected: logconfig.log.cloud.tencent.com "dl-logger" deleted

# 删除测试 Pod
kubectl delete pod log-test -n NAMESPACE --ignore-not-found
# expected: pod "log-test" deleted
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `UnauthorizedOperation` | `tccli cls DescribeLogsets --region <Region>` | CAM 缺少 `cls:DescribeLogsets` 权限 | 为主账号/UIN 添加 CLS 只读策略：`tccli cam AttachUserPolicy --Uin <uin> --PolicyId <id>` |
| `ResourceNotFound.TopicNotExist` | `tccli cls DescribeTopics --region <Region> --TopicId xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx` | 日志主题 ID 不存在或已删除 | 检查 TopicId 是否正确，或通过控制台重建日志主题 |
| `InvalidParameter.LogsetConflict` | 创建日志集时 `LogsetName` 与已有日志集同名 | CLS 日志集名称在同一地域内需唯一 | 使用不同名称或先 `DeleteLogset` 删除旧的再创建 |
| `LimitExceeded.Logset` | `tccli cls DescribeLogsets --region <Region>` | 日志集数量达到配额上限（默认 20） | 清理不再使用的日志集，或提交工单提升配额 |
| `FailedOperation.TopicIsolated` | `tccli cls SearchLog --region <Region>` | 日志主题因欠费被隔离 | 充值账户余额，等待主题自动恢复（约 5 分钟） |

### 创建成功但状态异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 日志采集已开启但 CLS 无日志 | `tccli cls SearchLog --TopicId xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx --Query "*"` | LogConfig 未创建或未关联正确的日志主题 | `kubectl apply -f logconfig.yaml` 创建并关联正确的 LogsetId 和 TopicId |
| CLS 检索无结果 | `tccli cls DescribeIndex --TopicId xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx` | 日志主题未配置索引，无法检索 | `tccli cls CreateIndex --cli-input-json file://create-index.json` 创建全文索引 |
| 检索速度极慢（> 10s） | `tccli cls SearchLog --Query "keyword"` 并观察响应耗时 | 索引未覆盖查询字段，触发全文扫描 | 为高频查询字段配置键值索引，缩小时间范围 |
| 日志延迟超过 5 分钟 | 写入测试日志后多次 `SearchLog` 对比写入时间与检索时间 | 采集组件资源不足或 CLS 写入限流 | `kubectl describe pod -n kube-system -l app=log-collector` 检查资源使用，必要时扩容采集组件 |
| 日志内容乱码/截断 | `tccli cls SearchLog --Query "*" --Limit 10` 检查返回内容 | 容器日志输出包含非 UTF-8 字符或单行超过 16KB | 在应用层限制单行日志长度，确保输出为 UTF-8 编码 |
| 深度学习训练日志格式分裂 | `tccli cls SearchLog` 查看多行日志序列 | 训练框架进度条（如 tqdm）使用 `\r` 覆盖输出，CLS 按行采集导致碎片 | 配置 LogConfig 的多行合并规则，或使用 `--logfile` 输出到持久卷 |
| Pod 销毁后日志丢失 | Pod 删除后 `tccli cls SearchLog` 查不到 Pod 生命周期日志 | stdout 日志随容器一起销毁，采集组件未及时投递 | 确保 LogConfig 在 Pod 创建前已部署，且 Pod 有足够的 terminationGracePeriodSeconds |
| LogConfig Status 非 Running | `kubectl describe logconfig LOG_CONFIG_NAME -n NAMESPACE` | 日志集或主题 ID 配置错误、采集组件无权限访问 CLS | 检查 LogsetId 和 TopicId 是否正确，确认集群绑定的服务角色有 CLS 写入权限 |

## 下一步

- [TKE 日志采集最佳实践](../../../../日志/TKE%20日志采集最佳实践/tccli%20操作.md)：日志采集的完整配置与最佳实践
- [公网访问相关](../公网访问相关/tccli%20操作.md)：EKS 公网访问排障（CLS 投递依赖公网/内网网络）
- [CLS 产品文档](https://cloud.tencent.com/document/product/614)：日志服务的完整产品说明
- [CLS 检索语法](https://cloud.tencent.com/document/product/614/47044)：CLS 检索分析语法参考
- [LogConfig CRD 说明](https://cloud.tencent.com/document/product/457/78446)：TKE 日志采集 CRD 字段说明

## 控制台替代

| 操作 | 控制台路径 | 等效 CLI |
|------|-----------|----------|
| 查看日志集 | CLS 控制台 > 日志集管理 | `tccli cls DescribeLogsets` |
| 查看日志主题 | CLS 控制台 > 日志主题 | `tccli cls DescribeTopics` |
| 检索日志 | CLS 控制台 > 检索分析 | `tccli cls SearchLog` |
| 配置索引 | CLS 控制台 > 日志主题 > 索引配置 | `tccli cls CreateIndex` |
| 创建日志主题 | CLS 控制台 > 日志主题 > 新建 | `tccli cls CreateTopic` |
| 删除日志主题 | CLS 控制台 > 日志主题 > 删除 | `tccli cls DeleteTopic` |
| 开启/关闭日志采集 | TKE 控制台 > Serverless 集群 > 日志 | `tccli tke EnableClusterAudit` |
| 查看采集状态 | TKE 控制台 > Serverless 集群 > 日志采集 | `tccli tke DescribeEksLogSwitches` |
| 管理 LogConfig | kubectl（需 VPN/IOA） | `kubectl get logconfigs -n NAMESPACE` |
