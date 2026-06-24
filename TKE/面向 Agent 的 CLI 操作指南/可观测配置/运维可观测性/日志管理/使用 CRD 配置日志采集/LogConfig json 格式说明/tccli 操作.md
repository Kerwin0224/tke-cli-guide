# LogConfig JSON 格式说明（tccli）

> 对照官方：[LogConfig JSON 格式说明](https://cloud.tencent.com/document/product/457/111541) · page_id `111541`

## 概述

LogConfig 是 TKE 日志采集功能的自定义资源（CRD），apiVersion 为 `cls.cloud.tencent.com/v1`。通过 kubectl apply 即可配置日志采集规则，支持容器标准输出（container_stdout）、容器文件路径（container_file）、节点文件路径（host_file）三种采集类型，支持 CLS 和 Kafka 两种消费端。

## 前置条件

- `tccli` 已配置凭证
- kubectl 不可达：演示集群 cls-xxxxxxxx 为托管集群，公网端点可能受 CAM 策略限制（strategyId:240463971）；内网无 VPN 时不可达。以下 kubectl 命令作为文档参考。
- 已开启日志采集功能（`tccli tke InstallLogAgent`）

```bash
tccli tke DescribeClusters --ClusterIds '["cls-xxxxxxxx"]' --region ap-guangzhou
```

```json
{
  "RequestId": "1e8b2c3d-4a5f-6789-abcd-ef0123456789",
  "TotalCount": 1,
  "Clusters": [
    {
      "ClusterId": "cls-xxxxxxxx",
      "ClusterName": "my-test-cluster",
      "ClusterType": "MANAGED_CLUSTER",
      "ClusterStatus": "Running",
      "ClusterVersion": "1.30.0",
      "ClusterNodeNum": 2
    }
  ]
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 创建采集配置 | `kubectl apply -f logconfig.yaml` | 是 |
| 查看采集配置 | `kubectl get logconfig -A` | 是 |
| 查看配置详情（YAML） | `kubectl get logconfig <Name> -o yaml` | 是 |
| 查看配置详情（JSON） | `kubectl get logconfig <Name> -o json` | 是 |
| 更新采集配置 | `kubectl apply -f logconfig.yaml` | 是 |
| 删除采集配置 | `kubectl delete logconfig <Name>` | 否 |
| 查看同步状态 | `kubectl get logconfig <Name> -o jsonpath='{.status.status}'` | 是 |

## 操作步骤

### LogConfig CRD 顶级结构

```yaml
apiVersion: cls.cloud.tencent.com/v1
kind: LogConfig
metadata:
  name: <logconfig-name>    # 必填，LogConfig 名称，唯一标识
spec:
  clsDetail: {}             # CLS 消费端配置（与 kafkaDetail 二选一）
  kafkaDetail: {}           # Kafka 消费端配置（与 clsDetail 二选一）
  inputDetail: {}           # 必填，日志采集规则
status: {}                  # 只读，LogConfig 同步状态
```

### inputDetail 字段说明

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `inputDetail.type` | String | 是 | 采集类型：`container_stdout` / `container_file` / `host_file` |
| `inputDetail.containerStdout` | Object | 条件 | type=container_stdout 时必填 |
| `inputDetail.containerFile` | Object | 条件 | type=container_file 时必填 |
| `inputDetail.hostFile` | Object | 条件 | type=host_file 时必填 |

### containerStdout 子字段

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `namespace` | String | 否 | 命名空间，为空表示所有命名空间 |
| `allContainers` | Bool | 否 | 是否采集所有容器，默认 false |
| `workloads` | Array | 否 | 工作负载列表 |
| `workloads[].kind` | String | 是 | 工作负载类型：Deployment / StatefulSet / DaemonSet / Job / CronJob |
| `workloads[].name` | String | 是 | 工作负载名称 |
| `workloads[].namespace` | String | 是 | 工作负载所在命名空间 |
| `container` | String | 否 | 容器名称 |
| `includeLabels` | Object | 否 | Pod 标签选择器（key-value map） |
| `excludeLabels` | Object | 否 | 排除的 Pod 标签选择器 |
| `metadataLabels` | Array | 否 | 附加的 Kubernetes 元数据标签 |

### containerFile 子字段

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `namespace` | String | 是 | 命名空间 |
| `workload` | Object | 是 | 工作负载信息 |
| `container` | String | 否 | 容器名称 |
| `logPath` | String | 是 | 日志文件夹路径 |
| `filePattern` | String | 是 | 日志文件匹配模式，如 `*.log` |
| `includeLabels` | Object | 否 | Pod 标签选择器 |
| `customLabels` | Object | 否 | 自定义 Key-Value 元数据标签 |

### hostFile 子字段

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `logPath` | String | 是 | 日志文件夹绝对路径 |
| `filePattern` | String | 是 | 日志文件匹配模式，如 `*.log` |
| `customLabels` | Object | 否 | 自定义 Key-Value 标签 |

### clsDetail 字段说明

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `clsDetail.logsetName` | String | 条件 | 日志集名称（与 topicId 互斥：如使用 topicId 则无需 logsetName） |
| `clsDetail.topicName` | String | 条件 | 日志主题名称（与 topicId 互斥） |
| `clsDetail.topicId` | String | 条件 | 日志主题 ID（与 topicName 互斥，优先级高于 topicName） |
| `clsDetail.logType` | String | 是 | 日志解析格式：`json_log` / `minimalist_log` / `delimiter_log` / `multiline_log` / `fullregex_log` |
| `clsDetail.extractRule` | Object | 否 | 日志提取规则（JSON 格式时无需） |
| `clsDetail.period` | Int | 否 | 日志保存时间，默认 30 天 |
| `clsDetail.logFormat` | String | 否 | 日志格式，默认 `default` |
| `clsDetail.tags` | Array | 否 | 日志主题标签 |

### kafkaDetail 字段说明

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `kafkaDetail.brokers` | String | 是 | Kafka 代理地址 |
| `kafkaDetail.topic` | String | 是 | Kafka 主题 |
| `kafkaDetail.kafkaType` | String | 是 | Kafka 类型：`CKafka` / `userKafka` |
| `kafkaDetail.instanceId` | String | 条件 | CKafka 实例 ID（kafkaType=CKafka 时必填） |
| `kafkaDetail.logType` | String | 是 | 日志解析格式 |
| `kafkaDetail.outputType` | String | 否 | 输出格式，默认 `json` |
| `kafkaDetail.userKafka` | Object | 否 | 自建 Kafka 配置（kafkaType=userKafka 时） |
| `kafkaDetail.timestamp` | Bool | 否 | 是否添加时间戳 |

### 查看 LogConfig CRD 定义

```bash
kubectl get crd logconfigs.cls.cloud.tencent.com -o yaml
```

```text
NAME  STATUS  AGE
...
```

### 查看所有 LogConfig

```bash
kubectl get logconfig -A
```

```text
NAME                 AGE
test-stdout          5m
test-container-file  3m
```

### 查看 LogConfig JSON 格式

```bash
kubectl get logconfig test-stdout -o json
```

```json
{
  "apiVersion": "cls.cloud.tencent.com/v1",
  "kind": "LogConfig",
  "metadata": {
    "name": "test-stdout",
    "creationTimestamp": "2026-06-18T10:00:00Z"
  },
  "spec": {
    "clsDetail": {
      "logsetName": "example-logset",
      "topicName": "example-topic",
      "logType": "minimalist_log",
      "period": 30
    },
    "inputDetail": {
      "type": "container_stdout",
      "containerStdout": {
        "allContainers": true,
        "namespace": "default"
      }
    }
  },
  "status": {
    "status": "Synced"
  }
}
```

### 查看同步状态

```bash
kubectl get logconfig test-stdout -o jsonpath='{.status.status}'
```

```text
Synced
```

`status.status` 可能的值：
- `""` (空)：初始状态，还未处理
- `Synced`：采集配置已成功同步到 LogListener
- `Stale`：配置有误，同步失败

## 验证

### Data plane (kubectl) — 参考

```bash
kubectl get logconfig <Name> -o jsonpath='{.status}'
```

```text
NAME  STATUS  AGE
...
```

确认 `status.status` 为 `Synced`。

## 清理

### Data plane (kubectl) — 参考

```bash
kubectl delete logconfig <Name>
```

## 排障

| 现象 | 处理 |
|------|------|
| status 为 Stale | 检查 inputDetail 配置是否正确；确认日志路径在容器中存在 |
| status 为空 | 初始状态，等待 LogListener 处理（最长 10 秒） |
| metadataLabels 不生效 | LogConfig status 为空表示初始状态，正确配置后变为 Synced |
| Kafka 采集限制 | 单规则最多监听约 256 个文件；超出需提交工单 |
| 日志单行长度限制 | 标准输出最大 16KB；容器/节点文件最大 2MB |
| kubectl 不可达 | 托管集群公网端点可能受 CAM 策略限制（strategyId:240463971）；内网无 VPN 时不可达。CRD 配置方式的 kubectl 命令作为文档参考；采集规则也可通过控制台或 `tccli tke CreateCLSLogConfig` 创建 |

## 下一步

- [使用 CRD 进行日志采集配置指导](../使用 CRD 进行日志采集配置指导/tccli 操作.md) — CRD 采集配置完整示例
- [采集容器日志到 CLS](../../采集容器日志到 CLS/tccli 操作.md) — 控制台方式配置日志采集

## 控制台替代

登录 [容器服务控制台](https://console.cloud.tencent.com/tke2) > 选择集群 > 日志 > 业务日志 > 新增采集配置，控制台会生成等效的 LogConfig CRD。
