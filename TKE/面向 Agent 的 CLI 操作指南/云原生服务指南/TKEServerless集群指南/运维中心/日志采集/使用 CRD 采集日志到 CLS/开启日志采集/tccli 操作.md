# 开启日志采集

> 对照官方：[开启日志采集](https://cloud.tencent.com/document/product/457/56751) · page_id `56751`

## 概述

在 TKE Serverless（EKS）集群中开启日志采集，将容器 stdout/stderr 日志投递至腾讯云 **CLS（日志服务）**。采集流程分为 **CLS 侧**（创建日志集和日志主题）和 **TKE 侧**（开启采集开关并关联 CLS 资源）两步。本文档通过 tccli 完成端到端配置，支持 API 驱动的自动化运维。

采集链路：**Pod stdout/stderr -> log-agent（平台管理）-> CLS 日志主题 -> 检索分析**。

TKE Serverless 集群由平台自动部署和管理 log-agent，无需手动安装采集组件。

## 前置条件

- [环境准备](../../../../../../环境准备.md)
- 至少一个运行中的 TKE Serverless 集群
- 已开通 CLS（日志服务），且 CAM 角色具有 `cls:CreateLogset`、`cls:CreateTopic`、`cls:CreateIndex` 权限

### 环境检查

```bash
tccli --version
# expected: tccli version 3.0.x

tccli configure list
# expected: secretId, secretKey, region 均已配置

# 验证 CAM Action — CLS 写权限
tccli cls DescribeLogsets --region <Region> --Limit 1
# expected: (含有 Logsets 数组和 RequestId，无 UnauthorizedOperation)

# 验证 CAM Action — TKE EKS 读权限
tccli tke DescribeEKSClusters --region <Region> --Limit 1
# expected: (含有 Clusters 数组和 RequestId，无错误)

# 确认 EKS 集群 Running
tccli tke DescribeEKSClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterStatus 为 Running
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
| 创建日志集 | `tccli cls CreateLogset` | 否（同名不可重复创建） |
| 创建日志主题 | `tccli cls CreateTopic` | 否（同名同 LogsetId 下不可重复） |
| 创建全文索引 | `tccli cls CreateIndex` | 是（覆盖写） |
| 开启 TKE 日志采集 | `tccli tke EnableEksLog` | 是（重复操作无副作用） |
| 查看日志采集状态 | `tccli tke DescribeEksLogSwitches` | 是 |
| 查看日志集列表 | `tccli cls DescribeLogsets` | 是 |
| 查看日志主题列表 | `tccli cls DescribeTopics` | 是 |

## 操作步骤

### 占位符说明

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|----------|
| `REGION` | 地域 | 如 ap-guangzhou | 固定值，需与集群一致 |
| `CLUSTER_ID` | EKS 集群 ID | 格式 cls-xxxxxxxx | `DescribeEKSClusters` |
| `LOGSET_NAME` | CLS 日志集名称 | 1-255 字符，同地域唯一 | 自定义 |
| `TOPIC_NAME` | CLS 日志主题名称 | 1-255 字符 | 自定义 |
| `LOGSET_ID` | CLS 日志集 ID | UUID 格式 | `CreateLogset` 返回值 |
| `TOPIC_ID` | CLS 日志主题 ID | UUID 格式 | `CreateTopic` 返回值 |
| `PERIOD` | 日志保存天数 | 1-3600，或 3640（永久） | 自定义，建议 30 |

### 步骤1: 创建 CLS 日志集

日志集（Logset）是日志主题的容器，每个地域最多创建 20 个日志集。

```bash
tccli cls CreateLogset \
    --region <Region> \
    --LogsetName LOGSET_NAME \
    --Period 30
# expected:
# {
#     "LogsetId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
#     "RequestId": "xxx"
# }

# 记录返回的 LogsetId，后续步骤使用
LOGSET_ID="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
```

### 步骤2: 创建 CLS 日志主题

日志主题（Topic）是日志存储和检索的基本单元，关联到日志集下。

```bash
tccli cls CreateTopic \
    --region <Region> \
    --LogsetId LOGSET_ID \
    --TopicName TOPIC_NAME \
    --Period 30 \
    --StorageType hot
# expected:
# {
#     "TopicId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
#     "RequestId": "xxx"
# }

# 记录返回的 TopicId
TOPIC_ID="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
```

### 步骤3: 为日志主题创建全文索引

日志采集后需要创建索引才能检索。全文索引将日志全文作为可检索字段。

```bash
cat > create-index.json <<'EOF'
{
    "TopicId": "TOPIC_ID",
    "Rule": {
        "FullText": {
            "CaseSensitive": false,
            "Tokenizer": " \t\n\r,;:.|[]{}()!@#$%^&*"
        }
    },
    "Status": true
}
EOF

# 替换占位符
sed -i '' "s/TOPIC_ID/$TOPIC_ID/g" create-index.json

tccli cls CreateIndex \
    --region <Region> \
    --cli-input-json file://create-index.json
# expected:
# {
#     "RequestId": "xxx"
# }
```

### 步骤4: 开启 TKE 集群日志采集开关

将 EKS 集群的 stdout 日志投递到 CLS 日志主题。

```bash
cat > enable-eks-log.json <<'EOF'
{
    "ClusterId": "CLUSTER_ID",
    "LogsetId": "LOGSET_ID",
    "TopicId": "TOPIC_ID",
    "LogType": "container_stdout"
}
EOF

# 替换占位符
sed -i '' "s/CLUSTER_ID/$CLUSTER_ID/g" enable-eks-log.json
sed -i '' "s/LOGSET_ID/$LOGSET_ID/g" enable-eks-log.json
sed -i '' "s/TOPIC_ID/$TOPIC_ID/g" enable-eks-log.json

tccli tke EnableEksLog \
    --region <Region> \
    --cli-input-json file://enable-eks-log.json
# expected:
# {
#     "RequestId": "xxx"
# }
```

## 验证

### 控制面（tccli）

```bash
# 1. 验证 CLS 日志集已创建
tccli cls DescribeLogsets \
    --region <Region> \
    --Filters '[{"Key":"logsetName","Values":["LOGSET_NAME"]}]'
# expected: TotalCount >= 1，Logsets[0].LogsetId 与创建返回一致

# 2. 验证 CLS 日志主题已创建并关联日志集
tccli cls DescribeTopics \
    --region <Region> \
    --Filters '[{"Key":"logsetId","Values":["LOGSET_ID"]}]'
# expected: TotalCount >= 1，Topics[0].Status="1"

# 3. 验证 TKE 日志采集开关已开启
tccli tke DescribeEksLogSwitches \
    --region <Region> \
    --ClusterId CLUSTER_ID
# expected:
# {
#     "SwitchSet": [
#         {
#             "LogType": "container_stdout",
#             "Switch": true,
#             "LogsetId": "LOGSET_ID",
#             "TopicId": "TOPIC_ID"
#         }
#     ],
#     "RequestId": "xxx"
# }

# 4. 验证索引已启用
tccli cls DescribeIndex \
    --region <Region> \
    --TopicId TOPIC_ID
# expected: Status=true，Rule.FullText 不为空
```

```json
{"RequestId": "..."}
```

### 数据面（需 VPN/IOA）

```bash
# 5. 验证 log-agent 组件正常运行
kubectl get pods -n kube-system -l app=log-agent
# expected: 所有 Pod STATUS=Running, READY=1/1

# 6. 创建测试 Pod 验证日志投递
kubectl run cls-log-test --image=busybox:1.28 --restart=Never -n default -- \
    /bin/sh -c 'echo "CLS_TEST_$(date +%s)"; sleep 3600'
# expected: pod/cls-log-test created

# 7. 等待 2-3 分钟后检索 CLS（控制面可执行）
tccli cls SearchLog \
    --region <Region> \
    --TopicId TOPIC_ID \
    --From $(python3 -c "import time; print(int((time.time()-600)*1000))") \
    --To $(python3 -c "import time; print(int(time.time()*1000))") \
    --Query "CLS_TEST"
# expected: Results 数组含 CLS_TEST_xxx 日志条目
```

```text
NAME  STATUS  AGE
...
```

## 清理

> **计费警告**：CLS 按日志写入量（写流量）、存储量（存储空间）和索引流量计费。开启采集后持续产生费用，按存储天数保留日志。请确保清理不再使用的日志主题和日志集。

> **副作用警告**：删除日志主题后，该主题下的所有历史日志将被永久删除且不可恢复。删除日志集将同时删除其下所有日志主题。

```bash
# 1. 关闭 TKE 日志采集开关（停止投递）
cat > disable-eks-log.json <<'EOF'
{
    "ClusterId": "CLUSTER_ID",
    "LogType": "container_stdout",
    "Enable": false
}
EOF
sed -i '' "s/CLUSTER_ID/$CLUSTER_ID/g" disable-eks-log.json

tccli tke DisableEksLog \
    --region <Region> \
    --cli-input-json file://disable-eks-log.json
# expected:
# {
#     "RequestId": "xxx"
# }

# 2. 删除日志主题
tccli cls DeleteTopic \
    --region <Region> \
    --TopicId TOPIC_ID
# expected:
# {
#     "RequestId": "xxx"
# }

# 3. 删除日志集（必须先删除其下所有主题）
tccli cls DeleteLogset \
    --region <Region> \
    --LogsetId LOGSET_ID
# expected:
# {
#     "RequestId": "xxx"
# }

# 4. 清理测试 Pod
kubectl delete pod cls-log-test -n default --ignore-not-found
# expected: pod "cls-log-test" deleted
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `CreateLogset` 返回 `InvalidParameter.LogsetConflict` | 查询已有日志集 `tccli cls DescribeLogsets --region <Region>` | 日志集名称在同地域已存在 | 使用不同名称或先删除同名日志集 |
| `CreateLogset` 返回 `LimitExceeded.Logset` | 查询日志集数量 | 日志集数量达到配额上限（默认 20） | 清理不再使用的日志集，或提交工单提升配额 |
| `DescribeEksLogSwitches` 返回 `ResourceNotFound` | 检查 `ClusterId` 格式 | 集群 ID 格式错误或集群不存在 | 确认 `ClusterId` 为 `cls-xxxxxxxx` 格式，且集群状态为 Running |
| `UnauthorizedOperation` 执行 CLS 操作时 | `tccli configure list` 确认凭证 | CAM 缺少 CLS 写权限（CreateLogset、CreateTopic 等） | 为主账号添加 `QcloudCLSFullAccess` 策略 |

### 创建成功但状态异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 采集开关已开启但 CLS 无日志 | `tccli cls SearchLog --TopicId TOPIC_ID --Query "*"` | log-agent 未运行或投递延迟 | `kubectl get pods -n kube-system -l app=log-agent` 确认组件正常，等待 2-3 分钟重试 |
| CLS 检索无结果但日志已投递 | `tccli cls DescribeIndex --TopicId TOPIC_ID` | 索引未配置或配置有误 | 重新执行 `CreateIndex` 创建全文索引 |
| 日志投递延迟超过 5 分钟 | 对比写入时间和检索到的日志时间戳 | log-agent 资源不足或 CLS 写入限流 | 联系平台检查 log-agent 资源使用，确认是否有 CLS 限流告警 |
| 日志内容出现乱码 | `tccli cls SearchLog --TopicId TOPIC_ID --Query "*" --Limit 10` | 容器输出包含非 UTF-8 字符 | 在应用层确保日志输出为 UTF-8 编码 |

## 下一步

- [通过控制台配置日志采集](../通过控制台配置日志采集/tccli%20操作.md)：使用 kubectl 创建 LogConfig CRD 细化采集规则
- [通过 YAML 配置日志采集](../通过 YAML 配置日志采集/tccli%20操作.md)：LogConfig CRD 完整 YAML 参考
- [使用 CRD 采集日志到 Kafka](../../使用%20CRD%20采集日志到%20Kafka/tccli%20操作.md)：采集日志到自建 Kafka
- [CLS 检索语法](https://cloud.tencent.com/document/product/614/47044)：CLS 检索分析语法参考

## 控制台替代

控制台路径：**CLS 控制台 > 日志集管理 > 新建日志集**，创建日志集和主题后，进入 **TKE 控制台 > Serverless 集群 > 集群 ID > 日志采集 > 开启日志采集**，选择日志集和主题并开启。tccli 通过 API 编排创建日志集、主题、索引并绑定到集群，适合 CI/CD 流水线和基础设施即代码管理。
