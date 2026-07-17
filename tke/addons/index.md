---
doc_type: Overview
---
# 集群组件与应用

> TKE 将集群扩展能力分为系统组件、增强组件、应用市场和云原生服务；不同类型的安装与生命周期边界不同。
>
> 官方文档：[组件与应用概述](https://cloud.tencent.com/document/product/457/81234) · [VPC-CNI（eniipamd）组件变更记录](https://cloud.tencent.com/document/product/457/64920)

## 是什么

- **系统组件**：集群基础能力，默认安装且无法删除。
- **增强组件**：按需安装到标准 TKE 集群，可通过 Addon API 管理受支持的组件。
- **应用市场**：基于 Helm 3.0 发布和管理应用。
- **云原生服务**：例如 Prometheus 监控服务，从对应服务入口配置。

## 触发条件

- 你要查询当前集群已安装的 addon — 用 `DescribeAddon`；不传 `AddonName` 时返回该集群的全部 addon，见 [快速检查](#快速检查)
- 你已从集群详情的组件管理页取得 `AddonName`，需要安装、更新或删除增强组件 — 去 [组件管理](manage.md)（`InstallAddon`/`UpdateAddon`/`DeleteAddon`）
- 你要查询某个已知 addon 的当前参数和 chart 默认参数 — 用 `DescribeAddonValues`
- 你要查询 TKE 支持的 App/Chart — 用 `GetTkeAppChartList`；该结果不代表某个集群当前已安装的 addon
- 你要发布自定义应用 — 去 [应用发布](../releases/manage.md)
- 你要配置 Prometheus 监控服务 — 去 [可观测](../observability/index.md)
- addon 处于 `InstallFailed` 或 `UpgradFailed` — 看 [故障排查](../troubleshooting.md)

## 核心概念

| 概念 | 含义 | 为什么重要 |
|:-----|:-----|:-----|
| Addon | Addon API 管理的集群组件 | 与系统组件、应用市场和云原生服务区分 |
| AddonName | addon 名称（如 `eniipamd`） | 安装、查询和变更的标识 |
| AddonVersion | addon 版本 | 决定功能与兼容性 |
| Phase | addon 状态 | 区分进行中、成功与失败 |
| RawValues | addon 配置（base64 编码的 JSON） | 自定义 addon 行为 |

## 状态判定

| 状态类型 | `Phase` | 处理方式 |
|:---------|:--------|:---------|
| 正常进行中 | `Installing`、`Upgrading`、`Terminating` | 等待操作完成后重新查询 |
| 成功 | `Succeeded` | 安装或升级已成功 |
| 失败 | `InstallFailed`、`UpgradFailed` | 查看 `Reason` 并进入故障排查 |

## 常见增强组件

以下名称均有 Addon API 安装或更新示例：

| 意图 | AddonName | 作用 |
|:-----|:----------|:-----|
| 存储 | `cbs` | CBS 云硬盘存储能力 |
| 网络 | `eniipamd` | VPC-CNI 网络能力 |
| 镜像 | `tcr` | 集群侧 TCR 拉取凭证 |
| 日志 | `tke-log-agent` | 日志采集 |
| 调度/弹性 | `cluster-autoscaler` | 自动扩缩容 |

> `monitoragent`、`ip-masq-agent` 属于系统组件，不应进入 `DeleteAddon` 删除流程。具体增强组件名称从集群详情的组件管理页获取；已知名称的参数可用 `DescribeAddonValues` 查询。

## 快速检查 {#快速检查}

```bash
# 查看当前集群已安装的全部 addon
tccli tke DescribeAddon --region <REGION> --ClusterId "<CLUSTER_ID>" \
  --filter "Addons[].{name:AddonName,ver:AddonVersion,phase:Phase,reason:Reason}"
# expected: 返回 Addons；各项含名称、版本、Phase 和 Reason
```

```bash
# 按已知名称查看 addon
tccli tke DescribeAddon --region <REGION> --ClusterId "<CLUSTER_ID>" --AddonName eniipamd \
  --filter "Addons[].{name:AddonName,ver:AddonVersion,phase:Phase,reason:Reason}"
# expected: 已安装时返回 eniipamd 的信息
```

## 不适用场景

- 系统组件的删除 — 系统组件默认安装且无法删除
- 自定义应用发布 — 使用 [应用发布](../releases/manage.md)
- Prometheus 监控服务配置 — 使用 [可观测](../observability/index.md)

## 文档

- [组件管理](manage.md) — 安装、更新或删除 Addon API 支持的增强组件
- [应用发布](../releases/manage.md) — 发布和管理 Helm 应用
- [可观测](../observability/index.md) — 配置 Prometheus 等可观测能力
- [故障排查](../troubleshooting.md) — addon 失败状态诊断
