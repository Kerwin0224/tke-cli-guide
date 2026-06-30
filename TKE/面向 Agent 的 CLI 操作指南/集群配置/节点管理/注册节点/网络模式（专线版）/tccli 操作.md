# 网络模式（专线版）

> 对照官方：[网络模式（专线版）](https://cloud.tencent.com/document/product/457/79748) · page_id `79748` · tccli ≥1.0.0 · API 2018-05-25

## 概述

在专线场景下，TKE 注册节点支持**两种网络模式**，通过 `EnableExternalNodeSupport` 的 `NetworkType` 参数选择。两种模式都要求 IDC 与腾讯云 VPC 之间已通过**专线或 VPN** 建立互通，网络流量不经公网传输。

| 模式 | API 枚举 | 适用场景 | ClusterCIDR | Pod IP 可达性 |
|------|----------|---------|:-----------:|:------------:|
| **Cilium-Overlay**（推荐 IDC 场景） | `CiliumBGP` | IDC IP 资源不足，需 Overlay 网络 | 必须指定 | Pod IP 集群外不可直接访问 |
| **GR / VPC-CNI** | `HostNetwork` | IDC IP 充足，Pod IP 需直接可达 | 无需填写 | Pod 直接使用节点网络 |

> **关键概念**：控制台页面名称"专线版"（DirectConnect）对应的 API 枚举值是 `HostNetwork` 和 `CiliumBGP`，**不是** `DirectConnect`。传 `DirectConnect` 将触发 `InvalidParameter.NetworkType`。

### Cilium-Overlay（`CiliumBGP`）

- 云上节点和注册节点共用容器网段，使用 Cilium VXLan 隧道封装协议构建 Overlay 网络
- 容器网段分配灵活，不占用 VPC 其他网段；无需更改 IDC 基础网络设施
- **注意事项**：约 10% 性能损耗；Pod IP 集群外不可达（Pod 访问集群外时 SNAT 成节点 IP）；不支持固定 Pod IP；不要同时创建超级节点池（Pod 间无法互访）

### GR / VPC-CNI（`HostNetwork`）

- 注册节点上 Pod 直接使用节点所在网络（不封装），适用于需要 Pod IP 直接可达的场景
- **注意事项**：需要 IDC 有足够 IP 分配给 Pod；不同网络环境间 Pod 互通受限于底层网络路由；具体限制因网络类型（GlobalRouter vs VPC-CNI）而异

> 本文档中所有 ID（集群 ID、节点池 ID、子网 ID 等）均为占位符，请替换为实际值。

## 前置条件

- [环境准备](../../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:DescribeClusters
#    tke:EnableExternalNodeSupport
#    tke:DescribeExternalNodeSupportConfig
#    tke:DescribeExternalNodePools
# 验证：执行 DescribeClusters 确认权限
tccli tke DescribeClusters --region <Region>
# expected: exit 0，返回集群列表（可为空）
```

**预期输出**（权限验证通过，返回集群列表）：

```json
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "cls-example",
            "ClusterName": "example-cluster",
            "ClusterStatus": "Running",
            "ClusterType": "MANAGED_CLUSTER",
            "ClusterVersion": "1.32.2",
            "EnableExternalNode": false
        }
    ]
}
```

### 资源检查

```bash
# 4. 确认目标集群存在
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["<ClusterId>"]'
# expected: TotalCount >= 1，ClusterType 为 MANAGED_CLUSTER

# 5. 查询 VPC 和子网（注册节点子网需与集群同 VPC 或可经由专线互通）
tccli vpc DescribeVpcs --region <Region>
# expected: 至少返回 1 个 VPC

tccli vpc DescribeSubnets --region <Region> \
    --Filters '[{"Name":"vpc-id","Values":["<VpcId>"]}]'
# expected: 至少返回 1 个子网

# 6. 确认专线/VPN 已建立（IDC 与 VPC 互通）
# 无对应的 tccli 直接检查命令，请登录控制台或联系网络管理员确认
```

**预期输出**（集群查询）：

```json
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "cls-example",
            "ClusterName": "example-cluster",
            "ClusterStatus": "Running",
            "ClusterType": "MANAGED_CLUSTER",
            "ClusterNetworkSettings": {
                "ClusterCIDR": "10.0.0.0/16",
                "ServiceCIDR": "10.1.0.0/20",
                "VpcId": "vpc-example",
                "SubnetId": "subnet-example",
                "MaxNodePodNum": 64,
                "Cni": true
            },
            "EnableExternalNode": false
        }
    ]
}
```

**预期输出**（VPC 查询）：

```json
{
    "TotalCount": 1,
    "VpcSet": [
        {
            "VpcId": "vpc-example",
            "VpcName": "example-vpc",
            "CidrBlock": "172.24.0.0/20"
        }
    ]
}
```

**预期输出**（子网查询）：

```json
{
    "TotalCount": 1,
    "SubnetSet": [
        {
            "SubnetId": "subnet-example",
            "SubnetName": "example-subnet",
            "CidrBlock": "172.24.0.0/20",
            "AvailableIpAddressCount": 4086
        }
    ]
}

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看注册节点配置 | `DescribeExternalNodeSupportConfig` | 是 |
| 开启注册节点支持（Cilium-Overlay） | `EnableExternalNodeSupport --NetworkType CiliumBGP` | 是 |
| 开启注册节点支持（GR/VPC-CNI） | `EnableExternalNodeSupport --NetworkType HostNetwork` | 是 |
| 查看注册节点池 | `DescribeExternalNodePools` | 是 |

### 关键字段说明

`EnableExternalNodeSupport` 参数：

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `ClusterId` | String | 是 | 集群 ID，格式 `cls-xxxxxxxx` | 集群不存在 → `InvalidParameter.ClusterId` |
| `NetworkType` | String | 是 | `HostNetwork`（GR/VPC-CNI）或 `CiliumBGP`（Cilium-Overlay）。控制台概念名 `DirectConnect`/`专线版` 不是合法值 | 填 `DirectConnect` → `InvalidParameter.NetworkType` |
| `SubnetId` | String | 是 | 已存在的子网 ID，格式 `subnet-xxxxxxxx`，需与 VPC 网络可达 | 子网不存在 → `InvalidParameter.SubnetId` |
| `ClusterCIDR` | String | 条件必填 | `NetworkType=CiliumBGP` 时必填（Pod IP 段），不能与 VPC CIDR/已有集群 CIDR 重叠。`NetworkType=HostNetwork` 时**无需填写**（API help 明确标注） | CiliumBGP 模式下 CIDR 冲突 → `InvalidParameter.ClusterCIDRSettings` |

`DescribeExternalNodeSupportConfig` 输出关键字段：

| 字段 | 类型 | 含义 |
|------|------|------|
| `NetworkType` | String | 网络类型：`HostNetwork`（GR/VPC-CNI）或 `CiliumBGP`（Cilium-Overlay） |
| `SubnetId` | String | 配置的子网 ID |
| `ClusterCIDR` | String | 分配的 Pod IP 段（HostNetwork 模式下为空字符串 `""`） |
| `Enabled` | Boolean | 注册节点功能是否已启用 |
| `Status` | String | 启用状态：`Enabled`（已启用）、`Disabled`（未启用）。**注意：是 `Enabled` 不是 `Running`** |
| `FailedReason` | String | 如状态异常，此处说明失败原因 |
| `Progress` | Array | 9 步初始化流水线（异步操作），每步含 `Name` 和 `Status`（`success` 或 `failed`） |
| `EnabledPublicConnect` | Boolean | 是否开启公网连接 |
| `PublicConnectUrl` | String | 公网连接地址（格式 `IP:Port`） |
| `Master` | String | Master 组件 IP |
| `Proxy` | String | Proxy 组件 IP |

## 操作步骤

本页为概念说明页，具体创建注册节点操作参见 [创建注册节点（专线版）](../创建注册节点（专线版）/tccli%20操作.md)。本节提供 `EnableExternalNodeSupport` 的参数模板和 `DescribeExternalNodeSupportConfig` 的查询与轮询方法。

### 选择依据

*为什么选这个 NetworkType：*

- **Cilium-Overlay（`CiliumBGP`）**：推荐 IDC 场景。使用 VXLan 隧道封装，Pod 有独立 IP 段（需指定 `ClusterCIDR`）。适合 IDC IP 资源不足的场景。约 10% 性能损耗，Pod IP 集群外不可直接访问，不支持固定 Pod IP。
- **GR / VPC-CNI（`HostNetwork`）**：Pod 直接使用节点网络（不封装），Pod IP 可直接可达。适合 IDC IP 充足且需要 Pod IP 直接可达的场景。`HostNetwork` 模式**无需指定 `ClusterCIDR`**（API help 明确标注"网络模式 HostNetwork 时无需填写"），查询输出中该字段为空字符串。
- **控制台概念 vs API 枚举**：控制台的"专线版"（DirectConnect）对应 API 枚举 `HostNetwork` 和 `CiliumBGP`，不是 `DirectConnect`。填 `DirectConnect` 会触发 `InvalidParameter.NetworkType`。

### 查看当前网络模式配置

查询前，先确认当前是否已开启：

```bash
tccli tke DescribeExternalNodeSupportConfig --region <Region> \
    --ClusterId <ClusterId>
# expected: exit 0，返回当前网络模式和启用状态
```

**预期输出**（HostNetwork 模式，已启用）：

```json
{
    "ClusterCIDR": "",
    "NetworkType": "HostNetwork",
    "SubnetId": "subnet-example",
    "Enabled": false,
    "AS": "",
    "SwitchIP": "",
    "Status": "Enabled",
    "FailedReason": "",
    "Master": "172.24.0.2",
    "Proxy": "172.24.0.19",
    "Progress": [
        {
            "Name": "EnsureNodeMasterCommunicate",
            "Status": "success",
            "StartAt": "2026-06-23T08:10:00Z",
            "EndAt": "2026-06-23T08:10:05Z",
            "Message": "",
            "Detail": ""
        },
        {
            "Name": "EnsureInClusterAccess",
            "Status": "success",
            "StartAt": "2026-06-23T08:10:05Z",
            "EndAt": "2026-06-23T08:10:07Z",
            "Message": "",
            "Detail": ""
        },
        {
            "Name": "EnsureExternalNodeProxy",
            "Status": "success",
            "StartAt": "2026-06-23T08:10:07Z",
            "EndAt": "2026-06-23T08:10:30Z",
            "Message": "",
            "Detail": ""
        },
        {
            "Name": "EnsureBootStrap",
            "Status": "success",
            "StartAt": "2026-06-23T08:10:30Z",
            "EndAt": "2026-06-23T08:10:31Z",
            "Message": "",
            "Detail": ""
        },
        {
            "Name": "EnsureAdmissionController",
            "Status": "success",
            "StartAt": "2026-06-23T08:10:32Z",
            "EndAt": "2026-06-23T08:10:35Z",
            "Message": "",
            "Detail": ""
        },
        {
            "Name": "AdaptNetworkPlugin",
            "Status": "success",
            "StartAt": "2026-06-23T08:10:35Z",
            "EndAt": "2026-06-23T08:10:37Z",
            "Message": "",
            "Detail": ""
        },
        {
            "Name": "InstallUnderlayCilium",
            "Status": "success",
            "StartAt": "2026-06-23T08:10:37Z",
            "EndAt": "2026-06-23T08:10:40Z",
            "Message": "",
            "Detail": ""
        },
        {
            "Name": "AdaptTkePlatApp",
            "Status": "success",
            "StartAt": "2026-06-23T08:10:40Z",
            "EndAt": "2026-06-23T08:11:00Z",
            "Message": "",
            "Detail": ""
        },
        {
            "Name": "CompleteEnable",
            "Status": "success",
            "StartAt": "2026-06-23T08:11:00Z",
            "EndAt": "2026-06-23T08:11:01Z",
            "Message": "",
            "Detail": ""
        }
    ],
    "EnabledPublicConnect": true,
    "PublicConnectUrl": "43.144.20.123:9000"
}
```

**关键字段解读**：

- `ClusterCIDR`：HostNetwork 模式下为空字符串（此模式不需要指定 Pod IP 段）；CiliumBGP 模式下为此处填写的 CIDR 值
- `Status`：`Enabled` 表示启用完成（**不是 `Running`**）；`Disabled` 表示未启用
- `Enabled`：注意与 `Status` 区分——`Enabled` 是布尔开关字段（`true`/`false`），`Status` 是状态描述字段
- `Progress`：`EnableExternalNodeSupport` 是异步操作，完成后 `Progress` 展示 9 步初始化流水线，每步 `Status` 为 `success`
- `Master` / `Proxy`：注册节点连回集群所需的 Master 和 Proxy 组件的 IP 地址
- `EnabledPublicConnect` / `PublicConnectUrl`：公网连接配置

**提前识别**：`NetworkType` 为 `HostNetwork` 时 `ClusterCIDR` 为空字符串（不是 `null`，不是未返回）。这是正确行为，不是异常。如果预期是 CiliumBGP 模式但看到空 `ClusterCIDR`，说明当前选的是 HostNetwork。

### EnableExternalNodeSupport JSON 模板

**HostNetwork 模式**（`enable-hostnetwork.json`）：

```json
{
    "ClusterId": "<ClusterId>",
    "ClusterExternalConfig": {
        "NetworkType": "HostNetwork",
        "SubnetId": "<SubnetId>"
    }
}
```

> `ClusterCIDR` 不填——API help 明确标注"网络模式 HostNetwork 时无需填写"。

**CiliumBGP 模式**（`enable-ciliumbgp.json`）：

```json
{
    "ClusterId": "<ClusterId>",
    "ClusterExternalConfig": {
        "NetworkType": "CiliumBGP",
        "SubnetId": "<SubnetId>",
        "ClusterCIDR": "10.200.0.0/16"
    }
}
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<ClusterId>` | 集群 ID | 格式 `cls-xxxxxxxx` | `tccli tke DescribeClusters --region <Region>` |
| `<SubnetId>` | 子网 ID | 格式 `subnet-xxxxxxxx`，需与 VPC 网络可达 | `tccli vpc DescribeSubnets --region <Region>` |

**CIDR 规划约束**（仅 CiliumBGP 模式）：

- `ClusterCIDR` 为注册节点上 Pod 使用的 IP 段，不能与以下重叠：
  - VPC CIDR（`tccli vpc DescribeVpcs --region <Region>` 查询）
  - 集群 Pod CIDR（`DescribeClusters` 输出的 `ClusterNetworkSettings.ClusterCIDR`）
  - 集群 Service CIDR（`DescribeClusters` 输出的 `ClusterNetworkSettings.ServiceCIDR`）
  - 其他已开启注册节点集群的 `ClusterCIDR`
- 推荐掩码范围 `/16` 到 `/20`

### Progress 初始化流水线

`EnableExternalNodeSupport` 是异步操作，后台执行 9 步初始化流水线。用 `DescribeExternalNodeSupportConfig` 轮询 `Progress` 数组确认所有步骤完成：

| 步骤 | 名称 | 含义 |
|:---:|------|------|
| 1 | `EnsureNodeMasterCommunicate` | 确保节点与 Master 通信 |
| 2 | `EnsureInClusterAccess` | 确保集群内访问 |
| 3 | `EnsureExternalNodeProxy` | 部署外部节点代理 |
| 4 | `EnsureBootStrap` | 引导初始化 |
| 5 | `EnsureAdmissionController` | 部署准入控制器 |
| 6 | `AdaptNetworkPlugin` | 适配网络插件 |
| 7 | `InstallUnderlayCilium` | 安装 Underlay Cilium |
| 8 | `AdaptTkePlatApp` | 适配平台应用 |
| 9 | `CompleteEnable` | 完成启用 |

全部步骤 `Status=success` 后，`Status` 字段变为 `Enabled`。单步 `Status=failed` 时检查 `FailedReason` 定位根因。

### 轮询启用状态

```bash
# 轮询直到 Status=Enabled（注意：不是 Running）且 Progress 全部 success
tccli tke DescribeExternalNodeSupportConfig --region <Region> \
    --ClusterId <ClusterId>
# expected: Status: "Enabled"，Progress 中全部步骤 Status: "success"
```

```json
{
    "ClusterCIDR": "",
    "NetworkType": "HostNetwork",
    "SubnetId": "subnet-example",
    "Enabled": false,
    "AS": "",
    "SwitchIP": "",
    "Status": "Enabled",
    "FailedReason": "",
    "Master": "172.24.0.2",
    "Proxy": "172.24.0.19",
    "Progress": [
        {
            "Name": "EnsureNodeMasterCommunicate",
            "Status": "success",
            "StartAt": "2026-06-23T08:10:00Z",
            "EndAt": "2026-06-23T08:10:05Z",
            "Message": "",
            "Detail": ""
        },
        {
            "Name": "EnsureInClusterAccess",
            "Status": "success",
            "StartAt": "2026-06-23T08:10:05Z",
            "EndAt": "2026-06-23T08:10:07Z",
            "Message": "",
            "Detail": ""
        },
        {
            "Name": "EnsureExternalNodeProxy",
            "Status": "success",
            "StartAt": "2026-06-23T08:10:07Z",
            "EndAt": "2026-06-23T08:10:30Z",
            "Message": "",
            "Detail": ""
        },
        {
            "Name": "EnsureBootStrap",
            "Status": "success",
            "StartAt": "2026-06-23T08:10:30Z",
            "EndAt": "2026-06-23T08:10:31Z",
            "Message": "",
            "Detail": ""
        },
        {
            "Name": "EnsureAdmissionController",
            "Status": "success",
            "StartAt": "2026-06-23T08:10:32Z",
            "EndAt": "2026-06-23T08:10:35Z",
            "Message": "",
            "Detail": ""
        },
        {
            "Name": "AdaptNetworkPlugin",
            "Status": "success",
            "StartAt": "2026-06-23T08:10:35Z",
            "EndAt": "2026-06-23T08:10:37Z",
            "Message": "",
            "Detail": ""
        },
        {
            "Name": "InstallUnderlayCilium",
            "Status": "success",
            "StartAt": "2026-06-23T08:10:37Z",
            "EndAt": "2026-06-23T08:10:40Z",
            "Message": "",
            "Detail": ""
        },
        {
            "Name": "AdaptTkePlatApp",
            "Status": "success",
            "StartAt": "2026-06-23T08:10:40Z",
            "EndAt": "2026-06-23T08:11:00Z",
            "Message": "",
            "Detail": ""
        },
        {
            "Name": "CompleteEnable",
            "Status": "success",
            "StartAt": "2026-06-23T08:11:00Z",
            "EndAt": "2026-06-23T08:11:01Z",
            "Message": "",
            "Detail": ""
        }
    ],
    "EnabledPublicConnect": true,
    "PublicConnectUrl": "43.144.20.123:9000"
}
```

| 维度 | 命令 | 预期 |
|------|------|------|
| 启用状态 | `DescribeExternalNodeSupportConfig` 检查 `Status` | `Enabled`（**不是 `Running`**） |
| 初始化流水线 | 同上，检查 `Progress` | 全部 9 步 `Status: "success"` |
| 网络类型 | 同上，检查 `NetworkType` | 与开启时指定一致（`HostNetwork` 或 `CiliumBGP`） |
| CIDR | 同上，检查 `ClusterCIDR` | HostNetwork 模式为空字符串，CiliumBGP 模式与开启时指定一致 |
| 连通性 | 同上，检查 `Master` 和 `Proxy` | 非空 IP 地址 |

## 验证

### 验证配置正确性

```bash
tccli tke DescribeExternalNodeSupportConfig --region <Region> \
    --ClusterId <ClusterId> \
    | jq '{NetworkType, SubnetId, ClusterCIDR, Status, Progress}'
# expected: Status="Enabled"，Progress 全部 success
```

```json
{
    "NetworkType": "HostNetwork",
    "SubnetId": "subnet-example",
    "ClusterCIDR": "",
    "Status": "Enabled",
    "Progress": [
        {
            "Name": "EnsureNodeMasterCommunicate",
            "Status": "success",
            "StartAt": "2026-06-23T08:10:00Z",
            "EndAt": "2026-06-23T08:10:05Z",
            "Message": "",
            "Detail": ""
        },
        {
            "Name": "EnsureInClusterAccess",
            "Status": "success",
            "StartAt": "2026-06-23T08:10:05Z",
            "EndAt": "2026-06-23T08:10:07Z",
            "Message": "",
            "Detail": ""
        },
        {
            "Name": "EnsureExternalNodeProxy",
            "Status": "success",
            "StartAt": "2026-06-23T08:10:07Z",
            "EndAt": "2026-06-23T08:10:30Z",
            "Message": "",
            "Detail": ""
        },
        {
            "Name": "EnsureBootStrap",
            "Status": "success",
            "StartAt": "2026-06-23T08:10:30Z",
            "EndAt": "2026-06-23T08:10:31Z",
            "Message": "",
            "Detail": ""
        },
        {
            "Name": "EnsureAdmissionController",
            "Status": "success",
            "StartAt": "2026-06-23T08:10:32Z",
            "EndAt": "2026-06-23T08:10:35Z",
            "Message": "",
            "Detail": ""
        },
        {
            "Name": "AdaptNetworkPlugin",
            "Status": "success",
            "StartAt": "2026-06-23T08:10:35Z",
            "EndAt": "2026-06-23T08:10:37Z",
            "Message": "",
            "Detail": ""
        },
        {
            "Name": "InstallUnderlayCilium",
            "Status": "success",
            "StartAt": "2026-06-23T08:10:37Z",
            "EndAt": "2026-06-23T08:10:40Z",
            "Message": "",
            "Detail": ""
        },
        {
            "Name": "AdaptTkePlatApp",
            "Status": "success",
            "StartAt": "2026-06-23T08:10:40Z",
            "EndAt": "2026-06-23T08:11:00Z",
            "Message": "",
            "Detail": ""
        },
        {
            "Name": "CompleteEnable",
            "Status": "success",
            "StartAt": "2026-06-23T08:11:00Z",
            "EndAt": "2026-06-23T08:11:01Z",
            "Message": "",
            "Detail": ""
        }
    ]
}
```

```bash
# 检查 CIDR 与现有集群 CIDR 不冲突（仅 CiliumBGP 模式需要）
tccli tke DescribeClusters --region <Region> \
    | jq '.Clusters[] | {Name: .ClusterName, PodCIDR: .ClusterNetworkSettings.ClusterCIDR, SvcCIDR: .ClusterNetworkSettings.ServiceCIDR}'
# expected: 列出所有集群的 CIDR 信息，确认无冲突
```

**预期输出**：

```json
{
    "Name": "my-production-cluster",
    "PodCIDR": "10.0.0.0/16",
    "SvcCIDR": "10.96.0.0/20"
}
{
    "Name": "my-staging-cluster",
    "PodCIDR": "172.16.0.0/16",
    "SvcCIDR": "172.20.0.0/20"
}
```

> **判断标准**：注册节点的 `ClusterCIDR`（如 `10.200.0.0/16`）不应与任何集群的 `PodCIDR` 或 `SvcCIDR` 范围重叠。如果计划使用的 CIDR 不在上述列表中，则为安全。

## 清理

> **⚠️ 警告**：关闭注册节点支持会**影响已注册的所有节点**，节点将失去与集群的连接。
> 注册节点功能无法通过单独的 API 显式关闭，需删除所有注册外部节点和节点池后自动失效。
> 本页为概念说明页，通常不需要执行清理操作。

### 清理前状态检查

```bash
tccli tke DescribeExternalNodeSupportConfig --region <Region> \
    --ClusterId <ClusterId>
# 确认当前配置，记录 NetworkType、SubnetId、ClusterCIDR
```

本页为概念说明页，通常不需要清理。如确需关闭注册节点支持，需先删除所有注册节点和节点池。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `EnableExternalNodeSupport` 返回 `InvalidParameter.NetworkType` | 检查 JSON 中 `NetworkType` 值 | 填了控制台概念 `DirectConnect` 或 `专线版` | 用 `"HostNetwork"`（GR/VPC-CNI）或 `"CiliumBGP"`（Cilium-Overlay）。注意：是 `HostNetwork` 不是 `DirectConnect` |
| `EnableExternalNodeSupport` 返回 `InvalidParameter.ClusterCIDRSettings` | `tccli tke DescribeClusters --region <Region> \| jq '.Clusters[].ClusterNetworkSettings'` 列出已有 CIDR | Pod CIDR 与 VPC CIDR 或已有集群 CIDR 冲突 | 选择一个不与任何已有 CIDR 重叠的新 IP 段。如果是 HostNetwork 模式，移除 `ClusterCIDR` 字段 |
| `EnableExternalNodeSupport` 返回 `InvalidParameter.SubnetId` | `tccli vpc DescribeSubnets --region <Region> --SubnetIds '["<SubnetId>"]'` | 子网不存在或不属于可用 VPC | 用 `tccli vpc DescribeSubnets --region <Region>` 获取正确的子网 ID |
| `EnableExternalNodeSupport` 返回 `FailedOperation.OperationForbidden: cannot enable external node support when in task: enableExternalNode` | `tccli tke DescribeExternalNodeSupportConfig --region <Region> --ClusterId <ClusterId>` 检查当前 `Status` 和 `Progress` | 集群已有一个 `EnableExternalNodeSupport` 任务在执行中（可能由同 session 其他操作触发）。此为状态冲突，非命令参数错误 | 等待当前任务完成：轮询 `DescribeExternalNodeSupportConfig` 直到 `Status=Enabled` 且 `Progress` 全部 `success`。不要重复调用 `EnableExternalNodeSupport` |
| 轮询脚本永远等不到 `Status=Running` | `tccli tke DescribeExternalNodeSupportConfig --region <Region> --ClusterId <ClusterId>` 检查 `Status` 字段 | 错误地检查 `"Running"` 而非 `"Enabled"`——API 真实输出中启用完成后的 Status 是 `"Enabled"` | 将轮询目标值从 `"Running"` 改为 `"Enabled"` |

### 开启成功但状态异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 开启返回 RequestId 但 10 分钟后 `Status` 仍非 `Enabled` | `tccli tke DescribeExternalNodeSupportConfig --region <Region> --ClusterId <ClusterId>` 检查 `Status`、`FailedReason` 和 `Progress` | 后端初始化缓慢或某一步骤卡住。检查 `Progress` 中各步骤状态，定位卡住的具体步骤 | 根据 `FailedReason` 和 `Progress` 中失败步骤修正后重新开启；保留 region、ClusterId、RequestId、Progress 快照备查 |
| `Status` 为 `Failed` | 同上，检查 `FailedReason` 和 `Progress` | 通常为子网不足、CIDR 冲突、或网络不可达 | 查看 `FailedReason` → 修正配置后重新调用 `EnableExternalNodeSupport`；如无法解决 → 保留 region、ClusterId、RequestId、Progress 快照提交工单 |
| `Progress` 某步 `Status=failed` 但 `Status` 仍为 `Enabled` | 同上，检查具体失败的步骤 | 部分步骤失败但核心功能已启用（如 `AdaptTkePlatApp` 失败不影响基本网络） | 根据失败步骤评估影响 → 如需修复，重新调用 `EnableExternalNodeSupport`；保留 Progress 快照 |

## 下一步

- [创建注册节点（专线版）](https://cloud.tencent.com/document/product/457/57917) — 专线版创建注册节点完整流程
- [流量接入](https://cloud.tencent.com/document/product/457/79749) — 注册节点流量接入方式
- [注册节点常见问题](https://cloud.tencent.com/document/product/457/79750) — 排障与 FAQ

## 控制台替代

[TKE 控制台 → 集群 → 节点管理 → 注册节点](https://console.cloud.tencent.com/tke2/cluster)：在控制台中选择专线模式并配置 CIDR。
