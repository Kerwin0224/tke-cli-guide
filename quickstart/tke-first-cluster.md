---
doc_type: Quickstart
fused: true
---

# TKE: 5 分钟创建第一个集群

> **Traceability**: [TKE 产品文档](https://cloud.tencent.com/document/product/457)
>
> ⚠️ **计费警告**: 创建托管集群（MANAGED_CLUSTER）即开始计收**集群管理费**（L5 为最低等级）。
> 完成本 Quickstart 后务必执行 [Step 3: 删除集群](#step-3-删除集群清理) 以避免持续计费。
>
> **目标读者**: DevOps / SRE — 用 tccli 管理 TKE 集群。
>
> **阅读路径**: 本文 → [TKE 概览](../tke/index.md) → [创建集群详解](../tke/clusters/create.md)
>
> **时间估计**: ~5 分钟（集群创建等待约 3-5 分钟）

---

## 准备工作

逐条验证，每条后附可执行命令。

| # | 条件 | 验证命令 | 预期结果 |
|:--|:-----|:--------|:---------|
| 1 | tccli 已安装 (≥ 3.0) | `tccli --version` | `3.x.x` 或更高 |
| 2 | 凭证已配置 | `tccli tke DescribeRegions` | 返回 `"RequestId"`，无 `Error` |
| 3 | 目标地域可用 | `tccli tke DescribeRegions` | 输出含 `ap-guangzhou`、`Status` 非空 |
| 4 | VPC 和子网已就绪 | `tccli vpc DescribeSubnets --region <REGION>` | 某子网 `AvailableIpAddressCount ≥ 10` |
| 5 | kubectl 已安装（验证集群用） | `kubectl version --client` | Client Version 显示版本号 |

```bash
tccli --version
# expected: 3.1.117.1

tccli tke DescribeRegions \
    --filter "RegionInstanceSet[0].{name:RegionName,status:Status}" --output text
# expected: ap-guangzhou    alluser
```

> 未安装 tccli → [安装 tccli](../getting-started/install.md)。凭证未配置 → [配置凭证](../getting-started/credentials.md)（`tccli configure` 或 `tccli auth login`）。缺少 VPC/子网 → [准备 VPC 与子网](../getting-started/prepare-vpc.md)。未装 kubectl → [kubectl 安装](https://kubernetes.io/docs/tasks/tools/)（验证集群用，非创建集群必需）。

---

## Step 0: 环境检查

查询当前地域可用的 K8s 版本，确认最新版本号。

```bash
tccli tke DescribeVersions --region <REGION> \
    --filter "VersionInstanceSet[-3:].Version" --output text
# expected (示例):
# 1.30.0
# 1.32.2
# 1.34.1
```

| 占位符 | 含义 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<REGION>` | 目标地域 | 如 `ap-guangzhou` | `tccli tke DescribeRegions` 查看 `RegionName` |

> 本 Quickstart 使用 **1.34.1**（当前最新可用版本）。用 `tccli tke DescribeVersions --region <REGION> --output json` 查看完整版本列表及 `Remark`。

---

## Step 1: 创建集群

### 决策：集群类型

| 特性 | **MANAGED_CLUSTER (托管)** | INDEPENDENT_CLUSTER (独立) |
|:-----|:--------------------------|:---------------------------|
| Master 节点 | 腾讯云托管，不占用户资源 | 用户自管 3 个 Master 节点 |
| 管理费 | 按等级收取（L5 起） | 无管理费，但需付费 Master CVM |
| 运维 | 低 — 升级/备份由云平台处理 | 高 — 自行维护 |
| 高可用 | 内置 HA | 需自行配置 3 节点 |
| SLO | 99.95% | 取决于用户配置 |

> **推荐**: `MANAGED_CLUSTER` — 适用绝大多数场景。

### 创建集群

命令使用 VPC-CNI 网络模式（Pod 直接从 VPC 子网获取 IP），最小规格 L5，无节点。

```bash
tccli tke CreateCluster \
    --region <REGION> \
    --ClusterType MANAGED_CLUSTER \
    --ClusterBasicSettings '{
        "ClusterName": "<CLUSTER_NAME>",
        "ClusterVersion": "1.34.1",
        "VpcId": "<VPC_ID>",
        "SubnetId": "<SUBNET_ID>",
        "ClusterLevel": "L5"
    }' \
    --ClusterCIDRSettings '{
        "ServiceCIDR": "10.100.0.0/17",
        "MaxNodePodNum": 64,
        "MaxClusterServiceNum": 32768,
        "IgnoreClusterCIDRConflict": true,
        "IgnoreServiceCIDRConflict": true,
        "EniSubnetIds": ["<SUBNET_ID>"]
    }' \
    --ClusterAdvancedSettings '{
        "ContainerRuntime": "containerd",
        "DeletionProtection": false,
        "NetworkType": "VPC-CNI",
        "VpcCniType": "tke-route-eni",
        "IsNonStaticIpMode": true
    }'
# expected: {"ClusterId":"cls-xxxxxxxx","RequestId":"..."}
```

```json
{
    "ClusterId": "cls-xxxxxxxx",
    "RequestId": "..."
}
```

**占位符说明**：

| 占位符 | 含义 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<REGION>` | 目标地域 | 如 `ap-guangzhou` | `tccli tke DescribeRegions` |
| `<CLUSTER_NAME>` | 集群名称 | 1-60 字符，字母开头 | 自定义 |
| `<VPC_ID>` | VPC ID | 格式 `vpc-xxxxxxxx` | `tccli vpc DescribeVpcs --region <REGION>` |
| `<SUBNET_ID>` | 子网 ID | 格式 `subnet-xxxxxxxx`，VPC-CNI 模式必填 | `tccli vpc DescribeSubnets --region <REGION>` |

**关键参数**：

| 参数 | 说明 |
|:-----|:-----|
| `ClusterVersion` | K8s 版本，Step 0 中 `DescribeVersions` 查得的最新值 |
| `ClusterLevel` | 集群等级，`L5` 为最低（成本最小） |
| `ServiceCIDR` | Service 网段，**不得与 VPC CIDR 重叠**。若报 `CIDR_CONFLICT_WITH_VPC_CIDR` 则换网段 |
| `EniSubnetIds` | VPC-CNI 模式必需，填子网 ID |
| `ClusterCIDR` | **VPC-CNI 模式禁止传此参数**。若同时传 `ClusterCIDR` + `NetworkType: VPC-CNI` → `FailedOperation.Param: VPC-CNI cluster doesn't need clusterCIDR` |
| `NetworkType` | `VPC-CNI`：Pod 直接从 VPC 子网获取 IP。也可选 `GR`（Global Router），此时需传 `ClusterCIDR` |
| `DeletionProtection` | `false` — Quickstart 用完即删 |
| `ContainerRuntime` | `containerd` — 当前默认运行时 |

### 等待就绪

集群创建是异步的，用 `--waiter` 等待状态变为 `Running`。

```bash
tccli tke DescribeClusterStatus \
    --region <REGION> \
    --ClusterIds '["<CLUSTER_ID>"]' \
    --waiter '{"expr":"ClusterStatusSet[0].ClusterState","to":"Running","timeout":600,"interval":10}' \
    --output json
# expected: ClusterState = "Running"
```

```json
{
    "ClusterStatusSet": [
        {
            "ClusterId": "cls-xxxxxxxx",
            "ClusterState": "Running",
            "ClusterInstanceState": "AllNormal",
            "ClusterBMonitor": false,
            "ClusterInitNodeNum": 0,
            "ClusterRunningNodeNum": 0,
            "ClusterFailedNodeNum": 0,
            "ClusterClosedNodeNum": 0,
            "ClusterClosingNodeNum": 0,
            "ClusterDeletionProtection": false,
            "ClusterAuditEnabled": false
        }
    ],
    "TotalCount": 1,
    "RequestId": "..."
}
```

| waiter 参数 | 值 | 说明 |
|:-----------|:----|:-----|
| `expr` | `ClusterStatusSet[0].ClusterState` | 指向状态字段的 JMESPath（注意字段名是 `ClusterState`，非 `ClusterStatus`） |
| `to` | `Running` | 目标状态 |
| `timeout` | `600` | 超时秒数（首次创建较慢） |
| `interval` | `10` | 轮询间隔秒数 |

> `--waiter` 参数必须用 **JSON 双引号**格式，不要用 Python dict 单引号。Shell 中整个 waiter JSON 用单引号包裹即可，内部 JSON key/value 用双引号。

### 验证

从四个维度确认集群创建成功并可用。

| 维度 | 命令 | 预期 |
|:-----|:-----|:-----|
| 1. 状态 | `DescribeClusterStatus` → `ClusterState` | `Running` |
| 2. 元数据 | `DescribeClusters` → `ClusterId`/`ClusterName`/`ClusterVersion`/`ClusterLevel` | 与创建参数一致 |
| 3. 端点 | `DescribeClusterSecurity` → `Domain` | `cls-xxxxxxxx.ccs.tencent-cloud.com` |
| 4. kubeconfig | `DescribeClusterSecurity` → `Kubeconfig` | 有效 YAML，可写文件 |

```bash
# 维度 1: 状态 = Running，删除保护 = false
tccli tke DescribeClusterStatus --region <REGION> --ClusterIds '["<CLUSTER_ID>"]' \
    --filter "ClusterStatusSet[0].{state:ClusterState,protection:ClusterDeletionProtection}" --output json
# expected: {"state":"Running","protection":false}
```
```json
{"state": "Running", "protection": false}
```

```bash
# 维度 2: 详情字段完整
tccli tke DescribeClusters --region <REGION> --ClusterIds '["<CLUSTER_ID>"]' \
    --filter "Clusters[0].{id:ClusterId,name:ClusterName,ver:ClusterVersion,level:ClusterLevel}" --output json
# expected: id/name/ver/level 与创建参数一致
```
```json
{"id": "cls-xxxxxxxx", "name": "<CLUSTER_NAME>", "ver": "1.34.1", "level": "L5"}
```

```bash
# 维度 3: 集群端点可达
tccli tke DescribeClusterSecurity --region <REGION> --ClusterId <CLUSTER_ID> \
    --filter "Domain" --output text
# expected: cls-xxxxxxxx.ccs.tencent-cloud.com
```
```text
cls-xxxxxxxx.ccs.tencent-cloud.com
```

```bash
# 维度 4: kubeconfig 可用
tccli tke DescribeClusterSecurity --region <REGION> --ClusterId <CLUSTER_ID> \
    --filter "Kubeconfig" --output text > ~/.kube/config-qs
# expected: 文件写入成功，kubeconfig 为有效 YAML
export KUBECONFIG=~/.kube/config-qs
kubectl cluster-info
# expected: Kubernetes control plane is running at https://cls-xxxxxxxx.ccs.tencent-cloud.com
```

> ⚠️ kubeconfig 包含访问凭证，勿提交到 git 或公开分享。

---

## Step 2: 查看集群

### 列表查询

筛选所有 `Running` 状态的集群，以表格形式展示。

```bash
tccli tke DescribeClusters --region <REGION> \
    --filter "Clusters[?ClusterStatus=='Running'].{id:ClusterId,name:ClusterName,ver:ClusterVersion,type:ClusterType}" \
    --output text
# expected: Tab 分隔的集群列表
```
```text
cls-xxxxxxxx    <CLUSTER_NAME>    1.34.1    MANAGED_CLUSTER
```

### 单集群详情

```bash
tccli tke DescribeClusters --region <REGION> --ClusterIds '["<CLUSTER_ID>"]' \
    --filter "Clusters[0].{id:ClusterId,name:ClusterName,version:ClusterVersion,status:ClusterStatus,runtime:ContainerRuntime,vpc:ClusterNetworkSettings.VpcId}" \
    --output json
# expected: 返回集群完整元数据
```
```json
{
    "id": "cls-xxxxxxxx",
    "name": "<CLUSTER_NAME>",
    "version": "1.34.1",
    "status": "Running",
    "runtime": "containerd",
    "vpc": "<VPC_ID>"
}
```

> 列表查询的状态字段是 `Clusters[].ClusterStatus`（如 `"Running"`），与单集群健康查询 `DescribeClusterStatus` 的 `ClusterState` 字段名不同，注意区分。

---

## Step 3: 删除集群（清理）

> ⚠️ **不可逆**: 集群删除后元数据**无法恢复**，工作负载全部丢失。
> `--InstanceDeleteMode terminate` 会销毁关联的 CVM 节点。
> **不会自动删除**的关联资源: CBS 云硬盘、EIP、CLB —— 请手动清理。

### 前置检查

确认删除保护已关闭。

```bash
tccli tke DescribeClusterStatus --region <REGION> --ClusterIds '["<CLUSTER_ID>"]' \
    --filter "ClusterStatusSet[0].ClusterDeletionProtection" --output text
# expected: false
```
```text
false
```

若输出 `true`，先关闭保护：

```bash
tccli tke DisableClusterDeletionProtection --ClusterId <CLUSTER_ID>
# expected: {"RequestId":"..."}
```

### 执行删除

```bash
tccli tke DeleteCluster \
    --region <REGION> \
    --ClusterId <CLUSTER_ID> \
    --InstanceDeleteMode terminate \
    --output json
# expected: {"RequestId":"..."}
```
```json
{"RequestId": "..."}
```

| 参数 | 说明 |
|:-----|:-----|
| `--InstanceDeleteMode terminate` | **推荐**: 销毁关联 CVM 节点，避免孤立计费 |
| `--InstanceDeleteMode retain` | 仅删管理面，CVM 保留为孤立实例（继续计费） |

### 验证删除

```bash
tccli tke DescribeClusters --region <REGION> --ClusterIds '["<CLUSTER_ID>"]' \
    --output json
# expected: TotalCount = 0，集群已从列表消失
```
```json
{"TotalCount": 0, "Clusters": [], "RequestId": "..."}
```

`TotalCount: 0` → 集群已删除。删除是异步的，查询可能短暂仍有记录，等几秒后 `TotalCount` 归零即确认删除完成。

---

## 故障恢复

### 命令返回错误 (exit ≠ 0)

| 错误码 / 信号 | 诊断 | 根因 | 修复 |
|:-------------|:-----|:-----|:-----|
| `FailedOperation.Param: VPC-CNI cluster doesn't need clusterCIDR` | 检查 `--ClusterCIDRSettings` 是否含 `ClusterCIDR` | VPC-CNI 模式下 Pod IP 来自 VPC 子网，不需要 `ClusterCIDR` | 移除 `ClusterCIDRSettings` 中的 `ClusterCIDR` 字段 |
| `InvalidParameter.CidrConflictWithVpcCidr` | 对比 `ServiceCIDR` 与 VPC CIDR | Service 网段与 VPC CIDR 重叠 | 换不重叠 CIDR（如 `10.100.0.0/17`） |
| `EniSubnetIds must be set` | 检查是否传了 `EniSubnetIds` | VPC-CNI 模式缺弹性网卡子网 | 在 `ClusterCIDRSettings` 中加 `"EniSubnetIds":["<SUBNET_ID>"]` |
| `DeleteCluster` 返回 exit 252 | 检查命令是否含 `--InstanceDeleteMode` | 缺少必填参数 `InstanceDeleteMode` | 补 `--InstanceDeleteMode terminate` 或 `retain` |
| 删除被 API 拒绝 | `DescribeClusterStatus` 查 `ClusterDeletionProtection` | 删除保护开启中 | 先执行 `tccli tke DisableClusterDeletionProtection --ClusterId <CLUSTER_ID>` |
| 集群卡 `Creating` > 10min | `DescribeTasks --region <REGION> --ClusterId <CLUSTER_ID>` 查任务状态 | 子网 IP 不足或后端异常 | 换子网重试；若持续失败则提工单并附 `RequestId` |

### 命令成功但状态不对 (exit = 0)

| 现象 | 诊断 | 根因 | 说明 |
|:-----|:-----|:-----|:-----|
| 删除后 `DescribeClusters` 仍有记录 | `DescribeClusters --ClusterIds '["<CLUSTER_ID>"]' --filter "TotalCount" --output text` | 删除是异步操作，短暂窗口内列表未刷新 | 等几秒再查，`TotalCount: 0` 即确认删除（**无需等 30s**） |
| 删除后 CVM 仍存在 | `tccli cvm DescribeInstances --region <REGION>` | 删除时可能用了 `--InstanceDeleteMode retain` | 手动终止残留 CVM，下次使用 `terminate` 模式 |
| `DescribeClusterSecurity` 返回空字段 | `DescribeClusterStatus` 查 `ClusterState` | 集群非 `Running` 时端点不可用 | 等待集群 `Running` 后再查 |

---

## 下一步

- [创建集群详解](../tke/clusters/create.md) — 完整参数、高级配置（Global Router 模式、双栈、IPVS）
- [查询和过滤集群](../tke/clusters/query.md) — JMESPath 高级过滤技巧
- [删除集群详解](../tke/clusters/delete.md) — 残留资源清理、批量删除
- [给集群添加节点](../tke/nodes/nodepool-create.md) — 创建节点池、扩容
- [TKE 拉取 TCR 镜像](../cross-product/tke-pull-tcr.md) — 跨产品：集群工作负载使用 TCR 私有镜像

---

## 控制台替代

[TKE 控制台创建集群](https://console.cloud.tencent.com/tke2/cluster/create?rid=1)

---
