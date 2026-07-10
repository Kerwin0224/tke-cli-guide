---
doc_type: Overview
---
# TKE + TCR: TCCLI 文档

> 腾讯云容器服务 (TKE) 和容器镜像服务 (TCR) 的 TCCLI 命令行操作指南。
> 零基础知识即可上手，每条命令可复制执行。

## 是什么

本指南覆盖用 `tccli` 命令行管理 TKE 集群和 TCR 镜像仓库的**全部操作**。

## 触发条件

- 你要用**命令行**（而非控制台/Terraform）管理腾讯云 TKE 集群或 TCR 镜像仓库 — 本指南是入口
- 你已在终端装好 TCCLI 并配好凭证，想找一个具体操作的**可复制命令** — 直接看下方快速导航
- 你是 agent，需要一条 `tccli (tke|tcr) <Action>` 命令的权威写法与可执行验证 — 每篇操作文档都给命令+`# expected:`+故障恢复

## 准备工作

第一次使用？先完成以下两步，否则任何 `tccli` 命令都会因无凭证失败：

| 步骤 | 去看 |
|---------|------|
| 安装 TCCLI | [安装 TCCLI](getting-started/install.md) |
| 保持 TCCLI 最新 | `uv tool upgrade tccli`（旧版可能缺新接口/字段） |
| 配置 CAM 凭证 | [配置凭证](getting-started/credentials.md) |
| 不认识的术语 | [术语表](getting-started/glossary.md) |

## 快速导航

| 我想... | 去看 |
|---------|------|
| 5 分钟创建一个集群 | [TKE 快速入门](quickstart/tke-first-cluster.md) |
| 5 分钟推送第一个镜像 | [TCR 快速入门](quickstart/tcr-first-registry.md) |
| 查看所有 TKE 操作 | [TKE 文档](tke/index.md) |
| 查看所有 TCR 操作 | [TCR 文档](tcr/index.md) |
| 理解 TKE 双 API 版本 | [TKE 概览 - API 版本选择](tke/index.md#api-版本选择) |
| 了解 Agent 优化 | [附录](appendix/agent-optimization.md) |

## 何时不用这套文档？

- 你用的是腾讯云控制台（非命令行）→ [控制台文档](https://console.cloud.tencent.com)
- 你用的是 Terraform / Pulumi → 去看对应的 Provider 文档
- 你需要的是 K8s 原生 kubectl 操作 → [Kubernetes 文档](https://kubernetes.io/docs/)

## 本指南不覆盖哪些操作？

并非所有 `tccli tke` / `tccli tcr` 的 Action 都在本指南范围内。以下 **27** 个 Action 经评估**有意排除**（非文档缺失；与 `tools/data/semantic-clustering-audit.json` 排除域同源）——遇到它们时不要在本指南查找 runbook，直接去对应去向：

| 排除域 | Action | 排除理由 | 去向 |
|:-------|:-------|:---------|:-----|
| TKE 预留券/规模价格（计费域，7 个） | `CreateReservedInstances` / `DeleteReservedInstances` / `DescribeReservedInstances` / `DescribeReservedInstanceUtilizationRate` / `ModifyReservedInstanceScope` / `RenewReservedInstances` / `GetClusterLevelPrice` | 包年包月资源计费管理，属计费域，非容器运维 | [预留实例计费文档](https://cloud.tencent.com/document/product/457) |
| TKE 计费用量（6 个） | `DescribeRIUtilizationDetail` / `DescribePodChargeInfo` / `DescribePodDeductionRate` / `DescribePodsBySpec` / `DescribePostNodeResources` / `DescribeResourceUsage` | 用量/抵扣查询；不写 runbook，仅作参考 | [配额参考](tke/reference/quotas.md) |
| TKE Edge 下线（5 个） | `DescribeAvailableTKEEdgeVersion` / `DescribeEdgeAvailableExtraArgs` / `DescribeTKEEdgeExternalKubeconfig` / `DescribeTKEEdgeScript` / `ForwardTKEEdgeApplicationRequestV3` | 4 个 `status=deprecated`；`ForwardTKEEdgeApplicationRequestV3` 真机全域 `UnsupportedRegion` | Edge 已下线；边缘场景见 [EKS](tke/specialized/eks-cluster.md) / [注册节点](tke/nodes/external-nodes.md) |
| TCR AI 模型（Beta，4 个） | `ListAIModels` / `ListAIModelVersions` / `DescribeAIModelVersionDetail` / `DeleteAIModel` | Beta，尚未稳定 | [TCR 产品文档](https://cloud.tencent.com/document/product/1141) |
| TCR Skill（Beta，5 个） | `ListSkills` / `ListSkillVersions` / `DescribeSkillDetail` / `DescribeSkillDownloadInfo` / `DeleteSkill` | Beta，尚未稳定 | [TCR 产品文档](https://cloud.tencent.com/document/product/1141) |

> 本指南覆盖上述排除域之外的**全部 TKE + TCR TCCLI 操作**。若你使用的 Action 既不在本指南、也不在上表，请通过 [GitHub 反馈](https://github.com/) 提交。
>
> **覆盖口径（可复现）**：覆盖率按命令在场计——非排除 Action 须在文档 fenced 代码块内出现字面 `tccli (tke|tcr) <Action>` 调用。概念归属见 `tools/data/semantic-clustering-audit.json`；命令覆盖见 `command/COVERAGE.md`。
>
> **覆盖规模（tccli 3.1.124.1）**：TKE + TCR 非排除 Action 在 docs 字面 command-presence **全覆盖**（TKE 并集 275 + TCR 115；排除计费/用量/Edge 下线/AI·Skill 共 27，见上表）。清单见 `command/COVERAGE.md`。
>
> **调用边界**：① 部分账号 CAM 可能拒绝 `tke:CreateCluster` 等写操作——以 `help --detail` 核入参，并以真机返回的 `Error.Code` 为准；响应字段以实际响应为准，不预写未验证字段。② 个别 Action 命令行展开参数可能解析失败，改用 `--cli-input-json file://` 传参。

## 快速检查

> 前置：已完成 [配置凭证](getting-started/credentials.md)。若未配置，此命令会返回 `AuthFailure`。

```bash
tccli tke DescribeClusters --region ap-guangzhou --Limit 1
# expected: { "TotalCount": ..., "Clusters": [...] }（tccli 默认剥离 Response 包装层）
```
