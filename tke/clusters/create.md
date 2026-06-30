---
doc_type: How-to
subtype: 6A
fused: true
---
# 创建集群

> 创建新的 TKE Kubernetes 集群。异步操作，通常 5-10 分钟完成。
> 控制台: [容器服务 - 集群](https://console.cloud.tencent.com/tke2/cluster) | page_id: `tke-cluster-create`

## 概述

创建 TKE 集群是在腾讯云上运行 K8s 的第一步。TKE 提供了两种集群类型:

| 选项 | 最佳场景 | 关键限制 | 升级路径 |
|------|---------|:---:|---------|
| MANAGED_CLUSTER (托管) | 生产环境，免运维 Master | Master 不可 SSH | 控制台/API 一键升级 |
| INDEPENDENT_CLUSTER (独立) | 完全控制 Master 节点 | 需自行维护 Master HA | 手动升级 Master |

**默认推荐**: 托管集群。99% 的场景下托管集群是最优解 —— Master 由腾讯云运维，你只需要管理工作节点。

操作是**异步**的: 命令返回 `ClusterId` 即表示创建已提交，集群就绪需等待 5-10 分钟。

## 准备工作

### 环境检查

```bash
tccli --version
# expected: 版本号 (如 1.2.0+)

tccli cvm DescribeRegions --region ap-guangzhou
# expected: { "RegionSet": [...] }  → 凭证有效（tccli 默认剥离 Response 包装层）
```

### 资源检查

```bash
# 1. 确认集群名未被占用
tccli tke DescribeClusters --region ap-guangzhou \
  --Filters '[{"Name":"ClusterName","Values":["<CLUSTER_NAME>"]}]'
# expected: { "TotalCount": 0 }  → 名称可用

# 2. 确认 VPC 存在
tccli vpc DescribeVpcs --region ap-guangzhou --VpcIds '["<VPC_ID>"]'
# expected: { "VpcSet": [{ "VpcId": "<VPC_ID>" }] }  → VPC 可用

# 3. 确认集群配额未满
tccli tke DescribeClusters --region ap-guangzhou
# expected: TotalCount < 配额上限 (默认 50)
```

## 关键字段

> 来源: `tccli tke CreateCluster --generate-cli-skeleton` + tccli `api.json` 的 `required` 字段（实测 P7）。注意：API 层 `required` 与业务层"必需"不同——`ClusterBasicSettings` 在 API 层非必填，但不传 `ClusterName`/`VpcId` 集群无法正常使用；下表"必填"列按 API 层 `required` 标注，"业务必需"在约束列说明。

### 顶层参数（API 必填项）

| 字段 | 类型 | 必填 | 约束 | 填错时的错误 |
|-------|------|:--------:|------------|---------------|
| ClusterType | string | 是 | `MANAGED_CLUSTER` / `INDEPENDENT_CLUSTER` | `InvalidParameterValue.ClusterType` |
| ClusterCIDRSettings | object | 是 | ClusterCIDR + ServiceCIDR（不传报 `the following arguments are required: --ClusterCIDRSettings`） | `InvalidParameterValue.ClusterCIDR` |

> ⚠️ 实测 `ClusterType` 与 `ClusterCIDRSettings` 是仅有的两个 API 层必填项。缺 `ClusterCIDRSettings` 命令直接 exit 252 失败，不会进入业务校验。

### ClusterBasicSettings（业务必需，API 非必填）

| 字段 | 类型 | 必填 | 约束 | 填错时的错误 |
|-------|------|:--------:|------------|---------------|
| ClusterName | string | 否 | ≤60 字符, 字母数字连字符；业务必需，不传集群无名 | `InvalidParameterValue.ClusterName` |
| ClusterVersion | string | 否 | 如 `1.34.1`，默认 `1.10.5`；业务必需 | `InvalidParameterValue.ClusterVersion` |
| VpcId | string | 否 | 已有 VPC ID；创建托管集群时业务必需（"创建托管空集群时必传"） | `ResourceNotFound.VpcId` |
| ClusterOs | string | 否 | `ubuntu20.04x86_64` / `tlinux3.1x86_64` 等；不传用默认 OS | `InvalidParameterValue.ClusterOs` |
| ClusterLevel | string | 否 | 真实枚举 L5/L20/L50/L100/L200/L500（无 L10），默认 `L5` | `InvalidParameterValue.ClusterLevel` |
| SubnetId | string | 否 | VPC 内子网 ID | `ResourceNotFound.SubnetId` |
| ProjectId | integer | 否 | 项目 ID，默认 `0` | `InvalidParameterValue.ProjectId` |

### ClusterAdvancedSettings（建议设置）

| 字段 | 类型 | 推荐值 | 作用 |
|-------|------|--------|------|
| DeletionProtection | boolean | `true` | 防止误删集群 |
| AuditEnabled | boolean | `true` | 开启审计日志 |
| ContainerRuntime | string | `containerd` | 容器运行时 |

## 操作步骤

### 步骤 1：决策 — 选集群类型

#### 为什么选 MANAGED_CLUSTER

- **托管 vs 独立**: 托管集群的 Master 由腾讯云负责 (HA、升级、备份)；独立集群需要自己维护 3 台 Master 节点
- **成本差异**: 托管集群收取集群管理费 (L5 约 ¥0.4/小时)；独立集群不收取管理费但需支付 3 台 Master 的 CVM 费用
- **默认推荐**: MANAGED_CLUSTER — 你不需要管理 Master
- **能改吗?**: 不能。集群类型创建后无法切换。选了独立就得一直维护 Master

### 步骤 2：创建 — 最小化

最少必需参数: ClusterType + ClusterCIDRSettings（API 必填）+ 集群名 + VPC + K8s 版本（业务必需）。

```bash
tccli tke CreateCluster \
  --region ap-guangzhou \
  --ClusterType MANAGED_CLUSTER \
  --ClusterBasicSettings '{
    "ClusterName": "<CLUSTER_NAME>",
    "ClusterVersion": "1.34.1",
    "ClusterOs": "ubuntu20.04x86_64",
    "VpcId": "<VPC_ID>"
  }' \
  --ClusterCIDRSettings '{
    "ClusterCIDR": "172.16.0.0/16",
    "ServiceCIDR": "10.96.0.0/20"
  }' \
  --ClusterAdvancedSettings '{
    "DeletionProtection": true
  }'
# expected: { "ClusterId": "cls-xxxxxxxx", "RequestId": "..." }（tccli 默认剥离 Response 包装层，顶层直接是 ClusterId/RequestId）
```

> ⚠️ **此命令创建的是无工作节点的空集群**（无 `RunInstancesForNode`，`ClusterRunningNodeNum=0`）。集群会进入 `Running` 状态但**不能直接运行 Pod**——必须接着 [创建节点池](../nodes/nodepool-create.md) 添加工作节点。`ClusterCIDR` 不得与 VPC CIDR 冲突。

| 占位符 | 含义 | 约束 | 如何获取 |
|------------|------|------|---------|
| `<CLUSTER_NAME>` | 集群名称 | ≤60 字符，全局唯一 | 自己定义 |
| `<VPC_ID>` | VPC 实例 ID | 必须存在 | `tccli vpc DescribeVpcs` → `VpcSet[].VpcId` |
| `<CLUSTER_CIDR>` / `<SERVICE_CIDR>` | 容器/服务网段 | 不得与 VPC CIDR 冲突 | 自己定义，常见 `172.16.0.0/16` / `10.96.0.0/20` |

### 步骤 3：创建 — 增强：容量规划与审计

在最小化基础上规划 Pod/Service 容量（`MaxNodePodNum`/`MaxClusterServiceNum`）、提升集群等级（`ClusterLevel`）、开启审计与指定运行时（适合生产）:

```bash
tccli tke CreateCluster \
  --region ap-guangzhou \
  --ClusterType MANAGED_CLUSTER \
  --ClusterBasicSettings '{
    "ClusterName": "<CLUSTER_NAME>",
    "ClusterVersion": "1.34.1",
    "ClusterOs": "ubuntu20.04x86_64",
    "VpcId": "<VPC_ID>",
    "ClusterLevel": "L20"
  }' \
  --ClusterCIDRSettings '{
    "ClusterCIDR": "172.16.0.0/16",
    "ServiceCIDR": "10.96.0.0/20",
    "MaxNodePodNum": 64,
    "MaxClusterServiceNum": 4096
  }' \
  --ClusterAdvancedSettings '{
    "DeletionProtection": true,
    "AuditEnabled": true,
    "ContainerRuntime": "containerd"
  }'
# expected: { "ClusterId": "cls-xxxxxxxx", "RequestId": "..." }
```

这启用了自定义 CIDR（适合多集群/VPC 互联场景）、审计日志、集群等级 L20（支持更多节点）。

### 步骤 4：验证

异步操作: 检查 ≥4 个维度确认集群可用。

```bash
# 轮询状态直到 Running
tccli tke DescribeClusterStatus \
  --region ap-guangzhou \
  --ClusterIds '["<CLUSTER_ID>"]'
```

| 维度 | 命令 | 预期 |
|-----------|---------|----------|
| Status | `DescribeClusterStatus` | `ClusterState: "Running"` |
| 集群属性 | `DescribeClusters --ClusterIds '["<ID>"]'` | ClusterName、Version、VpcId 与创建参数一致 |
| 删除保护 | `DescribeClusterStatus` → `ClusterStatusSet[].ClusterDeletionProtection` | `true`（布尔值，非 Enabled/Disabled） |
| kubeconfig | `DescribeClusterKubeconfig --ClusterId "<ID>"` | 返回有效 kubeconfig |
| 节点数 | `DescribeClusterStatus` → `ClusterStatusSet[].ClusterRunningNodeNum` | ≥0（空集群为 0，字段名是 ClusterRunningNodeNum 非 NodeCount） |

## 清理

> ⚠️ **Side-effect 警告**: 删除集群会:
> - 销毁集群内所有**工作节点** (CVM)，数据不可恢复
> - **不自动删除**: CBS 云硬盘、弹性公网 IP (EIP)、CLB 负载均衡器
> - 托管集群的 Master 由腾讯云自动回收
> - 计费停止时间: 集群删除即停止收取管理费；CVM、CBS、EIP 按各自计费规则继续

### 1. 清理前检查

```bash
# 确认要删除的是正确的集群
tccli tke DescribeClusters \
  --region ap-guangzhou \
  --ClusterIds '["<CLUSTER_ID>"]'
# expected: 确认 ClusterName 和 ClusterId 匹配你的目标
```

### 2. 关闭删除保护

```bash
tccli tke DisableClusterDeletionProtection \
  --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>"
# expected: exit 0
```

### 3. 删除

```bash
tccli tke DeleteCluster \
  --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>"
# expected: exit 0
```

### 4. 验证已删除

```bash
tccli tke DescribeClusters \
  --region ap-guangzhou \
  --ClusterIds '["<CLUSTER_ID>"]'
# expected: { "TotalCount": 0, "Clusters": [] }
```

> **Billing warning**: 集群管理费在删除后立即停止。但 CBS 盘和 EIP 会持续扣费，需手动清理:
> ```bash
> tccli cbs DescribeDisks --region ap-guangzhou  # 检查残留 CBS
> tccli vpc DescribeAddresses --region ap-guangzhou  # 检查残留 EIP
> ```

## 故障恢复

### 命令返回错误（exit ≠ 0）

| 现象 | 诊断 | 根因 | 修复 |
|---------|----------|------------|-----|
| `AuthFailure.SecretIdNotFound` | `tccli cvm DescribeRegions` | 凭证未配置或已过期 | `tccli configure` 重新配置 SecretId/SecretKey |
| `InvalidParameterValue.ClusterName` | 检查名称格式 | 名称含特殊字符或超长 | 使用 ≤60 字符的字母数字连字符 |
| `InvalidParameterValue.ClusterVersion` | 名称格式正确但仍报错 | 版本号不存在 | `tccli tke DescribeVersions` 查询可用版本列表 |
| `ResourceNotFound.VpcId` | `tccli vpc DescribeVpcs` | VPC ID 不存在或地域错误 | 确认 VPC 在该地域存在，或 `tccli vpc CreateVpc` 新建 |
| `LimitExceeded.ClusterQuota` | `tccli tke DescribeClusters` | 集群数量达上限 (默认 50) | 删除闲置集群或提交工单提升配额 |
| `InvalidParameterValue.ClusterCIDR` | CIDR 格式: `A.B.C.D/掩码` | CIDR 与现有 VPC 或集群冲突 | 换一个不重叠的 CIDR 段 |

### 命令成功但状态不对（exit = 0）

| 现象 | 诊断 | 根因 | 修复 |
|---------|----------|------------|-----|
| 集群卡在 `Creating` > 30 分钟 | `tccli tke DescribeClusterStatus` | 可用区资源不足或 VPC 配置冲突 | 1. 等待 10 分钟观察 2. 若超 30 分钟，删除重建并换可用区 |
| 状态为 `Running` 但 `DescribeClusterKubeconfig` 失败 | 检查返回的 kubeconfig 内容 | 集群就绪延迟 (API Server 未完全就绪) | 等待 1-2 分钟后重试 |
| 状态为 `Abnormal` | `tccli tke DescribeClusterStatus` | Master 组件异常 (罕见) | `DeleteCluster` → 重建。若重建仍异常，提交工单 |
| 删除保护无法关闭 | `tccli tke DisableClusterDeletionProtection` | CAM 权限不足 | 确认账号有 `tke:DisableClusterDeletionProtection` 权限 |

## 下一步

- [创建节点池](../nodes/nodepool-create.md) — 给集群添加工作节点（集群没有节点不能运行 Pod）
- [查询集群](query.md) — 学习 filter + JMESPath 表达式
- [获取 kubeconfig](../security/auth.md) — 配置 kubectl 访问集群
- [独立集群 Master 运维](master-ops.md) — 选了独立集群时，扩缩容 Master/etcd 节点

## 控制台替代方案

[容器服务控制台 - 创建集群](https://console.cloud.tencent.com/tke2/cluster/create)

## Action 清单

| Action | 类型 | 版本 | 说明 |
|:-------|:-----|:-----|:-----|
| `CreateCluster` | 主操作 | 2018-05-25 | 创建集群（异步，返回 ClusterId） |
| `DescribeClusterKubeconfig` | 验证 | 2018-05-25 | 获取 kubeconfig |
| `DescribeClusters` | 验证 | 2018-05-25 | 查询集群列表/确认名称可用与配额 |
| `DescribeClusterStatus` | 验证 | 2018-05-25 | 轮询集群状态到 Running |
| `DescribeRegions` | 验证 | 2018-05-25 | 凭证有效性检查 |
| `DescribeVersions` | 验证 | 2018-05-25 | 查询可用 K8s 版本 |
| `DisableClusterDeletionProtection` | 验证 | 2018-05-25 | 删除前置：关闭删除保护 |
| `DeleteCluster` | 清理 | 2018-05-25 | 删除集群（详见 delete.md） |
| `vpc:DescribeVpcs` | 跨产品 | vpc | 确认 VPC 存在 |
| `vpc:CreateVpc` | 跨产品 | vpc | 新建 VPC |
| `cbs:DescribeDisks` | 跨产品 | cbs | 检查残留 CBS 云硬盘 |
| `vpc:DescribeAddresses` | 跨产品 | vpc | 检查残留 EIP |
