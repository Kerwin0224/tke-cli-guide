# 使用 Dataplane V2（tccli）

> 对照官方：[使用 Dataplane V2](https://cloud.tencent.com/document/product/457/113228) · page_id `113228`

## 概述

Dataplane V2 是 TKE 的新一代容器网络转发模式，基于 eBPF 技术替代传统 kube-proxy（iptables/IPVS）实现 Service 流量转发。启用后集群不再安装 kube-proxy，Service ClusterIP 流量由 VPC-CNI 数据面直接处理，降低转发延迟和规则复杂度。必须在**创建集群时**启用，存量集群不可切换。

### Dataplane V2 与传统 kube-proxy 对比

| 维度 | Dataplane V2 | kube-proxy (iptables) | kube-proxy (IPVS) |
|------|:--:|:--:|:--:|
| 转发机制 | eBPF 内核态直接转发 | iptables 规则链匹配 | IPVS 哈希表查找 |
| 大规模 Service 性能 | 优异（O(1) 查找） | 差（O(n) 规则遍历） | 良好（O(1) 查找） |
| 规则数量影响 | 几乎无影响 | 随 Service 数线性增长 | 影响较小 |
| 运维可见性 | 无 iptables 规则 | `iptables -L -n -t nat` | `ipvsadm -L -n` |
| kube-proxy Pod | 不存在 | 每节点 1 个 DaemonSet Pod | 每节点 1 个 DaemonSet Pod |

## 前置条件

- [环境准备](../../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId、secretKey、region 均已配置

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:DescribeClusters, tke:CreateCluster, tke:DescribeVersions
#    vpc:DescribeVpcs, vpc:DescribeSubnets, cvm:DescribeInstances
#    验证：执行 DescribeClusters 确认权限
tccli tke DescribeClusters --region <Region>
# expected: exit 0，返回集群列表（可为空）
```

### 资源检查

```bash
# 4. 查询可用 K8s 版本（Dataplane V2 要求 ≥1.24）
tccli tke DescribeVersions --region <Region>
# expected: 返回 VersionInstanceSet，含 ≥1.24 版本
```

**预期输出**：

```json
{
  "VersionInstanceSet": [
    {"Version": "1.24.4", "Remark": "SUITABLE_FOR_INDEPENDENT_CLUSTER"},
    {"Version": "1.26.1", "Remark": "SUITABLE_FOR_INDEPENDENT_CLUSTER"},
    {"Version": "1.28.3", "Remark": "SUITABLE_FOR_INDEPENDENT_CLUSTER"},
    {"Version": "1.30.0", "Remark": "SUITABLE_FOR_INDEPENDENT_CLUSTER"},
    {"Version": "1.32.2", "Remark": "SUITABLE_FOR_INDEPENDENT_CLUSTER"}
  ]
}
```

```bash
# 5. 查询操作系统镜像（Dataplane V2 需 TencentOS Server 3.1/3.2）
tccli tke DescribeOSImages --region <Region>
# expected: 返回含 tlinux3.1 或 tlinux3.2 的镜像列表
```

> **已知行为**：部分 region（如 ap-guangzhou）DescribeOSImages 返回空列表属正常。生产环境请在目标 region 确认镜像可用性。

```bash
# 6. 确认当前集群 Dataplane V2 状态
tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
# expected: DataPlaneV2=false（当前未启用）
```

**预期输出**：

```json
{
  "TotalCount": 1,
  "Clusters": [
    {
      "ClusterId": "cls-example",
      "ClusterName": "my-dpv2-cluster",
      "ClusterVersion": "1.32.2",
      "ClusterOs": "ubuntu16.04.1 LTSx86_64",
      "ClusterType": "MANAGED_CLUSTER",
      "ClusterNetworkSettings": {
        "ClusterCIDR": "10.0.0.0/16",
        "IgnoreClusterCIDRConflict": false,
        "MaxNodePodNum": 64,
        "MaxClusterServiceNum": 4096,
        "Ipvs": false,
        "VpcId": "vpc-example",
        "Cni": true,
        "KubeProxyMode": "",
        "ServiceCIDR": "10.1.0.0/20",
        "Subnets": ["subnet-example-1"],
        "IgnoreServiceCIDRConflict": false,
        "IsDualStack": false,
        "Ipv6ServiceCIDR": "",
        "CiliumMode": "",
        "SubnetId": "subnet-example",
        "DataPlaneV2": false
      },
      "ClusterNodeNum": 0,
      "ClusterStatus": "Running",
      "Property": "{\"NodeNameType\":\"lan-ip\",\"NetworkType\":\"GR\",\"VpcCniType\":\"tke-route-eni\",\"IsSupportMultiENI\":true,\"IsNetworkWithApp\":true}",
      "ContainerRuntime": "containerd",
      "DeletionProtection": false,
      "ClusterLevel": "L5"
    }
  ],
  "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

> **注意**：`DescribeClusters` 返回值中 `DataPlaneV2` 出现在 `ClusterNetworkSettings` 内——这是**查询输出**。创建集群时该参数位置不同，见[关键字段说明](#关键字段说明)。

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看 Dataplane V2 状态 | `DescribeClusters`（`DataPlaneV2`） | 是 |
| 查看集群版本 | `DescribeClusters`（`ClusterVersion`） | 是 |
| 启用 Dataplane V2 | `CreateCluster`（`DataPlaneV2=true`） | 否（创建时一次性选择） |

## 关键字段说明

`CreateCluster` 中 Dataplane V2 相关参数。`DataPlaneV2`、`NetworkType`、`VpcCniType` 均位于 `ClusterAdvancedSettings` 下（**不是** `ClusterNetworkSettings`——后者仅出现在 `DescribeClusters` 返回值中，不作为 `CreateCluster` 入参）。

> `ClusterNetworkSettings` 仅在 `DescribeClusters` 返回值中出现，不作为 `CreateCluster` 入参——这是最常见的参数路径错误。

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `ClusterAdvancedSettings.DataPlaneV2` | Boolean | 否 | `true`/`false`，默认 `false` | 忘设 → 使用传统 kube-proxy，**创建后不可切换** |
| `ClusterAdvancedSettings.NetworkType` | String | 条件必填 | 须为 `"VPC-CNI"`（Dataplane V2 仅在 VPC-CNI 下支持） | 非 `VPC-CNI` → `DataPlaneV2=true` 被忽略，仍用 kube-proxy |
| `ClusterAdvancedSettings.VpcCniType` | String | 条件必填 | 须为 `"tke-route-eni"`（共享网卡多 IP 模式） | 其他模式 → 不兼容，集群创建可能失败 |
| `ClusterBasicSettings.ClusterVersion` | String | 是 | 须 ≥ `1.24`。通过 `tccli tke DescribeVersions` 查询 | 低于 1.24 → `InvalidParameter.ClusterVersion` |
| `ClusterCIDRSettings.EniSubnetIds` | Array | 条件必填 | VPC-CNI 子网 ID 列表，用于 Pod IP 分配。通过 `tccli vpc DescribeSubnets` 获取 | 子网无效或不可用 → 创建失败；误用 `ClusterNetworkSettings.Subnets`（该字段在 CreateCluster 中不存在）→ 参数被忽略 |

`DescribeClusters` 返回值中 Dataplane V2 相关字段：

| 字段 | `true` 含义 | `false` 含义 |
|------|------------|-------------|
| `ClusterNetworkSettings.DataPlaneV2` | 已启用，集群无 kube-proxy | 未启用，使用传统 kube-proxy |

## 操作步骤

### 使用限制

- 仅支持 **Kubernetes ≥ 1.24** 版本
- 仅支持 **TencentOS Server 3.1 / 3.2** 操作系统；启用后不可添加其他 OS 节点
- 仅支持 **VPC-CNI 共享网卡多 IP 模式**（`VpcCniType=tke-route-eni`）
- 不支持 **NodeLocalDNSCache**
- 默认节点上限 500，超出需 [提交工单](https://cloud.tencent.com/online-service)
- **新集群创建时启用，存量集群不可切换**

### 步骤 1：查看当前集群是否启用 Dataplane V2

```bash
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]'
# expected: DataPlaneV2=false（当前集群未启用）
```

**预期输出**（以 `cls-example` 为例）：

```json
{
  "TotalCount": 1,
  "Clusters": [
    {
      "ClusterId": "cls-example",
      "ClusterName": "my-dpv2-cluster",
      "ClusterVersion": "1.32.2",
      "ClusterType": "MANAGED_CLUSTER",
      "ClusterNetworkSettings": {
        "ClusterCIDR": "10.0.0.0/16",
        "MaxNodePodNum": 64,
        "MaxClusterServiceNum": 4096,
        "Ipvs": false,
        "VpcId": "vpc-example",
        "Cni": true,
        "KubeProxyMode": "",
        "ServiceCIDR": "10.1.0.0/20",
        "Subnets": ["subnet-example-1"],
        "SubnetId": "subnet-example",
        "DataPlaneV2": false
      },
      "ClusterNodeNum": 0,
      "ClusterStatus": "Running",
      "Property": "{\"NodeNameType\":\"lan-ip\",\"NetworkType\":\"GR\",\"VpcCniType\":\"tke-route-eni\",\"IsSupportMultiENI\":true,\"IsNetworkWithApp\":true}",
      "ContainerRuntime": "containerd",
      "DeletionProtection": false,
      "ClusterLevel": "L5"
    }
  ],
  "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

- `DataPlaneV2: true` -- 已启用，kube-proxy 不在集群中运行
- `DataPlaneV2: false` -- 未启用，使用传统 kube-proxy
- 上例中 `ClusterNetworkSettings.DataPlaneV2=false`，且 `Property` 中 `NetworkType="GR"`（非 VPC-CNI）——这意味着即使 `Cni=true`，该集群也不满足 Dataplane V2 的网络模式要求。需要新建 VPC-CNI 集群。

### 步骤 2：创建启用 Dataplane V2 的集群

#### 选择依据

- **DataPlaneV2=true**：适合大规模 Service 场景（数百+ Service），减少 iptables 规则爆炸带来的转发延迟。当前共享集群使用 GR 模式，Dataplane V2 需 VPC-CNI，因此需**新建集群**。
- **为什么选 VPC-CNI**：Dataplane V2 仅在 VPC-CNI 共享网卡模式下支持，GR 模式无法启用。
- **存量不可切换**：启用 Dataplane V2 需在创建时决定，已运行的 GR 集群不可切换。

`create-dpv2-minimal.json`：

```json
{
  "ClusterType": "MANAGED_CLUSTER",
  "ClusterBasicSettings": {
    "ClusterName": "CLUSTER_NAME",
    "ClusterVersion": "1.32.2",
    "VpcId": "VPC_ID",
    "SubnetId": "SUBNET_ID"
  },
  "ClusterCIDRSettings": {
    "ClusterCIDR": "10.1.0.0/16",
    "ServiceCIDR": "10.2.0.0/20",
    "EniSubnetIds": ["SUBNET_ID_1"]
  },
  "ClusterAdvancedSettings": {
    "NetworkType": "VPC-CNI",
    "VpcCniType": "tke-route-eni",
    "DataPlaneV2": true
  }
}
```

```bash
tccli tke CreateCluster --region <Region> --cli-input-json file://create-dpv2-minimal.json
# expected: exit 0，返回 ClusterId
```

**预期输出**：

```json
{
    "ClusterId": "cls-example",
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `CLUSTER_NAME` | 集群名称 | 长度 1-60 字符，字母开头 | 自定义 |
| `VPC_ID` | VPC ID | 格式 `vpc-xxxxxxxx` | `tccli vpc DescribeVpcs --region <Region>` |
| `SUBNET_ID` | 子网 ID（Master） | 在目标 VPC 内 | `tccli vpc DescribeSubnets --region <Region>` |
| `SUBNET_ID_1` | VPC-CNI EniSubnetId | 用于 Pod IP 分配（EniSubnetIds 数组） | 同上 |

### 步骤 3：轮询确认 Dataplane V2 已启用

集群创建是异步操作。轮询直到状态为 `Running`：

```bash
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterStatus="Running", DataPlaneV2=true
```

**预期输出**（启用 Dataplane V2 后）：

```json
{
  "TotalCount": 1,
  "Clusters": [
    {
      "ClusterId": "cls-example",
      "ClusterName": "my-dpv2-cluster",
      "ClusterVersion": "1.32.2",
      "ClusterType": "MANAGED_CLUSTER",
      "ClusterNetworkSettings": {
        "ClusterCIDR": "10.1.0.0/16",
        "MaxNodePodNum": 64,
        "MaxClusterServiceNum": 4096,
        "Ipvs": false,
        "VpcId": "VPC_ID",
        "Cni": true,
        "KubeProxyMode": "",
        "ServiceCIDR": "10.2.0.0/20",
        "Subnets": ["SUBNET_ID_1"],
        "SubnetId": "SUBNET_ID",
        "DataPlaneV2": true
      },
      "ClusterNodeNum": 0,
      "ClusterStatus": "Running",
      "Property": "{\"NodeNameType\":\"lan-ip\",\"NetworkType\":\"VPC-CNI\",\"VpcCniType\":\"tke-route-eni\",\"IsSupportMultiENI\":true,\"IsNetworkWithApp\":true}",
      "ContainerRuntime": "containerd",
      "DeletionProtection": false,
      "ClusterLevel": "L5"
    }
  ],
  "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

> **验证要点**：`ClusterNetworkSettings.DataPlaneV2=true`，`Property` 中 `NetworkType="VPC-CNI"`、`VpcCniType="tke-route-eni"`。最长等待约 10 分钟，超时参见[排障](#排障)。

### 步骤 4：获取 kubeconfig 并确认 Dataplane V2 效果（数据面，须同 VPC 网络可达）

```bash
# 获取内网 kubeconfig（公网端点可能被 CAM 策略禁用）
tccli tke DescribeClusterKubeconfig --region <Region> \
    --ClusterId CLUSTER_ID \
    --IsExtranet false \
    | jq -r '.Kubeconfig' > ~/.kube/config
# expected: exit 0，kubeconfig 写入成功
```

Dataplane V2 启用后，kube-proxy 不会安装：

```bash
# 确认 kube-proxy 不存在
kubectl get ds -n kube-system | grep kube-proxy
# expected: 无输出（kube-proxy 未安装）

# Service ClusterIP 流量由 VPC-CNI 数据面直接处理
kubectl get svc -A
# expected: Service 列表正常，ClusterIP 可达
```

## 验证

### 控制面（tccli）

```bash
# 确认 Dataplane V2 已启用
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]'
# expected: DataPlaneV2=true, ClusterStatus="Running"
```

**预期输出**（启用 Dataplane V2 的集群）：

```json
{
  "TotalCount": 1,
  "Clusters": [
    {
      "ClusterId": "cls-example",
      "ClusterName": "my-dpv2-cluster",
      "ClusterVersion": "1.32.2",
      "ClusterNetworkSettings": {
        "ClusterCIDR": "10.1.0.0/16",
        "ServiceCIDR": "10.2.0.0/20",
        "MaxNodePodNum": 64,
        "MaxClusterServiceNum": 4096,
        "Ipvs": false,
        "VpcId": "VPC_ID",
        "Cni": true,
        "KubeProxyMode": "",
        "Subnets": ["SUBNET_ID_1"],
        "SubnetId": "SUBNET_ID",
        "DataPlaneV2": true
      },
      "ClusterNodeNum": 0,
      "ClusterStatus": "Running",
      "Property": "{\"NodeNameType\":\"lan-ip\",\"NetworkType\":\"VPC-CNI\",\"VpcCniType\":\"tke-route-eni\"}",
      "ContainerRuntime": "containerd",
      "DeletionProtection": false
    }
  ],
  "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

### 数据面（须同 VPC 网络可达）

```bash
# 1. 确认 kube-proxy 不在集群中
kubectl get ds -n kube-system
# expected: 列表中无 kube-proxy

# 2. 确认 Service 流量正常
kubectl get svc -A
# expected: 所有 Service ClusterIP 正常
```

### 验证维度

| 维度 | 命令 | 预期 |
|------|------|------|
| Dataplane V2 开关 | `DescribeClusters` → `DataPlaneV2` | `true` |
| VPC-CNI 模式 | `DescribeClusters` → `Cni` | `true` |
| 共享网卡模式 | `DescribeClusters` → `VpcCniType` | `"tke-route-eni"` |
| kube-proxy 不存在 | `kubectl get ds -n kube-system` | 无 kube-proxy 条目 |
| K8s 版本 | `DescribeClusters` → `ClusterVersion` | ≥ 1.24 |

## 清理

> **⚠️ 警告**：Dataplane V2 为集群级不可逆配置，创建后无法降级为 kube-proxy 模式。新建启用 Dataplane V2 的测试集群会产生集群管理费（L5 最小规格）+ 节点 CVM 费用。
> `DeleteCluster` 配合 `InstanceDeleteMode terminate` 会**级联删除**所有关联 CVM 实例及磁盘，不可恢复。生产环境执行前务必先 `DescribeClusters` 确认集群名称和 ID。

```bash
# 1. 清理前确认
tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
# 确认是待删除的测试集群
```

```bash
# 2. 删除集群
tccli tke DeleteCluster --region <Region> \
    --ClusterId CLUSTER_ID \
    --InstanceDeleteMode terminate
# ⚠️ InstanceDeleteMode terminate 会级联删除所有关联 CVM 和磁盘
```

```bash
# 3. 验证已删除
tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
# expected: ResourceNotFound 或空列表
```

> 共享环境集群 `MANAGED_CLUSTER`（GR 模式，`DataPlaneV2=false`）不可切换，无需清理。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DescribeClusters` → `DataPlaneV2=false` | 查看 `Property` 中 `NetworkType` 和 `VpcCniType`：`tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'` | 存量集群未在创建时启用 Dataplane V2（如 `NetworkType="GR"`），或 `NetworkType` 非 `VPC-CNI` | Dataplane V2 仅创建时生效，存量集群不可切换。需新建 VPC-CNI 集群并在 `ClusterAdvancedSettings.DataPlaneV2=true` |
| `CreateCluster` 中 `DataPlaneV2=true` 但未生效 | 检查 `ClusterAdvancedSettings.NetworkType` 是否为 `"VPC-CNI"`，`VpcCniType` 是否为 `"tke-route-eni"` | `NetworkType` 非 `VPC-CNI` 或 `VpcCniType` 非 `tke-route-eni` 时 `DataPlaneV2` 参数被忽略 | 确保 `ClusterAdvancedSettings.NetworkType="VPC-CNI"` + `VpcCniType="tke-route-eni"` |
| `CreateCluster` 返回 `InvalidParameter.ClusterVersion` | 检查 `ClusterBasicSettings.ClusterVersion` 值 | 版本 < 1.24，不支持 Dataplane V2 | 通过 `tccli tke DescribeVersions --region <Region>` 确认可用版本，使用 ≥ 1.24 版本 |
| `CreateCluster` 参数 `ClusterNetworkSettings` 不生效 | 在 `CreateCluster` 请求 JSON 中搜索 `ClusterNetworkSettings` 字段，对照[关键字段说明](#关键字段说明)中的正确父路径 | `ClusterNetworkSettings` 不是 `CreateCluster` 的入参字段，仅出现在 `DescribeClusters` 返回值中 | 改用正确结构：`ClusterAdvancedSettings.DataPlaneV2`、`ClusterAdvancedSettings.NetworkType`、`ClusterCIDRSettings.EniSubnetIds` |
| `CreateClusterEndpoint` 返回 `InvalidParameter.Param`（公网端点） | 错误信息中查找 CAM 策略条件：检查 `strategyId` 和条件 key | 组织级 CAM 策略（如 `strategyId:240463971`）以 `tke:clusterExtranetEndpoint=true` 条件硬拒绝公网端点创建 | 此为环境限制，非命令错误。改用内网端点：`CreateClusterEndpoint --IsExtranet false`。kubectl 需同 VPC CVM 或通过 IOA/VPN/专线访问 |
| 创建后节点加入失败 | `tccli tke DescribeClusterInstances --region <Region> --ClusterId CLUSTER_ID` | OS 非 TencentOS Server 3.1/3.2。Dataplane V2 启用后不可添加其他 OS 节点 | 仅使用 TencentOS Server 3.1 (TK4) 或 3.2 创建节点。通过 `tccli tke DescribeOSImages --region <Region>` 查询可用镜像 |

### 功能异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| kube-proxy 不可见 | `kubectl get ds -n kube-system` | 正常行为：Dataplane V2 模式下不安装 kube-proxy | 无需修复，Service 转发由 eBPF 数据面直接处理 |
| NodeLocalDNSCache 报错 | `kubectl describe pod -n kube-system -l k8s-app=node-local-dns` | Dataplane V2 当前不支持 NodeLocalDNSCache | 移除 NodeLocalDNSCache，使用 CoreDNS 默认模式 |
| 节点数超过 500 | `tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]' \| jq '.Clusters[0].ClusterNodeNum'` | Dataplane V2 默认节点上限 500 | [提交工单](https://cloud.tencent.com/online-service) 申请提升 |
| 无法回退到 kube-proxy | -- | Dataplane V2 为创建时不可逆配置 | 新建集群前确认需求；测试环境充分验证后再用于生产 |
| kubectl 无法连接集群 | `tccli tke DescribeClusterEndpoints --ClusterId CLUSTER_ID --region <Region>` | 集群仅有内网端点（公网端点被 CAM 策略禁用），本机不在集群 VPC 内 | 在同 VPC 的 CVM 上执行 kubectl，或通过 IOA/VPN/专线访问内网端点。参见 `tccli tke DescribeClusterKubeconfig --IsExtranet false` |

## 下一步

- [Dataplane V2 介绍](../../Dataplane%20V2%20转发模式/Dataplane%20V2%20介绍/tccli%20操作.md) -- Dataplane V2 原理
- [VPC-CNI 模式介绍](../../VPC-CNI%20模式/VPC-CNI%20模式介绍/tccli%20操作.md) -- VPC-CNI 网络
- [集群启用 IPVS](../../集群启用%20IPVS/tccli%20操作.md) -- kube-proxy IPVS 模式（非 Dataplane V2 方案）
- [创建集群](../../../集群管理/创建集群/tccli%20操作.md) -- 完整创建参数

## 控制台替代

[容器服务控制台 → 新建集群](https://console.cloud.tencent.com/tke2/cluster/create) → 网络配置 → 选择 VPC-CNI → 高级设置 → 勾选「Dataplane v2」。
