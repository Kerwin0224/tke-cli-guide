---
doc_type: Overview
---
# 可观测性

> 集群的监控、告警、日志。决定如何发现集群问题、排查故障。
>
> 官方文档：[容器服务可观测体系概述](https://cloud.tencent.com/document/product/457/118975)

## 是什么

TKE 可观测按三条能力线组织：**指标**（云监控默认集成 + TMP/Prometheus + 可选自建）、**日志**（应用日志 CLS + 事件/审计）、**链路追踪**（APM）。用外部输出（指标/日志/追踪）推断集群与应用内部状态。

> **入口**：控制台把 Prometheus 放在「云原生服务」；日志采集/查看以**集群详情**侧栏为主（运维中心多为跨集群开关，不是日志正文入口）。下文以 `tccli` 可调的 Prometheus/日志 Action 为主；装默认 `monitoragent` 见 [插件](../addons/index.md)。

## 何时阅读
- 你要为集群创建 Prometheus 实例并关联采集 Agent — 用 `RunPrometheusInstance`，看 [Prometheus 监控入门](prometheus.md)
- 你要配置告警策略与通知规则 — 看 [Prometheus 告警配置](prometheus-alerting.md)
- 你要开启控制面日志或采集业务日志 — 用 `EnableControlPlaneLogs`，看 [日志采集](logging.md)
- 你要同步 Prometheus 模板或配置记录规则 — 看 [Prometheus 配置与模板](prometheus-config.md)
- 你遇到监控无数据或日志缺失 — 看 [故障排查](../troubleshooting.md)

## 核心概念

| 概念 | 含义 | 为什么重要 |
|:-----|:-----|:-----|
| 指标可观测 | 数值型 Metrics：资源利用率、调度、控制面健康等 | 量化集群与应用状态；默认云监控 + 可选 TMP |
| 日志可观测 | 应用日志、K8s 事件、审计等文本流 | 排障与审计的证据链；常落 CLS |
| 链路追踪 | 分布式调用链（APM） | 定位跨服务延迟与故障根因 |
| Prometheus / TMP | 托管或自建 Prometheus；TKE 侧大量 `*Prometheus*` Action | 本目录操作文档主路径；须 CAM 单独授权 |
| ClusterAgent | 集群内采集代理 | 把集群指标上报到 Prometheus 实例 |
| 告警策略 | 聚合告警规则 | 按策略触发告警 |
| CLS 日志 | 业务日志投递到 CLS | 业务日志集中检索 |
| 控制面日志 | Master 组件日志 | 排查控制面问题 |

## Prometheus 文档分层

TKE 的 Prometheus 相关功能（48 个 Action）分四层，每层独立文档：

| 层 | 作用 | 文档 |
|:---|:-----|:-----|
| 入口 | 实例创建 + Agent 关联 + 目标查询 | [Prometheus 监控入门](prometheus.md) |
| 告警 | 告警策略 + 规则 + 通知 | [Prometheus 告警配置](prometheus-alerting.md) |
| 配置 | 模板同步 + 记录规则 + Dashboard | [Prometheus 配置与模板](prometheus-config.md) |
| Agent | Agent 安装/卸载/采集目标 | [Prometheus Agent 管理](prometheus-agent.md) |

> Prometheus 是 TKE 2018-05-25 旧版独有功能（2022-05-01 新版无），命令须带 `--version 2018-05-25`。

## 日志体系

| 日志类型 | 采集方式 | 存储 | 文档 |
|:---------|:---------|:-----|:-----|
| 业务日志 | CLS Agent 采集 Pod 日志 | CLS | [日志采集](logging.md) |
| 控制面日志 | `EnableControlPlaneLogs` 开启 | CLS | [日志采集](logging.md) |
| 审计日志 | `EnableClusterAudit` 开启 | CLS | [审计日志](../security/audit.md) |

## 三条能力线（对照）

| 能力线 | 默认/常见路径 | 相关文档 |
|:-------|:--------------|:-----------|
| 指标 | 新建集群默认集成云监控；进阶用 TMP（`RunPrometheusInstance` + Agent）或自建 Prometheus | [Prometheus 入门](prometheus.md) 及同目录告警/配置/Agent |
| 日志 | CLS 采集容器日志；事件见事件日志专文；审计见 [审计日志](../security/audit.md) | [日志采集](logging.md) · [审计日志](../security/audit.md) |
| 链路 | 腾讯云 APM（探针采集调用链） | 控制台/APM 产品文档；不走本目录 `*Prometheus*` Action |

## 不适用场景

- 测试集群、不需指标告警 → 跳过 TMP/Prometheus 实例创建（云监控默认仍可能已装）
- 只要控制台基础监控曲线 → 用云监控即可；本目录 Prometheus 操作文档面向 TMP/自建路径
- 只要调用链、不要 Prometheus → 走 APM，不调用本目录 `*Prometheus*` Action
- 不需业务日志检索 → 跳过日志采集

## 快速检查

```bash
# 查看集群监控状态
tccli tke DescribeClusterStatus --region <REGION> --filter "ClusterStatusSet[?ClusterId=='<CLUSTER_ID>'] | [0].ClusterBMonitor"
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
