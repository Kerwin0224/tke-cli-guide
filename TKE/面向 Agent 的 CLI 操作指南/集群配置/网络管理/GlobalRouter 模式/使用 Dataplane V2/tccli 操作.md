# 使用 Dataplane V2（tccli）

> 对照官方：[使用 Dataplane V2](https://cloud.tencent.com/document/product/457/113228) · page_id `113228`

## 概述

Dataplane V2 是 TKE 集群的替代网络转发模式。启用后，集群不再安装 kube-proxy 组件，由底层数据面直接完成 Service 流量转发，消除 kube-proxy 带来的性能开销和规则膨胀问题。

**核心约束：**

- **仅创建时可设**：DataPlaneV2 通过 `ClusterAdvancedSettings.DataPlaneV2` 在 `CreateCluster` 时指定。集群创建后不可切换，如需变更需重建集群。
- **无 kube-proxy**：启用后集群无 kube-proxy DaemonSet，控制台「工作负载 > DaemonSet」中不可见 `kube-proxy`。
- **操作系统限制**：仅支持 TencentOS Server 3.1（TK4 内核）或 TencentOS Server 3.2。
- **Kubernetes 版本要求**：集群 Kubernetes 版本 >= 1.24。
- **节点上限**：默认 500 节点，如需提升需提交工单。
- **不支持 NodeLocalDNSCache**。

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
#    tke:DescribeClusters, tke:CreateCluster
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

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看 Dataplane V2 状态 | `DescribeClusters`（查看 `DataPlaneV2` 字段） | 是 |
| 创建启用 Dataplane V2 的集群 | `CreateCluster`（`ClusterAdvancedSettings.DataPlaneV2=true`） | 否 |
| 切换 Dataplane V2 开关 | 不支持（创建后不可切换） | — |

## 关键字段说明

`DescribeClusters` 返回体中 `ClusterNetworkSettings` 内含 `DataPlaneV2` 字段：

| 字段 | 类型 | 含义 | 取值与约束 |
|------|------|------|------|
| `DataPlaneV2` | Boolean | 是否启用 Dataplane V2 转发模式 | `true` 已启用；`false` 未启用（默认） |

`CreateCluster` 入参中 `ClusterAdvancedSettings` 内含 `DataPlaneV2` 字段：

| 字段 | 类型 | 含义 | 取值与约束 |
|------|------|------|------|
| `DataPlaneV2` | Boolean | 创建时指定是否启用 Dataplane V2 | `true` 启用；不传或 `false` 不启用。创建后不可变更 |

## 操作步骤

### 查询集群 Dataplane V2 状态

```bash
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]' \
    | jq '.Clusters[0].ClusterNetworkSettings.DataPlaneV2'
# expected: false（未启用）或 true（已启用）
```

**未启用示例输出：**

```json
false
```

**已启用示例输出（含完整网络配置）：**

```json
{
    "ClusterId": "cls-example",
    "ClusterNetworkSettings": {
        "ClusterCIDR": "10.100.0.0/16",
        "IgnoreClusterCIDRConflict": false,
        "MaxNodePodNum": 64,
        "MaxClusterServiceNum": 4096,
        "Ipvs": false,
        "VpcId": "vpc-example",
        "Cni": true,
        "KubeProxyMode": "",
        "ServiceCIDR": "10.101.0.0/20",
        "Subnets": ["subnet-example"],
        "IgnoreServiceCIDRConflict": false,
        "IsDualStack": false,
        "Ipv6ServiceCIDR": "",
        "CiliumMode": "",
        "SubnetId": "",
        "DataPlaneV2": true
    },
    "Property": "{\"NetworkType\":\"VPC-CNI\"}"
}
```

> **注意**：上述示例中的 `Region`、`ClusterId` 等均为占位符，请替换为实际值。所有示例仅供格式参考，不作真值依据。

### 创建启用 Dataplane V2 的集群

```bash
tccli tke CreateCluster \
    --region <Region> \
    --ClusterType "MANAGED_CLUSTER" \
    --ClusterName "CLUSTER_NAME" \
    --VpcId "VPC_ID" \
    --ClusterCIDR "CLUSTER_CIDR" \
    --ClusterVersion "CLUSTER_VERSION" \
    --ClusterAdvancedSettings '{"DataPlaneV2": true}'
# expected: exit 0，返回 ClusterId
```

> **关键约束**：`DataPlaneV2` 仅 `CreateCluster` 时可设，创建后不支持修改。如需为存量集群启用，需重建集群并迁移工作负载。

### 批量检查多个集群的 Dataplane V2 状态

```bash
tccli tke DescribeClusters --region <Region> \
    | jq '[.Clusters[] | {ClusterId, ClusterName, DataPlaneV2: .ClusterNetworkSettings.DataPlaneV2}]'
# expected: 返回所有集群的 DataPlaneV2 状态汇总
```

**示例输出：**

```json
[
    {
        "ClusterId": "cls-example1",
        "ClusterName": "prod-cluster",
        "DataPlaneV2": false
    },
    {
        "ClusterId": "cls-example2",
        "ClusterName": "test-cluster",
        "DataPlaneV2": true
    }
]
```

## 验证

### 验证 Dataplane V2 状态

```bash
# 单个集群
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]' \
    | jq '.Clusters[0].ClusterNetworkSettings.DataPlaneV2'
# expected（已启用）: true
# expected（未启用）: false
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

### 验证 kube-proxy 不存在（Dataplane V2 集群）

对于已启用 Dataplane V2 的集群，kube-proxy DaemonSet 不应存在：

```bash
kubectl get daemonset -n kube-system | grep kube-proxy
# expected: 无输出（kube-proxy 不存在）
```

```text
NAME  STATUS  AGE
...
```

### Dataplane V2 验证维度总览

| 验证维度 | 命令/方法 | 预期 |
|---------|----------|------|
| API 字段 | `DescribeClusters → .ClusterNetworkSettings.DataPlaneV2` | `true` |
| kube-proxy | `kubectl get ds -n kube-system \| grep kube-proxy` | 无结果 |
| Kubernetes 版本 | `DescribeClusters → .ClusterVersion` | >= 1.24 |

## 清理

无需清理。本页为概念说明与只读查询。若创建了测试集群，可用 `DeleteCluster` 删除：

```bash
tccli tke DeleteCluster --region <Region> \
    --ClusterId "CLUSTER_ID" \
    --InstanceDeleteMode "terminate"
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DescribeClusters` 返回 `UnauthorizedOperation` | `tccli configure list` 检查当前凭据 | 子账号缺少 `tke:DescribeClusters` 权限 | 联系主账号授予 `QcloudTKEFullAccess` 或最小权限 `tke:DescribeClusters` |
| `CreateCluster` 报 `DataPlaneV2` 参数不支持 | 检查集群版本和 OS | 版本 < 1.24 或 OS 非 TencentOS Server 3.1/3.2 | 使用 >= 1.24 版本 + 支持的 OS 重建集群 |
| `DataPlaneV2` 字段在输出中为 `false` 或不出现 | 集群创建时未设置 `ClusterAdvancedSettings.DataPlaneV2=true` | DataPlaneV2 仅创建时可设，存量集群无法启用 | 创建新集群并设置 `DataPlaneV2: true` |

### 功能行为疑问

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 现有集群想启用 Dataplane V2 | `tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'` | 创建后不可修改 | 新建集群并迁移工作负载 |
| 启用后 Pod 间网络异常 | 检查集群网络类型 | Dataplane V2 仅支持 VPC-CNI 共享网卡多 IP 模式 | 创建集群时确保选择 VPC-CNI 网络模式 |
| 节点加入失败 | 检查节点 OS | Dataplane V2 仅支持 TencentOS Server 3.1/3.2 | 使用符合要求的 OS 创建节点 |

## 下一步

- [GlobalRouter 模式介绍](../GlobalRouter%20模式介绍/tccli%20操作.md) -- page_id `50354`
- [集群启用 IPVS](../集群启用%20IPVS/tccli%20操作.md) -- IPVS 转发模式
- [容器网络概述](../../容器网络概述/tccli%20操作.md) -- 三种方案原理与对比

## 控制台替代

[TKE 控制台](https://console.cloud.tencent.com/tke2/cluster)：创建集群时，在网络配置 > 高级设置中勾选「Dataplane v2」开关。已有集群的基本信息页可查看 Dataplane V2 启用状态。
