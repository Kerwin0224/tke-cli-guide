# Node-Healing 说明（tccli）

> 对照官方：[Node-Healing 说明](https://cloud.tencent.com/document/product/457/127504) · page_id `127504`

## 概述

Node-Healing 是 TKE 的节点自愈组件。该组件自动检测集群中的异常节点，并根据预定义的策略执行自动化修复操作。它与 Node Problem Detector（NPD）配合使用，形成从检测到修复的完整闭环。

### 支持的故障类型

支持 **5 大类共 40 种故障类型**：

#### 内核/系统类（7 种）— 完整流程

| 类型 | 说明 | 自愈策略 |
|------|------|---------|
| KernelDeadlock | 内核死锁 | 完整流程 |
| ReadonlyFilesystem | 文件系统只读 | Cordon+Drain+UnCordon |
| StuckProcess | 进程卡死 | Cordon+Drain+UnCordon |
| ProcessProblem | 大量 D 状态进程 | Cordon+Drain+UnCordon |
| XFSError | XFS 文件系统错误 | Cordon+Drain+UnCordon |
| DiskReadOnly | 磁盘只读 | Cordon+Drain+UnCordon |
| NetDeviceProblem | 网卡硬件故障 | 仅 Cordon+Drain |

#### 资源压力类（7 种）— Cordon+Drain+UnCordon

| 类型 | 说明 |
|------|------|
| CustomPIDPressure | 自定义 PID 压力 |
| FDPressure | 文件描述符压力 |
| ThreadPressure | 系统线程压力 |
| SystemLoadProblem | 系统负载异常 |
| OverlayDiskPressure | Overlay 磁盘压力 |
| FDProblem | FD 泄漏 |
| ConntrackProblem | Conntrack 表满 |

#### 服务健康类（8 种）— 完整流程

| 类型 | 说明 |
|------|------|
| ContainerdProblem | Containerd 服务异常 |
| ContainerRuntimeInterfaceUnhealthy | CRI 不健康 |
| ContainerRuntimeUnhealthy | 运行时不可用 |
| ContainerdRuntimeUnhealthy | Containerd 运行时异常 |
| DockerdProblem | Docker 服务异常 |
| KubeletProblem | Kubelet 服务异常 |
| HasGhostContainer | 幽灵容器 |
| CorruptDockerOverlay2 | Overlay2 损坏 |

#### K8s 组件扩展（6 种）

| 类型 | 说明 | 自愈策略 |
|------|------|---------|
| ConnectApiServer | 无法连接 API Server | 完整流程 |
| PLEGNotHealthy | PLEG 不健康 | 完整流程 |
| KubeletUnhealthy | Kubelet 不健康 | 完整流程 |
| FrequentKubeletRestart | Kubelet 频繁重启 | Cordon+UnCordon |
| FrequentDockerRestart | Docker 频繁重启 | Cordon+UnCordon |
| FrequentContainerdRestart | Containerd 频繁重启 | Cordon+UnCordon |

#### GPU 监控（12 种）

| 类型 | 说明 | 自愈策略 |
|------|------|---------|
| GPUDeviceLost | GPU 设备丢失 | 完整流程 |
| GPUDoubleBitEccError | GPU 双比特 ECC 错误（不可纠正） | 仅 Cordon+Drain |
| GPUHighSingleBitEccError | GPU 单比特 ECC 超阈值 | 仅 Cordon+Drain |
| GPUInfoROMCorrupted | GPU InfoROM 损坏 | 仅 Cordon+Drain |
| GPUNvlinkError | NVLink 通信异常 | 完整流程 |
| GPUPCIeError | PCIe 接口异常 | 完整流程 |
| GPUPendingRetiredPages | GPU 待退役页面 | 完整流程 |
| GPUPowerError | GPU 电源故障 | 仅 Cordon+Drain |
| GPURemappingRowsError | GPU 内存行重映射错误 | 仅 Cordon+Drain |
| GPUTempError | GPU 温度过高 | 仅 Cordon+Drain |
| GPUXIDFatalError | GPU 致命 XID 错误 | 仅 Cordon+Drain |
| GPUXIDWarningError | GPU 警告 XID 错误 | 完整流程 |

### 自愈策略说明

| 策略 | 别名 | 说明 |
|------|------|------|
| **完整流程** | Cordon → Drain → Reboot → UnCordon | 用于可软件恢复的故障 |
| **Cordon+Drain+UnCordon** | 隔离+排空+恢复 | 用于资源压力类故障 |
| **仅 Cordon+Drain** | 隔离+排空 | 用于硬件故障，需人工介入 |
| **Cordon+UnCordon** | 隔离+恢复 | 用于频繁重启类场景 |

### 部署的 Kubernetes 对象

| 对象名称 | 类型 | 资源 | 命名空间 |
|---------|------|------|---------|
| node-healing-controller | Deployment | 100m CPU / 128Mi Memory | kube-system |
| node-healing | ServiceAccount | — | kube-system |
| node-healing-role | ClusterRole | — | — |
| node-healing-rolebinding | ClusterRoleBinding | — | — |
| machinehealingtemplates.node.healing.tke.cloud.tencent.com | CRD | — | — |
| nodehealers.node.healing.tke.cloud.tencent.com | CRD | — | — |
| healingtasks.node.healing.tke.cloud.tencent.com | CRD | — | — |

## 前置条件

- [环境准备](../../../../环境准备.md)
- 已 [连接集群](../../../../集群配置/集群管理/连接集群/tccli 操作.md)，kubectl 可正常访问 APIServer
- Kubernetes 版本 **>= 1.18**
- **必须已安装 NodeProblemDetectorPlus（NPD）组件**，Node-Healing 依赖 NPD 的检测结果
- **仅支持原生节点**，不支持超级节点、第三方节点（注册节点）、普通节点
- 集群状态 `Running`

```bash
# 检查集群状态
tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'
# expected: ClusterStatus "Running"
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

- 需要 CAM 权限：`tke:DescribeClusters`、`tke:DescribeAddon`、`tke:InstallAddon`、`tke:UpdateAddon`、`tke:DeleteAddon`

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 查看集群信息 | `tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'` | 是 |
| 查看 Node-Healing 组件详情 | `tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName node-healing` | 是 |
| 安装 Node-Healing 组件 | `tccli tke InstallAddon --region <Region> --ClusterId <ClusterId> --AddonName node-healing` | 否 |
| 升级 Node-Healing 组件 | `tccli tke UpdateAddon --region <Region> --ClusterId <ClusterId> --AddonName node-healing --AddonVersion <AddonVersion>` | 否 |
| 卸载 Node-Healing 组件 | `tccli tke DeleteAddon --region <Region> --ClusterId <ClusterId> --AddonName node-healing` | 是 |
| 创建 NodeHealer（自愈策略） | `kubectl apply -f node-healer.yaml` | 否（同名报错） |
| 查看自愈任务 | `kubectl get healingtasks` | 是 |
| 添加节点标签 | `kubectl label node <NodeName> ...` | 是 |

## 操作步骤

### 步骤 1：安装 NodeProblemDetectorPlus（控制面，前置依赖）

```bash
tccli tke InstallAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName node-problem-detector-plus
# expected: exit 0
```

### 步骤 2：安装 Node-Healing 组件（控制面）

```bash
tccli tke InstallAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName node-healing
# expected: exit 0，组件安装请求已提交
```

### 步骤 3：为节点打标签（数据面）

NodeHealer 通过 nodeSelector 选择管理哪些节点。需要为目标原生节点打上标签：

```bash
kubectl label node <NodeName> node-healing.tke.cloud.tencent.com/belong-to=cluster-node-healer
# expected: node/<NodeName> labeled
```

> **说明：** 替换 `<NodeName>` 为实际原生节点名称。可对多个节点重复执行。

> **注意**：kubectl 数据面当前因公网端点被 CAM 策略 strategyId:240463971 拒绝而不可达。以下 kubectl 命令为参考格式，需在内网/VPN 环境执行。

### 步骤 4：创建 MachineHealingTemplate（数据面）

定义自愈模板，指定不同故障类型的修复步骤：

```yaml
apiVersion: node.healing.tke.cloud.tencent.com/v1
kind: MachineHealingTemplate
metadata:
  name: default-healing-template
  namespace: kube-system
spec:
  nodeConditions:
    - type: KernelDeadlock
      status: "True"
      enableHealing: true
      priority: 100
      durationTime: "5m"
      steps:
        - CordonNode
        - DrainNode
        - RebootNode
        - UnCordonNode
    - type: KubeletProblem
      status: "True"
      enableHealing: true
      priority: 85
      durationTime: "5m"
      steps:
        - CordonNode
        - DrainNode
        - RebootNode
        - UnCordonNode
    - type: FDPressure
      status: "True"
      enableHealing: true
      priority: 70
      durationTime: "5m"
      steps:
        - CordonNode
        - DrainNode
        - UnCordonNode
    - type: GPUDoubleBitEccError
      status: "True"
      enableHealing: true
      priority: 90
      durationTime: "1m"
      steps:
        - CordonNode
        - DrainNode
```

| 自愈步骤 | 说明 |
|---------|---------|
| `CordonNode` | 隔离节点，不再调度新 Pod |
| `DrainNode` | 排空节点上的 Pod |
| `RebootNode` | 重启节点 |
| `UnCordonNode` | 恢复节点可调度 |

```bash
kubectl apply -f machine-healing-template.yaml
# expected: machinehealingtemplate.node.healing.tke.cloud.tencent.com/default-healing-template created
```

### 步骤 5：创建 NodeHealer（数据面）

定义哪些节点需要自愈以及使用哪个模板和策略：

```yaml
apiVersion: node.healing.tke.cloud.tencent.com/v1
kind: NodeHealer
metadata:
  name: cluster-node-healer
  namespace: kube-system
spec:
  nodeSelector:
    matchLabels:
      node-healing.tke.cloud.tencent.com/belong-to: cluster-node-healer
      cloud.tencent.com/provider: tencentcloud
  machineHealingTemplate:
    name: default-healing-template
    kind: MachineHealingTemplate
    apiVersion: node.healing.tke.cloud.tencent.com/v1
  unhealthyConditions:
    - type: KernelDeadlock
      enableCheck: true
      enableHealing: true
      durationTime: "5m"
      priority: 100
    - type: KubeletProblem
      enableCheck: true
      enableHealing: true
      durationTime: "5m"
      priority: 85
    - type: FDPressure
      enableCheck: true
      enableHealing: true
      durationTime: "5m"
      priority: 70
    - type: GPUDoubleBitEccError
      enableCheck: true
      enableHealing: true
      durationTime: "1m"
      priority: 90
  nodeHealingPolicy:
    maxHealNodesPerDay: 10
    maxHealNodesPerHour: 3
    maxHealConcurrency: 2
    minNodeCoolingMinutes: 60
```

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `maxHealNodesPerDay` | 10 | 每天最大自愈节点数 |
| `maxHealNodesPerHour` | 3 | 每小时最大自愈节点数 |
| `maxHealConcurrency` | 2 | 最大并发自愈数 |
| `minNodeCoolingMinutes` | 60 | 同一节点两次自愈的最小冷却时间（分钟） |

```bash
kubectl apply -f node-healer.yaml
# expected: nodehealer.node.healing.tke.cloud.tencent.com/cluster-node-healer created
```

### 组件 RBAC 权限（控制面已自动创建）

组件安装时自动创建 ClusterRole `node-healing-role`：

| API Group | 资源 | 操作 | 用途 |
|-----------|------|------|------|
| `""` (core) | nodes | get, list, watch, patch, update | 节点状态监控和修复 |
| `""` (core) | nodes/status | patch, update | 修改节点 Condition |
| `""` (core) | events | create, update, patch | 发送事件通知 |
| `""` (core) | pods | get, list, watch, delete | Pod 驱逐 |
| `""` (core) | pods/eviction | create | Pod 驱逐请求 |
| `node.healing.tke.cloud.tencent.com` | machinehealingtemplates, nodehealers, healingtasks | 完整 CRUD + watch | 管理自愈资源 |

## 验证

### 控制面（tccli）

```bash
tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName node-healing \
    | jq '.Addons[0] | {AddonName, Status, AddonVersion}'
# expected: Status "Running"，AddonName "node-healing"
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
kubectl get deploy -n kube-system node-healing-controller
# expected: READY 1/1

kubectl get crd | grep node.healing.tke.cloud.tencent.com
# expected: 列出 healingtasks、machinehealingtemplates、nodehealers 三个 CRD

kubectl get nodehealers -n kube-system
kubectl describe nodehealer cluster-node-healer -n kube-system
# expected: NodeHealer 已创建，Status 正常

kubectl get nodes -l node-healing.tke.cloud.tencent.com/belong-to=cluster-node-healer
# expected: 列出已打标签的原生节点
```

```text
NAME  STATUS  AGE
...
```

> **注意**：kubectl 数据面当前因公网端点被 CAM 策略 strategyId:240463971 拒绝而不可达。以下 kubectl 命令为参考格式，需在内网/VPN 环境执行。

### 监控指标

组件暴露以下 Prometheus 指标：

| 指标名 | 类型 | 说明 |
|--------|------|------|
| `component_request_duration_millisecond` | Histogram | 组件请求耗时（ms） |
| `controller_request_total` | Counter | 控制器请求总数 |
| `controller_request_duration_millisecond` | Summary（P25/P50/P75/P90/P99） | 控制器请求耗时分位数 |

## 清理

### 数据面（kubectl）

按创建逆序删除：

```bash
kubectl delete nodehealer cluster-node-healer -n kube-system
kubectl delete machinehealingtemplate default-healing-template -n kube-system
kubectl label node <NodeName> node-healing.tke.cloud.tencent.com/belong-to-
# expected: 资源已删除
```

### 控制面（tccli）

```bash
tccli tke DeleteAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName node-healing
# expected: exit 0，组件卸载请求已提交

# 验证已卸载
tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName node-healing \
    | jq '.Addons[] | select(.AddonName == "node-healing")'
# expected: 空结果
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

> **说明：** 卸载 Node-Healing 前建议先删除 NodeHealer 和 MachineHealingTemplate 资源。NPD 组件可选择保留或一并卸载。

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| NodeHealer `NotReady` 状态 | `kubectl get ds -n kube-system node-problem-detector` 确认 NPD 运行；`kubectl get nodes -l node-healing.tke.cloud.tencent.com/belong-to` 确认节点标签 | 未安装 NPD 或节点未打标签 | 安装 NPD；为目标节点打上 `node-healing.tke.cloud.tencent.com/belong-to` 标签 |
| 自愈不触发 | `kubectl describe nodehealer cluster-node-healer -n kube-system` 检查 Condition 配置 | `enableCheck` 或 `enableHealing` 未开启，或 `durationTime` 时间内故障已恢复 | 检查对应 Condition 的 `enableCheck` 和 `enableHealing` 为 `true`，`durationTime` 合理（如 `"5m"`） |
| 节点被间歇性 Cordoning | `kubectl get healingtasks` 查看自愈历史 | `minNodeCoolingMinutes` 设置过小，同一节点短时间被多次触发 | 增大 `minNodeCoolingMinutes` 值（如 `120` 即 2 小时冷却） |
| Controller Pod `CrashLoopBackOff` | `kubectl describe pod -n kube-system -l app=node-healing-controller` 查看 Events | 节点类型不支持（超级节点/注册节点） | Node-Healing 仅支持原生节点，移除不支持节点上的 NodeHealer 匹配规则 |
| `Unable to connect to the server` | `kubectl cluster-info` 检查连通性 | 公网端点被 CAM 策略 strategyId:240463971 以 `tke:clusterExtranetEndpoint=true` 条件拒绝 | 通过 IOA/VPN 连接内网，或使用同 VPC CVM 执行 kubectl 命令 |

## 下一步

- [NodeProblemDetectorPlus 说明](../NodeProblemDetectorPlus 说明/tccli 操作.md) — 节点异常检测组件
- [GPU 故障检测和自愈](https://cloud.tencent.com/document/product/457/127503) — GPU 节点故障自愈详细指南
- [组件的生命周期管理](../组件的生命周期管理/tccli 操作.md) — 组件安装、升级与卸载
- [OOMGuard 说明](../OOMGuard 说明/tccli 操作.md) — 内存 OOM 防护组件

## 控制台替代

通过 [TKE 控制台安装 Node-Healing 组件](https://cloud.tencent.com/document/product/457/127504) 操作。
