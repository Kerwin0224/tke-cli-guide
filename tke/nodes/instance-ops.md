---
doc_type: How-to
subtype: 6A
fused: true
---
# 节点实例运维

> 节点的查询、启停、删除。本节命令跨 TKE 两个 API 版本——按操作类型选择版本，所有命令显式带 `--version`。
> 完整版本选择见 [API 版本选择](../index.md#api-版本选择)。

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
# expected: MachineSet[] 含 InstanceId/InstanceState/ExistedLabels
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
  --filter "NodePoolSet[].{id:NodePoolId,name:Name,replicas:Replicas}" --output text
# expected: 节点池列表 id/name/replicas
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

> `CreateClusterInstances` 是新建 CVM 作节点（`RunInstancePara` 透传 CVM RunInstances JSON），与 `AddExistedInstances`（接入已有实例）区别。ECM/Edge 实例属专用场景，见 [边缘集群](../specialized/edge-cluster.md)。

```bash
# 新建 CVM 作节点（RunInstancePara 透传 CVM RunInstances JSON 字符串）
tccli tke CreateClusterInstances --version 2018-05-25 \
  --ClusterId "<CLUSTER_ID>" --region ap-guangzhou \
  --RunInstancePara '<CVM_RUNINSTANCES_JSON>'
# expected: { "Response": { "InstanceIdSet": ["ins-xxxxxxxx"] } }
```

| 占位符 | 含义 | 如何获取 |
|:-------|:-----|:---------|
| `<CVM_RUNINSTANCES_JSON>` | CVM RunInstances 入参 JSON | 含 InstanceType/ImageId/Placement/InstanceCount 等，可先用 `tccli cvm RunInstances --generate-cli-skeleton` 生成模板 |

> `CreateClusterInstances`（新建 CVM）vs `AddExistedInstances`（接入已有 CVM）：前者透传 CVM `RunInstances` JSON 让 TKE 代为创建，后者把已存在实例加入集群。参数 `RunInstancePara` 是 JSON 字符串，`SkipValidateOptions[]` 可跳过指定校验项。

## 下一步

- [创建节点池](nodepool-create.md) — 节点池生命周期
- [扩缩容节点池](nodepool-scale.md) — 调整节点数量
- [API 版本选择](../index.md#api-版本选择) — 理解 TKE 双版本

## Action 清单

| Action | 类型 | 版本 | 说明 |
|:-------|:-----|:-----|:-----|
| `AddExistedInstances` | 主操作 | 2018-05-25 | 接入已有 CVM 作节点 |
| `CreateClusterInstances` | 主操作 | 2018-05-25 | 新建 CVM 作节点（透传 RunInstances JSON） |
| `UpgradeClusterInstances` | 主操作 | 2018-05-25 | 升级节点版本 |
| `ModifyNodePoolDesiredCapacityAboutAsg` | 主操作 | 2018-05-25 | 改节点池期望数（旧版） |
| `DeleteClusterInstances` | 清理 | 2018-05-25 | 删除节点（Instance 抽象） |
| `RemoveNodeFromNodePool` | 清理 | 2018-05-25 | 移出节点池（不销毁 CVM） |
| `DrainClusterVirtualNode` | 主操作 | 2018-05-25 | 排水节点（驱逐 Pod） |
| `DescribeClusterInstances` | 验证 | 2018-05-25 | 节点列表（InstanceIds/InstanceRole） |
| `DescribeExistedInstances` | 验证 | 2018-05-25 | 查可接入的已有实例 |
| `StopMachines` | 主操作 | 2022-05-01 | 停止原生节点 |
| `StartMachines` | 主操作 | 2022-05-01 | 启动原生节点 |
| `RebootMachines` | 主操作 | 2022-05-01 | 重启原生节点（上限 100） |
| `ScaleNodePool` | 主操作 | 2022-05-01 | 设置节点池期望副本数 |
| `SetMachineLogin` | 主操作 | 2022-05-01 | 绑定 SSH 密钥（单数 MachineName） |
| `ModifyClusterMachine` | 主操作 | 2022-05-01 | 修改 Machine 粒度配置（复数 MachineNames） |
| `DeleteClusterMachines` | 清理 | 2022-05-01 | 删除节点（Machine 抽象） |
| `DescribeClusterMachines` | 验证 | 2022-05-01 | Machine 语义查询 |
| `DescribeNodePools` | 验证 | 2022-05-01 | 节点池列表 |
| `DescribeGPUInfo` | 验证 | 2022-05-01 | GPU 驱动/CUDA/cuDNN 版本（不绑集群） |
| `cvm:StopInstances` | 跨产品 | cvm | 旧版回退：停止节点 |
