---
doc_type: Overview
---
# TKE + TCR: TCCLI 文档

> 腾讯云容器服务 (TKE) 与容器镜像服务 (TCR) 的 `tccli` 命令行操作指南。  
> 命令可复制执行。人类按页操作；Agent 应先检索本手册再调用接口。问题请反馈至文档源仓库。

> **MCP 与 Agent skill 接入**  
> 手册 MCP、Agent skill 的安装与配置说明见源仓库首页：  
> https://github.com/Kerwin0224/tke-cli-guide

## 使用纪律（Agent）

完成任务前遵守下列纪律。完成标准：能指出将使用的 **page_url / Action**，或已判定手册缺失并准备反馈。

1. **先手册，再 `tccli`** — 通过文档站 MCP（`askQuestion` / `searchDocumentation` / `getPage` 等）定位页面；取得 `page_url` 后再调用命令。  
2. **调用范式** — 查询默认使用 `--filter` 与 `--output text`；复杂入参采用 `skeleton` → `--cli-input-json`；长任务使用 `--waiter`；多 Agent 场景使用 `--profile` / `--request-client`。详细路径见 skill `tencentcloud-tccli-skill`。  
3. **有问题则反馈** — 手册错误或缺失、工具异常、流程建议，按 skill `tccli-agent-sop` 向 **本仓库**（`Kerwin0224/tke-cli-guide`）提交 Issue。纯文档问题且无需追踪时，可使用 `sendFeedback`。  
4. **响应以实际返回为准** — 成功时读 stdout 业务字段；失败时读 **stderr** 的 `code:`/`message:`（勿只在 stdout 找 `Error`）；不预写未验证字段。

## 准备工作

首次使用前请完成下列步骤，否则任何 `tccli` 命令会因缺少凭证而失败：

| 步骤 | 说明 |
|---------|------|
| 安装 TCCLI | [安装 TCCLI](getting-started/install.md) |
| 保持 TCCLI 最新 | `uv tool upgrade tccli`（旧版可能缺少新接口或字段） |
| 配置 CAM 凭证 | [配置凭证](getting-started/credentials.md) |
| 术语说明 | [术语表](getting-started/glossary.md) |
| Agent：flag 组合范式 | [附录 · Agent 优化](appendix/agent-optimization.md) |

## 快速导航

| 目标 | 文档 |
|---------|------|
| 5 分钟创建集群 | [TKE 快速入门](quickstart/tke-first-cluster.md) |
| 5 分钟推送第一个镜像 | [TCR 快速入门](quickstart/tcr-first-registry.md) |
| 查看全部 TKE 操作 | [TKE 文档](tke/index.md) |
| 查看全部 TCR 操作 | [TCR 文档](tcr/index.md) |
| 理解 TKE 双 API 版本 | [TKE 概览 · API 版本选择](tke/index.md#api-版本选择) |
| 了解 Agent 优化 | [附录](appendix/agent-optimization.md) |

## 何时不使用本指南

- 通过腾讯云控制台操作（非命令行）→ [控制台](https://console.cloud.tencent.com)
- 使用 Terraform / Pulumi 等 IaC 工具 → 查阅对应 Provider 文档
- 仅需 Kubernetes 原生 `kubectl` 操作 → [Kubernetes 文档](https://kubernetes.io/docs/)

## 本指南不覆盖的操作 {#本指南不覆盖哪些操作}

并非全部 `tccli tke` / `tccli tcr` Action 均纳入本指南。下列 Action 不在覆盖范围，请按去向处理：

| 类别 | Action | 去向 |
|:-----|:-------|:-----|
| TKE 预留券/规模价格（计费域，7 个） | `CreateReservedInstances` / `DeleteReservedInstances` / `DescribeReservedInstances` / `DescribeReservedInstanceUtilizationRate` / `ModifyReservedInstanceScope` / `RenewReservedInstances` / `GetClusterLevelPrice` | [预留实例计费文档](https://cloud.tencent.com/document/product/457) |
| TKE 计费用量（6 个） | `DescribeRIUtilizationDetail` / `DescribePodChargeInfo` / `DescribePodDeductionRate` / `DescribePodsBySpec` / `DescribePostNodeResources` / `DescribeResourceUsage` | [配额参考](tke/reference/quotas.md) |
| TKE Edge（已下线，5 个） | `DescribeAvailableTKEEdgeVersion` / `DescribeEdgeAvailableExtraArgs` / `DescribeTKEEdgeExternalKubeconfig` / `DescribeTKEEdgeScript` / `ForwardTKEEdgeApplicationRequestV3` | Edge 已下线；边缘场景见 [EKS](tke/specialized/eks-cluster.md) / [注册节点](tke/nodes/registered-nodes/overview.md) |
| **超出 TCCLI 范围（控制台/独立产品面，无 tccli Action，9 项）** | 成本洞察 / Node Map / Workload Map / 服务网格 / 云原生 etcd / 备份中心 / 节点迁移 / EIS 推理三件套 / 弹性推理 / 交付流水线 | 由控制台或独立产品提供，非 TCCLI 可达。其中：服务网格=[TCM](https://cloud.tencent.com/document/product/1251)；云原生 etcd=[云原生 etcd](https://cloud.tencent.com/document/product/457/58176)；EIS 推理=[TI-ONE/EIS](https://cloud.tencent.com/document/product/851) |

> 本指南覆盖上表之外的 TKE + TCR TCCLI 操作。若所用 Action 既不在本指南、也不在上表，请通过 **本仓库 Issues**（推荐经 `tccli-agent-sop`）反馈。
>
> **调用边界**：① 部分账号的 CAM 策略可能拒绝 `tke:CreateCluster` 等写操作——以 `help --detail` 核对入参；失败时以 stderr 的 `code:`/`message:` 为准；响应字段以实际 stdout 为准，不预写未验证字段。② 个别 Action 在命令行展开参数时可能解析失败，请改用 `--cli-input-json file://` 传参。

## 快速检查

> 前置条件：已完成 [配置凭证](getting-started/credentials.md)。未配置时，下列命令将返回 `AuthFailure`。

```bash
tccli tke DescribeClusters --region ap-guangzhou --Limit 1
# expected: { "TotalCount": ..., "Clusters": [...] }（tccli 默认剥离 Response 包装层）
```
