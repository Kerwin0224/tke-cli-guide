---
doc_type: Reference
subtype: 8B
---
# TKE 共享字段参考

> TKE 多个 Action 共用的嵌套字段结构。这些字段在创建集群、添加节点、节点池等操作中重复出现，统一在此引导一次，各文档引用本页，不重复展开。
>
> 官方文档：[API 概览](https://cloud.tencent.com/document/product/457/31884)

## 何时查本页

- 写 `CreateCluster` / `AddExistedInstances` / `CreateClusterNodePool` / `ScaleOutClusterMaster` 等命令时，遇到 `InstanceAdvancedSettings` / `EnhancedService` / `LoginSettings` / `Label` / `Taint` / `TagSpecification` / `Filter` 等参数
- 需确认某字段是顶层传还是嵌套传、用 `--cli-unfold-argument` 还是 JSON

> 完整字段结构以 `tccli tke <Action> help --detail` 为准；本页只列跨 Action 通用的语义与传参模式。

## Filter 查询过滤 {#filter-查询过滤}

`Filters` 是部分查询 Action 提供的过滤参数。本节以 `DescribeClusters` 为例；其他 Action 是否支持、参数名以及合法 `Name`，须以该 Action 的 `help --detail` 为准。

| 字段 | 类型 | 说明 |
|------|------|------|
| Name | string | `DescribeClusters` 支持 `ClusterName`、`ClusterType`、`ClusterStatus`、`vpc-id`、`tag-key`、`tag-value`、`Tags`（大小写须精确） |
| Values | array | 属性值列表，同 Filter 内多值是 OR，多 Filter 间是 AND |

```bash
# 多 Filter AND，同 Filter 多 Value OR；ClusterName 按集群名过滤
tccli tke DescribeClusters --region <REGION> \
  --Filters '[
    {"Name":"ClusterName","Values":["prod"]},
    {"Name":"vpc-id","Values":["vpc-aaa","vpc-bbb"]}
  ]'
# expected: Clusters[] 只含名称匹配 prod 且 VPC 为 vpc-aaa/vpc-bbb 的集群
```

> 按集群 ID 查询应使用顶层参数 `--ClusterIds '["<CLUSTER_ID>"]'`，不要虚构 `cluster-id` Filter。各 Describe Action 支持的 Filter `Name` 不同，用 `tccli tke <Action> help --detail` 查该 Action 的合法 Name；不要把一个 Action 的 Name 迁移到另一个 Action。传错 Name 返回 `InvalidParameter`。

## Label / Taint / Annotation 节点调度标记 {#label-taint-annotation}

三者结构相似，用途不同：Label 打标签供选择器匹配，Taint 驱逐/排斥 Pod，Annotation 存非调度元数据。

### Label

| 字段 | 类型 | 说明 |
|------|------|------|
| Name | string | 标签键（如 `env`） |
| Value | string | 标签值（如 `production`） |

### Taint

| 字段 | 类型 | 说明 |
|------|------|------|
| Key | string | 污点键（如 `dedicated`） |
| Value | string | 污点值（如 `gpu`） |
| Effect | string | 效果：`NoSchedule`/`PreferNoSchedule`/`NoExecute` |

### Annotation

结构与 Label 相同（Name/Value）。

```bash
# Label + Taint 传参 (cli-unfold-argument 展开数组)
tccli tke CreateNodePool --version 2022-05-01 --region <REGION> \
  --cli-unfold-argument \
  --ClusterId "<CLUSTER_ID>" --Name "<POOL_NAME>" --Type Native \
  --Native.SubnetIds '["<SUBNET_ID>"]' --Native.InstanceTypes '["S5.MEDIUM4"]' \
  --Native.InstanceChargeType POSTPAID_BY_HOUR \
  --Native.SystemDisk '{"DiskType":"CLOUD_PREMIUM","DiskSize":50}' \
  --Native.Scaling '{"MinReplicas":1,"MaxReplicas":1,"CreatePolicy":"ZoneEquality"}' \
  --Labels.0.Name env --Labels.0.Value production \
  --Taints.0.Key dedicated --Taints.0.Value gpu --Taints.0.Effect NoSchedule
# expected: { "NodePoolId": "np-xxxxxxxx", "RequestId": "..." }
```

## TagSpecification 云标签

腾讯云标签（CAM/计费层），非 K8s Label。绑定资源类型 + 标签对。

| 字段 | 类型 | 说明 |
|------|------|------|
| ResourceType | string | 资源类型（如 `cluster`/`machine`） |
| Tags | array | `Tag` 数组（Key/Value） |

```bash
--Tags.0.ResourceType cluster --Tags.0.Tags.0.Key team --Tags.0.Tags.0.Value backend
```

## InstanceAdvancedSettings 节点高级设置 {#instanceadvancedsettings-节点高级设置}

出现在 `CreateCluster`/`AddExistedInstances`/`CreateClusterNodePool`/`CreateExternalNodePool`/`ScaleOutClusterMaster` 等 6+ Action，控制节点的运行时行为。

| 字段 | 类型 | 说明 |
|------|------|------|
| DesiredPodNumber | int | 单节点 Pod 上限（PodCIDR 自定义模式） |
| PreStartUserScript | string | 初始化前脚本（base64） |
| UserScript | string | 用户脚本（base64） |
| Taints | array | 节点污点（见 Taint） |
| Labels | array | 节点标签（见 Label） |
| Unschedulable | int | 是否不可调度（0/1） |
| DockerGraphPath | string | dockerd --graph 路径 |
| MountTarget | string | 数据盘挂载点 |
| GPUArgs | object | GPU 驱动参数（见下） |
| ExtraArgs | object | 自定义 K8s 组件参数 |

```bash
# 透传为 JSON (顶层 Action 用 --InstanceAdvancedSettings, 嵌套用 --X.InstanceAdvancedSettings)
--InstanceAdvancedSettings '{"Taints":[{"Key":"dedicated","Value":"gpu","Effect":"NoSchedule"}],"Unschedulable":0}'
```

### GPUArgs (嵌套)

GPU 驱动相关，含 `CUDA`/`CUDNN`/`CustomDriver`/`Driver`/`MIGEnable`。仅 GPU 机型需要。详见 [GPU 节点](../nodes/instance-ops.md#查询-gpu-驱动版本-2022-05-01)。

## EnhancedService 云服务增强

出现在 `AddExistedInstances`/`CreateECMInstances`/`ScaleOutClusterMaster`（透传 CVM），控制云安全/监控/自动化。

| 字段 | 类型 | 说明 |
|------|------|------|
| SecurityService | object | 云安全服务（默认开） |
| MonitorService | object | 云监控服务（默认开） |
| AutomationService | object | TAT 自动化助手 |

```bash
--EnhancedService '{"SecurityService":{},"MonitorService":{},"AutomationService":{}}'
```

> 传空对象 `{}` 表示开启默认配置；不传该字段也默认开启。

## LoginSettings 实例登录设置 {#loginsettings-实例登录设置}

| 字段 | 类型 | 说明 |
|------|------|------|
| Password | string | 登录密码 |
| KeyIds | array | 密钥 ID 列表；字段虽为数组，当前仅支持一个 KeyId；Windows 实例不支持密钥登录 |
| KeepImageLogin | string | 保持镜像原始登录设置；仅自定义镜像、共享镜像或外部导入镜像可设为 `true`，且与 Password/KeyIds 互斥 |

```bash
# 三种登录方式选一种；示例使用密钥登录
--LoginSettings '{"KeyIds":["skey-xxx"]}'
```

> `Password`、`KeyIds` 与 `KeepImageLogin` 是不同登录方式，不要在同一请求中混用。`KeyIds` 通过 `tccli cvm DescribeKeyPairs` 获取；虽然类型是数组，当前只支持传一个 KeyId，且 Windows 实例不支持密钥登录。`KeepImageLogin=true` 仅适用于自定义镜像、共享镜像或外部导入镜像。密码和私钥不得写入共享日志或版本库。

## 传参模式速查

| 场景 | 写法 |
|------|------|
| 单值标量 | `--ClusterId "<ID>"` |
| 对象数组（少元素） | `--cli-unfold-argument --Labels.0.Name env --Labels.0.Value prod` |
| 对象数组（多元素/复杂） | `--Native '{"SubnetIds":[...],"SystemDisk":{...}}'` 整段 JSON |
| 透传 CVM 参数 | 按目标 Action 的 `help --detail` 使用其实际字段名（常见为 `RunInstancePara`），传入 CVM `RunInstances` JSON 字符串 |

> 嵌套字段用点号展开 `--<Parent>.<Child>` 需配 `--cli-unfold-argument`；或用整段 JSON `--<Parent> '<JSON>'`。两者择一，不可混用。
