# 使用 CLS 告警异常资源（tccli）

> 对照官方：[使用 CLS 告警异常资源](https://cloud.tencent.com/document/product/457/82037) · page_id `82037`

## 概述

将 TKE 事件持久化到 CLS 后，在 CLS 中创建告警策略，当出现异常事件（Pod Crash、Node NotReady、OOMKilled）时触发通知。

## 前置条件

- 事件持久化已开启（`EnableEventPersistence`）
- CLS 告警策略已创建

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 开启事件持久化 | `tccli tke EnableEventPersistence --ClusterId <ClusterId>` | 是 |
| 创建 CLS 告警 | CLS 控制台 | 否 |
| 查看事件 | `kubectl get events -A -w` | 是 |

## 操作步骤

### 1. 确保事件持久化已开启

```bash
tccli tke EnableEventPersistence --region ap-guangzhou --ClusterId <ClusterId> --LogsetId <LogsetId> --TopicId <TopicId>
```

### 2. CLS 告警检索语句示例

```
# Pod Crash 告警
reason:"BackOff" OR reason:"CrashLoopBackOff"

# 节点异常告警
reason:"NodeNotReady" OR reason:"NodeHasDiskPressure"

# OOM 告警
reason:"OOMKilling"
```

### 3. 在 CLS 控制台创建告警策略

配置告警触发条件、通知渠道（短信/邮件/企业微信）。

## 验证

```bash
kubectl get events -A --field-selector type=Warning | tail -5
```

```text
NAME  STATUS  AGE
...
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl` 命令超时或连接被拒绝 | `tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]'`（控制面返回 Running） | CAM 策略 strategyId:240463971 拒绝公网端点，数据面需内网访问 | 通过 VPN/IOA 接入 VPC 内网后执行 kubectl；或使用控制台替代 |
| 事件持久化未启用 | `tccli tke DescribeClusterKubeconfig --ClusterId <ClusterId> > /dev/null` 后 `kubectl get events -A` 为空 | EventPersistence 未开启 | `tccli tke EnableEventPersistence --region ap-guangzhou --ClusterId <ClusterId>` |
| 告警未触发 | CLS 控制台 → 告警策略 → 查看触发历史 | 告警条件过严或通知渠道未配置 | 调整告警触发条件；确认通知渠道（短信/邮件/企业微信）已配置 |
| CLS 检索返回 `UnauthorizedOperation` | `tccli configure list` 检查当前凭据 | 子账号缺少 `cls:SearchLog` 权限 | 联系主账号授予 CLS 只读权限（`QcloudCLSReadOnlyAccess`） |

## 清理

```bash
tccli tke DisableEventPersistence --region ap-guangzhou --ClusterId <ClusterId>
```

## 下一步

- [TKE 日志采集最佳实践](../TKE%20日志采集最佳实践/tccli%20操作.md)
- [使用 TKE 审计和事件服务快速排查问题](../../运维/使用%20TKE%20审计和事件服务快速排查问题/tccli%20操作.md)

## 控制台替代

CLS 控制台 → 告警策略 → 新建 → 选择事件日志主题。
