# 创建节点池（tccli）

> 对照官方：[创建节点池](https://cloud.tencent.com/document/product/457/43735) · page_id `43735`

## 概述

**节点池**（NodePool）是 TKE 中一组同质 CVM 节点的逻辑分组，底层绑定**弹性伸缩组**（ASG）。创建节点池需同时配置**启动配置**（机型、磁盘、安全组）和**伸缩组参数**（VPC、子网、容量范围），并使用 `--cli-input-json` 传入（参数量 >= 4）。

## 前置条件

- 集群状态为 `Running`，且集群内子网 IP 充足。
- [环境准备](../../../../环境准备.md)：`tccli` 已配置、地域与凭证已完成。
- 可选：[DescribeSupportedRuntime](#选择运行时) 查询集群兼容的容器运行时。

## 控制台与 CLI 参数映射

| 控制台 / 官方 | 含义 | tccli / JSON 入参 | 幂等 |
|---------------|------|-------------------|------|
| 节点池名称 | 池名称 | `Name`（字符串） | 是 |
| 机型 | CVM 机型 | `LaunchConfigurePara.InstanceType` | 否 |
| 系统盘 | 云硬盘类型与大小 | `LaunchConfigurePara.SystemDisk`（`DiskType` + `DiskSize`） | 否 |
| VPC/子网 | 节点所属网络 | `AutoScalingGroupPara.VpcId` / `SubnetIds`（数组） | 否 |
| 安全组 | 防火墙规则 | `LaunchConfigurePara.SecurityGroupIds`（**必填**） | 否 |
| 期望节点数 | ASG 初始容量 | `AutoScalingGroupPara.DesiredCapacity` | 否 |
| 最小/最大节点数 | 伸缩范围 | `AutoScalingGroupPara.MinSize` / `MaxSize` | 否 |
| 弹性伸缩开关 | 是否启用自动伸缩 | `EnableAutoscale`（Boolean） | 否 |
| 容器运行时 | containerd/docker | `ContainerRuntime` + `RuntimeVersion` | 否 |
| 节点 OS | 操作系统 | `NodePoolOs` | 否 |
| Label | 节点标签 | `Labels`（`Name`/`Value` 数组） | 否 |
| Taint | 节点污点 | `Taints`（`Key`/`Value`/`Effect` 数组） | 否 |
| 主机名模式 | 自动命名规则 | `InstanceAdvancedSettings` | 是 |
| 启动脚本 | 节点初始化脚本 | `InstanceAdvancedSettings.UserScript` | 否 |

> **API 入参真源**：`AutoScalingGroupPara` 和 `LaunchConfigurePara` 均为 **JSON 字符串**，非嵌套对象。`LaunchConfigurePara` 中**不传** `LaunchConfigurationName`、`ImageId`、`InstanceName`。

## 操作步骤

---

### 步骤 1：选择运行时

`CreateClusterNodePool` 需要 `ContainerRuntime` 和 `RuntimeVersion`。先查询集群支持的运行时版本：

```bash
tccli tke DescribeSupportedRuntime --ClusterId <ClusterId> --region ap-guangzhou --output json
```

```json
{
  "OptionalRuntimes": [
    { "RuntimeName": "containerd", "RuntimeVersion": "1.6.9" },
    { "RuntimeName": "containerd", "RuntimeVersion": "1.7.28" }
  ]
}
```

**选择依据**：
- **为什么选用 containerd 而不是 Docker**：Kubernetes 1.24+ 已移除 Dockershim，TKE 新增集群仅支持 containerd。
- **为什么 RuntimeVersion 选 1.6.9**：K8s 1.32.2 兼容 containerd 1.6.9 和 1.7.28，1.6.9 为更长验证的稳定版本；如需新版 containerd 特性可选 1.7.28。
- **为什么 `EnableAutoscale` 建议初始设为 `false`**：创建后先验证节点状态、网络连通性，确认无误后再通过 `ModifyClusterNodePool` 开启弹性伸缩。

---

### 步骤 2：准备 JSON 入参文件

**最小示例**（仅必填字段）：

```json
{
  "ClusterId": "<ClusterId>",
  "Name": "example-np",
  "EnableAutoscale": false,
  "AutoScalingGroupPara": "{\"MaxSize\":2,\"MinSize\":1,\"DesiredCapacity\":1,\"VpcId\":\"<VpcId>\",\"SubnetIds\":[\"<SubnetId>\"]}",
  "LaunchConfigurePara": "{\"InstanceType\":\"S5.MEDIUM2\",\"InstanceChargeType\":\"POSTPAID_BY_HOUR\",\"SystemDisk\":{\"DiskType\":\"CLOUD_PREMIUM\",\"DiskSize\":50},\"SecurityGroupIds\":[\"<SecurityGroupId>\"]}",
  "InstanceAdvancedSettings": { "Unschedulable": 0 },
  "ContainerRuntime": "containerd",
  "RuntimeVersion": "1.6.9",
  "NodePoolOs": "TKE_CUSTOM_OS",
  "Tags": [{ "Key": "env", "Value": "dev" }],
  "Labels": [{ "Name": "pool", "Value": "example" }]
}
```

**增强示例**（含数据盘、多子网、多安全组）：

```json
{
  "ClusterId": "<ClusterId>",
  "Name": "example-np-enhanced",
  "EnableAutoscale": false,
  "AutoScalingGroupPara": "{\"MaxSize\":5,\"MinSize\":0,\"DesiredCapacity\":2,\"VpcId\":\"<VpcId>\",\"SubnetIds\":[\"<SubnetId1>\",\"<SubnetId2>\"],\"RetryPolicy\":\"IMMEDIATE_RETRY\"}",
  "LaunchConfigurePara": "{\"InstanceType\":\"S5.MEDIUM4\",\"InstanceChargeType\":\"POSTPAID_BY_HOUR\",\"SystemDisk\":{\"DiskType\":\"CLOUD_SSD\",\"DiskSize\":100},\"DataDisks\":[{\"DiskType\":\"CLOUD_PREMIUM\",\"DiskSize\":200}],\"SecurityGroupIds\":[\"<SecurityGroupId1>\",\"<SecurityGroupId2>\"],\"InternetAccessible\":{\"InternetMaxBandwidthOut\":1,\"PublicIpAssigned\":false}}",
  "InstanceAdvancedSettings": {
    "Unschedulable": 0,
    "MountTarget": "/var/lib/containerd",
    "UserScript": "IyEvYmluL2Jhc2gKZWNobyAibm9kZSBpbml0aWFsaXplZCI="
  },
  "Tags": [
    { "Key": "env", "Value": "prod" },
    { "Key": "team", "Value": "platform" }
  ],
  "Labels": [
    { "Name": "pool", "Value": "enhanced" },
    { "Name": "disktype", "Value": "ssd" }
  ],
  "Taints": [
    { "Key": "dedicated", "Value": "enhanced-pool", "Effect": "NoSchedule" }
  ]
}
```

> `UserScript` 值为 Base64 编码的 shell 脚本内容。上例解码后为 `#!/bin/bash\necho "node initialized"`。脚本路径：`/usr/local/qcloud/tke/userscript`。

---

### 步骤 3：调用 API 创建节点池

```bash
tccli tke CreateClusterNodePool --cli-input-json file://CreateClusterNodePool.json --region ap-guangzhou --output json
```

```json
{ "NodePoolId": "np-example", "RequestId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890" }
```

---

### 步骤 4：验证节点池已创建

```bash
tccli tke DescribeClusterNodePools --ClusterId <ClusterId> --region ap-guangzhou --output json
```

```json
{
  "NodePoolSet": [
    {
      "NodePoolId": "np-example",
      "Name": "example-np",
      "ClusterId": "cls-example",
      "LifeState": "normal",
      "NodeCountSummary": {
        "ManuallyAdded": { "Joined": 0 },
        "AutoscalingAdded": { "Joining": 1, "Total": 1 }
      }
    }
  ],
  "RequestId": "b2c3d4e5-f6a7-8901-bcde-f12345678901"
}
```

# expected: exit 0, contains "np-example"

---

### 包年包月节点池

官方建议：如需包年包月节点，宜**新建独立的包月节点池**，勿将按量节点池中的实例转为包月（会导致计费混叠）。计费模式在 `LaunchConfigurePara` 中通过 `InstanceChargeType` 控制（如 `PREPAID`，需附带 `InstanceChargePrepaid` 参数）。

---

### 创建时注意事项

- `AutoScalingGroupPara` 必须包含：`VpcId`、`SubnetIds`、`MaxSize`、`MinSize`、`DesiredCapacity`。
- `LaunchConfigurePara` 必须包含：`InstanceChargeType`（`POSTPAID_BY_HOUR` 或 `PREPAID`）、`InstanceType`、`SystemDisk`、`SecurityGroupIds`。
- `LaunchConfigurePara` 中**禁止**传入：`LaunchConfigurationName`、`ImageId`、`InstanceName`（这些字段会导致 `PARAM_ERROR(bad launch configuration para)`）。
- `SecurityGroupIds` 为数组，**必须填**，否则触发 `PARAM_ERROR(security group ids is not set)`。
- `MinSize` 和 `MaxSize` 需满足 `0 <= MinSize <= DesiredCapacity <= MaxSize`，违反时返回 `PARAM_ERROR(MinSize and MaxSize)`。
- 对于 GlobalRouter 模式集群，**禁止**在 `InstanceAdvancedSettings` 中设置 `DesiredPodNumber`（会触发 "customized Pod CIDR" 错误）。
- `Unschedulable` 字段类型为 `int64`（`0` / `1`），不是 boolean。

## 验证

### Control plane (tccli)

```bash
tccli tke DescribeClusterNodePoolDetail --ClusterId <ClusterId> --NodePoolId <NodePoolId> --region ap-guangzhou --output json
```

```json
{
  "NodePool": "<NodePool>",
  "NodePoolId": "<NodePoolId>",
  "Name": "<Name>",
  "ClusterInstanceId": "<ClusterInstanceId>",
  "LifeState": "<LifeState>",
  "LaunchConfigurationId": "<LaunchConfigurationId>",
  "AutoscalingGroupId": "<AutoscalingGroupId>",
  "Labels": []
}
```

- `LifeState` 为 `normal`。
- `NodeCountSummary.AutoscalingAdded` 节点数达到 `DesiredCapacity`。

### Data plane (kubectl)

```bash
kubectl get nodes -l pool=example
# expected: exit 0, contains "Ready"
```

```text
NAME  STATUS  AGE
...
```

> **可达性要求**：kubectl 需集群 APIServer 端点可达。公网端点受 CAM 策略限制（如 strategyId:240463971 拒绝外网访问），内网端点需通过 IOA/VPN 连通 VPC。若 kubectl 不可达，控制面的 `DescribeClusterInstances` 可替代验证节点状态。

## 清理

```bash
tccli tke DeleteClusterNodePool --cli-input-json '{"ClusterId":"<ClusterId>","NodePoolIds":["<NodePoolId>"],"KeepInstance":false}' --region ap-guangzhou --output json
```

⚠️ **级联清理警告**：删除节点池会级联删除池内所有节点及其关联云硬盘（`KeepInstance: false`）。若仅需移出节点保留 CVM，设 `KeepInstance: true`，之后在 CVM 控制台处理。

## 排障

| 错误码 / 现象 | 原因 | 处理 |
|---------------|------|------|
| `PolicyServerRuntimeNotMatchK8sVersionError` | `ContainerRuntime` / `RuntimeVersion` 与 K8s 版本不兼容 | 先 `DescribeSupportedRuntime` 获取集群兼容列表 |
| `AS_COMMON_ERROR` | 伸缩组参数有误或账号下 AS 资源受限 | 检查 `AutoScalingGroupPara` VpcId/SubnetIds 有效性，确认账号 AS 配额 |
| `PARAM_ERROR(MinSize and MaxSize)` | `MinSize` 和 `MaxSize` 大小关系不合法 | 确保 `0 <= MinSize <= DesiredCapacity <= MaxSize` |
| `PARAM_ERROR(image id can not be set)` | `LaunchConfigurePara` 中包含了 `ImageId` 字段 | 删除该字段，镜像由 TKE 托管指定 |
| `PARAM_ERROR(security group ids is not set)` | `LaunchConfigurePara` 中缺少 `SecurityGroupIds` | 添加安全组 ID 数组 |
| `PARAM_ERROR(bad launch configuration para)` | `LaunchConfigurePara` JSON 格式错误或含多余字段 | 检查 JSON 合法性，删除 `LaunchConfigurationName`/`InstanceName`/`ImageId` |
| "customized Pod CIDR" 错误 | GlobalRouter 集群传入了 `DesiredPodNumber` | 删除 `InstanceAdvancedSettings.DesiredPodNumber` 字段 |
| `InstanceChargeType` 不合法 | `LaunchConfigurePara` 缺少或传入了无效的计费类型 | 设为 `POSTPAID_BY_HOUR`（按量）或 `PREPAID`（包年包月） |
| 创建后节点一直 `Joining` | 安全组规则不完整或子网 IP 耗尽 | 检查安全组是否放通 10250 端口（kubelet），确认子网可用 IP 数量 |

## 下一步

- [查看节点池](../查看节点池/tccli%20操作.md)：确认池配置和节点状态
- [调整节点池](../调整节点池/tccli%20操作.md)：修改期望节点数、Labels 等
- [查看节点池伸缩记录](../查看节点池伸缩记录/tccli%20操作.md)：查看弹性伸缩活动

## 控制台替代

控制台 → **集群** → 目标集群 → **节点管理** → **新建节点池**，按向导填写机型、网络、伸缩组参数后确认。控制台等价于 `CreateClusterNodePool` 的完整 JSON 入参。
