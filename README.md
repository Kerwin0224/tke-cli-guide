---
doc_type: Overview
---
# TKE + TCR: TCCLI 文档

> 腾讯云容器服务 (TKE) 和容器镜像服务 (TCR) 的 `tccli` 命令行操作指南。  
> 命令可复制执行。人读命令；Agent 先查手册再调用；问题反馈到文档源仓库。

> 尚未接入手册 MCP 或 Agent skill？  
> 见：https://github.com/Kerwin0224/tke-cli-guide  
> （安装步骤只在源仓 GitHub 首页说明，本页不重复。）

## 使用纪律（Agent）

完成任务前遵守下列纪律。完成标准：能指出将用的 **page_url / Action**，或已判定手册缺失并准备反馈。

1. **先手册，再 `tccli`** — 用文档站 MCP（`askQuestion` / `searchDocumentation` / `getPage` 等）定向；有 `page_url` 再调命令。  
2. **调用范式** — 查询默认 `--filter` + `--output text`；复杂入参 `skeleton` → `--cli-input-json`；长任务 `--waiter`；多 agent 用 `--profile` / `--request-client`。深路径见 skill `tencentcloud-tccli-skill`。  
3. **有问题则反馈** — 手册错/缺、工具异常、流程建议 → 按 skill `tccli-agent-sop` 向 **本仓**（`Kerwin0224/tke-cli-guide`）提 issue。纯文档、无需追踪可用 `sendFeedback`。  
4. **响应以实际返回为准** — 字段与 `Error.Code` 以实际返回为准；不预写未验证字段。

## 准备工作

第一次使用？先完成以下步骤，否则任何 `tccli` 命令都会因无凭证失败：

| 步骤 | 去看 |
|---------|------|
| 安装 TCCLI | [安装 TCCLI](getting-started/install.md) |
| 保持 TCCLI 最新 | `uv tool upgrade tccli`（旧版可能缺新接口/字段） |
| 配置 CAM 凭证 | [配置凭证](getting-started/credentials.md) |
| 不认识的术语 | [术语表](getting-started/glossary.md) |
| Agent：flag 组合范式 | [附录 · Agent 优化](appendix/agent-optimization.md) |

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

并非所有 `tccli tke` / `tccli tcr` 的 Action 都在本指南范围内。以下 Action 不在本指南——遇到它们时直接去对应去向：

| 类别 | Action | 去向 |
|:-----|:-------|:-----|
| TKE 预留券/规模价格（计费域，7 个） | `CreateReservedInstances` / `DeleteReservedInstances` / `DescribeReservedInstances` / `DescribeReservedInstanceUtilizationRate` / `ModifyReservedInstanceScope` / `RenewReservedInstances` / `GetClusterLevelPrice` | [预留实例计费文档](https://cloud.tencent.com/document/product/457) |
| TKE 计费用量（6 个） | `DescribeRIUtilizationDetail` / `DescribePodChargeInfo` / `DescribePodDeductionRate` / `DescribePodsBySpec` / `DescribePostNodeResources` / `DescribeResourceUsage` | [配额参考](tke/reference/quotas.md) |
| TKE Edge（已下线，5 个） | `DescribeAvailableTKEEdgeVersion` / `DescribeEdgeAvailableExtraArgs` / `DescribeTKEEdgeExternalKubeconfig` / `DescribeTKEEdgeScript` / `ForwardTKEEdgeApplicationRequestV3` | Edge 已下线；边缘场景见 [EKS](tke/specialized/eks-cluster.md) / [注册节点](tke/nodes/registered-nodes/overview.md) |
| **超 TCCLI 范围（控制台/独立产品面，无 tccli Action，9 项）** | 成本洞察 / Node Map / Workload Map / 服务网格 / 云原生 etcd / 备份中心 / 节点迁移 / EIS 推理三件套 / 弹性推理 / 交付流水线 | 由控制台或独立产品提供，非 TCCLI 可达。其中：服务网格=[TCM](https://cloud.tencent.com/document/product/1251)；云原生 etcd=[云原生 etcd](https://cloud.tencent.com/document/product/457/58176)；EIS 推理=[TI-ONE/EIS](https://cloud.tencent.com/document/product/851) |

> 本指南覆盖上表之外的 TKE + TCR TCCLI 操作。若你使用的 Action 既不在本指南、也不在上表，请通过 **本仓库 Issues**（推荐经 `tccli-agent-sop`）反馈。
>
> **调用边界**：① 部分账号 CAM 可能拒绝 `tke:CreateCluster` 等写操作——以 `help --detail` 核入参，以实际返回的 `Error.Code` 为准；响应字段以实际响应为准，不预写未验证字段。② 个别 Action 命令行展开参数可能解析失败，改用 `--cli-input-json file://` 传参。

## 快速检查

> 前置：已完成 [配置凭证](getting-started/credentials.md)。若未配置，此命令会返回 `AuthFailure`。

```bash
tccli tke DescribeClusters --region ap-guangzhou --Limit 1
# expected: { "TotalCount": ..., "Clusters": [...] }（tccli 默认剥离 Response 包装层）
```
