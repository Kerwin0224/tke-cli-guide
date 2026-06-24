# 事件仪表盘（tccli）

> 对照官方：[事件仪表盘](https://cloud.tencent.com/document/product/457/50512) · page_id `50512`

## 概述

容器服务 TKE 在开启事件日志功能后，自动为集群配置三类事件仪表盘：事件总览、异常事件聚合检索、全局检索。仪表盘基于 CLS 日志服务，支持按集群 ID、命名空间、级别、原因、资源类型等维度过滤事件，并支持基于仪表盘配置告警。

## 前置条件

- `tccli` 已配置凭证
- kubectl 不可达：演示集群 cls-xxxxxxxx 为托管集群，公网端点可能受 CAM 策略限制（strategyId:240463971）；内网无 VPN 时不可达。以下 kubectl 命令作为文档参考。
- 已开启事件日志功能（`tccli tke EnableEventPersistence`）
- 事件日志已投递到 CLS

```bash
tccli tke DescribeLogSwitches --ClusterIds '["cls-xxxxxxxx"]' --region ap-guangzhou
```

```json
{
  "RequestId": "f2a3b4c5-d6e7-8901-2345-678901234567",
  "SwitchSet": [
    {
      "ClusterId": "cls-xxxxxxxx",
      "EventLog": {
        "Enable": true,
        "LogsetId": "logset-xxxxxxxx",
        "TopicId": "topic-xxxxxxxx",
        "TopicRegion": "ap-guangzhou"
      }
    }
  ]
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 开启事件日志（前提） | `tccli tke EnableEventPersistence` | 是 |
| 查看事件日志状态 | `tccli tke DescribeLogSwitches` | 是 |
| 在 CLS 检索事件 | CLS 检索 API（`cls:SearchLog`） | 是 |
| 配置告警 | CLS 告警策略 API（`cls:CreateAlarm`） | 是 |

## 操作步骤

### 开启事件持久化（前提）

```bash
tccli tke EnableEventPersistence --ClusterId cls-xxxxxxxx \
  --LogsetId logset-xxxxxxxx \
  --TopicId topic-xxxxxxxx \
  --TopicRegion ap-guangzhou \
  --region ap-guangzhou
```

### 事件总览 — 查看事件总数及级别分布

```bash
tccli cls SearchLog --TopicId topic-xxxxxxxx \
  --From 1749600000000 \
  --To 1749686400000 \
  --Query "* | SELECT type, count(*) AS cnt GROUP BY type ORDER BY cnt DESC" \
  --region ap-guangzhou
```

### 查看异常事件原因分布

```bash
tccli cls SearchLog --TopicId topic-xxxxxxxx \
  --From 1749600000000 \
  --To 1749686400000 \
  --Query 'type:"Warning" | SELECT reason, count(*) AS cnt GROUP BY reason ORDER BY cnt DESC' \
  --region ap-guangzhou
```

### 查看 Pod OOM 事件

```bash
tccli cls SearchLog --TopicId topic-xxxxxxxx \
  --From 1749600000000 \
  --To 1749686400000 \
  --Query 'reason:"OOMKilling"' \
  --region ap-guangzhou
```

### 异常事件聚合检索 — 按 reason 和 object 分布趋势

```bash
tccli cls SearchLog --TopicId topic-xxxxxxxx \
  --From 1749600000000 \
  --To 1749686400000 \
  --Query 'type:"Warning" | SELECT reason, involvedObject.kind AS objectKind, count(*) AS cnt GROUP BY reason, objectKind ORDER BY cnt DESC' \
  --region ap-guangzhou
```

## 验证

### Control plane (tccli)

```bash
tccli tke DescribeLogSwitches --ClusterIds '["cls-xxxxxxxx"]' --region ap-guangzhou
```

```json
{
  "SwitchSet": [],
  "Audit": "<Audit>",
  "Enable": true,
  "ErrorMsg": "<ErrorMsg>",
  "LogsetId": "<LogsetId>",
  "Status": "<Status>",
  "TopicId": "<TopicId>",
  "TopicRegion": "<TopicRegion>"
}
```

确认 `EventLog.Enable` 为 `true`。

## 清理

### Control plane (tccli)

```bash
tccli tke DisableEventPersistence --ClusterId cls-xxxxxxxx --region ap-guangzhou
```

仪表盘随事件日志关闭而不可用。

## 排障

| 现象 | 处理 |
|------|------|
| 仪表盘无数据 | 确认已开启事件持久化功能；事件日志投递有短暂延迟 |
| 事件总览无节点异常数据 | 检查集群节点状态是否正常 |
| 告警未触发 | 需在 CLS 控制台配置告警策略，在仪表盘右上角单击"添加到监控告警" |
| kubectl 不可达 | 托管集群公网端点可能受 CAM 策略限制（strategyId:240463971）；内网无 VPN 时不可达；事件仪表盘通过 CLS SearchLog API 检索，不依赖 kubectl |

## 下一步

- [事件日志](../事件日志/tccli 操作.md) — 开启/关闭事件持久化
- [审计仪表盘](../../审计管理/审计仪表盘/tccli 操作.md) — 审计日志可视化仪表盘

## 控制台替代

登录 [容器服务控制台](https://console.cloud.tencent.com/tke2) > 选择集群 > 日志 > 事件日志 > 各仪表盘页签（事件总览/异常事件聚合检索/全局检索）。
