# VPC-CNI模式Pod数量限制（tccli）

> 对照官方：[VPC-CNI模式Pod数量限制](https://cloud.tencent.com/document/product/457/64917) · page_id `64917`

## 概述

VPC-CNI 模式下 Pod 的数量限制取决于实例类型、可用区和 VpcCniType。不同实例规格支持不同的 ENI 数量及辅助 IP 数量，从而影响该节点可运行的 Pod 数量上限。通过 DescribeVpcCniPodLimits API 可查询指定实例类型在特定可用区的 Pod 数量限制。以 S5.MEDIUM2 在 ap-guangzhou-3 为例：TKERouteENINonStaticIP=27、TKERouteENIStaticIP=27、TKEDirectENI=9、TKESubENI=100。

## 前置条件

- 集群所在区域已开通 VPC-CNI 服务
- 已知目标实例族（InstanceFamily）和实例类型（InstanceType）
- 已知目标可用区（Zone），可用区需支持对应实例类型

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 查询 Pod 数量限制 | `tccli tke DescribeVpcCniPodLimits --region ap-guangzhou --Zone ap-guangzhou-3 --InstanceFamily S5 --InstanceType S5.MEDIUM2` | 是 |
| 查询集群 VPC-CNI 状态 | `tccli tke DescribeIPAMD --ClusterId cls-xxxxxxxx --region ap-guangzhou` | 是 |

## 操作步骤

### 1. 查询集群 VPC-CNI 状态

```bash
tccli tke DescribeIPAMD \
  --ClusterId cls-xxxxxxxx \
  --region ap-guangzhou
```

```json
{
  "EnableIPAMD": true,
  "EnableCustomizedPodCidr": true,
  "DisableVpcCniMode": true,
  "Phase": "<Phase>",
  "Reason": "<Reason>",
  "SubnetIds": [],
  "ClaimExpiredDuration": "<ClaimExpiredDuration>",
  "EnableTrunkingENI": true
}
```

当前集群 cls-xxxxxxxx 为 GR 模式，EnableIPAMD 为 false。Pod 数量限制查询无需依赖当前集群 VPC-CNI 状态，可直接查询。若返回 `FailedOperati

```json
{
  "TotalCount": 0,
  "PodLimitsInstanceSet": [],
  "Zone": "<Zone>",
  "InstanceFamily": "<InstanceFamily>",
  "InstanceType": "<InstanceType>",
  "PodLimits": "<PodLimits>",
  "TKERouteENINonStaticIP": 0,
  "TKERouteENIStaticIP": 0
}
```on.EnableVPCCNIFailed` 说明集群尚未开通 VPC-CNI，需先执行 EnableVpcCniNetworkType。

### 2. 查询指定实例类型的 Pod 数量限制

```bash
tccli tke DescribeVpcCniPodLimits \
  --region ap-guangzhou \
  --Zone ap-guangzhou-3 \
  --InstanceFamily S5 \
  --InstanceType S5.MEDIUM2
```

返回示例：

```json
{
    "TKERouteENINonStaticIP": 27,
    "TKERouteENIStaticIP": 27,
    "TKEDirectENI": 9,
    "TKESubENI": 100,
    "RequestId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
}
```

### 3. 解读返回结果

各字段含义：

| 字段 | 含义 | 说明 |
|------|------|------|
| `TKERouteENINonStaticIP` | route-eni 模式非固定 IP Pod 数量 | 使用 tke-route-eni 类型时，非固定 IP 的 Pod 数量上限 |
| `TKERouteENIStaticIP` | route-eni 模式固定 IP Pod 数量 | 使用 tke-route-eni 类型时，固定 IP 的 Pod 数量上限 |
| `TKEDirectENI` | direct-eni 模式 Pod 数量 | 使用 tke-direct-eni 类型时，Pod 独占 ENI 的数量上限（通常较小，受限于实例的 ENI 数量） |
| `TKESubENI` | sub-eni 模式 Pod 数量 | 使用 tke-sub-eni（Trunking ENI）类型时，Pod 数量上限（通过辅助 ENI 扩展） |

选择 VpcCniType 时参考：tke-route-eni 适用于高密度部署（单节点 27 个 Pod），tke-direct-eni 适用于需要 Pod 独占 ENI 性能的场景（单节点 9 个 Pod），tke-sub-eni 适用于超高密度场景（单节点 100 个 Pod）。

## 验证

- 多次执行 `tccli tke DescribeVpcCniPodLimits`，确认返回结果一致
- 更换 `--Zone`、`--InstanceFamily`、`--InstanceType` 参数组合，交叉验证限制数据
- 将查询结果与官方文档中的实例规格表对照，确认数据一致性

## 清理

本操作为纯查询类操作，不创建任何资源，无需清理步骤。

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 查询返回 0 或空 | 检查 `--Zone`、`--InstanceFamily`、`--InstanceType` 参数是否正确 | 参数错误导致 API 无法匹配实例类型 | 通过 `tccli cvm DescribeInstanceTypeConfigs` 或 `DescribeInstanceTypes` 核对可用实例类型 |
| Zone not available | 实例族在该可用区不可用 | 实例族不支持当前可用区 | 更换 `--Zone` 为支持该实例族的可用区 |
| 返回值与预期不符 | 不同实例规格 ENI 数量不同 | 实例规格限制决定 Pod 上限 | 查询更大规格的同族实例（如 S5.LARGE8）对比上限差异 |
| API 返回 UnknownParameter | 确认 CLI 参数拼写 | 参数名称或大小写错误 | 核实参数名：Zone（大驼峰）、InstanceFamily（大驼峰）、InstanceType（大驼峰） |
| Subnet shortage 错误 | 节点实际创建 Pod 时子网 IP 不足 | VPC 子网 IP 已耗尽（与查询限制无关） | 通过 AddVpcCniSubnets 添加更多子网 CIDR |

## 下一步

- [多Pod共享网卡](https://cloud.tencent.com/document/product/457/50356) -- page_id `50356`
- [独占网卡](https://cloud.tencent.com/document/product/457/50357) -- page_id `50357`
- [VPC-CNI模式介绍](https://cloud.tencent.com/document/product/457/50355) -- page_id `50355`

## 控制台替代

在 TKE 控制台创建集群或节点池时，选择 VPC-CNI 网络模式后，控制台会自动根据所选机型展示该机型支持的 Pod 数量上限，无需手动调用 API 查询。
