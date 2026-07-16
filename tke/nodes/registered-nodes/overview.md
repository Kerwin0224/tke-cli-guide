---
doc_type: Overview
---
> 官方文档：[节点概述](https://cloud.tencent.com/document/product/457/32201) · [节点池概述](https://cloud.tencent.com/document/product/457/104096) · [注册节点价格和配额说明](https://cloud.tencent.com/document/product/457/79747)
>
> 配额：专线/公网带宽配额、注册节点数量与节点规格受账号限制。[配额说明](https://cloud.tencent.com/document/product/457/9087)
>
> ⚠️ **高危操作**：公网模式安全组策略不当致节点失联；DNS 劫持风险；节点脚本过期致无法注册；脚本执行失败节点无法注册。[常见高危操作](https://cloud.tencent.com/document/product/457/39539)

# 注册节点

> 将自建 IDC、其他云或边缘机器作为节点接入已有 TKE 标准集群，在云上云下统一管理混合工作负载。
> 控制台: [容器服务 - 节点池 - 注册节点](https://console.cloud.tencent.com/tke2/nodepool)

注册节点（原生节点 API 之外的一种节点形态）把非腾讯云机器接入标准集群，复用集群的调度、日志、监控与网络能力。接入后节点与云上节点统一纳管，但计费、运维边界与能力范围不同。

## 是什么

注册节点 = 在目标机器上执行注册脚本，将机器注册为集群节点。集群不创建 CVM 实例，机器由你自备并维护。两条接入路径：

- **专线版**：目标机器通过专线 / 内网 / VPN 连通集群 VPC，网络稳定，支持 CLB 流量接入与 Prometheus 监控。
- **公网版**：目标机器跨公网接入，部署简单但有安全风险，不支持 CLB 与 Prometheus。

边缘集群（TKE-Edge）已下线，新建边缘场景改用[注册节点公网版](https://cloud.tencent.com/document/product/457/57916)。

## 触发条件

- 你要把自建 IDC、其他云或边缘机器接入已有 TKE 标准集群（本地资源利旧、混合云）— 专线版见[创建注册节点（专线版）](dedicated-line.md)，公网版见[创建注册节点（公网版）](public-network.md)
- 你要修改注册节点池的 Labels / Taints / 删除保护 — [编辑注册节点池](edit-pool.md)
- 你要升级或移除已接入的注册节点 — [升级注册节点](upgrade.md) / [移除注册节点](remove.md)
- 你要让注册节点对外暴露服务（CLB 四 / 七层流量接入）— [流量接入](traffic-access.md)
- 你遇到注册失败、节点不在线或注册节点与注册集群混淆 — [注册节点常见问题](faq.md)

## 核心概念

| 概念 | 含义 | 为什么重要 |
|:-----|:-----|:-----|
| 注册节点池 | 一组注册节点的集合，统一 Labels / Taints / 删除保护 | 批量纳管与运维的基本单位 |
| 注册脚本 | 按节点池生成的接入命令，含过期时间 | 在目标机器执行后节点上线 |
| 专线版 | 内网 / 专线接入 | 支持 CLB、Prometheus，生产推荐 |
| 公网版 | 公网接入 | 部署简单，不支持 CLB、Prometheus，有安全风险 |
| 云上节点 | 腾讯云 CVM / 原生节点 | 部分系统组件必须运行在云上节点 |

## 适用范围与约束

- **操作系统**：仅支持 TencentOS Server 3.1 与 TencentOS Server 2.4（TK4）。其他操作系统会导致注册失败。
- **必须存在云上节点**：集群内部分系统组件必须运行在云上节点，因此开启注册节点特性的集群内必须已有云上节点。
- **独立部署与托管均支持**：注册节点特性可用于托管形态与独立部署形态的集群。
- **配额**：单集群默认支持创建 5 个注册节点；超出需[提交工单](https://console.cloud.tencent.com/workorder/category)申请。
- **费用**：公测期间仅收取 TKE 托管集群及其他云资源费用，暂不收取注册节点管理费用。

## 专线版与公网版能力差异

| 能力 | 专线版 | 公网版 |
|:-----|:-----:|:-----:|
| 网络接入 | 专线 / 内网 / VPN | 公网 |
| CLB 流量接入（四/七层） | 支持 | 不支持 |
| Prometheus 监控接入 | 支持 | 不支持 |
| 日志投递 CLS | 内网（占专线带宽），可改公网 | 公网 |
| 安全建议 | 生产推荐 | 优先内网 / VPN |

## 节点类型定位

TKE 标准集群支持 4 种节点类型，注册节点与常规 CVM 节点、原生节点、超级节点并列：

| 类型 | 特点 | 适用场景 |
|:-----|:-----|:-----|
| 普通节点 | 腾讯云 CVM，基于弹性伸缩组 | 通用工作负载 |
| 原生节点 | MachineSet 声明式管理 | 降本、提升利用率 |
| 超级节点 | Serverless，按 Pod 计费 | 弹性业务、强隔离 |
| 注册节点 | 自备机器接入 | 本地资源利旧、混合云 |

> 注册节点 ≠ 注册集群。注册集群是另一独立产品形态（控制台标「即将下线」），二者计费与配额不可混用。

## 快速检查

```bash
# 列出集群下注册节点池（顶层键 NodePoolSet，非 ExternalNodePoolSet）
tccli tke DescribeExternalNodePools --region ap-guangzhou --ClusterId "<CLUSTER_ID>" \
  --filter "NodePoolSet[].{id:NodePoolId,name:Name,life:LifeState}" --output text
# expected: 有池时列出 id/name/life；无池时 TotalCount=0 且 NodePoolSet 为空数组
```

> 注册节点特性是否已在集群开启，用 `DescribeExternalNodeSupportConfig --ClusterId` 看 **`Status`**（`Disabled`/`Initializing`/`Enabled`/`InitFailed`）。`Enabled` 布尔可能与 `Status` 不同步，**以 `Status` 为准**。创建与接入步骤见下方链接。

## 快速开始

- 专线版接入：[创建注册节点（专线版）](dedicated-line.md)
- 公网版接入：[创建注册节点（公网版）](public-network.md)
- 修改节点池：[编辑注册节点池](edit-pool.md)
- 升级节点组件：[升级注册节点](upgrade.md)
- 下线节点：[移除注册节点](remove.md)
- 对外暴露服务：[流量接入](traffic-access.md)
- 常见问题：[注册节点常见问题](faq.md)
