# 日志采集

> 对照官方：[日志采集](https://cloud.tencent.com/document/product/457/72148) · page_id `72148`

## 概述

注册集群支持将容器标准输出、文件日志采集到 CLS（日志服务），实现统一日志管理。日志采集依赖两个组件：TKE 管理面配置（通过 `tccli tke` 开启日志采集功能并指定 CLS 日志集/主题）和外部集群 agent（通过 `kubectl` 部署 LogConfig CRD 及采集规则）。

本文演示：**tccli** 侧在 TKE 管理面为注册集群开启日志采集并绑定 CLS 日志主题，**kubectl** 侧在外部集群配置 LogConfig 规则指定待采集的日志源。两类操作分别在各自环境执行。

## 前置条件

- [环境准备](../../../../环境准备.md)
- 已完成 [创建注册集群](../../注册集群管理/创建注册集群/tccli%20操作.md)，外部集群 agent 已部署且状态正常
- 已创建 CLS 日志集和日志主题（参见 [CLS 快速入门](https://cloud.tencent.com/document/product/614/34340)）

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
# 1. 查询 CLS 日志集
tccli cls DescribeLogsets --region <Region>
# expected: 返回日志集列表，确认目标 LogsetId 存在

# 2. 查询 CLS 日志主题
tccli cls DescribeTopics --region <Region> \
  --Filters '[{"Key":"logsetId","Values":["LOGSET_ID"]}]'
# expected: 返回日志主题列表，确认目标 TopicId 存在

# 3. 确认外部集群 agent 正常（在外部集群执行）
kubectl get pods -n tke-xxx
# expected: agent Pod STATUS 为 Running

# 4. 检查外部集群是否已有 LogConfig CRD（在外部集群执行）
kubectl get crd | grep logconfigs
# expected: logconfigs 存在（若不存在，需先安装日志采集组件）
```

```text
NAME  STATUS  AGE
...
```

### CLS 资源准备

- **日志集（Logset）**：日志的逻辑容器，一个日志集下可有多个日志主题
- **日志主题（Topic）**：日志的实际存储单元，采集的日志写入指定主题
- **地域**：日志集和 TKE 注册集群应在同一地域，否则会产生跨地域传输费用

## 控制台与 CLI 参数映射

| 控制台操作 | tccli / kubectl 命令 | 幂等 |
|-----------|-----------|:--:|
| 为注册集群开启日志采集 | `tccli tke EnableExternalClusterLog --region <Region> --ClusterId CLUSTER_ID --LogsetId LOGSET_ID --TopicId TOPIC_ID` | 是 |
| 查看日志采集配置 | `tccli tke DescribeExternalClusterLog --region <Region> --ClusterId CLUSTER_ID` | 是（只读） |
| 关闭日志采集 | `tccli tke DisableExternalClusterLog --region <Region> --ClusterId CLUSTER_ID` | 是 |
| 创建 LogConfig 采集规则 | `kubectl apply -f logconfig.yaml`（外部集群执行） | 是 |
| 查看采集规则 | `kubectl get logconfigs --all-namespaces`（外部集群执行） | 是（只读） |
| 查询 CLS 日志 | `tccli cls SearchLog --region <Region> --TopicId TOPIC_ID --From <ts> --To <ts> --Query "<query>"` | 是（只读） |

## 操作步骤

### 步骤1：在 TKE 管理面为注册集群开启日志采集

```bash
# 创建 CLS 日志主题（如尚未创建）
tccli cls CreateTopic --region <Region> \
  --LogsetId LOGSET_ID \
  --TopicName "TOPIC_NAME" \
  --output json
# expected: exit 0，返回 TopicId
```

```bash
# 为注册集群绑定日志采集配置
tccli tke EnableExternalClusterLog --region <Region> \
  --ClusterId CLUSTER_ID \
  --LogsetId LOGSET_ID \
  --TopicId TOPIC_ID \
  --output json
# expected: exit 0，返回 RequestId
```

### 步骤2：验证 TKE 侧日志采集配置

```bash
tccli tke DescribeExternalClusterLog --region <Region> \
  --ClusterId CLUSTER_ID --output json
# expected: 返回 LogsetId、TopicId，状态为 Enabled
```

预期输出：

```json
{
    "LogsetId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
    "TopicId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
    "Status": "Enabled",
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

### 步骤3：在外部集群安装日志采集组件（如未安装）

> 以下命令在**外部集群 kubectl 管理节点**执行。

```bash
# 检查是否已安装日志采集组件
kubectl get pods -n kube-system | grep cls
# expected: 若有 cls-provisioner Pod 则已安装；若无输出则需安装

# 检查 LogConfig CRD
kubectl get crd logconfigs.cloud.tencent.com
# expected: 返回 CRD 详情（已安装）；NotFound 则需安装

# 若未安装，在 TKE 控制台 "日志采集" 页面获取组件安装命令后执行
```

### 步骤4：创建日志采集规则（LogConfig）

创建 `logconfig-example.yaml`：

```yaml
apiVersion: cloud.tencent.com/v1
kind: LogConfig
metadata:
  name: LOG_CONFIG_NAME
  namespace: TARGET_NAMESPACE
spec:
  inputDetail:
    type: container_stdout
    containerStdout:
      namespace: TARGET_NAMESPACE
      allContainers: true
  outputDetail:
    type: cls
    cls:
      logsetId: "LOGSET_ID"
      topicId: "TOPIC_ID"
```

```bash
# 应用采集规则（在外部集群执行）
kubectl apply -f logconfig-example.yaml
# expected: logconfig.cloud.tencent.com/LOG_CONFIG_NAME created

# 查看采集规则状态
kubectl get logconfigs -n TARGET_NAMESPACE
# expected: NAME 为 LOG_CONFIG_NAME，AGE 记录创建时间
```

```text
NAME  STATUS  AGE
...
```

### 步骤5：验证日志已采集到 CLS

```bash
# 等待 2-3 分钟后查询 CLS 日志
tccli cls SearchLog --region <Region> \
  --TopicId TOPIC_ID \
  --From $(date -v-10M +%s)000 \
  --To $(date +%s)000 \
  --Query "*" \
  --Limit 5 \
  --output json
# expected: 返回日志条目，Results 非空
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `REGION` | 地域 | 如 `ap-guangzhou` | `tccli configure list` |
| `CLUSTER_ID` | 注册集群 ID | 格式 `cls-xxxxxxxx` | `tccli tke DescribeExternalClusters --region <Region>` |
| `LOGSET_ID` | CLS 日志集 ID | UUID 格式 | `tccli cls DescribeLogsets --region <Region>` |
| `TOPIC_ID` | CLS 日志主题 ID | UUID 格式 | `tccli cls DescribeTopics --region <Region>` |
| `TOPIC_NAME` | 日志主题名称 | 长度 1-255 | 自定义 |
| `LOG_CONFIG_NAME` | LogConfig 名称 | K8s 命名规范 | 自定义 |
| `TARGET_NAMESPACE` | 目标命名空间 | 外部集群中存在的命名空间 | `kubectl get ns`（外部集群执行） |

## 验证

### TKE 管理面验证

```bash
tccli tke DescribeExternalClusterLog --region <Region> \
  --ClusterId CLUSTER_ID
# expected: Status 为 Enabled，LogsetId / TopicId 与配置一致
```

```json
{"RequestId": "..."}
```

### 外部集群验证（kubectl）

```bash
# LogConfig 规则已生效
kubectl get logconfigs --all-namespaces
# expected: 列出全部 LogConfig 规则

# 日志采集组件 Pod 运行正常
kubectl get pods -n kube-system | grep cls
# expected: cls-provisioner 及相关 Pod STATUS 为 Running
```

```text
NAME  STATUS  AGE
...
```

### CLS 验证

```bash
# 确认 CLS 有最新日志
tccli cls SearchLog --region <Region> \
  --TopicId TOPIC_ID \
  --From $(date -v-5M +%s)000 \
  --To $(date +%s)000 \
  --Query "*" \
  --Limit 3 \
  --output json | jq '.Results | length'
# expected: > 0（有日志条目输出）
```

## 清理

> **警告**：关闭日志采集后，外部集群将停止向 CLS 写入日志。已采集到 CLS 的历史日志不受影响（按 CLS 数据保留策略管理）。
> **计费提醒**：关闭日志采集停止写入后，CLS 日志主题仍按存储量计费。如需完全停止计费，需删除 CLS 日志主题。

### 1. 停止 TKE 日志采集

```bash
tccli tke DisableExternalClusterLog --region <Region> \
  --ClusterId CLUSTER_ID --output json
# expected: exit 0
```

### 2. 删除 LogConfig 规则（在外部集群执行）

```bash
kubectl delete logconfig LOG_CONFIG_NAME -n TARGET_NAMESPACE
# expected: logconfig "LOG_CONFIG_NAME" deleted
```

### 3. 清理 CLS 资源（可选，不可逆）

```bash
# > 警告：删除日志主题会永久删除所有已采集日志，不可恢复
tccli cls DeleteTopic --region <Region> --TopicId TOPIC_ID
# expected: exit 0
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `EnableExternalClusterLog` 返回 `ResourceNotFound` | `DescribeExternalClusters` 确认集群存在 | 集群 ID 错误或 agent 未部署 | 确认 `ClusterId` 和 `--region` 正确，检查 agent 状态 |
| `EnableExternalClusterLog` 返回 `InvalidParameter.LogsetId` | `tccli cls DescribeLogsets --region <Region>` 验证 | LogsetId 不存在或地域不匹配 | 确认日志集在目标地域，使用正确的 LogsetId |
| `EnableExternalClusterLog` 返回 `InvalidParameter.TopicId` | `tccli cls DescribeTopics` 验证 | TopicId 不存在或不属于指定日志集 | 确认日志主题在目标日志集下 |
| `EnableExternalClusterLog` 返回 `AuthFailure` | `tccli configure list` 检查凭据 | CAM 缺少 CLS 或 TKE 权限 | 授予 `cls:*` 和 `tke:EnableExternalClusterLog` 权限 |
| `kubectl apply -f logconfig.yaml` 返回 `no matches for kind` | `kubectl get crd` 检查 LogConfig CRD | 日志采集组件未安装 | 先在外部集群安装 TKE 日志采集组件 |

### 操作成功但异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| TKE 侧配置成功但 CLS 查不到日志 | 等待 3-5 分钟后重试 `SearchLog` | 首次采集有延迟，或 LogConfig 未匹配到日志源 | 检查 LogConfig 中的 namespace 和容器选择条件，确认目标容器正在产生日志 |
| CLS 日志有数据但内容为空 | `SearchLog` 查看具体日志内容 | 容器输出为空或日志 agent 解析异常 | 检查容器日志输出格式，确认日志采集组件版本与 LogConfig spec 兼容 |
| 日志采集组件 Pod CrashLoopBackOff | `kubectl describe pod -n kube-system <cls-pod>` | 资源不足或 CLS endpoint 不可达 | 增加 Pod 资源配额，确认外部集群可访问 CLS endpoint |
| 删除 LogConfig 后仍持续写入 CLS | `kubectl get logconfigs --all-namespaces` 确认真实状态 | 有多条 LogConfig 规则指向同一日志源 | 检查并删除所有相关 LogConfig 规则 |

## 下一步

- [集群审计](../集群审计/tccli%20操作.md) — 启用集群审计日志（page_id `72149`）
- [事件存储](../事件存储/tccli%20操作.md) — 持久化 Kubernetes Events（page_id `72150`）
- [CLS 控制台](https://console.cloud.tencent.com/cls) — 管理日志集、主题和检索

## 控制台替代

[容器服务控制台 → 集群列表](https://console.cloud.tencent.com/tke2/external) 选中注册集群 → **运维功能 → 日志采集** → 开启并配置 CLS 日志集/主题 → 创建采集规则。
