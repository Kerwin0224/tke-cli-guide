---
doc_type: Overview
---
# 集群插件

> TKE 集群插件管理——扩展集群功能的组件，如 CBS CSI、Ingress Controller、eniipamd。
>
> 官方文档：[组件与应用概述](https://cloud.tencent.com/document/product/457/81234) · [VPC-CNI（eniipamd）组件变更记录](https://cloud.tencent.com/document/product/457/64920)

## 是什么

插件是封装好的 Kubernetes 组件，一键安装到集群。TKE 提供官方插件列表，每个插件有版本与配置。

## 触发条件

- 你要给集群安装/更新/卸载组件（存储 `cbs`、监控 `monitoragent`、网络 `ip-masq-agent`/`eniipamd` 等）— 去 [插件管理](manage.md)（`InstallAddon`/`UpdateAddon`/`DeleteAddon`）
- 你要查某插件当前状态/版本/配置（如 `eniipamd` 是否 `Phase=Succeeded`）— 用 `DescribeAddon`，见 [快速检查](#快速检查)
- 你不确定当前集群有哪些可用插件及版本 — 用 `GetTkeAppChartList`/`DescribeAddonValues` 核对，见 [常见插件](#常见插件)
- 你要装的不是官方插件而是自定义应用 — 插件本质是 Helm Release，去 [应用发布](../releases/manage.md)
- 插件 `Phase` 异常（非 `Succeeded`）— 看 [故障排查](../troubleshooting.md)

## 核心概念

| 概念 | 含义 | 为什么重要 |
|:-----|:-----|:-----|
| Addon | 集群插件（封装的 Helm Chart） | 扩展集群功能 |
| AddonName | 插件名（如 `eniipamd`） | 安装/查询的标识 |
| AddonVersion | 插件版本 | 决定功能与兼容性 |
| Phase | 插件状态 | 判断插件是否正常运行 |
| RawValues | 插件配置（base64 编码的 JSON） | 自定义插件行为 |

## 常见插件

> 创建确认步**默认会装**：`cbs`（存储）+ `monitoragent`（监控）+ `ip-masq-agent`（网络）。其余按需 `InstallAddon`。控制台增强组件按类分桶（监控/镜像/DNS/调度/网络/GPU/安全/认证授权/其他等），合计 **30+**——下表是高频入口，**不是全集**；安装前用 `GetTkeAppChartList` / `DescribeAddonValues` 核对当前集群可用的 `AddonName` 与版本。

### 按意图分桶（高频）

| 意图 | 典型 AddonName | 作用 |
|:-----|:---------------|:-----|
| 存储 | `cbs`（部分 chart 为 `cbs-csi`） | CBS 云硬盘 |
| 监控 | `monitoragent` | 节点/集群监控 Agent |
| 网络 | `ip-masq-agent`、`eniipamd` | 伪装/VPC-CNI 弹性网卡 |
| 镜像 | `tcr` | 集群侧 TCR 拉取凭证 |
| 日志 | `tke-log-agent` | 日志采集 Agent |
| 调度/弹性 | `cluster-autoscaler` | 自动扩缩容 |
| 安全/策略 | `gatekeeper` | OPA 策略引擎 |
| 巡检 | `kubejarvisservice` | 集群巡检 |

> 插件本质是 Helm Release，用 `DescribeClusterReleases` 也能查到。见 [应用发布](../releases/manage.md)。Prometheus 监控在控制台属「云原生服务」入口，文档见 [可观测](../observability/index.md)，不要只在插件列表里找。

## 不适用场景

- 不需扩展集群功能 → 跳过插件
- 用 Helm 直接装应用 → 看 [应用发布](../releases/manage.md)
- 需要非官方插件 → 用 Helm Release 直接部署

## 快速检查

```bash
# 查看某插件状态（eniipamd Phase=Succeeded）
tccli tke DescribeAddon --region <REGION> --ClusterId "<CLUSTER_ID>" --AddonName eniipamd \
  --filter "Addons[].{name:AddonName,ver:AddonVersion,phase:Phase}"
# expected: 插件信息，Phase 含 Succeeded
```

## 文档

- [插件管理](manage.md) — 安装/更新/卸载插件
- [应用发布](../releases/manage.md) — Helm Release（插件本质）
- [故障排查](../troubleshooting.md) — 插件 Phase 异常诊断
