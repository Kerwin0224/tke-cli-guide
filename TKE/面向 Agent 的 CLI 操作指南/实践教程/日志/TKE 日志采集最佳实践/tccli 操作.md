# TKE 日志采集最佳实践（tccli）

> 对照官方：[TKE 日志采集最佳实践](https://cloud.tencent.com/document/product/457/48836) · page_id `48836`

## 概述

TKE 通过 `InstallLogAgent` 安装日志采集组件，配合 `CreateCLSLogConfig` 创建采集规则，将容器标准输出和文件日志采集到 CLS。

## 前置条件

- 已开通 [CLS 日志服务](https://console.cloud.tencent.com/cls)

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 安装日志组件 | `tccli tke InstallLogAgent --ClusterId <ClusterId>` | 是 |
| 创建采集规则 | `tccli tke CreateCLSLogConfig --cli-input-json file://config.json` | 否 |
| 查看采集状态 | `tccli tke DescribeLogConfigs --ClusterId <ClusterId>` | 是 |
| 修改规则 | `tccli tke ModifyLogConfig` | 否 |

## 操作步骤

### 1. 安装日志采集组件

```bash
tccli tke InstallLogAgent --region ap-guangzhou --ClusterId <ClusterId>
```

### 2. 创建采集规则

```bash
tccli tke CreateCLSLogConfig --region ap-guangzhou --cli-input-json file://log-config.json
```

```json
{
    "ClusterId": "<ClusterId>",
    "LogConfigName": "app-stdout",
    "ClsLogsetId": "<LogsetId>",
    "ClsTopicId": "<TopicId>",
    "InputDetail": {"ContainerStdout": {"AllContainers": true}}
}
```

### 3. 验证 CLS 日志

在 CLS 控制台检索确认日志到达。

## 验证

```bash
tccli tke DescribeLogConfigs --region ap-guangzhou --ClusterId <ClusterId>
```

```json
{
  "Total": 0,
  "Message": "<Message>",
  "LogConfigs": "<LogConfigs>",
  "RequestId": "<RequestId>"
}
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl` 命令超时或连接被拒绝 | `tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]'`（控制面返回 Running） | CAM 策略 strategyId:240463971 拒绝公网端点，数据面需内网访问 | 通过 VPN/IOA 接入 VPC 内网后执行 kubectl；或使用控制台替代 |
| `DescribeLogConfigs` 返回空 | `tccli tke DescribeLogConfigs --region ap-guangzhou --ClusterId <ClusterId>` | LogConfig 未创建或名称不匹配 | 确认 `--LogConfigNames` 参数值与创建时一致 |
| 日志未投递到 CLS | `tccli cls SearchLog --TopicId <TopicId>` 检索最近 1 小时 | LogAgent 未安装或 CLS 主题不存在 | `tccli tke DescribeAddon --ClusterId <ClusterId> --AddonName tke-log-agent` 确认组件 running；确认 TopicId 有效 |
| CLS 检索返回 `UnauthorizedOperation` | `tccli configure list` 检查当前凭据 | 子账号缺少 `cls:SearchLog` 权限 | 联系主账号授予 CLS 只读权限（`QcloudCLSReadOnlyAccess`） |

## 清理

```bash
tccli tke DeleteLogConfigs --region ap-guangzhou --ClusterId <ClusterId> --LogConfigNames '["app-stdout"]'
tccli tke UninstallLogAgent --region ap-guangzhou --ClusterId <ClusterId>
```

## 下一步

- [NginxIngress 自定义日志](../NginxIngress%20自定义日志/tccli%20操作.md)
- [使用 CLS 告警异常资源](../使用%20CLS%20告警异常资源/tccli%20操作.md)

## 控制台替代

控制台：集群 → 日志管理 → 日志采集 → 新建采集规则。
