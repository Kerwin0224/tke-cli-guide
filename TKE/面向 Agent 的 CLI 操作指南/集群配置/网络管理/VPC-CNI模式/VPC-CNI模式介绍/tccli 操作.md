# VPC-CNI模式介绍（tccli）

> 对照官方：[VPC-CNI模式介绍](https://cloud.tencent.com/document/product/457/50355) · page_id `50355`

## 概述

VPC-CNI 模式是 TKE 容器网络的一种实现方式，Pod 直接从 VPC 的 CIDR 获取 IP 地址，使 Pod 成为 VPC 中的一等公民。与 GlobalRouter (GR) 模式不同，GR 模式下 Pod 使用容器专有 CIDR 进行 Overlay 封装通信，而 VPC-CNI 模式下 Pod IP 属于 VPC 子网，可直接与 VPC 内其他资源（如 CVM、CLB、CDB）互通，无需经过 NAT 或路由跳转。

### 子类型

VPC-CNI 模式支持三种子类型：

| 类型 | API 参数 | 说明 |
|------|---------|------|
| **多Pod共享网卡（tke-route-eni）** | `VpcCniType: "tke-route-eni"` | 多个 Pod 共享节点弹性网卡（ENI）的辅助 IP，Pod 密度高。例如 S5.MEDIUM2 支持 27 个非固定 IP Pod。 |
| **Pod间独占网卡（tke-direct-eni）** | `VpcCniType: "tke-direct-eni"` | 每个 Pod 独占一张 ENI，享有独享网卡的网络性能。但 Pod 密度较低，S5.MEDIUM2 仅支持 9 个直接 ENI Pod。 |
| **tke-sub-eni（Trunking ENI）** | `VpcCniType: "tke-sub-eni"` | 基于 Trunking ENI 技术的子网卡模式，兼顾高密度与独立网络性能。 |

### IP 分配模式

| 模式 | 说明 |
|------|------|
| **非固定IP** | Pod 重启或漂移后 IP 会变化，IP 从子网池中动态分配。 |
| **固定IP** | Pod 重启后保留原 IP，适用于依赖固定 IP 的服务（如数据库、有状态应用）。需要在 StatefulSet 中使用。 |

### 关键 API

| API | 用途 |
|-----|------|
| `DescribeIPAMD` | 查询集群 VPC-CNI 状态（`EnableIPAMD` 字段） |
| `EnableVpcCniNetworkType` | 启用 VPC-CNI 网络模式（GR 集群迁移到 VPC-CNI 的入口） |
| `DisableVpcCniNetworkType` | 关闭 VPC-CNI 网络模式 |
| `DescribeEnableVpcCniProgress` | 查询 VPC-CNI 开启/关闭进度 |
| `DescribeVpcCniPodLimits` | 查询不同机型在 VPC-CNI 模式下的 Pod 数量限制 |
| `AddVpcCniSubnets` | 向 VPC-CNI 集群添加子网 |

### 当前集群状态

集群 `cls-xxxxxxxx` 当前为 **GR 模式**，未启用 VPC-CNI。`DescribeIPAMD` 查询结果中 `EnableIPAMD` 为 `false`。

## 前置条件

- 集群状态为 **Running**
- VPC 下有可用子网（子网 CIDR 需有足够空闲 IP 规划给 Pod 使用）
- 目标子网需在与集群相同地域（如 `ap-guangzhou`）
- 了解不同子类型的 Pod 密度上限（通过 `DescribeVpcCniPodLimits` 查询）
- 注意：开启 VPC-CNI 后 Pod 网络模型发生变化，需重调度/重建 Pod

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 查询集群基本信息 | `tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]'` | 是 |
| 查询 VPC-CNI 状态 | `tccli tke DescribeIPAMD --region ap-guangzhou --ClusterId cls-xxxxxxxx` | 是 |
| 查询 Pod 数量限制 | `tccli tke DescribeVpcCniPodLimits --region ap-guangzhou --Zone ap-guangzhou-3 --InstanceFamily S5 --InstanceType S5.MEDIUM2` | 是 |
| 开启 VPC-CNI 模式 | `tccli tke EnableVpcCniNetworkType --region ap-guangzhou --ClusterId cls-xxxxxxxx --VpcCniType "tke-route-eni" --SubnetIds '["subnet-xxxxxxxx"]'` | 否 |
| 查询开启进度 | `tccli tke DescribeEnableVpcCniProgress --region ap-guangzhou --ClusterId cls-xxxxxxxx` | 是 |

## 操作步骤

本页为概念介绍页，无实际操作步骤。具体操作请参见各子类型的操作页面。

## 验证

> 验证集群当前的 VPC-CNI 状态。

```bash
tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]'
# expected: ClusterType: "MANAGED_CLUSTER", NetworkType: "GR" (GlobalRouter, 当前未启用 VPC-CNI)

tccli tke DescribeIPAMD --region ap-guangzhou --ClusterId cls-xxxxxxxx
# expected: "EnableIPAMD": false (说明 VPC-CNI 未启用)
```

```json
{
  "TotalCount": "<TotalCount>",
  "Clusters": "<Clusters>",
  "ClusterId": "<ClusterId>",
  "ClusterName": "<ClusterName>",
  "ClusterDescription": "<ClusterDescription>",
  "ClusterVersion": "<ClusterVersion>"
}
```

## 清理

本页为概念介绍页，无清理操作。

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| GR 集群直接查询 VPC-CNI 进度报错 `FailedOperation.EnableVPCCNIFailed: cluster is not vpc-cni cluster` | 查询 `DescribeIPAMD` 确认 `EnableIPAMD` 为 `false` | GR 集群未开启 VPC-CNI，无法查询 VPC-CNI 相关进度 | 先调用 `EnableVpcCniNetworkType` 启用 VPC-CNI 后再查询进度 |
| 开启 VPC-CNI 时报错 `LimitExceeded` | 查询 VPC 子网信息，确认子网中可用 IP 是否充足 | VPC-CNI 子网可用 IP 不足，无法为 Pod 分配 IP | 调用 `AddVpcCniSubnets` 添加新子网，或清理子网中空闲 IP |
| 切换网络模式后 Pod 无法通网 | 检查 Pod 事件和节点网络配置 | 切换网络模式后 Pod 网络模型改变，原 Pod 可能 IP 失效 | 重建 Pod 以获取新的网络配置；注意 `DisableVpcCniNetworkType` 会改变 Pod IP |

## 下一步

- [多Pod共享网卡模式](https://cloud.tencent.com/document/product/457/50356) -- page_id `50356`
- [Pod间独占网卡模式](https://cloud.tencent.com/document/product/457/50357) -- page_id `50357`
- [VPC-CNI模式Pod数量限制](https://cloud.tencent.com/document/product/457/64917) -- page_id `64917`
- [固定IP使用方法](https://cloud.tencent.com/document/product/457/34994) -- page_id `34994`

## 控制台替代

所有 VPC-CNI 相关操作均可在 TKE 控制台完成（集群详情 > 基本信息 > 网络信息 > VPC-CNI 模式设置）。控制台提供可视化界面选择子类型（共享网卡/独占网卡）并指定子网，同时可查看开启进度。CLI 方式适用于自动化脚本、批量配置和 CI/CD 集成场景。
