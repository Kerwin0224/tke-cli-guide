---
doc_type: Overview
---
# 可观测性

> 集群的监控、告警、日志。决定你怎么发现集群问题、排查故障。

## 这是什么

TKE 可观测分三块：Prometheus 监控（指标 + 告警）、日志采集（业务 + 控制面）、事件管理。覆盖"集群有没有出问题"和"出了什么问题"。

## 核心概念

| 概念 | 含义 | 为什么重要 |
|:-----|:-----|:-----|
| Prometheus | 指标采集与告警系统 | 监控集群与应用指标 |
| Prometheus 实例 | 独立的 Prometheus 服务 | 承载采集目标与告警规则 |
| ClusterAgent | 集群内的采集代理 | 把集群指标上报到 Prometheus 实例 |
| 告警策略 | 聚合告警规则 | 按策略触发告警 |
| CLS 日志 | 业务日志投递到 CLS | 业务日志集中检索 |
| 控制面日志 | Master 组件日志 | 排查控制面问题 |

## Prometheus 四层架构

TKE 的 Prometheus 相关功能（48 个 Action）分四层，每层独立文档：

| 层 | 作用 | 文档 |
|:---|:-----|:-----|
| L1 入口 | 实例创建 + Agent 关联 + 目标查询 | [Prometheus 监控入门](prometheus.md) |
| L2 告警 | 告警策略 + 规则 + 通知 | [Prometheus 告警配置](prometheus-alerting.md) |
| L2 配置 | 模板同步 + 记录规则 + Dashboard | [Prometheus 配置与模板](prometheus-config.md) |
| L2 Agent | Agent 安装/卸载/采集目标 | [Prometheus Agent 管理](prometheus-agent.md) |

> Prometheus 是 TKE 2018-05-25 旧版独有功能（2022-05-01 新版无），命令须带 `--version 2018-05-25`。

## 日志体系

| 日志类型 | 采集方式 | 存储 | 文档 |
|:---------|:---------|:-----|:-----|
| 业务日志 | CLS Agent 采集 Pod 日志 | CLS | [日志采集](logging.md) |
| 控制面日志 | `EnableControlPlaneLogs` 开启 | CLS | [日志采集](logging.md) |
| 审计日志 | `EnableClusterAudit` 开启 | CLS | [审计日志](../security/audit.md) |

## 不适用场景

- 不需监控（测试集群）→ 跳过 Prometheus
- 已用云监控 → 看 [Prometheus 入门](prometheus.md) 对比
- 不需业务日志检索 → 跳过日志采集

## 快速检查

```bash
# 查看集群监控状态
tccli tke DescribeClusterStatus --region <REGION> --ClusterIds '["<CLUSTER_ID>"]' \
  --filter "ClusterStatusSet[0].ClusterBMonitor"
# expected: true（已开启监控）

# 查看 Prometheus 实例
tccli tke DescribePrometheusInstancesOverview --region <REGION> --Limit 3
# expected: Prometheus 实例列表
```

## 文档

- [Prometheus 监控入门](prometheus.md) — 实例创建 + Agent 关联
- [Prometheus 告警配置](prometheus-alerting.md) — 告警策略与规则
- [Prometheus 配置与模板](prometheus-config.md) — 模板同步与记录规则
- [Prometheus Agent 管理](prometheus-agent.md) — Agent 安装与采集目标
- [日志采集](logging.md) — 业务日志与控制面日志
- [故障排查](../troubleshooting.md) — 监控无数据、日志缺失诊断
