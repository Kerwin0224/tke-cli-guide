---
doc_type: How-to
subtype: 6A
fused: true
---
# 节点实例运维

> 节点的查询、启停、删除。本节命令跨 TKE 两个 API 版本——按操作类型选择版本，所有命令显式带 `--version`。
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
- 你遇到节点运维问题需查诊断路径 — 看 [故障恢复] 或对应操作段


## 查询节点

```bash
# 2018-05-25 旧版: 用 InstanceIds/InstanceRole 过滤
tccli tke DescribeClusterInstances --version 2018-05-25 \
  --ClusterId "<CLUSTER_ID>" \
  --filter "InstanceSet[].{id:InstanceId,state:InstanceState,ip:LanIP}" \
  --output text
# expected: 节点列表, 每行 id/state/ip
```

> ⚠️ `DescribeClusterInstances` 两版同名但入参不兼容: 旧版用 `InstanceIds`/`InstanceRole`, 新版用 `SortBy`/`NeedTags`。若需 Machine 语义/排序/标签, 改用新版 `DescribeClusterMachines --version 2022-05-01`。切换前用 `--generate-cli-skeleton` 核契约。

```bash
# 2022-05-01 新版: 用 Filters/SortBy/NeedTags 查询（Machine 语义，含标签）
tccli tke DescribeClusterMachines --version 2022-05-01 \
  --ClusterId "<CLUSTER_ID>" --region ap-guangzhou \
  --Filters '[{"Name":"InstanceIds","Values":["<INSTANCE_ID>"]}]' \
  --Offset 0 --Limit 20
# expected: exit 0，返回 Machines[]+TotalCount（每项含 InstanceId/InstanceState/ExistedLabels 等 Machine 语义字段）
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

> 旧版 (2018-05-25) 回退方案: TKE 无节点启停 Action，走 CVM 服务 `tccli cvm StopInstances --InstanceIds '["<INSTANCE_ID>"]'`。

## 删除节点

```bash
# 2022-05-01 新版: DeleteClusterMachines (Machine 抽象)
tccli tke DeleteClusterMachines --version 2022-05-01 \
  --ClusterId "<CLUSTER_ID>" \
  --MachineNames '["<MACHINE_NAME>"]' --EnableScaleDown false
# expected: exit 0
```

> 旧版 (2018-05-25) 用 `DeleteClusterInstances --version 2018-05-25 --InstanceIds '["<INSTANCE_ID>"]'`（Instance 抽象）。两版抽象不同: 新版 Machine vs 旧版 Instance。

## 节点隔离与驱逐（kubectl，非 tccli）

> tke API 无 cordon/drain 普通节点 Action（仅 `DrainClusterVirtualNode` 超级节点 / `DrainExternalNode` 注册节点）。普通节点隔离用 **kubectl**（K8s 原生，非 tccli）。本段补全"故障→隔离→恢复"生命链——[健康检查](health-check.md) 检测到不可修复故障时，用此段隔离节点。

### 隔离节点（停止调度新 Pod）

> kubectl（K8s 原生命令，非 tccli；TCCLI 管 TKE 抽象层不提供 K8s 资源操作能力）
<!-- tccli无普通节点cordon能力(仅DrainClusterVirtualNode/DrainExternalNode)，kubectl管理K8s原生节点调度，非tccli边界 -->
```bash
kubectl cordon <NODE_NAME>
# expected: node <NODE_NAME> cordoned（节点标 SchedulingDisabled，已运行 Pod 不动）
```

### 驱逐节点上 Pod（维护前移走工作负载）

> kubectl（K8s 原生命令，非 tccli；TCCLI 管 TKE 抽象层不提供 K8s 资源操作能力）
<!-- tccli无普通节点drain能力(仅DrainClusterVirtualNode/DrainExternalNode)，kubectl管理K8s原生Pod驱逐，非tccli边界 -->
```bash
kubectl drain <NODE_NAME> --ignore-daemonsets --delete-emptydir-data
# expected: 逐个 evict Pod，DaemonSet Pod 保留（--ignore-daemonsets），emptyDir 数据删（--delete-emptydir-data）
```

> `kubectl drain` = `cordon` + 逐个 evict Pod。维护后恢复调度：

> kubectl（K8s 原生命令，非 tccli；TCCLI 管 TKE 抽象层不提供 K8s 资源操作能力）
<!-- kubectl管理K8s原生节点调度恢复，tccli无uncordon能力，非tccli边界 -->
```bash
kubectl uncordon <NODE_NAME>
# expected: node <NODE_NAME> uncordoned（恢复可调度）
```

| 操作 | 命令 | 作用 |
|:-----|:-----|:-----|
| 停止调度 | `kubectl cordon <NODE>` | 新 Pod 不调度到该节点 |
| 驱逐 Pod | `kubectl drain <NODE> --ignore-daemonsets --delete-emptydir-data` | 移走工作负载（维护/下线前） |
| 恢复调度 | `kubectl uncordon <NODE>` | 维护后恢复 |

> **生命链**：[健康检查](health-check.md) 检测故障 → 不可修则 `kubectl cordon` 隔离 → `kubectl drain` 驱逐 Pod → 删除/重建节点（见上文 [删除节点](#删除节点)）→ 新节点加入恢复容量。

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

## 查询 GPU 驱动版本 (2022-05-01)

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

### 新建 CVM 作节点（CreateClusterInstances）

> `CreateClusterInstances` 是新建 CVM 作节点（`RunInstancePara` 透传 CVM `RunInstances` JSON），与 `AddExistedInstances`（接入已有实例）区别。**不依赖 AS 节点池**，适合「只要 1 台普通 Worker 跑通」；缺 `AS_QCSRole` 时可用本路径绕过节点池。ECM/Edge 见 [边缘集群](../specialized/edge-cluster.md)。

```bash
# 最小可跑：1 台 POSTPAID + 指定子网/安全组/机型（字段以 cvm RunInstances 契约为准）
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
| `<PASSWORD>` | 登录密码 | 符合 CVM 复杂度；勿提交 git |
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

> **Tag 陷阱**：`RunInstancePara` 内 **不要**再塞 `TagSpecification`（含 `billing` 等）——TKE 可能自动打标，重复 key → `FailedOperation.CvmCommon` / `InvalidParameterValue.DuplicateTags`。标签改在集群/账号侧统一打，或创建后再 `cvm` 打标。

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

> `FileSystem` 取值以该字段契约为准（常见 `ext4`/`xfs`）。`MountTarget` 用本环境挂载点，勿复用他人环境路径。

## 收尾确认

> 本篇是多操作合集（查询/启停/删除/驱逐/扩缩/修改/GPU/接入），无统一收尾命令。每类操作执行后用下表对应命令确认该操作产物：

| 操作类型 | 确认命令 | 预期（②业务可用性端到端） |
|:---------|:---------|:--------------------------|
| 启停/重启 | `tccli tke DescribeClusterMachines --version 2022-05-01 --ClusterId "<CLUSTER_ID>" --Filters '[{"Name":"InstanceIds","Values":["<ID>"]}]'` | ②业务可用性: InstanceState=Stopped(停止后)/Running(启动后)；启动后 `kubectl get nodes` 节点须 Ready |
| 删除节点 | `tccli tke DescribeClusterInstances --version 2018-05-25 --ClusterId "<CLUSTER_ID>" --InstanceIds '["<ID>"]'` | ②业务可用性: 目标节点不在列表（已删）；剩余 `kubectl get nodes` 无 NotReady |
| 驱逐 (kubectl) | `kubectl get nodes <NODE_NAME>` | ②业务可用性: 节点 SchedulingDisabled + Pod 已迁移无 Pending |
| 扩缩容 | 见 [扩缩容节点池](nodepool-scale.md) 收尾确认 | ③跨步骤汇总: 新版 LifeState=Running + Replicas==ReadyReplicas；旧版 LifeState=normal + DesiredNodesNum==NodeCountSummary |
| 接入已有 CVM | `tccli tke DescribeClusterInstances --version 2018-05-25 --ClusterId "<CLUSTER_ID>" --InstanceIds '["<ID>"]'` | ②业务可用性: 接入节点 InstanceState=running 且 `kubectl get nodes` 含该节点 Ready |

---

## 下一步

- [创建节点池](nodepool-create.md) — 节点池生命周期
- [扩缩容节点池](nodepool-scale.md) — 调整节点数量
- [API 版本选择](../index.md#api-版本选择) — 理解 TKE 双版本
