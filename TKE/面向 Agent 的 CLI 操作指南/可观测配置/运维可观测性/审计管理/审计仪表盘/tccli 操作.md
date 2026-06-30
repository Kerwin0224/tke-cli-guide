# 审计仪表盘（tccli）

> 对照官方：[审计仪表盘](https://cloud.tencent.com/document/product/457/50510) · page_id `50510`

## 概述

容器服务 TKE 在开启集群审计功能后，自动为集群配置五类审计仪表盘：审计总览、节点操作概览、K8S 对象操作概览、聚合检索、全局检索。仪表盘基于 CLS 日志服务，支持用户自定义过滤项，并内置全局检索和告警配置能力。

## 前置条件

- `tccli` 已配置凭证
- kubectl 不可达：演示集群 cls-xxxxxxxx 为托管集群，公网端点可能受 CAM 策略限制（strategyId:240463971）；内网无 VPN 时不可达。以下 kubectl 命令作为文档参考。
- 已开启集群审计功能（`tccli tke EnableClusterAudit`）
- 审计日志已投递到 CLS

```bash
tccli tke DescribeLogSwitches --ClusterIds '["cls-xxxxxxxx"]' --region ap-guangzhou
```

```json
{
  "RequestId": "b8c9d0e1-f2a3-4567-8901-234567890123",
  "SwitchSet": [
    {
      "ClusterId": "cls-xxxxxxxx",
      "AuditLog": {
        "Enable": true,
        "LogsetId": "logset-xxxxxxxx",
        "TopicId": "topic-xxxxxxxx"
      }
    }
  ]
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 开启审计日志（前提） | `tccli tke EnableClusterAudit` | 是 |
| 查看审计日志状态 | `tccli tke DescribeLogSwitches` | 是 |
| 在 CLS 检索审计日志 | CLS 检索 API（`cls:SearchLog`） | 是 |
| 配置告警 | CLS 告警策略 API（`cls:CreateAlarm`） | 是 |

## 操作步骤

### 开启审计功能（前提）

```bash
tccli tke EnableClusterAudit --ClusterId cls-xxxxxxxx \
  --LogsetId logset-xxxxxxxx \
  --TopicId topic-xxxxxxxx \
  --region ap-guangzhou
```

### 审计总览 — 查看核心审计日志统计

审计总览展示核心审计日志的统计数、分布情况、重要操作趋势。可通过 CLS 检索 API 获取：

```bash
tccli cls SearchLog --TopicId topic-xxxxxxxx \
  --From 1749600000000 \
  --To 1749686400000 \
  --Query "* | SELECT count(*) AS total" \
  --region ap-guangzhou
```

### 节点操作概览 — 查看节点相关审计

Kubernetes 审计日志记录节点 create、delete、patch、update、封锁、驱逐等操作：

```bash
tccli cls SearchLog --TopicId topic-xxxxxxxx \
  --From 1749600000000 \
  --To 1749686400000 \
  --Query 'objectRef.resource:"nodes"' \
  --region ap-guangzhou
```

### K8S 对象操作概览 — 查看工作负载审计

```bash
tccli cls SearchLog --TopicId topic-xxxxxxxx \
  --From 1749600000000 \
  --To 1749686400000 \
  --Query 'objectRef.resource:"deployments"' \
  --region ap-guangzhou
```

### 聚合检索 — 按维度聚合分布

按操作人聚合统计：

```bash
tccli cls SearchLog --TopicId topic-xxxxxxxx \
  --From 1749600000000 \
  --To 1749686400000 \
  --Query "* | SELECT user.username, count(*) AS cnt GROUP BY user.username ORDER BY cnt DESC" \
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

确认 `AuditLog.Enable` 为 `true`。

## 清理

### Control plane (tccli)

```bash
tccli tke DisableClusterAudit --ClusterId cls-xxxxxxxx --region ap-guangzhou
```

仪表盘随审计功能关闭而不可用，日志数据保留在 CLS 中。

## 排障

| 现象 | 处理 |
|------|------|
| 仪表盘无数据 | 确认已开启审计功能；审计日志投递有短暂延迟 |
| 特定操作未记录 | 参考 TKE 审计日志策略：get/list/watch 记录 Request 级别；某些系统组件操作不记录 |
| 告警未触发 | 需在 CLS 控制台配置告警策略：监控告警 > 告警策略 > 新建 |
| kubectl 不可达 | 托管集群公网端点可能受 CAM 策略限制（strategyId:240463971）；内网无 VPN 时不可达；审计仪表盘通过 CLS SearchLog API 检索，不依赖 kubectl |

## 下一步

- [审计日志](../审计日志/tccli 操作.md) — 开启/关闭集群审计日志
- [事件日志](../../事件管理/事件日志/tccli 操作.md) — Kubernetes 事件持久化

## 控制台替代

登录 [容器服务控制台](https://console.cloud.tencent.com/tke2) > 选择集群 > 日志 > 审计日志 > 各仪表盘页签（审计总览/节点操作/K8S对象/聚合检索/全局检索）。
