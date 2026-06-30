# 设置节点 Label（tccli）

> 对照官方：[设置节点 Label](https://cloud.tencent.com/document/product/457/32768) · page_id `32768` · tccli ≥3.1.0 · API 2018-05-25

## 概述

为节点设置 **Label**（标签），供 Kubernetes 调度器通过 `nodeSelector`、`nodeAffinity` 等机制将 Pod 调度到特定节点。Label 可在两个粒度设置：

| 粒度 | 操作方式 | 影响范围 | 依赖 kubectl 可达 |
|------|---------|---------|:--:|
| **单节点** | `kubectl label nodes` | 指定节点 | 是 |
| **节点池级别** | `ModifyClusterNodePool` → `Labels` | 池内全部节点（含新增节点） | 否（控制面） |

> **kubectl 可达性提示**：`kubectl label` 需 APIServer 端点可达。若公网端点被 CAM 策略拒绝，需通过内网端点（IOA/VPN）连接。节点池级别 Label 通过 tccli 控制面设置，不依赖 kubectl 可达性。

## 前置条件

- [环境准备](../../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 3.1.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId、secretKey、region 均已配置

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:DescribeClusters, tke:DescribeClusterNodePoolDetail
#    tke:ModifyClusterNodePool, tke:DescribeClusterInstances
# 验证：执行 DescribeClusters 确认权限
tccli tke DescribeClusters --region <Region>
# expected: exit 0，返回集群列表

# 4. 检查 kubectl（单节点操作需要，须 VPN/IOA）
kubectl version --client
# expected: Client Version v1.28.x+
```

```json
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "cls-example",
            "ClusterName": "example-cluster",
            "ClusterStatus": "Running",
            "ClusterVersion": "1.32.2"
        }
    ],
    "RequestId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
}
```

### 资源检查

```bash
# 5. 查询目标集群
tccli tke DescribeClusters --region <Region>
# expected: 至少 1 个 Running 集群，记录 ClusterId

# 6. 查询集群节点（确认有可标记的节点）
tccli tke DescribeClusterInstances --region <Region> \
    --ClusterId <ClusterId>
# expected: 至少 1 个节点，InstanceState 为 running
```

```json
{
    "TotalCount": 1,
    "InstanceSet": [
        {
            "InstanceId": "ins-example",
            "InstanceRole": "WORKER",
            "InstanceState": "running",
            "LanIP": "172.24.0.34",
            "FailedReason": ""
        }
    ],
    "RequestId": "b2c3d4e5-f6a7-8901-bcde-f12345678901"
}
```

> **节点名获取**：kubectl 使用 `LanIP` 值作为节点名（如 `172.24.0.34`）。kubectl 不可达时，可用 `tccli tke DescribeClusterInstances` 输出的 `$.InstanceSet[0].LanIP` 作为 `<NodeName>`。

```bash
# 7. 查询节点池（节点池级别 Label 需要）
tccli tke DescribeClusterNodePools --region <Region> \
    --ClusterId <ClusterId>
# expected: 返回节点池列表，记录 NodePoolId
```

```json
{
    "TotalCount": 1,
    "NodePoolSet": [
        {
            "NodePoolId": "np-example",
            "Name": "default-pool",
            "NodeCountSummary": {
                "AutoscalingAdded": {"Total": 1},
                "ManuallyAdded": {"Total": 0}
            }
        }
    ],
    "RequestId": "c3d4e5f6-a7b8-9012-cdef-23456789012"
}
```

### kubectl 连接准备（单节点操作需要）

```bash
# 获取 kubeconfig（公网端点，如可达）
tccli tke DescribeClusterKubeconfig --region <Region> \
    --ClusterId <ClusterId> \
    --IsExtranet true
# expected: exit 0，返回 base64 编码的 kubeconfig
```

```text
# 输出为 base64 编码的完整 kubeconfig YAML，解码后保存:
# mkdir -p ~/.kube && tccli tke DescribeClusterKubeconfig ... | jq -r '.Kubeconfig' | base64 -d > ~/.kube/config
```

```bash
# 将输出中的 kubeconfig 保存到 ~/.kube/config 或通过 KUBECONFIG 环境变量指向
```

> **kubectl 不可达说明**：若公网端点被 CAM 策略拒绝（如 `strategyId:240463971` 拦截 `tke:clusterExtranetEndpoint=true`），且内网端点无 IOA/VPN 可达，则跳过所有 kubectl 操作，使用节点池级别 `ModifyClusterNodePool` 控制面设置 Label。

### 版本与规格选择

| 粒度 | 推荐场景 | 限制 |
|------|---------|------|
| 单节点（kubectl） | 临时标记、调试、少量节点 | 需 APIServer 可达；可能被节点池控制器覆盖同名 Label |
| 节点池（tccli） | 批量标记、生产环境、GPU/SSD 节点池 | 修改可能触发池内节点滚动更新，请确认业务可接受 |

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 / kubectl | 幂等 |
|-----------|----------------------|:--:|
| 单节点 Label 编辑 | `kubectl label nodes <NodeName> <Key>=<Value>` | 否 |
| 节点池 Label 编辑 | `ModifyClusterNodePool` → `Labels` | 否 |
| 查看单节点 Label | `kubectl get node <NodeName> --show-labels` | 是 |
| 查看节点池 Label | `DescribeClusterNodePoolDetail` → `NodePool.Labels` | 是 |
| 删除单节点 Label | `kubectl label nodes <NodeName> <Key>-` | 否 |
| 覆盖已有 Label | `kubectl label nodes <NodeName> <Key>=<Value> --overwrite` | 否 |

## 操作步骤

### 步骤 1：单节点 Label 设置（kubectl）

#### 选择依据

- **操作方式**：`kubectl label nodes` 直接操作 etcd，立即生效，无需重建节点。
- **适用场景**：临时标记（如 `disktype=ssd`）、调试、少量节点。生产环境推荐节点池级别 Label。
- **一致性风险**：若节点池已配置同名 Label，节点池控制器可能在滚动更新时覆盖手动设置的值。如需与池不同的 Label，将节点移出节点池或使用节点池级 Label 统一管理。

#### 最小操作：添加单个 Label

```bash
kubectl label nodes <NodeName> disktype=ssd
# expected: exit 0, 输出 node/<NodeName> labeled
```

**预期输出**：

```text
node/172.24.0.34 labeled
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<NodeName>` | 节点名称（节点 LAN IP） | 集群内唯一 | `kubectl get nodes`；不可达时用 `tccli tke DescribeClusterInstances` 的 `$.InstanceSet[].LanIP` |
| `disktype` | Label Key | 前缀/名称 ≤ 63 字符，仅含字母、数字、-、_、. | 自定义 |
| `ssd` | Label Value | ≤ 63 字符，仅含字母、数字、-、_、. | 自定义 |

#### 增强操作：覆盖、批量、删除

覆盖已有 Label（不加 `--overwrite` 会报错）：

```bash
kubectl label nodes <NodeName> disktype=hdd --overwrite
# expected: exit 0, 输出 node/<NodeName> labeled
```

一次性添加多个 Label：

```bash
kubectl label nodes <NodeName> env=staging team=backend
# expected: exit 0, 输出 node/<NodeName> labeled
```

删除 Label（Key 后加 `-`）：

```bash
kubectl label nodes <NodeName> disktype-
# expected: exit 0, 输出 node/<NodeName> unlabeled
```

#### 查看单节点 Label

```bash
kubectl get node <NodeName> --show-labels
# expected: exit 0, LABELS 列含设置的 Key=Value
```

**预期输出**：

```text
NAME          STATUS   ROLES    AGE   VERSION   LABELS
172.24.0.34   Ready    <none>   2d    v1.32.2   beta.kubernetes.io/arch=amd64,disktype=ssd,env=staging,...
```

### 步骤 2：节点池 Label 设置（tccli）

#### 选择依据

- **操作方式**：`ModifyClusterNodePool` 修改 `Labels` 字段，对池内**全部节点**生效（含新增节点）。
- **适用场景**：GPU 节点池（`gpu=true`）、SSD 节点池（`disktype=ssd`）、环境隔离（`env=prod`）。
- **滚动更新影响**：修改节点池 Label 会触发池内节点滚动更新。设置 `IgnoreExistedNode: true` 可跳过存量节点，仅对新节点生效。
- **参数数量**：`ModifyClusterNodePool` 含 `ClusterId`、`NodePoolId`、`Labels` 等 ≥4 个参数，使用 `--cli-input-json file://`。

#### 关键字段说明

| 字段名 | 类型 | 必填 | 取值与约束 | 错误后果 |
|--------|------|:--:|---------|---------|
| `ClusterId` | string | 是 | `cls-xxxxxxxx` 格式，目标集群 ID | 若不存在 → `ResourceNotFound` |
| `NodePoolId` | string | 是 | `np-xxxxxxxx` 格式，目标节点池 ID | 若不存在 → `ResourceNotFound` |
| `Labels` | array | 否 | `[{"Name":"key","Value":"val"}]` 格式，每个元素含 `Name`/`Value`；Key ≤63 字符，仅含字母、数字、-、_、. | 格式错误 → `InvalidParameter` |
| `IgnoreExistedNode` | bool | 否 | `true`：仅对新节点生效；`false`（默认）：触发存量节点滚动更新 | 设为 `false` 可能触发 Pod 驱逐和节点重建 |

#### 最小操作：设置单个 Label

`modify-nodepool-labels-minimal.json`：

```json
{
  "ClusterId": "<ClusterId>",
  "NodePoolId": "<NodePoolId>",
  "Labels": [
    {"Name": "env", "Value": "staging"}
  ]
}
```

```bash
tccli tke ModifyClusterNodePool --region <Region> \
    --cli-input-json file://modify-nodepool-labels-minimal.json
# expected: exit 0, 返回 RequestId
```

**预期输出**：

```json
{
    "RequestId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
}
```

#### 增强配置：多 Label + 跳过存量节点

`modify-nodepool-labels-enhanced.json`：

```json
{
  "ClusterId": "<ClusterId>",
  "NodePoolId": "<NodePoolId>",
  "Labels": [
    {"Name": "env", "Value": "staging"},
    {"Name": "disktype", "Value": "ssd"},
    {"Name": "gpu", "Value": "true"}
  ],
  "IgnoreExistedNode": true
}
```

```bash
tccli tke ModifyClusterNodePool --region <Region> \
    --cli-input-json file://modify-nodepool-labels-enhanced.json
# expected: exit 0, 返回 RequestId
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<ClusterId>` | 集群 ID | 格式 `cls-xxxxxxxx` | `tccli tke DescribeClusters` |
| `<NodePoolId>` | 节点池 ID | 格式 `np-xxxxxxxx` | `tccli tke DescribeClusterNodePools` |
| `IgnoreExistedNode` | 是否跳过存量节点 | `true`：仅对新节点生效；`false`（默认）：滚动更新所有节点 | 生产环境推荐 `true` |

#### 查看节点池已有 Label

```bash
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId <ClusterId> \
    --NodePoolId <NodePoolId>
# expected: exit 0, NodePool.Labels 含设置的 Key/Value
```

**预期输出**（截取关键字段）：

```json
{
    "NodePool": {
        "NodePoolId": "np-example",
        "Name": "gpu-pool",
        "ClusterInstanceId": "cls-example",
        "LifeState": "normal",
        "Labels": [
            {"Name": "env", "Value": "staging"},
            {"Name": "disktype", "Value": "ssd"},
            {"Name": "gpu", "Value": "true"}
        ],
        "Taints": []
    },
    "RequestId": "b2c3d4e5-f6a7-8901-bcde-f12345678901"
}
```

### 步骤 3：新建节点池时预设 Label

在 `CreateClusterNodePool` 的 JSON 中直接设置 `Labels`，新节点池的节点初始化时即带该 Label。

#### 选择依据

- **时机**：新建节点池时预设 Label，避免创建后再修改触发滚动更新。
- **参数数量**：`CreateClusterNodePool` 含 ≥4 个参数，使用 `--cli-input-json file://`。

> **副作用声明**：`CreateClusterNodePool` 会创建 CVM 实例（按量付费）、挂载数据盘（如配置）、分配公网 IP（如启用）等级联资源。使用 GPU 实例类型（如 `GN7.5XLARGE80`）会产生显著计费成本。建议先用 `DesiredCapacity: 0` 零节点模式验证参数，确认后再扩容。测试完成后务必执行清理。

#### 关键字段说明

| 字段名 | 类型 | 必填 | 取值与约束 | 错误后果 |
|--------|------|:--:|---------|---------|
| `ClusterId` | string | 是 | `cls-xxxxxxxx` 格式，目标集群 ID | 若不存在 → `ResourceNotFound` |
| `Name` | string | 是 | 节点池名称，集群内唯一 | 重名 → `InvalidParameter` |
| `EnableAutoscale` | bool | 否 | 是否启用弹性伸缩，默认 `false` | 设为 `true` 但不配置 ASG 参数 → `MissingParameter` |
| `AutoScalingGroupPara` | string | 是 | JSON 字符串，含 `VpcId`/`SubnetIds`/`MaxSize`/`MinSize`/`DesiredCapacity` | 格式错误或缺失必填子字段 → `MissingParameter` 或 `InvalidParameter` |
| `LaunchConfigurePara` | string | 是 | JSON 字符串，含 `InstanceType`/`InstanceChargeType`/`SystemDisk`/`SecurityGroupIds` | `InstanceType` 停售 → `InstanceTypeNotSupported`；`InstanceChargeType` 非法 → `InvalidParameter` |
| `InstanceType` | string | 是 | 见下方 GPU 机型查询方式 | 使用已停售机型 → `InstanceTypeNotSupported` |
| `ContainerRuntime` | string | 否 | `containerd`（推荐）或 `docker` | 与 K8s 版本不兼容 → `FailedOperation` |
| `RuntimeVersion` | string | 否 | containerd 版本，如 `1.6.9`；需与 K8s 版本兼容 | 与 K8s 版本不兼容 → `PolicyServerRuntimeNotMatchK8sVersionError` |
| `Labels` | array | 否 | `[{"Name":"key","Value":"val"}]` 格式 | 格式错误 → `InvalidParameter` |
| `InstanceAdvancedSettings.Unschedulable` | int | 否 | `0`：可调度；`1`：不可调度 | — |

> **GPU 机型查询**：`GN10S` 系列已停售。执行以下命令查询当前可用 GPU 机型：
> ```bash
> tccli cvm DescribeInstanceTypeConfigs --region <Region> \
>     --Filters '[{"Name":"zone","Values":["<Zone>"]},{"Name":"instance-family","Values":["GN7","GN7vw"]}]'
> ```
> 选择含 `"GPU": 1` 且 Status 可用的最小规格（如 `GN7.5XLARGE80` 或 `GN7vw.2XLARGE32`），替换 JSON 中的 `<GPUInstanceType>`。

#### 前置资源查询：获取 VPC、子网、安全组

```bash
# 获取集群的 VPC、子网、安全组信息
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["<ClusterId>"]'
# 从输出中提取:
#   $.Clusters[0].ClusterNetworkSettings.VpcId        → <VpcId>
#   $.Clusters[0].ClusterNetworkSettings.Subnets[0]    → <SubnetId>
#   安全组通过下面的命令查询
```

```bash
# 查询安全组列表
tccli vpc DescribeSecurityGroups --region <Region> \
    --Filters '[{"Name":"group-name","Values":["default"]}]'
# 或使用 DescribeClusterInstances 获取节点已使用的安全组
```

`create-nodepool-labels.json`：

```json
{
  "ClusterId": "<ClusterId>",
  "Name": "gpu-pool",
  "EnableAutoscale": false,
  "AutoScalingGroupPara": "{\"MaxSize\":2,\"MinSize\":0,\"DesiredCapacity\":0,\"VpcId\":\"<VpcId>\",\"SubnetIds\":[\"<SubnetId>\"]}",
  "LaunchConfigurePara": "{\"InstanceType\":\"<GPUInstanceType>\",\"InstanceChargeType\":\"POSTPAID_BY_HOUR\",\"SystemDisk\":{\"DiskType\":\"CLOUD_SSD\",\"DiskSize\":100},\"SecurityGroupIds\":[\"<SecurityGroupId>\"]}",
  "InstanceAdvancedSettings": {"Unschedulable": 0},
  "ContainerRuntime": "containerd",
  "RuntimeVersion": "1.6.9",
  "Labels": [
    {"Name": "gpu", "Value": "true"},
    {"Name": "gpu-type", "Value": "v100"}
  ]
}
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<ClusterId>` | 集群 ID | 格式 `cls-xxxxxxxx` | `tccli tke DescribeClusters` → `$.Clusters[0].ClusterId` |
| `<VpcId>` | VPC ID | 格式 `vpc-xxxxxxxx` | `tccli tke DescribeClusters` → `$.Clusters[0].ClusterNetworkSettings.VpcId` |
| `<SubnetId>` | 子网 ID | 格式 `subnet-xxxxxxxx` | `tccli tke DescribeClusters` → `$.Clusters[0].ClusterNetworkSettings.Subnets[0]` |
| `<SecurityGroupId>` | 安全组 ID | 格式 `sg-xxxxxxxx` | `tccli vpc DescribeSecurityGroups`；或复用集群已有安全组 |
| `<GPUInstanceType>` | GPU 实例类型 | 如 `GN7.5XLARGE80` 或 `GN7vw.2XLARGE32` | `tccli cvm DescribeInstanceTypeConfigs --Filters '[{"Name":"instance-family","Values":["GN7","GN7vw"]}]'` |

```bash
tccli tke CreateClusterNodePool --region <Region> \
    --cli-input-json file://create-nodepool-labels.json
# expected: exit 0, 返回 NodePoolId
```

**预期输出**：

```json
{
    "NodePoolId": "np-example",
    "RequestId": "c3d4e5f6-a7b8-9012-cdef-23456789012"
}
```

### 步骤 4：使用 Label 做调度

配合 Pod `nodeSelector` 将工作负载调度到带特定 Label 的节点。

#### 选择依据

- **`nodeSelector` vs `nodeAffinity`**：`nodeSelector` 为硬性约束（Pod 只调度到匹配节点），适合必须满足的条件（如 GPU）。`nodeAffinity` 支持软性偏好（`preferredDuringScheduling`），适合"尽量但非必须"的场景。本教程使用 `nodeSelector` 作为简单入门。
- **GPU Pod 兼容性**：Pod 通过 `resources.limits.nvidia.com/gpu` 声明 GPU 资源，需集群已安装 NVIDIA device plugin（`nvidia-gpu` Addon）。
- **GPU 设备插件**：`nodeSelector: gpu=true` 需节点上有可用的 `nvidia.com/gpu` 资源。若无 GPU 节点，Pod 将一直处于 `Pending` 状态。

`gpu-pod.yaml`：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-workload
spec:
  containers:
  - name: cuda-app
    image: nvidia/cuda:12.0-base
    resources:
      limits:
        nvidia.com/gpu: 1
  nodeSelector:
    gpu: "true"
```

```bash
kubectl apply -f gpu-pod.yaml
# expected: pod/gpu-workload created
```

```bash
kubectl get pod gpu-workload -o wide
# expected: NODE 列为带 gpu=true Label 的节点
```

**预期输出**：

```text
NAME           READY   STATUS    RESTARTS   AGE   IP            NODE          NOMINATED NODE   READINESS GATES
gpu-workload   1/1     Running   0          30s   10.1.0.123   172.24.0.34   <none>           <none>
```

## 验证

### 控制面（tccli）

```bash
# 1. 确认节点池 Label 已设置
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId <ClusterId> \
    --NodePoolId <NodePoolId>
# expected: NodePool.Labels 含设置的 Key/Value
```

```json
{
    "NodePool": {
        "NodePoolId": "np-example",
        "Name": "kerwinwjyan-audit-np",
        "ClusterInstanceId": "cls-example",
        "LifeState": "normal",
        "Labels": [
            {"Name": "env", "Value": "staging"},
            {"Name": "team", "Value": "platform"}
        ],
        "Taints": []
    },
    "RequestId": "d4e5f6a7-b8c9-0123-defa-456789012345"
}
```

### 数据面（kubectl）

```bash
# 2. 确认单节点 Label
kubectl get node <NodeName> --show-labels
# expected: LABELS 列含设置的 Key=Value
```

**预期输出**：

```text
NAME          STATUS   ROLES    AGE   VERSION   LABELS
172.24.0.34   Ready    <none>   2d    v1.32.2   disktype=ssd,env=staging,...
```

```bash
# 3. 确认节点池内所有节点均有该 Label（按 Label 过滤）
kubectl get nodes -l env=staging
# expected: 返回池内所有带 env=staging Label 的节点

# 4. 确认 Pod 调度到正确节点（如已部署 nodeSelector Pod）
kubectl get pod -o wide
# expected: Pod 调度到带对应 Label 的节点
```

**预期输出（kubectl get nodes -l env=staging）**：

```text
NAME          STATUS   ROLES    AGE   VERSION
172.24.0.34   Ready    <none>   2d    v1.32.2
```

**预期输出（kubectl get pod -o wide）**：

```text
NAME           READY   STATUS    RESTARTS   AGE   IP           NODE          NOMINATED NODE   READINESS GATES
gpu-workload   1/1     Running   0          30s   10.1.0.123   172.24.0.34   <none>           <none>
```

## 清理

> **警告**：删除 Label 不会删除节点或 Pod，但可能导致依赖该 Label 调度的 Pod 被重新调度。生产环境操作前确认无 Pod 依赖待删除的 Label。

### 数据面清理（kubectl，单节点 Label）

```bash
# 1. 清理前状态检查
kubectl get node <NodeName> --show-labels
# 确认节点当前 Label
```

```text
NAME          STATUS   ROLES    AGE   VERSION   LABELS
172.24.0.34   Ready    <none>   2d    v1.32.2   disktype=ssd,env=staging,...
```

```bash
# 2. 删除 Label（Key 后加 -）
kubectl label nodes <NodeName> disktype-
# expected: exit 0, 输出 node/<NodeName> unlabeled

# 3. 验证已删除
kubectl get node <NodeName> --show-labels
# expected: LABELS 列不再含 disktype
```

```text
NAME          STATUS   ROLES    AGE   VERSION   LABELS
172.24.0.34   Ready    <none>   2d    v1.32.2   env=staging,...
```

### 控制面清理（tccli，节点池 Label）

```bash
# 1. 清理前状态检查
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId <ClusterId> \
    --NodePoolId <NodePoolId>
# 确认节点池当前 Label
```

```json
{
    "NodePool": {
        "NodePoolId": "np-example",
        "Name": "gpu-pool",
        "Labels": [
            {"Name": "env", "Value": "staging"},
            {"Name": "disktype", "Value": "ssd"}
        ],
        "Taints": []
    },
    "RequestId": "e5f6a7b8-c9d0-1234-efab-567890123456"
}
```

```bash
# 2. 清空 Label（传入空数组）
#    modify-nodepool-labels-clear.json 内容见下方 JSON 代码块
tccli tke ModifyClusterNodePool --region <Region> \
    --cli-input-json file://modify-nodepool-labels-clear.json
# expected: exit 0, 返回 RequestId

# 3. 验证已清空
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId <ClusterId> \
    --NodePoolId <NodePoolId>
# expected: NodePool.Labels 为空数组
```

```json
{
    "NodePool": {
        "NodePoolId": "np-example",
        "Name": "gpu-pool",
        "Labels": [],
        "Taints": []
    },
    "RequestId": "f6a7b8c9-d0e1-2345-fabc-678901234567"
}
```

`modify-nodepool-labels-clear.json`：

```json
{
  "ClusterId": "<ClusterId>",
  "NodePoolId": "<NodePoolId>",
  "Labels": []
}
```

### 节点池清理（tccli，Step 3 创建的 gpu-pool）

```bash
# 删除 Step 3 创建的 GPU 节点池
tccli tke DeleteClusterNodePool --region <Region> \
    --ClusterId <ClusterId> \
    --NodePoolIds '["<gpu-pool的NodePoolId>"]' \
    --KeepInstance false
# expected: exit 0, 返回 RequestId

# 验证已删除
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId <ClusterId> \
    --NodePoolId <gpu-pool的NodePoolId>
# expected: ResourceNotFound 或空响应
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl label` 报 `Forbidden` | `kubectl auth can-i patch nodes` | kubeconfig 对应账号无 node label 写权限 | 授予 RBAC `nodes/patch` 或 `nodes/update` 权限，或使用节点池级别 `ModifyClusterNodePool`（tccli 控制面） |
| `kubectl label` 报 `already has a value` | `kubectl get node <NodeName> --show-labels` 查看现有值 | Label 已存在，未加 `--overwrite` | 加 `--overwrite` 覆盖：`kubectl label nodes <NodeName> disktype=hdd --overwrite` |
| `ModifyClusterNodePool` 返回 `InvalidParameter` | 检查 JSON 中 `Labels` 格式 | `Labels` 不是 `{"Name":"key","Value":"value"}` 数组，或 Key/Value 含非法字符 | 确认 `Labels` 为数组，每个元素含 `Name` 和 `Value` 字段，Key/Value 仅含字母、数字、-、_、. |
| `ModifyClusterNodePool` 返回 `ResourceNotFound` | `tccli tke DescribeClusterNodePools --region <Region> --ClusterId <ClusterId>` 确认 NodePoolId | ClusterId 或 NodePoolId 不存在 | 核对 ClusterId 和 NodePoolId，确认集群和节点池在同一地域 |
| `CreateClusterNodePool` 返回 `InstanceTypeNotSupported` | `tccli cvm DescribeInstanceTypeConfigs --region <Region> --Filters '[{"Name":"instance-family","Values":["GN7","GN7vw","GN10Xp"]}]'` 查询可用 GPU 机型 | 文档使用的 GPU 机型（如 `GN10S.2XLARGE40`）已停售 | 替换为查询结果中 `"GPU": 1` 的最小规格（如 `GN7.5XLARGE80`、`GN7vw.2XLARGE32`） |
| kubectl 不可达 | `kubectl cluster-info` 报 `dial tcp: lookup ...: no such host`<br>`tccli tke CreateClusterEndpoint --IsExtranet true` 返回 `InvalidParameter.Param` | 公网端点被组织级 CAM 策略 **strategyId:240463971** 以条件 `tke:clusterExtranetEndpoint=true` 硬拒绝；集群域名（`cls-xxx.ccs.tencent-cloud.com`）本地 DNS 不可解析（NXDOMAIN）；内网端点（`172.x.x.x`）仅 VPC 内可达，本地无 IOA/VPN | ① 使用节点池级别 `ModifyClusterNodePool`（tccli 控制面）设置 Label，不依赖 kubectl 可达性；<br>② 通过 IOA/VPN/专线连接内网端点后执行 kubectl 操作；<br>③ 在同 VPC 的 CVM 上执行 kubectl 命令；<br>④ 使用 `tccli tke DescribeClusterInstances` 输出的 `$.InstanceSet[0].LanIP` 作为 `<NodeName>` |

### 操作成功但状态异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Pod `nodeSelector` 不生效，Pod 一直 Pending | `kubectl get nodes -l env=staging` 确认节点数 | 没有节点匹配 Label，或 Label 拼写不一致 | 确认节点 Label 与 Pod `nodeSelector` 的 Key/Value 完全一致（区分大小写） |
| 修改节点池 Label 后 Pod 被驱逐 | `kubectl get pods -o wide` 查看 Pod 状态 | 节点池滚动更新触发了节点重建（`IgnoreExistedNode: false`） | 这是预期行为。生产环境建议设 `IgnoreExistedNode: true` 避免影响存量节点 |
| 手动 `kubectl label` 被覆盖 | `kubectl get node <NodeName> --show-labels` 对比节点池 Label | 节点池控制器定期同步池级 Label | 如需与池不同的 Label，将节点移出节点池，或使用节点池级 Label 统一管理 |

## 下一步

- [创建节点池](../../普通节点/创建节点池/tccli%20操作.md) -- page_id `43735`（新建含 Label 的节点池）
- [调整节点池](../../普通节点/调整节点池/tccli%20操作.md)（修改节点池全局配置）
- [驱逐或封锁节点](../驱逐或封锁节点/tccli%20操作.md)（封锁节点维护）
- [连接集群](../../../集群管理/连接集群/tccli%20操作.md)（获取 kubeconfig，用于 kubectl 操作）
- [Kubernetes Labels 官方文档](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/)

## 控制台替代

[容器服务控制台 - 节点管理](https://console.cloud.tencent.com/tke2/cluster?rid=1)
