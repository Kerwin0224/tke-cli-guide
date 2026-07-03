---
doc_type: Overview
---
# TKE + TCR: tccli 文档

> 腾讯云容器服务 (TKE) 和容器镜像服务 (TCR) 的 tccli 命令行操作指南。
> 零基础知识即可上手，每条命令可复制执行。

## 是什么

本指南覆盖用 `tccli` 命令行管理 TKE 集群和 TCR 镜像仓库的**全部操作**。

## 触发条件

- 你要用**命令行**（而非控制台/Terraform）管理腾讯云 TKE 集群或 TCR 镜像仓库 — 本指南是入口
- 你已在终端装好 tccli 并配好凭证，想找一个具体操作的**可复制命令** — 直接看下方快速导航
- 你是 agent，需要一条 `tccli (tke|tcr) <Action>` 命令的权威写法与可执行验证 — 每篇操作文档都给命令+`# expected:`+故障恢复

## 准备工作

第一次使用？先完成以下两步，否则任何 `tccli` 命令都会因无凭证失败：

| 步骤 | 去看 |
|---------|------|
| 安装 tccli | [安装 tccli](getting-started/install.md) |
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

并非所有 `tccli tke` / `tccli tcr` 的 Action 都在本指南范围内。以下 16 个 Action 经评估**有意排除**（非文档缺失）——遇到它们时不要在本指南查找，直接去对应去向：

| 排除域 | Action | 排除理由 | 去向 |
|:-------|:-------|:---------|:-----|
| TKE 预留实例（计费优惠，6 个） | `CreateReservedInstances` / `DeleteReservedInstances` / `DescribeReservedInstances` / `DescribeReservedInstanceUtilizationRate` / `ModifyReservedInstanceScope` / `RenewReservedInstances` | 包年包月资源计费管理，属计费域，非容器运维；用户不会用 tccli 管预留实例。注：预留实例的**查询用量**操作 `DescribeRIUtilizationDetail` 不在此排除域，已纳入 [配额参考](tke/reference/quotas.md)（与配额/用量查询同属成本核算任务） | [预留实例计费文档](https://cloud.tencent.com/document/product/457) |
| TKE 边缘集群 addon 转发（产品下线，1 个） | `ForwardTKEEdgeApplicationRequestV3` | TKE Edge 产品已下线，该接口全域不可用（6 个地域均报 `UnsupportedRegion`/`InvalidParameter: unsupported operation`），与 官方标记为已下线 | Edge 已下线，无替代；如需边缘计算见 [EKS 文档](tke/specialized/eks-cluster.md) |
| TCR AI 模型（Beta，4 个） | `ListAIModels` / `ListAIModelVersions` / `DescribeAIModelVersionDetail` / `DeleteAIModel` | Beta 特性，尚未稳定；用户不会主动搜索"AI 模型" | 待稳定后纳入，暂见 [TCR 产品文档](https://cloud.tencent.com/document/product/1141) |
| TCR Skill（Beta，5 个） | `ListSkills` / `ListSkillVersions` / `DescribeSkillDetail` / `DescribeSkillDownloadInfo` / `DeleteSkill` | Beta 特性，尚未稳定 | 待稳定后纳入，暂见 [TCR 产品文档](https://cloud.tencent.com/document/product/1141) |

> 本指南覆盖上述排除域之外的**全部 TKE + TCR tccli 操作**——非排除 Action **命令覆盖率 100%**：每个 Action 在文档里都有可复制运行的 `tccli (tke|tcr) <Action>` 命令行。若你使用的 Action 既不在本指南、也不在上表，请通过 [GitHub 反馈](https://github.com/) 提交。
>
> **覆盖口径（可复现）**：覆盖率按命令在场计——非排除 Action 须在文档 fenced 代码块内出现字面 `tccli (tke|tcr) <Action>` 调用。校验方法：以腾讯云 API 的 Action 清单为基准，逐个核对文档是否给出对应命令。Action→文档归属的追溯由机器可读索引 `tools/data/action-to-doc.json` 承担，文档正文不保留机械的全量 Action 清单表。

## 快速检查

> 前置：已完成 [配置凭证](getting-started/credentials.md)。若未配置，此命令会返回 `AuthFailure`。

```bash
tccli tke DescribeClusters --region ap-guangzhou --Limit 1
# expected: { "TotalCount": ..., "Clusters": [...] }（tccli 默认剥离 Response 包装层）
```
