# GPU 故障检测与自愈

> 对照官方：[GPU 故障检测与自愈](https://cloud.tencent.com/document/product/457/127503) · page_id `127503`

## 概述

TKE 提供完整的 GPU 故障检测与自愈方案：通过 **Node Problem Detector Plus（NPDPlus，组件名 `npdplus`）** 实时检测 12 种 GPU 故障并上报为 Node Condition，结合 **Node-Healing（组件名 `node-healing`）** 按 `MachineHealingTemplate` 中定义的步骤（`CordonNode` → `DrainNode` → `RebootNode` → `UnCordonNode`）自动执行隔离、排空、重启等自愈操作。NPDPlus 的 GPU 故障检测依赖 **nvidia-gpu 组件** 提供 GPU 驱动和设备插件。

整体流程：安装 nvidia-gpu（GPU 驱动）→ 安装 NPDPlus（故障检测）→ 安装 Node-Healing（自愈执行）→ 创建 MachineHealingTemplate（定义自愈步骤）→ 创建 NodeHealer（绑定节点和故障类型）→ 为 GPU 节点打标签。

> **注意**：控制台显示名为 **NodeProblemDetectorPlus** 和 **Node-Healing**，但 API AddonName 必须使用 `npdplus` 和 `node-healing`。使用控制台显示名作为 AddonName 将返回 `UnknownParameter: addon version not found`。正确的 AddonName 和版本通过 `tccli tke GetTkeAppChartList` 查询。

| 组件 | AddonName（API 名） | 控制台显示名 | 角色 | 控制面/数据面 |
|------|---------------------|-------------|------|:--:|
| nvidia-gpu | `nvidia-gpu` | nvidia-gpu | GPU 驱动 + 设备插件，注册 `nvidia.com/gpu` 资源 | 控制面 |
| NodeProblemDetectorPlus | `npdplus` | NodeProblemDetectorPlus | 检测 GPU 故障，上报 Node Condition | 控制面 |
| Node-Healing | `node-healing` | Node-Healing | 监听 Condition，执行自愈步骤 | 控制面 |
| MachineHealingTemplate | CRD | — | 定义故障类型对应的自愈步骤序列 | 数据面 |
| NodeHealer | CRD | — | 选择目标节点并应用模板 | 数据面 |

## 前置条件

- [环境准备](../../../环境准备.md)
- 已 [连接集群](../../集群管理/连接集群/tccli%20操作.md)，kubectl 可正常访问 APIServer

> **kubectl 连接说明**：本页面的 CRD 创建（`kubectl apply`）、节点标签（`kubectl label`）和验证（`kubectl get`）命令需要 kubectl 可访问集群 APIServer。公网端点可能被组织级 CAM 策略拒绝（如 `strategyId:240463971` 拒绝 `tke:clusterExtranetEndpoint=true`），此时需使用内网端点。内网端点仅 VPC 内可达——需通过 IOA/VPN/专线，或在同 VPC 的 CVM 上执行。检查端点状态：
> ```bash
> tccli tke DescribeClusterEndpoints --region <Region> --ClusterId CLUSTER_ID
> # expected: 至少有一个端点状态为 Created
> ```
> 如内网端点不可达，需在同 VPC CVM 上安装 kubectl 并配置 kubeconfig：`tccli tke DescribeClusterKubeconfig --region <Region> --ClusterId CLUSTER_ID --IsExtranet false`

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置，region 为目标地域

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:DescribeClusters, tke:InstallAddon, tke:DescribeAddon, tke:DeleteAddon
#    tke:DescribeClusterInstances, tke:DescribeClusterNodePoolDetail,
#    tke:DescribeClusterEndpoints, tke:DescribeClusterKubeconfig
# 验证：执行 DescribeClusters 确认权限
tccli tke DescribeClusters --region <Region>
# expected: exit 0，返回集群列表（可为空）

# 4. 检查 Addon 版本可用性 — 查询可用的 Addon 名称和版本
tccli tke GetTkeAppChartList --region <Region>
# expected: 返回 AppCharts 列表，含 npdplus 和 node-healing 的 Name 和 LatestVersion

# 5. 检查集群端点可达性
tccli tke DescribeClusterEndpoints --region <Region> --ClusterId CLUSTER_ID
# expected: 至少一个端点 Status 为 Created

# 6. 验证 kubectl 连通性（内网端点需在同 VPC CVM 或通过 VPN/专线执行）
kubectl cluster-info
# expected: Kubernetes control plane is running at https://ENDPOINT
```

### 资源检查

```bash
# 5. 确认集群状态为 Running 且版本 >= 1.18
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterStatus 为 Running，ClusterVersion >= 1.18

# 6. 确认集群中存在 GPU 原生节点（NPDPlus + Node-Healing 仅支持原生节点）
tccli tke DescribeClusterInstances --region <Region> \
    --ClusterId CLUSTER_ID \
    --Limit 50
# expected: InstanceSet 中包含原生节点（InstanceId 以 ins- 开头）

# 7. 确认节点已注册 nvidia.com/gpu 资源
kubectl describe node NODE_NAME | grep nvidia.com/gpu
# expected: nvidia.com/gpu >= 1

# 8. 确认 nvidia-gpu 组件尚未安装或已安装
tccli tke DescribeAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName nvidia-gpu
# expected: Phase 为 Succeeded（已安装）或空列表（未安装）
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

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `CLUSTER_ID` | 集群 ID | 格式 `cls-xxxxxxxx` | `tccli tke DescribeClusters --region <Region>` |
| `REGION` | 地域 | 如 `ap-guangzhou` | `tccli configure list` |
| `NODE_NAME` | GPU 原生节点名 | 节点须含 `cloud.tencent.com/provider=tencentcloud` 标签 | `kubectl get nodes` |

## 关键字段说明

### InstallAddon 参数

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `ClusterId` | String | 是 | 集群 ID，格式 `cls-xxxxxxxx`。仅支持标准 TKE 集群 | 集群不存在 → `ResourceNotFound.ClusterNotFound` |
| `AddonName` | String | 是 | 组件 API 名（非控制台显示名）：`nvidia-gpu`、`npdplus`、`node-healing`。**必须通过 `GetTkeAppChartList` 查询准确的 AddonName**，控制台显示名 `NodeProblemDetectorPlus` 不是有效 API 名 | 使用控制台显示名 → `UnknownParameter: addon version not found` |
| `AddonVersion` | String | 是 | 组件版本，从 `GetTkeAppChartList` 的 `LatestVersion` 字段获取。如 npdplus `1.1.4`、node-healing `1.0.0`。不传时可能安装非最新版 | 版本不存在 → `UnknownParameter: addon version not found` |
| `RawValues` | String | 否 | 组件自定义参数，JSON 字符串。GPU 故障检测场景传 `'{}'` 使用默认配置即可 | 格式错误 → `InvalidParameter` |

### NodeHealer CRD — unhealthyConditions 字段

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `type` | String | 是 | GPU 故障类型枚举（见下表） | 类型不存在 → NPDPlus 不上报该 Condition |
| `enableCheck` | Bool | 是 | 是否启用检测 | 设为 `false` 则不检测 |
| `enableHealing` | Bool | 是 | 是否启用自愈 | 设为 `false` 则仅检测不修复 |
| `durationTime` | String | 是 | 故障持续多久后触发自愈，如 `"1m"`、`"5m"` | 过短易误触发，过长延迟修复 |
| `priority` | Int | 是 | 优先级 0-100，值越大越优先处理 | 多故障同时发生时按优先级排序 |

### GPU 故障检测类型（12 种）

| 故障类型 | 说明 | 推荐自愈步骤 |
|---------|------|-------------|
| `GPUDeviceLost` | GPU 设备丢失，系统无法识别 | CordonNode → DrainNode → RebootNode → UnCordonNode |
| `GPUDoubleBitEccError` | GPU 双比特 ECC 错误（不可纠正） | CordonNode → DrainNode |
| `GPUHighSingleBitEccError` | GPU 单比特 ECC 错误超阈值 | CordonNode → DrainNode |
| `GPUInfoROMCorrupted` | GPU InfoROM 损坏 | CordonNode → DrainNode |
| `GPUNvlinkError` | NVLink 通信异常 | CordonNode → DrainNode → RebootNode → UnCordonNode |
| `GPUPCIeError` | PCIe 接口异常 | CordonNode → DrainNode → RebootNode → UnCordonNode |
| `GPUPendingRetiredPages` | GPU 待退役内存页面 | CordonNode → DrainNode → RebootNode → UnCordonNode |
| `GPUPowerError` | GPU 电源故障 | CordonNode → DrainNode |
| `GPURemappingRowsError` | GPU 内存行重映射错误 | CordonNode → DrainNode |
| `GPUTempError` | GPU 温度过高 | CordonNode → DrainNode |
| `GPUXIDFatalError` | GPU 致命 XID 错误 | CordonNode → DrainNode |
| `GPUXIDWarningError` | GPU 警告 XID 错误 | CordonNode → DrainNode → RebootNode → UnCordonNode |

### 自愈步骤

| 步骤 | 说明 |
|------|------|
| `CordonNode` | 隔离节点，标记为不可调度，不再分配新 Pod |
| `DrainNode` | 排空节点上的 Pod（安全驱逐） |
| `RebootNode` | 重启节点 |
| `UnCordonNode` | 恢复节点为可调度状态 |

### durationTime 建议值

| 故障类别 | 建议值 | 原因 |
|---------|--------|------|
| 严重硬件（ECC 错误、致命 XID、设备丢失） | `1m` | 快速响应，避免数据损毁或业务影响 |
| 一般硬件（NVLink、PCIe） | `5m` | 短暂观察，避免瞬时抖动误触发 |
| 可恢复（警告级错误） | `10m` | 延长观察，可能自行恢复 |

### nodeHealingPolicy 限流参数

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `maxHealNodesPerDay` | Int | 10 | 每天最大自愈节点数 |
| `maxHealNodesPerHour` | Int | 3 | 每小时最大自愈节点数 |
| `maxHealConcurrency` | Int | 2 | 最大并发自愈数 |
| `minNodeCoolingMinutes` | Int | 60 | 同一节点两次自愈最小冷却时间（分钟） |


## 控制台与 CLI 参数映射

| 控制台操作 | tccli / kubectl 命令 | 幂等 |
|-----------|---------------------|:--:|
| 查看已安装组件 | `tccli tke DescribeAddon` | 是 |
| 查询可用 Addon 及版本 | `tccli tke GetTkeAppChartList` | 是 |
| 安装 GPU 驱动组件 | `tccli tke InstallAddon --AddonName nvidia-gpu` | 是 |
| 安装故障检测组件 | `tccli tke InstallAddon --AddonName npdplus` | 是 |
| 安装自愈执行组件 | `tccli tke InstallAddon --AddonName node-healing` | 是 |
| 创建自愈模板 | `kubectl apply -f ... MachineHealingTemplate` | 是 |
| 创建节点自愈器 | `kubectl apply -f ... NodeHealer` | 是 |
| 卸载组件 | `tccli tke DeleteAddon` | 是 |

## 操作步骤

### 步骤 1：安装 nvidia-gpu 组件（GPU 驱动 + 设备插件）

#### 选择依据

- **nvidia-gpu 是 GPU 故障检测的前置依赖**：NPDPlus 通过 nvidia-gpu 部署的 device-plugin 和 exporter 获取 GPU 状态信息。未安装 nvidia-gpu 时，GPU 故障 Condition 不会上报。
- **存量集群检查**：若集群已存在 GPU 节点，首次安装前需检查是否已有旧版 `nvidia-device-plugin-daemonset`，如有需先删除。删除该 DaemonSet 不会影响已运行的 GPU 业务 Pod。

```bash
# 检查是否已存在旧版 device-plugin
kubectl get daemonset nvidia-device-plugin-daemonset -n kube-system 2>/dev/null
# expected: 无输出（不存在，可跳过删除）或返回该 DaemonSet 信息（需先删除）

# 安装 nvidia-gpu 组件
tccli tke InstallAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName nvidia-gpu
# expected: exit 0，返回 RequestId
```

**预期输出**：

```json
{
    "RequestId": "..."
}
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `CLUSTER_ID` | 集群 ID | 格式 `cls-xxxxxxxx` | `tccli tke DescribeClusters --region <Region>` |
| `REGION` | 地域 | 如 `ap-guangzhou` | `tccli configure list` |

确认安装状态（异步操作，轮询至 `Phase` 为 `Succeeded`）：

```bash
tccli tke DescribeAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName nvidia-gpu
# expected: Phase 为 Succeeded
```

**预期输出**：

```json
{
    "Addons": [
        {
            "AddonName": "nvidia-gpu",
            "AddonVersion": "1.60.0",
            "Phase": "Succeeded",
            "Reason": ""
        }
    ]
}
```

### 步骤 2：安装 NPDPlus 组件（GPU 故障检测）

#### 选择依据

- **NPDPlus 的 API AddonName 是 `npdplus`**（非控制台显示名 `NodeProblemDetectorPlus`）。通过 `tccli tke GetTkeAppChartList` 查询到的真实图表名是 `npdplus`。使用控制台显示名将返回 `UnknownParameter: addon version not found`。
- **版本选择**：从 `GetTkeAppChartList` 获取 `LatestVersion`。经真机验证，当前最新版本为 `1.1.4`。
- **NPDPlus 作为 DaemonSet 运行在每个节点**，实时检测 GPU 故障并更新 Node Condition。
- **必须先安装 nvidia-gpu**：NPDPlus 的 GPU 故障检测依赖 nvidia-gpu 提供的 GPU 状态数据。

#### 发现 Addon 名称和版本

```bash
# 查询可用的 Addon 列表，获取正确的 AddonName 和 LatestVersion
tccli tke GetTkeAppChartList --region <Region>
# expected: AppCharts 中包含 npdplus (LatestVersion: 1.1.4) 和 node-healing (LatestVersion: 1.0.0)
```

**预期输出**：

```json
{
    "AppCharts": [
        {
            "Name": "npdplus",
            "LatestVersion": "1.1.4"
        },
        {
            "Name": "node-healing",
            "LatestVersion": "1.0.0"
        }
    ],
    "RequestId": "..."
}
```

#### 最小安装（使用默认配置）

```bash
# 安装 NPDPlus 组件
tccli tke InstallAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName npdplus \
    --AddonVersion 1.1.4 \
    --RawValues '{}'
# expected: exit 0，返回 RequestId
```

**预期输出**：

```json
{
    "RequestId": "..."
}
```

确认安装状态：

```bash
tccli tke DescribeAddon --region <Region> \
    --ClusterId CLUSTER_ID
# expected: Addons 中包含 npdplus，Phase 为 Succeeded
```

**预期输出**：

```json
{
    "Addons": [
        {
            "AddonName": "npdplus",
            "AddonVersion": "1.1.4",
            "Phase": "Succeeded",
            "Reason": ""
        }
    ],
    "RequestId": "..."
}
```

### 步骤 3：安装 Node-Healing 组件（自愈执行器）

#### 选择依据

- **Node-Healing 的 API AddonName 是 `node-healing`**（非控制台显示名 `Node-Healing`，注意大小写和连字符）。通过 `GetTkeAppChartList` 可确认图表名。使用控制台显示名 `NodeHealing` 将返回 `UnknownParameter: addon version not found`。
- **版本选择**：从 `GetTkeAppChartList` 获取 `LatestVersion`。经真机验证，当前最新版本为 `1.0.0`。
- **Node-Healing 依赖 NPDPlus**：Node-Healing 监听 NPDPlus 上报的 Node Condition，按 `MachineHealingTemplate` 定义的步骤执行自愈。必须先完成步骤 2 安装 NPDPlus。
- **仅支持原生节点**：Node-Healing 通过云 API 重启节点，不支持超级节点、第三方节点（注册节点）、普通节点。

```bash
# 安装 Node-Healing 组件
tccli tke InstallAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName node-healing \
    --AddonVersion 1.0.0 \
    --RawValues '{}'
# expected: exit 0，返回 RequestId
```

**预期输出**：

```json
{
    "RequestId": "..."
}
```

确认安装状态：

```bash
tccli tke DescribeAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName node-healing
# expected: Phase 为 Succeeded
```

**预期输出**：

```json
{
    "Addons": [
        {
            "AddonName": "node-healing",
            "AddonVersion": "1.0.0",
            "Phase": "Succeeded",
            "Reason": ""
        }
    ]
}
```

### 步骤 4：创建 MachineHealingTemplate（定义 GPU 故障自愈步骤）

#### 选择依据

- **MachineHealingTemplate 定义每种故障的自愈步骤序列**。GPU 故障分两类策略：
  - **完整自愈流程**（`CordonNode → DrainNode → RebootNode → UnCordonNode`）：适用于可通过重启恢复的软件级 GPU 故障（如 NVLink、PCIe 瞬时错误、设备丢失）。
  - **仅隔离排空**（`CordonNode → DrainNode`）：适用于严重硬件故障（如 ECC 错误、电源故障、温度过高、致命 XID），需人工介入更换硬件，不自动重启。
- **`durationTime` 阈值**：严重故障（ECC、Xid）建议 `1m`，一般故障建议 `5m`，避免瞬时抖动误触发。

#### 最小创建（仅 GPUDeviceLost + GPUXIDFatalError 两种典型故障）

文件 `gpu-healing-template-minimal.yaml`：

```yaml
apiVersion: node.healing.tke.cloud.tencent.com/v1
kind: MachineHealingTemplate
metadata:
  name: gpu-healing-template
  namespace: kube-system
spec:
  nodeConditions:
    - type: GPUDeviceLost
      status: "True"
      enableHealing: true
      priority: 100
      durationTime: "1m"
      steps:
        - CordonNode
        - DrainNode
        - RebootNode
        - UnCordonNode
    - type: GPUXIDFatalError
      status: "True"
      enableHealing: true
      priority: 90
      durationTime: "1m"
      steps:
        - CordonNode
        - DrainNode
```

```bash
kubectl apply -f gpu-healing-template-minimal.yaml
# expected: machinehealingtemplate.node.healing.tke.cloud.tencent.com/gpu-healing-template created
```

#### 增强配置（覆盖 6 种常见 GPU 故障）

文件 `gpu-healing-template-enhanced.yaml`：

```yaml
apiVersion: node.healing.tke.cloud.tencent.com/v1
kind: MachineHealingTemplate
metadata:
  name: gpu-healing-template
  namespace: kube-system
spec:
  nodeConditions:
    - type: GPUDeviceLost
      status: "True"
      enableHealing: true
      priority: 100
      durationTime: "1m"
      steps:
        - CordonNode
        - DrainNode
        - RebootNode
        - UnCordonNode
    - type: GPUPowerError
      status: "True"
      enableHealing: true
      priority: 95
      durationTime: "1m"
      steps:
        - CordonNode
        - DrainNode
    - type: GPUTempError
      status: "True"
      enableHealing: true
      priority: 95
      durationTime: "1m"
      steps:
        - CordonNode
        - DrainNode
    - type: GPUPCIeError
      status: "True"
      enableHealing: true
      priority: 85
      durationTime: "5m"
      steps:
        - CordonNode
        - DrainNode
        - RebootNode
        - UnCordonNode
    - type: GPUNvlinkError
      status: "True"
      enableHealing: true
      priority: 85
      durationTime: "5m"
      steps:
        - CordonNode
        - DrainNode
        - RebootNode
        - UnCordonNode
    - type: GPUXIDFatalError
      status: "True"
      enableHealing: true
      priority: 90
      durationTime: "1m"
      steps:
        - CordonNode
        - DrainNode
```

```bash
kubectl apply -f gpu-healing-template-enhanced.yaml
# expected: machinehealingtemplate.node.healing.tke.cloud.tencent.com/gpu-healing-template configured
```

### 步骤 5：创建 NodeHealer（绑定 GPU 节点和故障类型）

#### 选择依据

- **NodeHealer 通过 `nodeSelector.matchLabels` 选择目标节点**：必须匹配 `cloud.tencent.com/provider=tencentcloud`（原生节点标识）和 `node.tke.cloud.tencent.com/accelerator-type=gpu`（GPU 节点标识），再叠加自定义的 `node-healing.tke.cloud.tencent.com/belong-to` 标签用于精细控制。
- **`machineHealingTemplate` 引用步骤 4 创建的模板**：NodeHealer 将模板中的故障检测规则应用到匹配的节点。
- **限流参数防止雪崩**：`maxHealConcurrency` 限制并发自愈数，`minNodeCoolingMinutes` 防止同一节点频繁重启。

文件 `gpu-node-healer.yaml`：

```yaml
apiVersion: node.healing.tke.cloud.tencent.com/v1
kind: NodeHealer
metadata:
  name: gpu-cluster-healer
  namespace: kube-system
spec:
  nodeSelector:
    matchLabels:
      node-healing.tke.cloud.tencent.com/belong-to: gpu-cluster-healer
      cloud.tencent.com/provider: tencentcloud
      node.tke.cloud.tencent.com/accelerator-type: gpu
  machineHealingTemplate:
    name: gpu-healing-template
    kind: MachineHealingTemplate
    apiVersion: node.healing.tke.cloud.tencent.com/v1
  unhealthyConditions:
    - type: GPUDeviceLost
      enableCheck: true
      enableHealing: true
      durationTime: "1m"
      priority: 100
    - type: GPUPowerError
      enableCheck: true
      enableHealing: true
      durationTime: "1m"
      priority: 95
    - type: GPUTempError
      enableCheck: true
      enableHealing: true
      durationTime: "1m"
      priority: 95
    - type: GPUPCIeError
      enableCheck: true
      enableHealing: true
      durationTime: "5m"
      priority: 85
    - type: GPUNvlinkError
      enableCheck: true
      enableHealing: true
      durationTime: "5m"
      priority: 85
    - type: GPUXIDFatalError
      enableCheck: true
      enableHealing: true
      durationTime: "1m"
      priority: 90
  nodeHealingPolicy:
    maxHealNodesPerDay: 10
    maxHealNodesPerHour: 3
    maxHealConcurrency: 2
    minNodeCoolingMinutes: 30
```

```bash
kubectl apply -f gpu-node-healer.yaml
# expected: nodehealer.node.healing.tke.cloud.tencent.com/gpu-cluster-healer created
```

### 步骤 6：为 GPU 节点添加自愈标签

NodeHealer 通过 `nodeSelector.matchLabels` 选择目标节点。需为 GPU 原生节点添加 `node-healing.tke.cloud.tencent.com/belong-to=gpu-cluster-healer` 标签。

```bash
# 批量标记所有原生 GPU 节点
kubectl label node -l node.tke.cloud.tencent.com/accelerator-type=gpu \
    node-healing.tke.cloud.tencent.com/belong-to=gpu-cluster-healer
# expected: node/gpu-node-1 labeled, node/gpu-node-2 labeled

# 确认节点同时具备 tencentcloud 标签
kubectl get nodes -l node-healing.tke.cloud.tencent.com/belong-to=gpu-cluster-healer \
    -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.metadata.labels.cloud\.tencent\.com/provider}{"\n"}{end}'
# expected: 每行列节点名和 provider 标签值（应为 tencentcloud）
```

```text
NAME  STATUS  AGE
...
```

## 验证

### 控制面（tccli）

确认三个组件均已安装成功：

```bash
# 验证 nvidia-gpu 组件状态
tccli tke DescribeAddon --region <Region> \
    --ClusterId CLUSTER_ID
# expected: Addons 中包含 nvidia-gpu，Phase 为 Succeeded

# 验证 NPDPlus 组件状态
tccli tke DescribeAddon --region <Region> \
    --ClusterId CLUSTER_ID
# expected: Addons 中包含 npdplus，Phase 为 Succeeded

# 验证 Node-Healing 组件状态
tccli tke DescribeAddon --region <Region> \
    --ClusterId CLUSTER_ID
# expected: Addons 中包含 node-healing，Phase 为 Succeeded
```

```json
{
  "Addons": "<Addons>",
  "AddonName": "<AddonName>",
  "AddonVersion": "<AddonVersion>",
  "RawValues": "<RawValues>",
  "Phase": "<Phase>",
  "Reason": "<Reason>"
}
```

### 数据面（kubectl）

```bash
# 1. 确认 nvidia-gpu DaemonSet 运行正常
kubectl get ds -n kube-system nvidia-device-plugin-daemonset
# expected: DESIRED 等于 CURRENT 等于 READY

# 2. 确认 NPDPlus DaemonSet 运行正常
kubectl get ds -n kube-system | grep node-problem-detector
# expected: DESIRED 等于 CURRENT 等于 READY

# 3. 确认 Node-Healing Controller 运行正常
kubectl get deploy -n kube-system node-healing-controller
# expected: READY = 1/1

# 4. 确认 CRD 已注册
kubectl get crd | grep node.healing.tke.cloud.tencent.com
# expected: 3 个 CRD（healingtasks、machinehealingtemplates、nodehealers）

# 5. 确认 NodeHealer 和 MachineHealingTemplate 资源已创建
kubectl get nodehealer -n kube-system
# expected: gpu-cluster-healer 存在

kubectl get machinehealingtemplate -n kube-system
# expected: gpu-healing-template 存在

# 6. 确认 GPU 节点已打标签且被 NodeHealer 选中
kubectl get nodes -l node-healing.tke.cloud.tencent.com/belong-to=gpu-cluster-healer
# expected: 列出已标记的 GPU 原生节点

# 7. 确认 GPU 节点 Condition 正常
kubectl get node NODE_NAME -o jsonpath='{.status.conditions}' | jq '.[] | select(.type | startswith("GPU"))'
# expected: 正常时无输出或无 GPU 故障（status 为 False）
```

```text
NAME  STATUS  AGE
...
```

验证维度总览：

| 维度 | 命令 | 预期 |
|------|------|------|
| 组件状态 | `DescribeAddon` 三个组件 | Phase 均为 `Succeeded` |
| GPU 资源 | `kubectl describe node NODE_NAME \| grep nvidia.com/gpu` | nvidia.com/gpu >= 1 |
| NPDPlus DaemonSet | `kubectl get ds -n kube-system \| grep node-problem-detector` | DESIRED 等于 CURRENT 等于 READY |
| Node-Healing Controller | `kubectl get deploy -n kube-system node-healing-controller` | READY = 1/1 |
| CRD 注册 | `kubectl get crd \| grep node.healing` | 3 个 CRD |
| NodeHealer | `kubectl get nodehealer -n kube-system` | gpu-cluster-healer 存在 |
| 节点标签 | `kubectl get nodes -l node-healing.tke.cloud.tencent.com/belong-to=gpu-cluster-healer` | 已标记节点列表 |

当 NPDPlus 检测到 GPU 故障时，对应 Condition 变为 True，Node-Healing 自动创建 HealingTask：

```bash
kubectl get healingtask -n kube-system
# expected: 故障发生时出现 HealingTask
```

```text
NAME  STATUS  AGE
...
```

## 清理

> **副作用警告**：卸载组件不会自动删除已创建的 NodeHealer、MachineHealingTemplate 资源和节点标签。需按依赖倒序手动清理：先删 NodeHealer → 删 MachineHealingTemplate → 移除标签 → 卸载组件。卸载 nvidia-gpu 后节点上的 `nvidia.com/gpu` 资源将不再注册，影响 GPU Pod 调度。
>
> **计费警告**：三个组件本身不产生额外费用，但自愈操作中的节点重启（RebootNode）会导致 Pod 短时不可用，清理前请确认无关键业务运行。
>
> **操作警告**：
> - 卸载 npdplus/node-healing 组件前，确认没有正在执行的自愈任务：`kubectl get healingtask -n kube-system`
> - 卸载组件会删除所有相关 CRD 资源和运行中的 Pod，不影响节点本身
> - 组件卸载后，GPU 故障检测与自愈功能完全停用，建议仅在维护窗口执行

### 清理前状态检查

```bash
# 确认当前自愈资源状态
kubectl get nodehealer -n kube-system
# expected: 列出 gpu-cluster-healer

kubectl get machinehealingtemplate -n kube-system
# expected: 列出 gpu-healing-template

# 确认当前无正在执行的自愈任务
kubectl get healingtask -n kube-system
# expected: 无任务或任务已完成
```

```text
NAME  STATUS  AGE
...
```

### 数据面（kubectl）— 按依赖倒序删除

```bash
# 1. 删除 NodeHealer（停止自愈）
kubectl delete nodehealer gpu-cluster-healer -n kube-system
# expected: nodehealer "gpu-cluster-healer" deleted

# 2. 删除 MachineHealingTemplate
kubectl delete machinehealingtemplate gpu-healing-template -n kube-system
# expected: machinehealingtemplate "gpu-healing-template" deleted

# 3. 移除所有 GPU 节点的自愈标签（批量）
kubectl label node -l node-healing.tke.cloud.tencent.com/belong-to=gpu-cluster-healer \
    node-healing.tke.cloud.tencent.com/belong-to-
# expected: node/gpu-node-1 labeled, node/gpu-node-2 labeled
```

### 控制面（tccli）— 按依赖倒序卸载组件

```bash
# 4. 卸载 Node-Healing 组件
tccli tke DeleteAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName node-healing
# expected: exit 0

# 5. 卸载 NPDPlus 组件
tccli tke DeleteAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName npdplus
# expected: exit 0

# 6. 卸载 nvidia-gpu 组件（如不再需要 GPU 资源调度）
tccli tke DeleteAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName nvidia-gpu
# expected: exit 0
```

### 验证已删除

```bash
# 确认三个组件均已卸载（一次 DescribeAddon 即可）
tccli tke DescribeAddon --region <Region> \
    --ClusterId CLUSTER_ID
# expected: Addons 列表中不含 npdplus、node-healing、nvidia-gpu
```

**预期输出**（部分）：

```json
{
    "Addons": [
        {"AddonName": "cbs", "AddonVersion": "1.1.15", "Phase": "Succeeded"},
        {"AddonName": "cluster-autoscaler", "AddonVersion": "2.1.9", "Phase": "Succeeded"},
        {"AddonName": "coredns", "AddonVersion": "1.0.4", "Phase": "Succeeded"}
    ],
    "RequestId": "..."
}
```

> npdplus 和 node-healing 不在返回的 Addons 列表中，确认已删除。

```bash
# 确认数据面资源已清理
kubectl get nodehealer -n kube-system
# expected: No resources found

kubectl get machinehealingtemplate -n kube-system
# expected: No resources found
```

```json
{
  "Addons": "<Addons>",
  "AddonName": "<AddonName>",
  "AddonVersion": "<AddonVersion>",
  "RawValues": "<RawValues>",
  "Phase": "<Phase>",
  "Reason": "<Reason>"
}
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `InstallAddon` 返回 `UnknownParameter: addon version not found` | `tccli tke GetTkeAppChartList --region <Region>` 查询正确的 AddonName 和 LatestVersion | 使用了控制台显示名（`NodeProblemDetectorPlus`）而非 API AddonName（`npdplus`），或使用了控制台显示名（`NodeHealing`）而非 API AddonName（`node-healing`） | 使用 `GetTkeAppChartList` 查询准确的 AddonName。npdplus 是 NodeProblemDetectorPlus 的图表名，node-healing 是 Node-Healing 的图表名。版本号使用 `LatestVersion` 字段 |
| `InstallAddon` 返回 `UnknownParameter: addon version not found`（AddonName 正确但版本错误） | `tccli tke GetTkeAppChartList --region <Region>` 查看对应 Addon 的 `LatestVersion` | 版本号不匹配。如 `InstallAddon --generate-cli-skeleton` 中的示例版本号可能不是当前可用版本 | 从 `GetTkeAppChartList` 的 `LatestVersion` 字段获取正确版本号。当前验证：npdplus `1.1.4`、node-healing `1.0.0`。不要使用 skeleton 中的示例值 |
| `CreateClusterEndpoint`（公网端点）返回 `InvalidParameter.Param: CAM deny` | `tccli tke DescribeClusterEndpoints --region <Region> --ClusterId CLUSTER_ID` 查看端点状态 | 组织级 CAM 策略 `strategyId:240463971` 以 `tke:clusterExtranetEndpoint=true` 条件硬拒绝创建公网端点。自建安全组也无法绕过 | Fallback 到内网端点（`--IsExtranet false`）。内网端点需通过 IOA/VPN/专线或在同 VPC CVM 上执行 kubectl 命令。`tccli tke DescribeClusterKubeconfig --ClusterId CLUSTER_ID --IsExtranet false` 获取内网 kubeconfig |
| `kubectl` 连接内网端点返回 `http: server gave HTTP response to HTTPS client` | `tccli tke DescribeClusterEndpoints --region <Region> --ClusterId CLUSTER_ID` 检查端点类型和 IP | 从 VPC 外访问内网端点（如 `172.24.0.12`）。内网端点仅在 VPC 内可达 | 通过 IOA/VPN/专线接入 VPC，或在同 VPC CVM 上执行 kubectl。如公网端点被 CAM 拒绝（见上条），这是唯一路径 |
| `InstallAddon` 返回 `FailedOperation.AddonAlreadyInstalled` | `tccli tke DescribeAddon --region <Region> --ClusterId CLUSTER_ID` 查看组件状态 | 组件已安装 | 跳过安装步骤，用 `DescribeAddon` 确认 `Phase` 为 `Succeeded` 即可 |
| `InstallAddon` 返回 `ResourceNotFound.ClusterNotFound` | `tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'` 确认集群存在 | ClusterId 错误或集群已删除 | 核对集群 ID，从 `DescribeClusters` 获取正确的 `ClusterId` |
| `InstallAddon` 返回 `InternalError` 或超时 | `tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'` 查看 `ClusterStatus` | 集群状态非 Running | 等待集群恢复 Running 后重试，保留 RequestId |
| `kubectl apply` 返回 `no matches for kind "NodeHealer"` | `kubectl get crd \| grep node.healing` 确认 CRD | Node-Healing 组件未安装或 CRD 未就绪 | 确认 `DescribeAddon` 中 `node-healing` 的 `Phase` 为 `Succeeded` |
| `LimitExceeded` 或 `CamNoAuth` | `tccli tke DescribeAddon --region <Region> --ClusterId CLUSTER_ID` 确认调用权限 | CAM 权限不足或资源配额已达上限 | 此为环境限制，非命令错误。确认 CAM 策略含 `tke:InstallAddon`、`tke:DeleteAddon`、`tke:GetTkeAppChartList` |

### 操作成功但状态异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 组件 `Phase` 为 `Failed` 或长时间 `Creating` | `tccli tke DescribeAddon --region <Region> --ClusterId CLUSTER_ID` 查看 `Reason` 字段 | 节点资源不足、镜像拉取失败、RBAC 冲突 | 根据 `Reason` 定位；资源不足则扩容节点；保留 RequestId 提交工单 |
| GPU 故障检测不生效（npdplus 安装成功） | `kubectl get ds -n kube-system \| grep node-problem-detector` 确认 DaemonSet | NPDPlus Pod 未运行或未调度到 GPU 节点 | 检查 NPDPlus Pod 日志 `kubectl logs -n kube-system -l app=node-problem-detector --tail=50` |
| GPU 故障检测不生效（节点无 GPU 资源） | `kubectl describe node NODE_NAME \| grep nvidia.com/gpu` 检查 | 未安装 nvidia-gpu 组件，节点不注册 `nvidia.com/gpu` 资源 | 先安装 nvidia-gpu：`tccli tke InstallAddon --AddonName nvidia-gpu --AddonVersion 1.0.5` |
| GPU Condition 出现但自愈未触发 | `kubectl get nodehealer -n kube-system -o yaml` 检查 matchLabels | 节点标签不匹配 NodeHealer 的 `matchLabels` | 核对标签：确保节点含 `cloud.tencent.com/provider=tencentcloud`、`node.tke.cloud.tencent.com/accelerator-type=gpu`、`node-healing.tke.cloud.tencent.com/belong-to=gpu-cluster-healer` |
| 自愈任务未触发（标签正确） | `kubectl get node NODE_NAME -o jsonpath='{.status.conditions}' \| jq '.[] \| select(.status=="True")'` 检查持续时间 | 故障持续时间未超过 `durationTime` | 等待达到阈值，或调小 `durationTime` |
| 自愈达限流上限 | `kubectl describe nodehealer gpu-cluster-healer -n kube-system` 检查限流 | 达到 `maxHealNodesPerDay` 或 `maxHealConcurrency` 上限 | 增大限流阈值，或等待冷却期结束 |
| NodeHealer enableHealing 为 false | `kubectl get machinehealingtemplate -n kube-system -o yaml` 查看配置 | 对应故障类型的 `enableHealing` 未开启 | 设为 `true` 并 `kubectl apply` 重新应用 |
| 完整自愈后节点未恢复 | `kubectl describe node NODE_NAME` 查看 Condition | 严重硬件故障（ECC）仅隔离排空不重启 | 登录节点执行 `nvidia-smi -q`，提交工单维修后 `kubectl uncordon` |
| nvidia-gpu 安装后 GPU 资源未注册 | `kubectl describe node NODE_NAME \| grep nvidia.com/gpu` 检查资源 | 驱动未安装或不兼容，或节点无 GPU | `kubectl logs -n kube-system -l app=nvidia-device-plugin --tail=50` 查看日志 |
| 节点标签与 NodeHealer nodeSelector 不匹配 | `kubectl get node NODE_NAME --show-labels` 查看标签 | 标签键值对与 `spec.nodeSelector.matchLabels` 不一致 | 确认 `kubectl label` 添加的标签键值对与 NodeHealer YAML 中的 `matchLabels` 完全一致 |
| 如何关闭单节点自愈 | `kubectl get node NODE_NAME --show-labels` 查看标签 | 节点标签匹配了 NodeHealer 的 `matchLabels` | `kubectl label node NODE_NAME node-healing.tke.cloud.tencent.com/belong-to-` |

### 端点不可达排查流程

当 kubectl 无法连接时，按以下顺序排查：

```bash
# 1. 检查集群端点状态
tccli tke DescribeClusterEndpoints --region <Region> --ClusterId CLUSTER_ID
# expected: 查看每个端点的 IsExtranet、Status、Domain

# 2. 公网端点被 CAM 拒绝（strategyId:240463971）
#    现象：CreateClusterEndpoint --IsExtranet true 返回 InvalidParameter.Param
#    诊断：组织级策略以 tke:clusterExtranetEndpoint=true 条件硬拒绝
#    解决：使用内网端点（--IsExtranet false）

# 3. 内网端点需 VPC 内可达
#    现象：kubectl 连接内网 IP 返回 http: server gave HTTP response to HTTPS client
#    解决：通过 IOA/VPN/专线 接入 VPC，或在同 VPC CVM 上执行

# 4. 如通过同 VPC CVM 执行
#    4a. 安装 kubectl
#    4b. 获取 kubeconfig
tccli tke DescribeClusterKubeconfig --region <Region> \
    --ClusterId CLUSTER_ID \
    --IsExtranet false | jq -r '.Kubeconfig' > ~/.kube/config
#    4c. 验证连通性
kubectl cluster-info
kubectl get ns
```

## 下一步

- [nvidia-gpu 说明](https://cloud.tencent.com/document/product/457/104593) — GPU 驱动与设备插件组件详情
- [NodeProblemDetectorPlus 说明](https://cloud.tencent.com/document/product/457/49422) — NPDPlus 组件支持的全部检测项
- [Node-Healing 说明](https://cloud.tencent.com/document/product/457/127504) — Node-Healing 组件支持的全部故障类型和自愈策略
- [GPU 监控指标获取](../GPU%20监控指标获取/tccli%20操作.md) — 监控 GPU 利用率与显存
- [故障自愈规则](https://cloud.tencent.com/document/product/457/78650) — 通用节点故障自愈

## 控制台替代

[控制台 → 集群 → 组件管理](https://console.cloud.tencent.com/tke2/cluster)：单击 **新建**，依次勾选 **nvidia-gpu**、**NodeProblemDetectorPlus**、**Node-Healing** 完成组件安装。MachineHealingTemplate 和 NodeHealer 资源需通过 kubectl 在集群内创建。
