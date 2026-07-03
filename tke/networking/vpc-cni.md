---
doc_type: How-to
subtype: 6B
fused: false
---
# 配置 VPC-CNI 网络

> 开启/关闭 VPC-CNI 网络模型。VPC-CNI 让 Pod 直接从 VPC 子网获取 IP，支持固定 IP 与安全组直通。配置型操作（改变行为，不创建/销毁资源）。

## 概述

VPC-CNI 是 TKE 的三种 Pod 网络模型之一（另两种：Global Router / CiliumOverlay）。开启后，新建 Pod 可从指定 VPC 子网获取 IP，与 CVM 同级，支持安全组直通与固定 IP。

| 模型 | Pod IP 来源 | 固定 IP | 安全组直通 | IP 消耗 | 后期扩网段 |
|:-----|:-----------|:------:|:----------|:--------|:----------:|
| Global Router（默认） | 容器网段 | ❌ | ❌ | 少 | ✅ `AddClusterCIDR` |
| VPC-CNI | VPC 子网 | ✅ | ✅ | 多（每 Pod 一个 VPC IP） | ❌ |
| CiliumOverlay | Overlay 隧道 | ❌ | ❌ | 少（不占 VPC IP） | ❌ |

> VPC-CNI 可与 Global Router 共存：Global Router 为主，VPC-CNI 子网补充。开启 VPC-CNI 不影响已有 Global Router Pod。CiliumOverlay 与两者互斥（创建时定型，不可切换），见 [配置 CiliumOverlay](cilium-overlay.md)。

## 决策依据

#### 为什么用 VPC-CNI

- **VPC-CNI vs Global Router vs CiliumOverlay**: VPC-CNI 让 Pod 拿 VPC IP，支持固定 IP（StatefulSet 稳定地址）与安全组直通（Pod 级网络策略）；Global Router 的 Pod 用容器网段 IP，不暴露到 VPC；CiliumOverlay 用 Overlay 隧道，不占 VPC IP 但无固定 IP/安全组直通，适合要 Cilium 数据面的场景（见 [配置 CiliumOverlay](cilium-overlay.md)）
- **默认推荐**: 不确定就用 Global Router（默认）。仅当需要固定 IP 或安全组直通时开启 VPC-CNI；仅当需要 Cilium 数据面且接受创建时定型时选 CiliumOverlay
- **能关闭吗?**: VPC-CNI 能，`DisableVpcCniNetworkType`（已有 VPC-CNI Pod 需先迁移）。CiliumOverlay 不能——创建时定型，无独立开关 Action

## 配置项

> 来源：`tccli tke EnableVpcCniNetworkType --generate-cli-skeleton`。

| 字段 | 类型 | 必填 | 默认值 | 有效值 | 填错的影响 |
|:------|------|:--------:|:------:|-------|-----------|
| ClusterId | string | 是 | — | `cls-xxxxxxxx` | `ResourceNotFound` |
| VpcCniType | string | 是 | — | `tke-route-eni` / `tke-direct-route-eni` | IP 分配方式不对 |
| EnableStaticIp | boolean | 否 | false | `true`/`false` | 固定 IP 不生效 |
| Subnets | list | 是 | — | VPC 子网 ID 列表 | `ResourceNotFound.SubnetId` |
| ExpiredSeconds | int | 否 | 0 | 固定 IP 回收秒数 | IP 回收时机不对 |
| SkipAddingNonMasqueradeCIDRs | boolean | 否 | false | `true`/`false` | 路由配置影响 |

> `VpcCniType`: `tke-route-eni`（弹性网卡路由，常用）/ `tke-direct-route-eni`（直连路由）。`EnableStaticIp=true` 开启固定 IP，配合 `ExpiredSeconds` 设回收时间。

## 应用

### 开启 VPC-CNI

```bash
tccli tke EnableVpcCniNetworkType --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" \
  --VpcCniType tke-route-eni \
  --EnableStaticIp true \
  --Subnets '["<SUBNET_ID>"]' \
  --ExpiredSeconds 300
# expected: exit 0, 返回 RequestId
```

| 占位符 | 含义 | 约束 | 如何获取 |
|:------------|:-----|:-----|:---------|
| `<CLUSTER_ID>` | 集群 ID | `cls-xxxxxxxx` | `tccli tke DescribeClusters` → `Clusters[].ClusterId` |
| `<SUBNET_ID>` | VPC 子网 ID | 须在集群 VPC 内，且有可用 IP | `tccli vpc DescribeSubnets` |

> 子网 IP 数量决定可运行的 VPC-CNI Pod 数。IP 不足时新 Pod 卡在 ContainerCreating。

### 查询 Pod 上限（按机型）

```bash
# 查某机型在该可用区的 VPC-CNI Pod 上限
tccli tke DescribeVpcCniPodLimits --region ap-guangzhou \
  --Zone "ap-guangzhou-3" --InstanceType "S5.MEDIUM4"
# expected: 该机型支持的 Pod 上限
```

> `DescribeVpcCniPodLimits` 入参是 `Zone`/`InstanceFamily`/`InstanceType`（CVM 机型维度），非 ClusterId。不同机型支持的 VPC-CNI Pod 数不同。

### 查询 IPAMD 状态

> `DescribeIPAMD` 诊断 VPC-CNI IPAMD 组件状态（IPAMD 负责 Pod IP 分配）。

```bash
tccli tke DescribeIPAMD --ClusterId "<CLUSTER_ID>" --region <REGION>
# expected: exit 0, EnableIPAMD/Phase/SubnetIds 反映 IPAMD 状态
```
```json
{
    "EnableIPAMD": false,
    "EnableCustomizedPodCidr": false,
    "DisableVpcCniMode": false,
    "Phase": "",
    "Reason": "",
    "SubnetIds": null,
    "ClaimExpiredDuration": "",
    "EnableTrunkingENI": false
}
```

> `EnableIPAMD=false` 表示未启用 IPAMD（集群非 VPC-CNI 或未开）。`Phase` 反映运行阶段，`Reason` 非空表示异常原因。`EnableTrunkingENI` 是中继弹性网卡模式。

### 集群路由表管理

> 集群路由表（`ClusterRouteTable`）是 Global Router 模式的底层路由，TKE 用它把 Pod 网段路由到节点。路由表用 `RouteTableName`（字符串名）标识，**不绑 ClusterId**。

```bash
# 查询所有集群路由表 (无入参)
tccli tke DescribeClusterRouteTables --region <REGION>
# expected: exit 0, RouteTableSet[] 含 RouteTableName/RouteTableCidrBlock/VpcId
```
```json
{
    "TotalCount": 2,
    "RouteTableSet": [
        {"RouteTableName": "rt-example", "RouteTableCidrBlock": "10.20.0.0/16", "VpcId": "vpc-example"}
    ]
}
```

```bash
# 查询路由表冲突 (建表前检查 CIDR 是否与 VPC 已有路由冲突)
tccli tke DescribeRouteTableConflicts --region <REGION> \
  --RouteTableCidrBlock "<CIDR>" --VpcId "<VPC_ID>"
# expected: exit 0, 返回冲突列表 (无冲突则空)

# 创建路由表
tccli tke CreateClusterRouteTable --region <REGION> \
  --RouteTableName "<RT_NAME>" --RouteTableCidrBlock "<CIDR>" --VpcId "<VPC_ID>" --IgnoreClusterCidrConflict 0
# expected: exit 0

# 查询路由表下的路由
tccli tke DescribeClusterRoutes --region <REGION> --RouteTableName "<RT_NAME>"
# expected: exit 0, TotalCount + 路由条目

# 添加路由
tccli tke CreateClusterRoute --region <REGION> \
  --RouteTableName "<RT_NAME>" --DestinationCidrBlock "<DEST_CIDR>" --GatewayIp "<GW_IP>"
# expected: exit 0

# 删除路由
tccli tke DeleteClusterRoute --region <REGION> \
  --RouteTableName "<RT_NAME>" --GatewayIp "<GW_IP>" --DestinationCidrBlock "<DEST_CIDR>"
# expected: exit 0

# 删除路由表
tccli tke DeleteClusterRouteTable --region <REGION> --RouteTableName "<RT_NAME>"
# expected: exit 0
```

> ⚠️ 路由操作用 `RouteTableName`（非 ClusterId，非路由表 ID）。`DeleteClusterRoute` 需同时传 `RouteTableName`+`GatewayIp`+`DestinationCidrBlock` 三者定位路由。`CreateClusterRouteTable` 的 `IgnoreClusterCidrConflict` 是 Integer（0/1）非 Boolean。建表前用 `DescribeRouteTableConflicts` 查冲突。

## 验证

```bash
# 查询开启进度
tccli tke DescribeEnableVpcCniProgress --region ap-guangzhou --ClusterId "<CLUSTER_ID>"
# expected: Status="Enabled"，进度 100%
```

| 维度 | 命令 | 预期 |
|:-----|:-----|:-----|
| 开启进度 | `DescribeEnableVpcCniProgress` → `Status` | `Enabled` |
| 网络类型 | `DescribeClusters` → `NetworkType` | 含 `VPC-CNI` |
| Pod 获 IP | `kubectl get pod -o wide` | Pod IP 在 VPC 子网段内 |

## 回滚

```bash
# 关闭 VPC-CNI（已有 VPC-CNI Pod 需先迁移）
tccli tke DisableVpcCniNetworkType --region ap-guangzhou --ClusterId "<CLUSTER_ID>"
# expected: exit 0
```

> 关闭前确认无 VPC-CNI Pod 运行，否则这些 Pod 会失联。建议先迁移到 Global Router 节点。

## 故障恢复

### 命令返回错误 (exit ≠ 0)

| 现象 | 诊断 | 根因 | 修复 |
|:--------|:----------|:------------|:-----|
| `ResourceNotFound.SubnetId` | `tccli vpc DescribeSubnets` | 子网不在集群 VPC | 用集群 VPC 内子网 |
| `InvalidParameterValue.VpcCniType` | 查枚举 | VpcCniType 拼错 | 用 `tke-route-eni` 或 `tke-direct-route-eni` |
| `UnsupportedOperation` | `DescribeClusters` → `NetworkType` | 已开启 VPC-CNI 或集群非 Running | 先 Disable 或等 Running |

### 命令成功但状态不对 (exit = 0)

| 现象 | 诊断 | 根因 | 修复 |
|:--------|:----------|:------------|:-----|
| 开启卡住进度不动 | `DescribeEnableVpcCniProgress` | 子网 IP 不足或路由冲突 | 加子网（`AddVpcCniSubnets`）或换子网 |
| Pod 卡在 ContainerCreating | `kubectl describe pod` | 子网 IP 耗尽 | `AddVpcCniSubnets` 增加子网 |
| 固定 IP 不生效 | `EnableStaticIp` 值 | 未设 true 或 StatefulSet 未配 | `EnableStaticIp=true`，StatefulSet 注解 `tke.cloud.tencent.com/vpc-ips` |

### 子网 IP 不足时增加子网

> VPC-CNI Pod 卡在 ContainerCreating 多因子网 IP 耗尽。`AddVpcCniSubnets` 给已开启的 VPC-CNI 追加子网，参数以 `--generate-cli-skeleton` 为准（`SubnetIds[]` 复数 + `VpcId`）。

```bash
# 给 VPC-CNI 追加子网（ClusterId + VpcId + SubnetIds[]）
tccli tke AddVpcCniSubnets --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" \
  --VpcId "<VPC_ID>" \
  --SubnetIds '["<SUBNET_ID>"]'
# expected: exit 0 返回 RequestId; 子网无效报 InvalidParameter.Param: subnet <ID> is invalid
```

| 占位符 | 含义 | 如何获取 |
|:-------|:-----|:---------|
| `<VPC_ID>` | 集群所在 VPC | `tccli tke DescribeClusters` → `Clusters[].ClusterNetworkSettings.VpcId` |
| `<SUBNET_ID>` | 追加的子网 ID | `tccli vpc DescribeSubnets --Filters '[{"Name":"vpc-id","Values":["<VPC_ID>"]}]'` |

> `AddVpcCniSubnets` 用复数 `SubnetIds[]`（可一次追加多个子网），需带 `VpcId` 标识子网所属 VPC。追加后用 `DescribeEnableVpcCniProgress` 确认子网已生效，新 Pod 将从新子网获 IP。

## 下一步

- [管理访问端点](endpoints.md) — API Server 访问入口
- [配置 CiliumOverlay](cilium-overlay.md) — 第三种网络模型，Cilium 数据面
- [创建集群](../clusters/create.md) — 建集群时选网络模型
- [故障排查](../troubleshooting.md) — Pod IP 不足诊断

## 控制台替代方案

[容器服务控制台 - 集群网络](https://console.cloud.tencent.com/tke2/cluster)
