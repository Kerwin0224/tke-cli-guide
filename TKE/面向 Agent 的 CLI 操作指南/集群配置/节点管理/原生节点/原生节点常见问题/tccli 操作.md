# 原生节点常见问题

> 对照官方：[原生节点常见问题](https://cloud.tencent.com/document/product/457/103600) · page_id `103600`

## 概述

本文汇总原生节点在使用过程中的常见问题，覆盖选型、计费、机型、版本兼容、创建失败、配额限制和故障自愈等维度。每个问题附带对应的 CLI 诊断命令，帮助快速定位根因。

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
#    tke:DescribeClusters, tke:DescribeClusterNodePools
#    tke:DescribeClusterNodePoolDetail, tke:DescribeClusterInstances
#    tke:CreateClusterNodePool, cvm:DescribeInstanceTypeConfigs
#    cvm:DescribeInstances
# 验证：执行 DescribeClusters 确认基础权限
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
# 4. 查看可用 CXM 机型列表
tccli cvm DescribeInstanceTypeConfigs --region <Region> \
    --Filters '[{"Name":"instance-type","Values":["CXM*"]}]'
# expected: 返回 CXM 族机型列表，确认地域至少有 1 种 CXM 机型可用

# 5. 查看当前集群和节点池数量
tccli tke DescribeClusters --region <Region>
# expected: 返回集群总数（用于配额判断）
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
| 查看集群列表 | `DescribeClusters` | 是 |
| 查看节点池列表 | `DescribeClusterNodePools` | 是 |
| 查看节点池详情 | `DescribeClusterNodePoolDetail` | 是 |
| 查看节点实例 | `DescribeClusterInstances` | 是 |
| 查看可用机型 | `cvm DescribeInstanceTypeConfigs` | 是 |

## 操作步骤

### 问题 1：原生节点与普通节点如何选型？

原生节点提供声明式管理、故障自愈、原地升降配等增强能力，适合生产业务；普通节点无增强能力，适合简单测试。

**诊断命令**：

```bash
# 查看集群内节点池类型分布
tccli tke DescribeClusterNodePools --region <Region> \
    --ClusterId CLUSTER_ID \
    | jq '.NodePoolSet[] | {Name: .Name, Type: .Type, NodeCount: .NodeCountSummary}'
# expected: 返回 Type=Native 和 Type=Regular 的节点池
```

```json
{
  "NodePoolSet": "<NodePoolSet>",
  "NodePoolId": "<NodePoolId>",
  "Name": "<Name>",
  "ClusterInstanceId": "<ClusterInstanceId>",
  "LifeState": "<LifeState>",
  "LaunchConfigurationId": "<LaunchConfigurationId>"
}
```

| 对比维度 | 原生节点（Native） | 普通节点（Regular） |
|---------|-------------------|-------------------|
| 管理模式 | 节点管家辅助运维 | 用户全自管 |
| 声明式管理 | 支持 API/K8s Annotations | 不支持 |
| Pod 原地升降配 | 支持 | 不支持 |
| 故障自愈 | 支持 AutoRepair | 不支持 |
| 机型要求 | CXM 机型 | 普通 CVM 机型 |
| 容器运行时 | containerd | containerd / docker |
| 最小 K8s 版本 | 1.24 | 1.10 |

### 问题 2：原生节点和普通节点的计费差异是什么？

原生节点底层使用 CXM 机型，按 CVM 实例计费，计费模式与普通节点一致（包年包月/按量计费），不额外收取管理费。区别在于机型单价：CXM 机型价格与同类普通 CVM 机型可能略有差异。

**诊断命令**：

```bash
# 查看节点池下 CVM 实例的计费模式
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId CLUSTER_ID \
    --NodePoolId NODE_POOL_ID \
    | jq '.NodePool.Native.InstanceChargeType'
# expected: "POSTPAID_BY_HOUR" 或 "PREPAID"
```

```json
{
  "NodePool": "<NodePool>",
  "NodePoolId": "<NodePoolId>",
  "Name": "<Name>",
  "ClusterInstanceId": "<ClusterInstanceId>",
  "LifeState": "<LifeState>",
  "LaunchConfigurationId": "<LaunchConfigurationId>"
}
```

### 问题 3：为什么创建原生节点池要求 CXM 机型？

CXM（Cloud eXtreme Machine）是腾讯云为容器化场景优化的机型族，具备更高的网络和存储性能，能更好地支持大规模容器集群。原生节点的节点管家组件依赖 CXM 机型底层能力。

**诊断命令**：

```bash
# 查看地域可用的 CXM 机型
tccli cvm DescribeInstanceTypeConfigs --region <Region> \
    --Filters '[{"Name":"instance-family","Values":["CXM"]}]' \
    | jq '.InstanceTypeConfigSet[] | {Type: .InstanceType, CPU: .CPU, Memory: .Memory}'
# expected: 返回 CXM 机型列表及其规格
```

### 问题 4：哪些 K8s 版本支持原生节点？

原生节点要求 K8s >= 1.24，仅支持托管集群（MANAGED_CLUSTER）。独立集群不支持原生节点。

**诊断命令**：

```bash
# 确认集群版本和类型是否满足要求
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]' \
    | jq '.Clusters[] | {Type: .ClusterType, Version: .ClusterVersion}'
# expected: ClusterType 为 MANAGED_CLUSTER，ClusterVersion >= 1.24
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

### 问题 5：原生节点池创建失败常见原因有哪些？

常见原因包括：CXM 机型不可用、子网 IP 不足、安全组配置错误、Management 参数无效、配额超限。

**诊断命令**：

```bash
# 综合诊断脚本
# 1. 检查 CXM 机型可用性
tccli cvm DescribeInstanceTypeConfigs --region <Region> \
    --Filters '[{"Name":"instance-family","Values":["CXM"]}]' \
    | jq '.InstanceTypeConfigSet | length'
# expected: > 0，至少有 1 种 CXM 机型

# 2. 检查子网可用 IP
tccli vpc DescribeSubnets --region <Region> --SubnetIds '["SUBNET_ID"]' \
    | jq '.SubnetSet[0].AvailableIpAddressCount'
# expected: >= 3（至少为节点池最小节点数留出 IP）

# 3. 检查已有节点池数量
tccli tke DescribeClusterNodePools --region <Region> \
    --ClusterId CLUSTER_ID \
    | jq '.NodePoolSet | length'
# expected: < 配额上限（通常 20）
```

```json
{
  "NodePoolSet": "<NodePoolSet>",
  "NodePoolId": "<NodePoolId>",
  "Name": "<Name>",
  "ClusterInstanceId": "<ClusterInstanceId>",
  "LifeState": "<LifeState>",
  "LaunchConfigurationId": "<LaunchConfigurationId>"
}
```

### 问题 6：原生节点配额有限制吗？

有。单个集群原生节点池数量上限通常为 20（与普通节点池共享配额），具体配额见各区域限制。

**诊断命令**：

```bash
# 查看当前节点池数量
tccli tke DescribeClusterNodePools --region <Region> \
    --ClusterId CLUSTER_ID \
    | jq '{Total: (.NodePoolSet | length), Pools: [.NodePoolSet[].Name]}'
# expected: Total < 20（或地域配额上限）
```

```json
{
  "NodePoolSet": "<NodePoolSet>",
  "NodePoolId": "<NodePoolId>",
  "Name": "<Name>",
  "ClusterInstanceId": "<ClusterInstanceId>",
  "LifeState": "<LifeState>",
  "LaunchConfigurationId": "<LaunchConfigurationId>"
}
```

### 问题 7：故障自愈效果如何？能处理哪些场景？

开启 `Management.AutoRepair` 并配置 `HealthCheckPolicyName` 后，节点管家能自动处理以下场景：
- 节点连续 NotReady 超时 → 自动重装/迁移
- 容器运行时异常 → 自动重启 containerd
- Kubelet 异常 → 自动重启 kubelet
- 内核 Panic → 自动迁移节点

处理时间通常为故障检测后 5-10 分钟。不能处理的场景：网络完全中断导致的节点失联、需要人工决策的业务级故障。

**诊断命令**：

```bash
# 查看自愈配置和策略
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId CLUSTER_ID \
    --NodePoolId NODE_POOL_ID \
    | jq '.NodePool.Native.Management | {AutoRepair, HealthCheckPolicyName}'
# expected: AutoRepair 为 true 时自愈生效；HealthCheckPolicyName 为已创建的策略名

# 查看节点池中节点异常时是否已触发自愈
tccli tke DescribeClusterInstances --region <Region> \
    --ClusterId CLUSTER_ID \
    | jq '[.InstanceSet[] | select(.InstanceState == "failed")] | length'
# expected: 如自愈正常，failed 节点数应随时间减少
```

```json
{
  "NodePool": "<NodePool>",
  "NodePoolId": "<NodePoolId>",
  "Name": "<Name>",
  "ClusterInstanceId": "<ClusterInstanceId>",
  "LifeState": "<LifeState>",
  "LaunchConfigurationId": "<LaunchConfigurationId>"
}
```

### 问题 8：原生节点 NotReady 如何排查？

节点 NotReady 可能由多种原因导致：资源不足、组件异常、网络问题、自愈未启用。

**诊断命令**：

```bash
# 1. 查看节点状态
tccli tke DescribeClusterInstances --region <Region> \
    --ClusterId CLUSTER_ID \
    | jq '.InstanceSet[] | select(.InstanceState != "running") | {InstanceId, InstanceState, FailedReason}'
# expected: 列出所有非 running 状态的节点

# 2. 通过 kubectl 查看节点详情
kubectl describe node NODE_NAME | grep -A 5 Conditions
# expected: 查看 K8s 层 Conditions（MemoryPressure、DiskPressure、PIDPressure、Ready）

# 3. 查看节点池自愈状态
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId CLUSTER_ID \
    --NodePoolId NODE_POOL_ID \
    | jq '.NodePool.Native.Management.AutoRepair'
# expected: 如为 false，说明自愈未开启，需手动处理
```

```json
{
  "TotalCount": "<TotalCount>",
  "InstanceSet": "<InstanceSet>",
  "InstanceId": "<InstanceId>",
  "InstanceRole": "<InstanceRole>",
  "FailedReason": "<FailedReason>",
  "InstanceState": "<InstanceState>"
}
```

### 问题 9：原生节点的密钥/密码管理有何特殊要求？

SSH 密钥通过 `LoginSettings.KeyIds` 下发到节点池内所有新建节点。修改密钥后仅影响新建节点，不影响存量节点。删除已使用的密钥不影响已部署的节点，但会导致新节点创建失败。

**诊断命令**：

```bash
# 查看节点池的登录配置
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId CLUSTER_ID \
    --NodePoolId NODE_POOL_ID \
    | jq '.NodePool.Native.LoginSettings'
# expected: 返回 KeyIds 数组或 KeepImageLogin 布尔值

# 确认密钥是否仍存在
tccli cvm DescribeKeyPairs --region <Region> \
    --KeyIds '["SSH_KEY_ID"]'
# expected: TotalCount = 1
```

```json
{
  "NodePool": "<NodePool>",
  "NodePoolId": "<NodePoolId>",
  "Name": "<Name>",
  "ClusterInstanceId": "<ClusterInstanceId>",
  "LifeState": "<LifeState>",
  "LaunchConfigurationId": "<LaunchConfigurationId>"
}
```

### 问题 10：节点删除后能否恢复？

原生节点删除后不可恢复。节点被删除时，其上的本地数据（非持久化存储）和 Pod 会被清理。建议使用持久化存储（如 CBS、CFS）保存业务数据。

**诊断命令**：

```bash
# 查看节点池的删除保护状态（如启用）
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId CLUSTER_ID \
    --NodePoolId NODE_POOL_ID \
    | jq '.NodePool.DeletionProtection'
# expected: 建议生产环境设为 true

# 查看节点实例列表（确认哪些节点将被删除）
tccli tke DescribeClusterInstances --region <Region> \
    --ClusterId CLUSTER_ID \
    --Filters '[{"Name":"node-pool-id","Values":["NODE_POOL_ID"]}]' \
    | jq '.InstanceSet[] | {InstanceId, InstanceState}'
# expected: 列出目标节点池的所有节点
```

```json
{
  "NodePool": "<NodePool>",
  "NodePoolId": "<NodePoolId>",
  "Name": "<Name>",
  "ClusterInstanceId": "<ClusterInstanceId>",
  "LifeState": "<LifeState>",
  "LaunchConfigurationId": "<LaunchConfigurationId>"
}
```

## 验证

### 验证诊断结果一致性

```bash
# 综合检查：集群类型、版本、节点池类型和数量
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]' \
    | jq '.Clusters[] | {Type: .ClusterType, Version: .ClusterVersion}'

tccli tke DescribeClusterNodePools --region <Region> \
    --ClusterId CLUSTER_ID \
    | jq '[.NodePoolSet[] | {Name, Type, LifeState}]'
# expected: Type 为 MANAGED_CLUSTER，Version >= 1.24；节点池 LifeState 均为 normal
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

| 验证维度 | 命令 | 预期 |
|---------|------|------|
| 集群兼容性 | `DescribeClusters` | ClusterType=MANAGED_CLUSTER, Version>=1.24 |
| CXM 机型可用 | `cvm DescribeInstanceTypeConfigs` | CXM 族 > 0 |
| 子网 IP 充足 | `vpc DescribeSubnets` | AvailableIpAddressCount >= 3 |
| 配额充足 | `DescribeClusterNodePools` → length | < 20 |
| 自愈状态 | `DescribeClusterNodePoolDetail` → Management | 确认 AutoRepair 与预期一致 |

## 清理

本页为概念说明页，无资源需清理。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DescribeClusterNodePools` 返回空列表 | `tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'` 确认集群存在且类型正确 | 集群下不存在任何节点池（原生或普通） | 先通过 `CreateClusterNodePool` 创建原生节点池 |
| `cvm DescribeInstanceTypeConfigs` 无 CXM 机型 | 检查 `--region` 是否正确；检查 `--Filters` 写法 | 当前地域未提供 CXM 机型，或筛选条件错误 | 换用其他支持 CXM 的地域（如 `ap-guangzhou`、`ap-shanghai`）；不带 Filters 先全量查看可用机型 |
| 创建原生节点池返回 `InvalidParameter.InstanceType` | 检查 `Native.InstanceTypes` 中填写的机型 | 填了非 CXM 机型（普通 CVM 机型如 S5、SA2 等） | 替换为 CXM 机型，`tccli cvm DescribeInstanceTypeConfigs --region <Region> --Filters '[{"Name":"instance-family","Values":["CXM"]}]'` 查询可用列表 |

### 节点状态异常排查

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 原生节点池创建成功但节点 NotReady | `kubectl describe node NODE_NAME` 查看 Conditions；`kubectl get events --all-namespaces` | Management 参数中的 KubeletArgs/KernelArgs 配置有误，导致 kubelet 启动失败 | 修改节点池去掉可疑的 KubeletArgs → 扩容新节点验证 → 逐个添加参数定位问题参数 |
| 节点数达到配额上限无法新建节点池 | `tccli tke DescribeClusterNodePools --region <Region> --ClusterId CLUSTER_ID \| jq '.NodePoolSet \| length'` 确认当前数量 | 已达到单集群节点池数量上限（此为环境限制，非命令错误） | 删除不再使用的节点池后重试 |

## 下一步

- [新建原生节点](../新建原生节点/tccli%20操作.md) — 完整的原生节点池创建指南
- [Management 参数介绍](../Management%20参数介绍/tccli%20操作.md) — page_id `79698`
- [故障自愈规则](../故障自愈规则/tccli%20操作.md) — page_id `78650`
- [原生节点扩缩容](../原生节点扩缩容/tccli%20操作.md) — page_id `78648`
- [原生节点登录方式](../原生节点登录方式/tccli%20操作.md) — page_id `88696`

## 控制台替代

[TKE 控制台 → 集群 → 节点管理](https://console.cloud.tencent.com/tke2/cluster)：在节点列表和节点池详情页查看状态，排查常见问题。对照[官方 FAQ](https://cloud.tencent.com/document/product/457/103600)逐项处理。
