# 集群启用 IPVS

> 对照官方：[集群启用 IPVS](https://cloud.tencent.com/document/product/457/32193) · page_id `32193`

## 概述

TKE 集群默认使用 iptables 模式的 kube-proxy 进行 Service 流量转发。IPVS（IP Virtual Server）基于 Linux 内核哈希表实现 L4 负载均衡，在 Service 数量较多时性能优于 iptables（O(1) 查找 vs O(n) 链表遍历）。IPVS 需在创建集群时通过 `ClusterAdvancedSettings.IPVS=true` 开启，创建后不可修改。

核心约束：
- 仅新建集群时可设置，存量集群不可修改 `Ipvs` 字段
- 全集群唯一模式（不可在部分节点混用 IPVS 和 iptables）
- 开启后不可关闭
- 仅 Kubernetes 1.10 及以上版本生效
- 若启用 Dataplane V2（eBPF），kube-proxy 被完全替代，IPVS 设置无效

## 前置条件

- [环境准备](../../../环境准备.md)

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
#    验证：执行 DescribeClusters 确认权限
tccli tke DescribeClusters --region <Region>
# expected: exit 0，返回集群列表（可为空）
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

### 资源检查

```bash
# 4. 确认目标集群存在并查看当前 IPVS 状态
tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterNetworkSettings.Ipvs 为 true 或 false
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

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 新建集群时选择 IPVS | `CreateCluster`（`ClusterAdvancedSettings.IPVS: true`） | 否 |
| 查看是否已启用 IPVS | `DescribeClusters`（`ClusterNetworkSettings.Ipvs` 字段） | 是 |
| 查看 kube-proxy 模式（数据面） | `kubectl get cm kube-proxy -n kube-system -o yaml` | 是 |

### 控制台概念与 API 参数映射

| 控制台概念 | API 参数 | 取值 | 说明 |
|-----------|---------|------|------|
| kube-proxy 转发模式 — ipvs | `ClusterAdvancedSettings.IPVS` | `true` | 创建时在高级设置中指定 |
| kube-proxy 转发模式 — iptables（默认） | `ClusterAdvancedSettings.IPVS` | `false` 或不传 | 默认值 |
| 集群网络设置 → 查询 IPVS 状态 | `ClusterNetworkSettings.Ipvs` | `true`/`false` | 创建后只读字段 |

## 关键字段说明

`DescribeClusters` 返回中与 IPVS 相关的核心字段，以及 `CreateCluster` 时的创建参数：

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `ClusterAdvancedSettings.IPVS`（创建时） | Boolean | 否 | `true` 启用 IPVS。仅创建时可设，创建后不可修改 | 不传默认 iptables |
| `ClusterNetworkSettings.Ipvs`（查询时） | Boolean | 仅输出 | `true` 已启用 IPVS / `false` 使用 iptables | — |
| `ClusterVersion` | String | 是（创建时） | 须 ≥ 1.10.0。`tccli tke DescribeVersions --region <Region>` 查询 | 低于 1.10 → IPVS 不生效 |
| `DataPlaneV2` | Boolean | 否 | Dataplane V2 与 IPVS 互斥。`true` 时 kube-proxy 不安装 | 同时启用 → IPVS 无效（按 Dataplane V2 为准） |

## 操作步骤

### 步骤 1：查询存量集群 IPVS 状态

选择依据：`DescribeClusters` 返回的 `ClusterNetworkSettings.Ipvs` 字段反映集群当前的 kube-proxy 转发模式。`Ipvs: false` 表示使用默认 iptables 模式。

```bash
# 查询单个集群的 IPVS 状态
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]' \
    | jq '.Clusters[0].ClusterNetworkSettings | {Ipvs, DataPlaneV2, KubeProxyMode}'
# expected: Ipvs 为 true 或 false，DataPlaneV2 为 true 或 false
```

预期输出（未启用 IPVS 的 GR 集群）：

```json
{
    "Ipvs": false,
    "DataPlaneV2": false,
    "KubeProxyMode": ""
}
```

### 步骤 2：列出所有启用 IPVS 的集群

```bash
tccli tke DescribeClusters --region <Region> \
    | jq '.Clusters[] | select(.ClusterNetworkSettings.Ipvs == true) | {ClusterId, ClusterName, Ipvs: .ClusterNetworkSettings.Ipvs, KubeProxyMode: .ClusterNetworkSettings.KubeProxyMode}'
# expected: 返回 Ipvs=true 的集群列表（可为空）
```

### 步骤 3：创建启用 IPVS 的新集群

最小创建配置（仅含网络必填字段 + IPVS）：

`cluster-ipvs-minimal.json`：

```json
{
    "ClusterType": "MANAGED_CLUSTER",
    "ClusterBasicSettings": {
        "ClusterName": "CLUSTER_NAME",
        "ClusterVersion": "1.30.0",
        "VpcId": "VPC_ID",
        "SubnetId": "SUBNET_ID"
    },
    "ClusterCIDRSettings": {
        "ClusterCIDR": "10.200.0.0/16",
        "ServiceCIDR": "10.201.0.0/20"
    },
    "ClusterAdvancedSettings": {
        "IPVS": true
    }
}
```

```bash
tccli tke CreateCluster --region <Region> --cli-input-json file://cluster-ipvs-minimal.json
# expected: exit 0，返回 ClusterId
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `CLUSTER_NAME` | 集群名称 | 长度 1-60 字符 | 自定义 |
| `VPC_ID` | VPC 实例 ID | 格式 `vpc-xxxxxxxx` | `tccli vpc DescribeVpcs --region <Region>` |
| `SUBNET_ID` | 子网 ID | 属于目标 VPC | `tccli vpc DescribeSubnets --region <Region>` |
| `REGION` | 地域 | 如 `ap-guangzhou` | `tccli tke DescribeRegions` |

> **注意**：`IPVS: true` 仅在 `ClusterAdvancedSettings` 中设置（注意全大写）。查询时返回字段为小写的 `Ipvs`（`ClusterNetworkSettings.Ipvs`）。

增强配置（加上标签、删除保护、运行时等可选字段）：

`cluster-ipvs-enhanced.json`：

```json
{
    "ClusterType": "MANAGED_CLUSTER",
    "ClusterBasicSettings": {
        "ClusterName": "CLUSTER_NAME",
        "ClusterVersion": "1.30.0",
        "VpcId": "VPC_ID",
        "SubnetId": "SUBNET_ID",
        "ClusterDescription": "CLUSTER_DESCRIPTION"
    },
    "ClusterCIDRSettings": {
        "ClusterCIDR": "10.200.0.0/16",
        "ServiceCIDR": "10.201.0.0/20",
        "MaxNodePodNum": 64,
        "MaxClusterServiceNum": 4096
    },
    "ClusterAdvancedSettings": {
        "IPVS": true,
        "ContainerRuntime": "containerd",
        "DeletionProtection": true
    }
}
```

## 验证

### 控制面（tccli）

创建集群是异步操作。轮询直到状态为 `Running`，然后核对 IPVS 已生效：

```bash
# 轮询集群状态
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterStatus: "Running"
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

```bash
# 验证 IPVS 已启用
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]' \
    | jq '.Clusters[0].ClusterNetworkSettings.Ipvs'
# expected: true
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

### 数据面（kubectl）

集群就绪后，获取 kubeconfig 并确认 kube-proxy 模式：

```bash
tccli tke DescribeClusterKubeconfig --region <Region> \
    --ClusterId CLUSTER_ID \
    | jq -r '.Kubeconfig' > ~/.kube/config

# 确认 kube-proxy 使用 IPVS 模式
kubectl get cm kube-proxy -n kube-system -o yaml | grep mode
# expected: mode: ipvs
```

```json
{
  "Kubeconfig": "<Kubeconfig>",
  "RequestId": "<RequestId>"
}
```

### IPVS 验证维度总览

| 验证维度 | 命令 | 预期 |
|---------|------|------|
| 状态 | `DescribeClusters → .ClusterStatus` | `Running` |
| `Ipvs` | `DescribeClusters → .Ipvs` | `true` |
| `KubeProxyMode` | `DescribeClusters → .KubeProxyMode` | `ipvs` 或 `""` |
| kube-proxy 配置 | `kubectl get cm kube-proxy -n kube-system -o yaml \| grep mode` | `mode: ipvs` |
| IPVS 规则 | `kubectl get svc --all-namespaces` 后 `ipvsadm -Ln`（需登录节点） | 有 IPVS 转发规则 |

## 清理

本页为说明与只读查询（IPVS 为不可逆配置，无法关闭）。若测试集群不再需要，执行删除命令：

> **警告**：`DeleteCluster` 配合 `InstanceDeleteMode: "terminate"` 会**释放/删除**集群关联的所有 CVM 实例。生产环境执行前务必确认。

```bash
# 清理前状态检查
tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
# 确认是待删除的目标集群

# 删除集群
tccli tke DeleteCluster --region <Region> \
    --ClusterId CLUSTER_ID \
    --InstanceDeleteMode terminate

# 验证已删除
tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
# expected: ResourceNotFound 或空列表
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

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DescribeClusters` 返回 `UnauthorizedOperation` | `tccli configure list` 检查凭据 | 子账号缺少 `tke:DescribeClusters` 权限（此为环境限制，非命令错误） | 联系主账号授予 `QcloudTKEFullAccess` 或最小权限 `tke:DescribeClusters` |
| `CreateCluster` 中 `IPVS` 参数不生效 | 检查 JSON 中是否用了小写 `ipvs` 或放在 `ClusterNetworkSettings` 下 | 创建时参数名是大写 `IPVS`，在 `ClusterAdvancedSettings` 下（而非 `ClusterNetworkSettings`） | 改为 `ClusterAdvancedSettings.IPVS: true` |
| `kubectl get cm kube-proxy` 返回 `not found` | `tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]' \| jq '.ClusterNetworkSettings.DataPlaneV2'` | 集群启用了 Dataplane V2，kube-proxy 被 eBPF 替代 | 确认 Dataplane V2 状态；若为 `true`，kube-proxy 不存在属正常行为 |

### IPVS 生效异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `Ipvs: false` 但创建时指定了 `IPVS: true` | 检查创建 JSON 中参数位置（是否在 `ClusterAdvancedSettings` 下）和大小写 | 参数位置错误（放在 `ClusterNetworkSettings` 下）或大小写错误 | 确保 `ClusterAdvancedSettings.IPVS` 为 `true`（大写）。保留 RequestId + 创建 JSON |
| 存量集群想改用 IPVS | `tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]' \| jq '.ClusterNetworkSettings.Ipvs'` | `Ipvs` 字段创建后不可修改 | 无法修改；需新建集群，在新集群上启用 IPVS |
| 启用 Dataplane V2 后 IPVS 不生效 | `tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]' \| jq '.ClusterNetworkSettings.{Ipvs, DataPlaneV2, KubeProxyMode}'` | Dataplane V2 不安装 kube-proxy，IPVS 设置不适用 | 二选一：Dataplane V2（高性能 eBPF）或 IPVS（传统内核模式） |

## 下一步

- [创建集群](../../集群管理/创建集群/tccli%20操作.md) -- 完整集群创建步骤（含所有高级设置）
- [Dataplane V2 介绍](../Dataplane%20V2%20%E8%BD%AC%E5%8F%91%E6%A8%A1%E5%BC%8F/Dataplane%20V2%20%E4%BB%8B%E7%BB%8D/tccli%20%E6%93%8D%E4%BD%9C.md) -- 下一代 eBPF 数据面方案
- [连接集群](../../集群管理/连接集群/tccli%20操作.md) -- 获取 kubeconfig 连接集群
- [GlobalRouter 模式介绍](../GlobalRouter%20%E6%A8%A1%E5%BC%8F/GlobalRouter%20%E6%A8%A1%E5%BC%8F%E4%BB%8B%E7%BB%8D/tccli%20%E6%93%8D%E4%BD%9C.md) -- GR 模式下的转发机制

## 控制台替代

[TKE 控制台 → 集群](https://console.cloud.tencent.com/tke2/cluster) 新建 → 网络配置 → 高级设置 → kube-proxy 转发模式选择「ipvs」。
