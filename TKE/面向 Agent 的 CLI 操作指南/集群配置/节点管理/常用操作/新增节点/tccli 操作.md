# 新增节点（tccli）

> 对照官方：[新增节点](https://cloud.tencent.com/document/product/457/103964) · page_id `103964` · tccli ≥3.1.107 · API 2022-05-01

## 概述

在集群中新增节点，由**节点池**统一管理。TKE 支持四种节点类型，本页为各类型的入口导航页：通过 tccli 先列出已有节点池，再按目标类型跳转到对应的专页操作。

| 节点类型 | 底层实现 | 适合场景 | 计费方式 |
|----------|----------|----------|----------|
| 普通节点 | CVM + 弹性伸缩组（ASG） | 标准容器负载，需自定义机型和磁盘 | 按 CVM 实例计费 |
| 原生节点 | MachineSet（声明式） | 声明式节点管理，支持原地升降配 | 按 CVM 实例计费 |
| 超级节点 | 虚拟节点（eklet） | Serverless 容器，无需管理底层 CVM | 按 Pod 计费 |
| 注册节点 | External Node（IDC/第三方云） | 混合云，将外部机器纳入 TKE 管理 | 不产生 TKE 计费（外部机器自管） |

> 如不确定选哪种节点类型，默认推荐**普通节点**（CVM + 弹性伸缩组），这是覆盖最广泛、最通用的类型。具体创建操作见 [创建节点池](../../普通节点/创建节点池/tccli%20操作.md) 或 [新增游离普通节点](../../普通节点/新增游离普通节点/tccli%20操作.md)。

## 前置条件

- [环境准备](../../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId、secretKey、region 均已配置，region 为目标地域

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:DescribeClusters, tke:DescribeClusterInstances, tke:DescribeClusterNodePools
# 验证：执行 DescribeClusters 确认权限
tccli tke DescribeClusters --region <Region>
# expected: exit 0，返回集群列表（可为空）
```

```json
{
  "TotalCount": 1,
  "Clusters": [
    {
      "ClusterId": "<ClusterId>",
      "ClusterName": "<ClusterName>",
      "ClusterStatus": "Running",
      "ClusterType": "MANAGED_CLUSTER",
      "ClusterVersion": "1.32.2",
      "ClusterNetworkSettings": {
        "VpcId": "<VpcId>",
        "ClusterCIDR": "10.0.0.0/16",
        "ServiceCIDR": "10.1.0.0/20"
      },
      "CreatedTime": "2026-06-23T05:24:09Z"
    }
  ],
  "RequestId": "<RequestId>"
}
```

### 资源检查

```bash
# 4. 确认目标集群存在且状态为 Running
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["<ClusterId>"]'
# expected: TotalCount >= 1，ClusterStatus "Running"
```

```json
{
  "TotalCount": 1,
  "Clusters": [
    {
      "ClusterId": "<ClusterId>",
      "ClusterName": "<ClusterName>",
      "ClusterStatus": "Running",
      "ClusterType": "MANAGED_CLUSTER",
      "ClusterVersion": "1.32.2",
      "ClusterNetworkSettings": {
        "VpcId": "<VpcId>",
        "ClusterCIDR": "10.0.0.0/16",
        "ServiceCIDR": "10.1.0.0/20"
      }
    }
  ],
  "RequestId": "<RequestId>"
}
```

```bash
# 5. 查询集群已有节点池，确定是扩容已有池还是新建
tccli tke DescribeClusterNodePools --region <Region> \
    --ClusterId <ClusterId>
# expected: exit 0，返回 NodePoolSet（可为空，表示集群尚无节点池）
```

```json
{
  "TotalCount": 2,
  "NodePoolSet": [
    {
      "NodePoolId": "<NodePoolId>",
      "Name": "<NodePoolName>",
      "LifeState": "normal",
      "NodeCountSummary": {
        "AutoscalingAdded": { "Joined": 0, "Total": 1 },
        "ManuallyAdded": { "Joined": 0 }
      }
    },
    {
      "NodePoolId": "<NodePoolId>",
      "Name": "<NodePoolName>",
      "LifeState": "normal",
      "NodeCountSummary": {
        "AutoscalingAdded": { "Joined": 0, "Total": 1 },
        "ManuallyAdded": { "Joined": 0 }
      }
    }
  ],
  "RequestId": "<RequestId>"
}
```

```bash
# 6. 检查 VPC-CNI 子网 IP 余量——Pod IP 从 VPC 子网分配，需确保目标可用区子网有空闲 IP
# <SubnetId> 从 DescribeClusters 输出 ClusterNetworkSettings.Subnets 字段获取
tccli vpc DescribeSubnets --region <Region> \
    --SubnetIds '["<SubnetId>"]'
# expected: exit 0，AvailableIpAddressCount > 0，确保有空闲 IP 供新增节点使用
```

```json
{
  "TotalCount": 1,
  "SubnetSet": [
    {
      "SubnetId": "<SubnetId>",
      "SubnetName": "<SubnetName>",
      "CidrBlock": "<CidrBlock>",
      "Zone": "<Zone>",
      "AvailableIpAddressCount": <Count>,
      "TotalIpAddressCount": <Count>
    }
  ],
  "RequestId": "<RequestId>"
}
```

> **注意**：本集群使用 VPC-CNI 网络模式，Pod IP 直接从 VPC 子网分配。新增节点时需确保目标可用区子网 `AvailableIpAddressCount` 足够。若 `ClusterExternalEndpoint` 为空，则集群仅开放内网访问，kubectl 需通过 IOA / VPN / 专线 连接内网端点。

### 版本与规格说明

- 网络模式：`VPC-CNI`（Pod IP 从 VPC 子网分配），新建节点时需所在可用区子网 IP 充足
- K8s 版本：集群版本 `1.32.2`，节点版本应匹配
- OS 镜像：推荐 `tlinux3.1x86_64`（腾讯云优化内核），可通过 `tccli tke DescribeOSImages --region <Region>` 查询可用镜像

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<Region>` | 地域 | 如 `ap-guangzhou`、`ap-beijing` | `tccli configure list` |
| `<ClusterId>` | 集群 ID | 格式 `cls-xxxxxxxx` | `tccli tke DescribeClusters --region <Region>` |

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看已有节点池列表 | `DescribeClusterNodePools` | 是 |
| 查看集群节点列表 | `DescribeClusterInstances` | 是 |
| 普通节点池扩容（批量） | `ModifyNodePoolDesiredCapacityAboutAsg` | 否 |
| 新建普通节点池 | `CreateClusterNodePool` | 否 |
| 新增游离普通节点 | `CreateClusterInstances` | 否 |
| 添加已有 CVM | `AddExistedInstances` | 否 |

> 本页为导航页，各节点类型的具体创建参数和操作步骤见下方"操作步骤"中的专页链接。

## 操作步骤

### 步骤 1：列出当前集群节点池

了解集群中已有哪些节点池，确定是扩容已有池还是新建：

```bash
tccli tke DescribeClusterNodePools --region <Region> \
    --ClusterId <ClusterId>
# expected: exit 0，返回 NodePoolSet 列表（可为空）
```

```json
{
    "TotalCount": 2,
    "NodePoolSet": [
        {
            "NodePoolId": "<NodePoolId>",
            "Name": "<NodePoolName>",
            "LifeState": "normal",
            "NodeCountSummary": {
                "AutoscalingAdded": { "Joined": 0, "Total": 1 },
                "ManuallyAdded": { "Joined": 0 }
            }
        },
        {
            "NodePoolId": "<NodePoolId>",
            "Name": "<NodePoolName>",
            "LifeState": "normal",
            "NodeCountSummary": {
                "AutoscalingAdded": { "Joined": 0, "Total": 1 },
                "ManuallyAdded": { "Joined": 0 }
            }
        }
    ]
}
```

> `NodePoolSet` 为空表示集群尚无节点池，需按目标节点类型新建。`LifeState` 为 `normal` 表示节点池可用。

---

### 步骤 2：按节点类型进入对应操作页

根据需要的节点类型，选择对应的 tccli 操作专页：

#### 普通节点

最常用类型，底层为 CVM + 弹性伸缩组（ASG）。需指定机型、网络、磁盘和伸缩参数。

| 操作 | 适用场景 | 专页 |
|------|----------|------|
| 创建节点池 | 批量管理，定义池配置后通过调整期望节点数扩缩容 | [创建节点池](../../普通节点/创建节点池/tccli%20操作.md) |
| 新增游离普通节点 | 临时单节点，直接添加 CVM 到集群 | [新增游离普通节点](../../普通节点/新增游离普通节点/tccli%20操作.md) |
| 添加已有 CVM | 将现有 CVM 加入集群 | [新增游离普通节点](../../普通节点/新增游离普通节点/tccli%20操作.md) |

#### 原生节点

声明式节点管理，支持原地升降配。

- [新建原生节点](../../原生节点/新建原生节点/tccli%20操作.md)

#### 超级节点

Serverless 容器，按 Pod 计费，无需管理底层 CVM。

- [新建超级节点](../../超级节点/使用超级节点/新建超级节点/tccli%20操作.md)

#### 注册节点

将 IDC 或其他云平台机器注册到 TKE，用于混合云场景。

- [创建注册节点（公网版）](../../注册节点/创建注册节点（公网版）/tccli%20操作.md) -- 通过公网注册
- [创建注册节点（专线版）](../../注册节点/创建注册节点（专线版）/tccli%20操作.md) -- 通过专线注册

---

### 步骤 3：验证新增节点

新增节点后，通过控制面和数据面双重验证。

```bash
# 控制面：确认集群节点数增加
tccli tke DescribeClusterInstances --region <Region> \
    --ClusterId <ClusterId> \
    --InstanceRole WORKER
# expected: TotalCount 较新增前增加，新节点 InstanceState "running"
```

```json
{
  "TotalCount": 3,
  "InstanceSet": [
    {
      "InstanceId": "<InstanceId>",
      "InstanceRole": "WORKER",
      "InstanceState": "running",
      "LanIP": "172.24.0.34",
      "NodePoolId": "<NodePoolId>",
      "CreatedTime": "2026-06-23T08:13:42Z"
    },
    {
      "InstanceId": "<InstanceId>",
      "InstanceRole": "WORKER",
      "InstanceState": "initializing",
      "LanIP": "172.24.0.22",
      "NodePoolId": "<NodePoolId>",
      "CreatedTime": "2026-06-23T08:16:29Z"
    }
  ],
  "RequestId": "<RequestId>"
}
```

```bash
# 数据面：确认节点已加入 kubernetes（须 VPN/IOA）
kubectl get nodes
# expected: 新节点出现在列表中，STATUS 为 Ready
```

```text
NAME             STATUS   ROLES    AGE     VERSION
node-example-1   Ready    worker   12d     v1.32.2
node-example-2   Ready    worker   5m      v1.32.2
```

## 验证

### 控制面（tccli）

```bash
# 1. 确认节点池节点数增加
tccli tke DescribeClusterNodePools --region <Region> \
    --ClusterId <ClusterId>
# expected: 目标节点池 NodeCountSummary.AutoscalingAdded.Joined 或 ManuallyAdded.Joined 增加

# 2. 确认新节点在集群中
tccli tke DescribeClusterInstances --region <Region> \
    --ClusterId <ClusterId> \
    --InstanceRole WORKER
# expected: TotalCount 增加，新节点 InstanceState "running"，FailedReason 含 "Ready:True" 或为空
```

```json
{
  "TotalCount": 2,
  "NodePoolSet": [
    {
      "NodePoolId": "<NodePoolId>",
      "Name": "<NodePoolName>",
      "LifeState": "normal",
      "NodeCountSummary": {
        "AutoscalingAdded": { "Joined": 1, "Total": 2 },
        "ManuallyAdded": { "Joined": 0 }
      }
    }
  ],
  "RequestId": "<RequestId>"
}
```

```json
{
  "TotalCount": 2,
  "InstanceSet": [
    {
      "InstanceId": "<InstanceId>",
      "InstanceRole": "WORKER",
      "FailedReason": "=Ready:True",
      "InstanceState": "running",
      "LanIP": "<LanIP>",
      "NodePoolId": "<NodePoolId>",
      "CreatedTime": "<CreatedTime>"
    }
  ],
  "RequestId": "<RequestId>"
}
```

### 数据面（须 VPN/IOA）

```bash
# 3. 确认节点已加入 kubernetes 且 Ready
kubectl get nodes
# expected: 新节点 STATUS 为 Ready

# 4. 确认节点可调度
kubectl describe node NODE_NAME
# expected: Conditions 中 Ready 类型为 True，Unschedulable 为 false
```

```text
NAME  STATUS  AGE
...
```

## 清理

新增节点后如需移除，按对应节点类型的移除方式清理：

### 普通节点清理

- **移出单个节点**：见 [移出节点](../移出节点/tccli%20操作.md) -- page_id `32204`
- **删除节点池**（整池删除）：见 [删除节点池](../../普通节点/删除节点池/tccli%20操作.md)
- **直接销毁 CVM**（绕过 TKE 状态限制）：`tccli cvm TerminateInstances --InstanceIds '["<InstanceId>"]' --region <Region>`，集群侧会自动移除节点

### 清理前检查

```bash
# 确认待清理节点状态
tccli tke DescribeClusterInstances --region <Region> \
    --ClusterId <ClusterId> \
    --InstanceIds '["<InstanceId>"]'
# expected: 确认 InstanceId、InstanceState、NodePoolId
```

```json
{
  "TotalCount": 1,
  "InstanceSet": [
    {
      "InstanceId": "<InstanceId>",
      "InstanceRole": "WORKER",
      "FailedReason": "=Ready:True",
      "InstanceState": "running",
      "LanIP": "<LanIP>",
      "NodePoolId": "<NodePoolId>",
      "CreatedTime": "<CreatedTime>"
    }
  ],
  "RequestId": "<RequestId>"
}
```

### 重要警告

> **警告 1**：`DeleteClusterInstances --InstanceDeleteMode terminate` 会**销毁 CVM 实例和系统盘**，操作不可逆。生产环境执行前务必确认节点 ID。
>
> **警告 2**：处于 `initializing` 状态的节点**无法通过 `DeleteClusterInstances` 删除**（返回 `FailedOperation.Param: not in right state`）。此时可通过 `tccli cvm TerminateInstances` 直接销毁 CVM 实例，集群侧会自动移除该节点。
>
> **警告 3**：CVM `TerminateInstances` 直接销毁实例后，`DescribeClusterInstances` 中该节点会自动消失。若要保留 CVM 只从集群移除，使用 `InstanceDeleteMode: "remove"`（仅移出不解绑）。

- 其他类型：见对应类型专页的清理章节

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DescribeClusterNodePools` 返回空 | `tccli tke DescribeClusterNodePools --region <Region> --ClusterId <ClusterId>` 确认返回 `TotalCount` | 集群尚无节点池（正常情况） | 按所需节点类型进入对应专页创建节点池 |
| 普通节点池扩容失败 | `tccli tke DescribeClusterInstances --region <Region> --ClusterId <ClusterId> --InstanceRole WORKER` 检查新节点 `FailedReason` | CVM 机型库存不足或子网 IP 耗尽 | 换机型/可用区，或扩容子网 CIDR；参考 [创建节点池](../../普通节点/创建节点池/tccli%20操作.md) 排障节 |
| 超级节点 Pod 调度失败 | `kubectl describe pod POD_NAME` 查看 Events | 超级节点（虚拟节点）不支持部分调度特性 | 参考 [新建超级节点](../../超级节点/使用超级节点/新建超级节点/tccli%20操作.md) 的调度说明 |
| 不知道该选哪种节点类型 | 参考本页"概述"对比表，确认负载特征 | 无 | 默认使用**普通节点**（CVM + ASG，最通用），除非有明确的 Serverless 或混合云需求 |
| 创建集群公网端点失败 `InvalidParameter.Param` | `tccli tke DescribeClusterEndpoints --region <Region> --ClusterId <ClusterId>` 查看端点状态 | 公网端点被组织级 CAM 策略拒绝（strategyId:240463971，`tke:clusterExtranetEndpoint=true` 条件硬拒绝）。自建安全组也无法绕过 | 改用内网端点（`--IsExtranet false`），kubectl 需通过 IOA / VPN / 专线 或同 VPC CVM 连接内网端点（如 `172.24.0.12`）。此为环境限制，非命令错误 |
| VPC-CNI 模式 `CreateClusterInstances` 返回 `FailedOperation.NetworkScaleError` | `tccli tke DescribeClusterEndpoints --region <Region> --ClusterId <ClusterId>` 确认为 VPC-CNI 模式；`tccli vpc DescribeSubnets --region <Region> --SubnetIds '["<SubnetId>"]'` 检查 `AvailableIpAddressCount` | VPC IP for pods in zone `<Zone>` is not enough。VPC-CNI 模式下 Pod IP 从 VPC 子网分配，该可用区子网空闲 IP 不足——非 quota 问题，是子网容量问题 | 方案 1（生产环境）：通过 `tccli tke AddVpcCniSubnets` 扩容子网。方案 2（临时跳过校验）：加 `--SkipValidateOptions '["VpcCniCIDRCheck"]'`，但仅跳过校验不解决 IP 不足 |
| `DeleteClusterInstances` 返回 `FailedOperation.Param`（`some instances is not in right state`） | `tccli tke DescribeClusterInstances --region <Region> --ClusterId <ClusterId> --InstanceIds '["<InstanceId>"]'` 查看 `InstanceState` | 节点处于 `initializing` 状态，`DeleteClusterInstances` 无法删除未就绪节点。`ForceDelete true` 也无效 | 方案 1：等待节点变为 `running` 后重试 `DeleteClusterInstances`。方案 2（立即清理）：`tccli cvm TerminateInstances --InstanceIds '["<InstanceId>"]' --region <Region>` 直接销毁 CVM，集群侧自动移除节点 |

### 新增成功但状态异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 新节点 `InstanceState: running` 但 kubectl 不可见 | `tccli tke DescribeClusterInstances --region <Region> --ClusterId <ClusterId> --InstanceIds '["<InstanceId>"]'` 检查 `FailedReason`；`kubectl get nodes` 确认 | 节点仍在初始化（kubelet 未就绪）或 kubectl APIServer 不可达。若 `ClusterExternalEndpoint` 为空，仅内网可访问 | 等待 1-2 分钟后重试 `kubectl get nodes`；若 APIServer 不可达，通过 IOA / VPN / 专线 连接内网端点（`<ClusterIntranetEndpoint>`） |
| 新节点 `FailedReason` 非空 | `tccli tke DescribeClusterInstances --region <Region> --ClusterId <ClusterId> --InstanceIds '["<InstanceId>"]'` 查看 `FailedReason` 字段 | 节点初始化失败（网络、镜像拉取、kubelet 注册问题） | 根据 `FailedReason` 内容排查，参考 [节点生命周期](../节点生命周期/tccli%20操作.md) -- page_id `32202`。保留 RequestId + InstanceId 以备工单查询 |

## 下一步

- [节点池概述](../../节点池概述/tccli%20操作.md) -- 理解节点池架构
- [移出节点](../移出节点/tccli%20操作.md) -- page_id `32204`（移出不再需要的节点）
- [节点生命周期](../节点生命周期/tccli%20操作.md) -- page_id `32202`（理解节点状态流转）
- [节点概述](../../节点概述/tccli%20操作.md) -- 各节点类型的适用场景
- [腾讯云 TKE 新增节点](https://cloud.tencent.com/document/product/457/103964)

## 控制台替代

[容器服务控制台 - 节点管理](https://console.cloud.tencent.com/tke2/cluster)
