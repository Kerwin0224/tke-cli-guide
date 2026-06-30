---
doc_type: Overview
---
# TKE + TCR: tccli 文档

> 腾讯云容器服务 (TKE) 和容器镜像服务 (TCR) 的 tccli 命令行操作指南。
> 零基础知识即可上手，每条命令可复制执行。

## 这是什么？

本指南覆盖用 `tccli` 命令行管理 TKE 集群和 TCR 镜像仓库的**全部操作**。

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

并非所有 `tccli tke` / `tccli tcr` 的 Action 都在本指南范围内。以下 15 个 Action 经评估**有意排除**（非文档缺失）——遇到它们时不要在本指南查找，直接去对应去向：

| 排除域 | Action | 排除理由 | 去向 |
|:-------|:-------|:---------|:-----|
| TKE 预留实例（计费优惠，6 个） | `CreateReservedInstances` / `DeleteReservedInstances` / `DescribeReservedInstances` / `DescribeReservedInstanceUtilizationRate` / `ModifyReservedInstanceScope` / `RenewReservedInstances` | 包年包月资源计费管理，属计费域，非容器运维；用户不会用 tccli 管预留实例 | [预留实例计费文档](https://cloud.tencent.com/document/product/457) |
| TCR AI 模型（Beta，4 个） | `ListAIModels` / `ListAIModelVersions` / `DescribeAIModelVersionDetail` / `DeleteAIModel` | Beta 特性，尚未稳定；用户不会主动搜索"AI 模型" | 待稳定后纳入，暂见 [TCR 产品文档](https://cloud.tencent.com/document/product/1141) |
| TCR Skill（Beta，5 个） | `ListSkills` / `ListSkillVersions` / `DescribeSkillDetail` / `DescribeSkillDownloadInfo` / `DeleteSkill` | Beta 特性，尚未稳定 | 待稳定后纳入，暂见 [TCR 产品文档](https://cloud.tencent.com/document/product/1141) |

> 本指南覆盖上述排除域之外的**全部 TKE + TCR tccli 操作**——非排除 Action **command-presence 401/401 = 100%**：每个 Action 在文档里都有可复制运行的 `tccli (tke|tcr) <Action>` 命令行（非仅文末清单列名）。若你使用的 Action 既不在本指南、也不在上表，请通过 [GitHub 反馈](https://github.com/) 提交。
>
> **覆盖口径（可复现）**：覆盖率按命令在场计——非排除 Action 须在文档 fenced 代码块内出现字面 `tccli (tke|tcr) <Action>` 调用。校验：取 `tccli tke/tcr help` 的 Action 全集，Python 单遍扫 `docs/*.md` 的 fenced block 匹配命令调用，416 个 Action（TKE 270+22 + TCR 124）扣 15 个排除项后 401/401 命令在场。文末 `## Action 清单` 表仅作追溯（证明 Action 归属哪篇文档），不计入覆盖。

## 快速检查

```bash
tccli tke DescribeClusters --region ap-guangzhou --Limit 1
# expected: { "Response": { "TotalCount": ..., "Clusters": [...] } }
```
