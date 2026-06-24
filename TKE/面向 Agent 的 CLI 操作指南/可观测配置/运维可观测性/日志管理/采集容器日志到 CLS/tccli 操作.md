# 采集容器日志到 CLS（tccli）

> 对照官方：[采集容器日志到 CLS](https://cloud.tencent.com/document/product/457/36771) · page_id `36771`

## 概述

容器服务 TKE 支持在控制台配置日志采集规则，将容器标准输出、容器文件、节点文件路径的日志投递到 CLS（腾讯云日志服务）。日志采集组件 LogListener 以 DaemonSet 方式运行。

## 前置条件

- `tccli` 已配置凭证
- kubectl 不可达：演示集群 cls-xxxxxxxx 为托管集群，公网端点可能受 CAM 策略限制（strategyId:240463971）；内网无 VPN 时不可达。以下 kubectl 命令作为文档参考。
- 已开启日志采集功能（参见 [日志采集概述](../日志采集概述/tccli 操作.md)）
- 已在 CLS 中创建日志集和日志主题
- 确认集群节点可访问 CLS；了解 [LogListener 限制说明](https://cloud.tencent.com/document/product/614/63513)

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
| 创建日志采集规则 | `tccli tke CreateCLSLogConfig` | 否（同名覆盖旧配置） |
| 查看采集规则 | `tccli tke DescribeLogConfigs` | 是 |
| 更新采集规则 | `tccli tke ModifyLogConfig` | 是 |
| 删除采集规则 | `tccli tke DeleteLogConfigs` | 是 |
| 开启日志采集 | `tccli tke InstallLogAgent` | 是 |
| 关闭日志采集 | `tccli tke UninstallLogAgent` | 否 |

## 操作步骤

### 创建日志采集规则（容器标准输出）

`--LogConfig` 参数为 JSON 字符串，格式如下：

```bash
tccli tke CreateCLSLogConfig --ClusterId cls-xxxxxxxx \
  --LogConfig '{"Name":"demo-stdout-rule","Input":{"type":"container_stdout","containerStdout":{"namespace":"default","allContainers":true}},"Output":{"type":"CLS","cls":{"logsetId":"logset-xxxxxxxx","topicId":"topic-xxxxxxxx"}},"LogFormat":"json_log"}' \
  --region ap-guangzhou
```

```json
{
  "RequestId": "d8e9f0a1-b2c3-4567-8901-234567890123"
}
```

`--LogConfig` JSON 结构说明：

| 字段 | 类型 | 说明 |
|------|------|------|
| `Name` | String | 日志采集规则名称 |
| `Input.type` | String | 采集类型：`container_stdout`（容器标准输出）、`container_file`（容器文件）、`host_file`（节点文件） |
| `Input.containerStdout.namespace` | String | 命名空间 |
| `Input.containerStdout.allContainers` | Bool | 是否采集所有容器 |
| `Output.type` | String | 消费端类型：`CLS` |
| `Output.cls.logsetId` | String | CLS 日志集 ID |
| `Output.cls.topicId` | String | CLS 日志主题 ID |
| `LogFormat` | String | 日志格式：`json_log`、`minimalist_log`、`delimiter_log` 等 |

### 查看采集规则列表

```bash
tccli tke DescribeLogConfigs --ClusterId cls-xxxxxxxx --region ap-guangzhou
```

```json
{
  "RequestId": "e9f0a1b2-c3d4-5678-9012-345678901234",
  "LogConfigs": [
    {
      "Name": "demo-stdout-rule",
      "Input": {
        "type": "container_stdout",
        "containerStdout": {
          "namespace": "default",
          "allContainers": true
        }
      },
      "Output": {
        "type": "CLS",
        "cls": {
          "logsetId": "logset-xxxxxxxx",
          "topicId": "topic-xxxxxxxx"
        }
      }
    }
  ]
}
```

### 更新日志规则

```bash
tccli tke ModifyLogConfig --ClusterId cls-xxxxxxxx \
  --LogConfig '{"Name":"demo-stdout-rule","Input":{"type":"container_stdout","containerStdout":{"namespace":"default","allContainers":false,"workloads":[{"kind":"Deployment","name":"nginx-deploy","namespace":"default"}]}},"Output":{"type":"CLS","cls":{"logsetId":"logset-xxxxxxxx","topicId":"topic-xxxxxxxx"}}}' \
  --region ap-guangzhou
```

```json
{
  "RequestId": "f0a1b2c3-d4e5-6789-0123-456789012345"
}
```

### 删除日志规则

```bash
tccli tke DeleteLogConfigs --ClusterId cls-xxxxxxxx --LogConfigNames "demo-stdout-rule" --region ap-guangzhou
```

```json
{
  "RequestId": "a1b2c3d4-e5f6-7890-1234-567890123456"
}
```

## 验证

### Control plane (tccli)

```bash
tccli tke DescribeLogConfigs --ClusterId cls-xxxxxxxx --region ap-guangzhou
```

```json
{
  "Total": 0,
  "Message": "<Message>",
  "LogConfigs": "<LogConfigs>",
  "RequestId": "<RequestId>"
}
```

确认采集规则列表中包含已创建的规则。

## 清理

### Control plane (tccli)

```bash
tccli tke DeleteLogConfigs --ClusterId cls-xxxxxxxx --LogConfigNames "demo-stdout-rule" --region ap-guangzhou
tccli tke UninstallLogAgent --ClusterId cls-xxxxxxxx --region ap-guangzhou
```

## 排障

| 现象 | 处理 |
|------|------|
| CLS 不支持跨地域 | CLS 不支持国内和境外地域之间跨地域日志投递；未开通的地域仅支持就近投递 |
| 日志主题冲突 | 一个日志主题仅支持一类日志（日志/审计/事件不可共用同一 topic） |
| 容器文件路径采集失败 | "容器文件路径"不能为符号链接或硬链接 |
| 节点文件路径采集失败 | "节点文件路径"不能为软链接或硬链接；一个节点日志文件只能被一个日志主题采集 |
| 日志集满 | 日志集下最多 500 个日志主题 |
| kubectl 不可达 | 托管集群公网端点可能受 CAM 策略限制（strategyId:240463971）；内网无 VPN 时不可达；采集规则操作需通过控制台或 tccli TKE API（`CreateCLSLogConfig`/`DescribeLogConfigs`/`DeleteLogConfigs`）替代 kubectl |

## 下一步

- [使用 CRD 进行日志采集配置指导](../使用 CRD 配置日志采集/使用 CRD 进行日志采集配置指导/tccli 操作.md) — CRD 方式配置日志采集
- [日志采集概述](../日志采集概述/tccli 操作.md) — 日志采集基本概念与开启/关闭

## 控制台替代

登录 [容器服务控制台](https://console.cloud.tencent.com/tke2) > 选择集群 > 日志 > 业务日志 > 新增采集配置 / 编辑采集规则。
