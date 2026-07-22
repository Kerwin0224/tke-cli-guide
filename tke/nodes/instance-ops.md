---
doc_type: How-to
subtype: 6A
fused: true
---
# 节点实例运维

> 节点的查询、启停、删除。本文命令跨 TKE 两个 API 版本——按操作类型选择版本，所有命令显式带 `--version`。
> 完整版本选择见 [API 版本选择](../index.md#api-版本选择)。
>
> 官方文档：[节点概述](https://cloud.tencent.com/document/product/457/32201) · [集群生命周期](https://cloud.tencent.com/document/product/457/32188) · [常见高危操作](https://cloud.tencent.com/document/product/457/39539)
>
> 配额：启停/重装/退还/销毁按 CVM 实例配额、重启节点上限 100。[配额说明](https://cloud.tencent.com/document/product/457/9087)
>
> ⚠️ **高危操作**：StopMachines 停机前需驱逐 Pod 防业务中断、DeleteClusterMachines 销毁不可逆、SetMachineLogin 修改登录凭据影响节点安全。[常见高危操作](https://cloud.tencent.com/document/product/457/39539)

## 触发条件

- DescribeClusterInstances/DescribeClusterMachines 返回节点列表，需对具体节点执行启停/删除/驱逐/扩缩/修改/GPU/接入操作 — 从 [查询节点](#查询节点) 开始定位节点
- 节点出现 NotReady 或需维护（驱逐 Pod 后重建）— 看 [节点隔离与驱逐](#节点隔离与驱逐kubectl非-tccli) 段
- 你遇到节点运维问题需查诊断路径 — 看各操作段内的故障表，或 [故障排查](../troubleshooting.md)

## 准备工作

```bash
# 确认集群 Running
tccli tke DescribeClusterStatus --region ap-guangzhou --ClusterIds '["<CLUSTER_ID>"]' \
  --filter "ClusterStatusSet[].ClusterState" --output text
# expected: Running

# 取得待运维节点（旧版 Instance 抽象）
tccli tke DescribeClusterInstances --version 2018-05-25 \
  --ClusterId "<CLUSTER_ID>" \
  --filter "InstanceSet[].{id:InstanceId,state:InstanceState,ip:LanIP}" --output text
# expected: 节点列表；记下目标 InstanceId
```

- 启停 / 删除 / 修改前确认节点所属集群与 API 版本（本文命令均显式 `--version`）
- 隔离 / 驱逐普通节点需 kubectl 已配置且可达该集群（见 [集群认证](../security/auth.md)）
- 接入已有 CVM 前：实例与集群同 VPC，且 `DescribeExistedInstances` 返回 `Usable=true`

## 查询节点 {#查询节点}

```bash
# 2018-05-25 旧版: 用 InstanceIds/InstanceRole 过滤
tccli tke DescribeClusterInstances --version 2018-05-25 \
  --ClusterId "<CLUSTER_ID>" \
  --filter "InstanceSet[].{id:InstanceId,state:InstanceState,ip:LanIP}" \
  --output text
# expected: 节点列表, 每行 id/state/ip
```

> ⚠️ 三套查询勿混用：
> - `DescribeClusterInstances --version 2018-05-25`：`InstanceIds`/`InstanceRole` 顶层参数；出参 `InstanceSet[].InstanceState`
> - `DescribeClusterInstances --version 2022-05-01`：`Filters`（含 `InstanceIds`/`InstanceStates`/`NodePoolIds` 等）+ 可选 `SortBy`/`NeedTags`
> - `DescribeClusterMachines --version 2022-05-01`：原生节点 Machine 语义；`Filters` **仅** `NodePoolsName`/`NodePoolsId`/`tags`/`tag:tag-key`（**无** `InstanceIds`）；出参 `Machines[].MachineName`/`MachineState`（**不是** `InstanceState`）

```bash
# 2022-05-01：按实例 ID 过滤用 DescribeClusterInstances（Filters.Name=InstanceIds）
tccli tke DescribeClusterInstances --version 2022-05-01 \
  --ClusterId "<CLUSTER_ID>" --region ap-guangzhou \
  --Filters '[{"Name":"InstanceIds","Values":["<INSTANCE_ID>"]}]' \
  --Offset 0 --Limit 20
# expected: exit 0，InstanceSet[]+TotalCount

# 2022-05-01：原生节点 Machine 列表（按节点池过滤；无 InstanceIds Filter）
tccli tke DescribeClusterMachines --version 2022-05-01 \
  --ClusterId "<CLUSTER_ID>" --region ap-guangzhou \
  --Filters '[{"Name":"NodePoolsId","Values":["<NODE_POOL_ID>"]}]' \
  --Offset 0 --Limit 20
# expected: exit 0，Machines[]+TotalCount（MachineName/MachineState/LanIP/InstanceType 等）
```

## 启动 / 停止 / 重启 (原生节点)

> 节点启停是 **2022-05-01 新版独有** Action（旧版无，需走 `cvm:StartInstances` 等）。必须显式 `--version 2022-05-01`，否则静默走旧版会报 `UnknownAction`。

```bash
# 停止 (仅 Running 节点)
tccli tke StopMachines --version 2022-05-01 \
  --ClusterId "<CLUSTER_ID>" --MachineNames '["<MACHINE_NAME>"]'
# expected: exit 0, RequestId 返回

# 启动 (仅 Stopped 节点)
tccli tke StartMachines --version 2022-05-01 \
  --ClusterId "<CLUSTER_ID>" --MachineNames '["<MACHINE_NAME>"]'
# expected: exit 0

# 重启 (上限 100)
tccli tke RebootMachines --version 2022-05-01 \
  --ClusterId "<CLUSTER_ID>" --MachineNames '["<MACHINE_NAME>"]'
# expected: exit 0
```

`StopMachines` 与 `RebootMachines` 共用 `StopType` 停机方式：

| `StopType` | 行为 |
|:-----------|:-----|
| `soft` | 仅执行软关机；`RebootMachines` 的静态默认值为此值 |
| `soft_first` | 先执行软关机，失败后再强制关机 |
| `hard` | 直接强制关机 |

`StopMachines` 的默认值以该 Action 的 `help --detail` 为准；需要固定行为时显式传 `--StopType <值>`。

> 旧版 (2018-05-25) 回退方案: TKE 无节点启停 Action，走 CVM 服务 `tccli cvm StopInstances --InstanceIds '["<INSTANCE_ID>"]'`。

## 删除节点 {#删除节点}

```bash
# 2022-05-01 新版: DeleteClusterMachines (Machine 抽象)
tccli tke DeleteClusterMachines --version 2022-05-01 \
  --ClusterId "<CLUSTER_ID>" \
  --MachineNames '["<MACHINE_NAME>"]' --EnableScaleDown false
# expected: exit 0
```

`InstanceDeleteMode` 决定删除后的实例资源（两版均可传；旧版 `DeleteClusterInstances` 为 **Required**，新版 `DeleteClusterMachines` 为 Optional）：

| 值 | 结果 |
|:---|:-----|
| `terminate` | 将节点移出集群并销毁实例；仅支持按量计费 CVM |
| `retain` | 仅将节点移出集群，保留 CVM 实例 |

> 旧版 (2018-05-25) 用 `DeleteClusterInstances --version 2018-05-25 --InstanceIds '["<INSTANCE_ID>"]' --InstanceDeleteMode retain`（Instance 抽象；`InstanceDeleteMode` 必填，不传 exit 252）。两版抽象不同: 新版 Machine vs 旧版 Instance。

## 节点隔离与驱逐（kubectl，非 tccli） {#节点隔离与驱逐kubectl非-tccli}

> tke API 无 cordon/drain 普通节点 Action（仅 `DrainClusterVirtualNode` 超级节点 / `DrainExternalNode` 注册节点）。普通节点隔离用 **kubectl**（K8s 原生，非 tccli）。本段给出"故障→隔离→恢复"操作路径——[健康检查](health-check.md) 检测到不可修复故障时，用此段隔离节点。

### 隔离节点（停止调度新 Pod）

> kubectl（K8s 原生命令，非 tccli；TCCLI 管 TKE 抽象层不提供 K8s 资源操作能力）
```bash
kubectl cordon <NODE_NAME>
# expected: node <NODE_NAME> cordoned（节点标 SchedulingDisabled，已运行 Pod 不动）
```

### 驱逐节点上 Pod（维护前移走工作负载）

> kubectl（K8s 原生命令，非 tccli；TCCLI 管 TKE 抽象层不提供 K8s 资源操作能力）
```bash
kubectl drain <NODE_NAME> --ignore-daemonsets --delete-emptydir-data
# expected: 逐个 evict Pod，DaemonSet Pod 保留（--ignore-daemonsets），emptyDir 数据删（--delete-emptydir-data）
```

> `kubectl drain` = `cordon` + 逐个 evict Pod。维护后恢复调度：

> kubectl（K8s 原生命令，非 tccli；TCCLI 管 TKE 抽象层不提供 K8s 资源操作能力）
```bash
kubectl uncordon <NODE_NAME>
# expected: node <NODE_NAME> uncordoned（恢复可调度）
```

| 操作 | 命令 | 作用 |
|:-----|:-----|:-----|
| 停止调度 | `kubectl cordon <NODE>` | 新 Pod 不调度到该节点 |
| 驱逐 Pod | `kubectl drain <NODE> --ignore-daemonsets --delete-emptydir-data` | 移走工作负载（维护/下线前） |
| 恢复调度 | `kubectl uncordon <NODE>` | 维护后恢复 |

> **故障→隔离→恢复路径**：[健康检查](health-check.md) 检测故障 → 不可修则 `kubectl cordon` 隔离 → `kubectl drain` 驱逐 Pod → 删除/重建节点（见上文 [删除节点](#删除节点)）→ 新节点加入恢复容量。

## 扩缩容节点池 (2022-05-01)

> `ScaleNodePool` 是 2022-05-01 新版 Action，设置节点池期望副本数（`Replicas`），旧版用 `ModifyNodePoolDesiredCapacityAboutAsg`。

```bash
tccli tke ScaleNodePool --version 2022-05-01 \
  --ClusterId "<CLUSTER_ID>" --NodePoolId "<NODEPOOL_ID>" --Replicas <REPLICAS>
# expected: exit 0, RequestId 返回
```
```bash
# 查询节点池 ID
tccli tke DescribeNodePools --version 2022-05-01 --ClusterId "<CLUSTER_ID>" \
  --filter "NodePools[].{id:NodePoolId,name:Name,replicas:Native.Replicas}" --output text
# expected: 节点池列表 id/name/replicas（新版 2022 集合名是 NodePools 非 NodePoolSet；Replicas 嵌套在 Native 对象内）
```

## 修改原生节点配置 (2022-05-01)

> `ModifyClusterMachine` 修改 Machine 粒度配置（系统盘/安全组/计费/显示名），旧版无此粒度。

```bash
tccli tke ModifyClusterMachine --version 2022-05-01 \
  --ClusterId "<CLUSTER_ID>" \
  --MachineNames '["<MACHINE_NAME>"]' \
  --SecurityGroupIDs '["<SG_ID>"]'
# expected: exit 0
```

> 入参含 `MachineNames[]`(复数)/`SystemDisk`(嵌套)/`SecurityGroupIDs[]`/`InstanceChargePrepaid`。`SetMachineLogin` 是配套的 SSH 密钥绑定:

```bash
tccli tke SetMachineLogin --version 2022-05-01 \
  --ClusterId "<CLUSTER_ID>" --MachineName "<MACHINE_NAME>" --KeyIds '["<KEY_ID>"]'
# expected: exit 0
```

> ⚠️ `SetMachineLogin` 用单数 `MachineName`，`ModifyClusterMachine` 用复数 `MachineNames`——同域字段名不一致。

## 查询 GPU 驱动版本 (2022-05-01) {#查询-gpu-驱动版本-2022-05-01}

> `DescribeGPUInfo` 不绑集群，按机型+OS 查询可用 GPU 驱动/CUDA/cuDNN 版本，创建 GPU 节点前用。

```bash
tccli tke DescribeGPUInfo --version 2022-05-01 \
  --InstanceType "<INSTANCE_TYPE>" --OsName "<OS_NAME>" --region <REGION>
# expected: exit 0, GPUParams[] 含 Driver/CUDA/CUDNN
```
```json
{
    "GPUParams": [
        {"Driver": "470.182.03", "CUDA": "11.4.3", "CUDNN": "8.2.4", "MIGEnable": false},
        {"Driver": "535.216.01", "CUDA": "12.4.1", "CUDNN": "9.5.1", "MIGEnable": false}
    ]
}
```

> 返回多个驱动版本组合，创建 GPU 节点时选其一。`InstanceType` 如 `GN10X.P.20XLARGE`，`OsName` 如 `ubuntu22.04x64_32`。

## 接入已有 CVM 实例 (2018-05-25)

> 将已存在的 CVM 实例加入集群作节点（非新建 CVM）。先查可接入实例，再加节点。

```bash
# 1. 查询可接入的已有实例 (Usable=true 才能加)
tccli tke DescribeExistedInstances --version 2018-05-25 \
  --ClusterId "<CLUSTER_ID>" --region <REGION> --Limit 10 \
  --filter "ExistedInstanceSet[?Usable==`true`].{id:InstanceId,name:InstanceName,ip:PrivateIpAddresses[0]}" --output text
# expected: 可接入实例列表 (Usable=false 的已在集群或不可用)
```
```json
{
    "ExistedInstanceSet": [
        {"Usable": false, "UnusableReason": "already in Cluster", "AlreadyInCluster": "cls-example",
         "InstanceId": "ins-example", "InstanceName": "tke_cls-example_worker"}
    ]
}
```

```bash
# 2. 接入已有实例作节点
tccli tke AddExistedInstances --version 2018-05-25 \
  --ClusterId "<CLUSTER_ID>" --InstanceIds '["<INSTANCE_ID>"]'
# expected: exit 0
```

### 接入已有实例字段约束 {#接入已有实例字段约束}

## 跨字段约束

| Action | 字段组合 | 精确关系 |
|:-------|:---------|:---------|
| `DescribeExistedInstances` | `ClusterId`、`InstanceIds`、`Filters` | `InstanceIds` 不能与 `ClusterId` 或 `Filters` 同传。传 `ClusterId` 时，服务会把该集群的 VPC ID 附加为过滤条件；若 `Filters` 也指定 `vpc-id`，其值必须与集群 VPC 相同 |
| `AddExistedInstances` | `InstanceIds[]`、`InstanceAdvancedSettings`、`InstanceAdvancedSettingsOverrides[]` | 覆盖数组与实例数组按下标同序对应；对应项覆盖公共设置，未提供覆盖项的实例沿用公共设置；覆盖数组长度不得大于实例数组 |

| 字段 | 所属 Action | 必填 | 条件说明 |
|:---|:---|:---:|:---|
| `HostName` | `AddExistedInstances` | 条件 | 仅在重装接入且集群使用 HostName 模式时必传 |
| `InstanceAdvancedSettingsOverrides[]` | `AddExistedInstances` | 否 | 按顺序与 `InstanceIds[]` 对应；长度不得大于 `InstanceIds[]`。数组较短时，未覆盖的实例使用公共 `InstanceAdvancedSettings` |

### 新建 CVM 作节点（CreateClusterInstances） {#新建-cvm-作节点createclusterinstances}

> `CreateClusterInstances` 是新建 CVM 作节点（`RunInstancePara` 透传 CVM `RunInstances` JSON），与 `AddExistedInstances`（接入已有实例）区别。**不依赖 AS 节点池**，适合「仅需 1 台普通 Worker 做最小验证」；缺 `AS_QCSRole` 时可用该路径绕过节点池。ECM/Edge 见 [边缘集群](../specialized/edge-cluster.md)。

```bash
# 最小验证：1 台 POSTPAID + 指定子网/安全组/机型（字段以 cvm RunInstances 入参为准）
# 机型无货时先 DescribeZoneInstanceConfigInfos 取 Status=SELL 最小规格
tccli tke CreateClusterInstances --version 2018-05-25 \
  --ClusterId "<CLUSTER_ID>" --region ap-guangzhou \
  --RunInstancePara '{
    "Placement":{"Zone":"<ZONE>","ProjectId":0},
    "InstanceType":"<INSTANCE_TYPE>",
    "ImageId":"<IMAGE_ID>",
    "SystemDisk":{"DiskType":"CLOUD_PREMIUM","DiskSize":50},
    "VirtualPrivateCloud":{"VpcId":"<VPC_ID>","SubnetId":"<SUBNET_ID>"},
    "InternetAccessible":{"InternetChargeType":"TRAFFIC_POSTPAID_BY_HOUR","InternetMaxBandwidthOut":1,"PublicIpAssigned":true},
    "InstanceCount":1,
    "InstanceName":"<INSTANCE_NAME>",
    "LoginSettings":{"Password":"<PASSWORD>"},
    "SecurityGroupIds":["<SECURITY_GROUP_ID>"],
    "InstanceChargeType":"POSTPAID_BY_HOUR"
  }' \
  --InstanceAdvancedSettings '{"Unschedulable":0}'
# expected: {"InstanceIdSet":["ins-xxxxxxxx"],"RequestId":"..."}
```

| 占位符 | 含义 | 如何获取 |
|:-------|:-----|:---------|
| `<ZONE>` | 可用区 | 与子网同区，如 `ap-guangzhou-7` |
| `<INSTANCE_TYPE>` | 机型 | `tccli cvm DescribeZoneInstanceConfigInfos` → `Status=SELL`（常见最小 2C2G 如 `SA2.MEDIUM2`） |
| `<IMAGE_ID>` | 公共镜像 | `tccli cvm DescribeImages`（如 TencentOS Server 3.1） |
| `<VPC_ID>` / `<SUBNET_ID>` | 集群 VPC/子网 | 须与集群同 VPC |
| `<SECURITY_GROUP_ID>` | 节点安全组 | 见 [创建节点池 — 安全组](nodepool-create.md#安全组节点加入前) |
| `<PASSWORD>` | 登录密码 | 符合 CVM 复杂度；不要提交到版本库 |
| `<INSTANCE_NAME>` | 实例名 | 可含业务前缀 |

```bash
# 等待节点加入集群（ClusterRunningNodeNum ≥ 1）
tccli tke DescribeClusterStatus --region ap-guangzhou \
  --ClusterIds '["<CLUSTER_ID>"]' \
  --waiter '{"expr":"ClusterStatusSet[0].ClusterRunningNodeNum","to":1,"timeout":600,"interval":15}'
# expected: running 节点数 ≥ 1

tccli tke DescribeClusterInstances --version 2018-05-25 \
  --ClusterId "<CLUSTER_ID>" \
  --filter "InstanceSet[].{id:InstanceId,state:InstanceState,lan:LanIP}" --output text
# expected: 新节点 InstanceState=running
```

> **Tag 约束**：`RunInstancePara` 内 **不要**再塞 `TagSpecification`（含 `billing` 等）——TKE 可能自动打标，重复 key → `FailedOperation.CvmCommon` / `InvalidParameterValue.DuplicateTags`。标签改在集群/账号侧统一打，或创建后再 `cvm` 打标。

> `CreateClusterInstances`（新建 CVM）vs `AddExistedInstances`（接入已有 CVM）：前者透传 `RunInstances` 让 TKE 代建，后者把已存在实例加入集群。`SkipValidateOptions[]` 可跳过指定校验项。

| 现象 | 诊断 | 修复 |
|:-----|:-----|:-----|
| `AuthFailure` / `UnauthorizedOperation` | 用户 CAM 无 `cvm:RunInstances` 等 | 给子账号补 CVM 权限（与 AS 服务角色无关） |
| `InvalidParameterValue.DuplicateTags` | RunInstancePara 含 TagSpecification | 去掉透传 Tags 后重试 |
| 实例已建但 `ClusterRunningNodeNum` 长期 0 | `DescribeClusterInstances` → `InstanceState` | 查安全组/镜像/初始化；节点 `failed` 则删后重建 |

### 多块数据盘节点

新建节点且需 ≥2 块数据盘时：`RunInstancePara` 内的 CVM `DataDisks` 负责开盘；`InstanceAdvancedSettings.DataDisks` 负责格式化与挂载。两侧块数一致。仅传 CVM `DataDisks` 时，盘可能未格式化、未挂到目标目录。

```bash
# 多块数据盘：CVM DataDisks 开盘 + InstanceAdvancedSettings.DataDisks 挂载
tccli tke CreateClusterInstances --version 2018-05-25 \
  --ClusterId "<CLUSTER_ID>" --region ap-guangzhou \
  --RunInstancePara '<CVM_RUNINSTANCES_JSON_WITH_DATADISKS>' \
  --InstanceAdvancedSettings '<IAS_WITH_DATADISKS>'
# expected: exit 0（或 CAM 拦截后授权）；返回 InstanceIdSet[]
```

| 占位符 | 含义 | 约束 |
|:-------|:-----|:-----|
| `<CVM_RUNINSTANCES_JSON_WITH_DATADISKS>` | CVM `RunInstances` JSON | 含 `DataDisks:[{DiskType,DiskSize},…]`（≥2 块）及 Placement/InstanceType/VPC 等 |
| `<IAS_WITH_DATADISKS>` | TKE 节点高级设置 | `DataDisks[]` 每项含 `DiskType`/`DiskSize`/`FileSystem`/`MountTarget`/`AutoFormatAndMount`；块数与 CVM 侧一致 |

`InstanceAdvancedSettings.DataDisks` 结构：

```json
{
  "DataDisks": [
    {"AutoFormatAndMount": true, "DiskSize": 50, "DiskType": "CLOUD_BSSD", "FileSystem": "ext4", "MountTarget": "<MOUNT_1>"},
    {"AutoFormatAndMount": true, "DiskSize": 50, "DiskType": "CLOUD_BSSD", "FileSystem": "ext4", "MountTarget": "<MOUNT_2>"}
  ]
}
```

> `FileSystem` 取值以该字段的 Action 入参为准（常见 `ext4`/`xfs`）。`MountTarget` 用本环境挂载点，不要复用他人环境路径。

## 验证

> 各写操作后按类型核对（与「收尾确认」表一致；本段给出最短命令）：

```bash
# 启停后：原生节点 MachineState（按节点池列 Machine，再对照 MachineName）
tccli tke DescribeClusterMachines --version 2022-05-01 \
  --ClusterId "<CLUSTER_ID>" \
  --Filters '[{"Name":"NodePoolsId","Values":["<NODE_POOL_ID>"]}]' \
  --filter "Machines[?MachineName=='<MACHINE_NAME>'].MachineState" --output text
# expected: Stopped（停止后）或 Running（启动后）；无 InstanceIds Filter、无 InstanceState 字段

# 删除后：目标节点不在列表
tccli tke DescribeClusterInstances --version 2018-05-25 \
  --ClusterId "<CLUSTER_ID>" --InstanceIds '["<INSTANCE_ID>"]'
# expected: InstanceSet 不含该 ID 或为空

# 接入后：节点 running 且 kubectl Ready（kubectl 为 K8s 原生，非 tccli）
tccli tke DescribeClusterInstances --version 2018-05-25 \
  --ClusterId "<CLUSTER_ID>" --InstanceIds '["<INSTANCE_ID>"]' \
  --filter "InstanceSet[].InstanceState" --output text
# expected: running
```

## 故障恢复

| 现象 | 诊断 | 根因 | 修复 |
|:-----|:-----|:-----|:-----|
| 启停后状态不符 | `DescribeClusterMachines` → `MachineState` | 异步未完成或 Filter/字段混用 | 等状态收敛；Filter 仅 NodePools* / tags；字段为 MachineState/MachineName |
| 删除后节点仍在列表 | `DescribeClusterInstances` | 删错 ID 或删保护 | 核对 InstanceIds；原生节点走 `DeleteClusterMachines` |
| 驱逐后 Pod 卡住 | `kubectl describe pod` / Events | PDB、本地卷、DaemonSet | 放宽 PDB 或按业务删 Pod；勿对普通节点误用虚拟/注册节点 Drain Action |
| 接入 CVM 后 NotReady | `DescribeClusterInstances` + `kubectl get nodes` | 网络/安全组/运行时 | 查实例状态与安全组；必要时 `retain` 后重建接入 |
| 扩缩容未达副本 | 见 [扩缩容节点池](nodepool-scale.md) | 配额/库存/健康检查 | 按该篇故障表处理 |

更广路径见 [故障排查](../troubleshooting.md)。健康检查触发的隔离见上文 [节点隔离与驱逐](#节点隔离与驱逐kubectl非-tccli)。

## 清理

> 本文多数操作是对**已有节点**的运维；「接入已有 CVM」「`CreateClusterInstances` 新建节点」会新增节点。下线测试节点用删除命令（不可逆时先 `retain` 或确认业务已迁出）。

```bash
# 新版：删除原生节点（EnableScaleDown=false 不联动缩容 AS）
tccli tke DeleteClusterMachines --version 2022-05-01 \
  --ClusterId "<CLUSTER_ID>" \
  --MachineNames '["<MACHINE_NAME>"]' --EnableScaleDown false
# expected: exit 0

# 旧版：移出集群；InstanceDeleteMode=retain 保留 CVM，terminate 销毁按量实例
tccli tke DeleteClusterInstances --version 2018-05-25 \
  --ClusterId "<CLUSTER_ID>" --InstanceIds '["<INSTANCE_ID>"]' \
  --InstanceDeleteMode "retain"
# expected: exit 0
```

> 扩缩容产生的节点生命周期归节点池，清理见 [扩缩容节点池](nodepool-scale.md) / [创建节点池](nodepool-create.md#清理)。

## 收尾确认

> 本文是多操作合集（查询/启停/删除/驱逐/扩缩/修改/GPU/接入），无统一收尾命令。每类操作执行后用下表对应命令确认该操作产物：

| 操作类型 | 确认命令 | 预期（端到端可用性） |
|:---------|:---------|:--------------------------|
| 启停/重启 | `tccli tke DescribeClusterMachines --version 2022-05-01 --ClusterId "<CLUSTER_ID>" --Filters '[{"Name":"NodePoolsId","Values":["<NODE_POOL_ID>"]}]'` | 目标 `MachineName` 的 `MachineState`=Stopped(停止后)/Running(启动后)；启动后 `kubectl get nodes` 节点须 Ready |
| 删除节点 | `tccli tke DescribeClusterInstances --version 2018-05-25 --ClusterId "<CLUSTER_ID>" --InstanceIds '["<ID>"]'` | 目标节点不在列表（已删）；剩余 `kubectl get nodes` 无 NotReady |
| 驱逐 (kubectl) | `kubectl get nodes <NODE_NAME>` | 节点 SchedulingDisabled + Pod 已迁移无 Pending |
| 扩缩容 | 见 [扩缩容节点池](nodepool-scale.md) 收尾确认 | 新版 LifeState=Running + Replicas==ReadyReplicas；旧版 LifeState=normal + DesiredNodesNum==NodeCountSummary |
| 接入已有 CVM | `tccli tke DescribeClusterInstances --version 2018-05-25 --ClusterId "<CLUSTER_ID>" --InstanceIds '["<ID>"]'` | 接入节点 InstanceState=running 且 `kubectl get nodes` 含该节点 Ready |

---

## 下一步

- [创建节点池](nodepool-create.md) — 节点池生命周期
- [扩缩容节点池](nodepool-scale.md) — 调整节点数量
- [API 版本选择](../index.md#api-版本选择) — 理解 TKE 双版本

## 接入与新建字段

| 字段 | 所属 Action | 必填 | 说明 |
|:---|:---|:---:|:---|
| `InstanceIds` | `AddExistedInstances` | 是 | 待接入实例 ID 列表 |
| `RunInstancePara` | `CreateClusterInstances` | 是 | 透传 CVM RunInstances 参数 |
