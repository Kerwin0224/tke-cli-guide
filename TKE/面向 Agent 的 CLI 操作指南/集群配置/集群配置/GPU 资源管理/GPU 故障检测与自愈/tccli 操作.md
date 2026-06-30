# GPU 故障检测与自愈

> 对照官方：[GPU 故障检测与自愈](https://cloud.tencent.com/document/product/457/127503) · page_id `127503`

## 概述

TKE 提供完整的 GPU 故障检测与自愈方案：通过 **Node Problem Detector Plus（NPDPlus）** 组件实时检测 GPU 异常，结合 **Node-Healing** 组件自动执行隔离、排空、重启等自愈操作。仅作用于原生节点。

## 前置条件

- TKE 集群包含 GPU 节点（原生节点）
- Kubernetes 版本 >= 1.18
- 已安装 NPDPlus 和 Node-Healing 组件
- kubectl 可达集群以 apply YAML

> **kubectl 可达性说明**：当前共享集群 kubectl 不可达。以下 YAML 完整，但 `kubectl apply` 无法真跑验证。共享集群无 GPU 节点，功能无法验证。

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 安装 NPDPlus 组件 | `tccli tke InstallAddon --AddonName NodeProblemDetectorPlus` | 否 |
| 安装 Node-Healing 组件 | `tccli tke InstallAddon --AddonName NodeHealing` | 否 |
| 创建 NodeHealer 自愈策略 | `kubectl apply -f <yaml>` | 是 |
| 为节点添加自愈标签 | `kubectl label node <name> <key>=<value>` | 是 |
| 查看 NodeHealer 状态 | `kubectl get nodehealer` | 是 |

## 操作步骤

### 安装 NPDPlus 组件

```bash
tccli tke InstallAddon --region <Region> \
    --cli-input-json '{"ClusterId":"<ClusterId>","AddonName":"NodeProblemDetectorPlus","AddonVersion":"<AddonVersion>"}'
# expected: exit 0
```

### 安装 Node-Healing 组件

```bash
tccli tke InstallAddon --region <Region> \
    --cli-input-json '{"ClusterId":"<ClusterId>","AddonName":"NodeHealing","AddonVersion":"<AddonVersion>"}'
# expected: exit 0
```

### 配置 GPU 节点自愈策略

```yaml
apiVersion: node.healing.tke.cloud.tencent.com/v1
kind: NodeHealer
metadata:
  name: gpu-cluster-healer
spec:
  nodeSelector:
    matchLabels:
      node-healing.tke.cloud.tencent.com/belong-to: gpu-cluster-healer
      cloud.tencent.com/provider: tencentcloud
      node.tke.cloud.tencent.com/accelerator-type: gpu
  machineHealingTemplate:
    name: "default-healing-template"
    kind: "MachineHealingTemplate"
    apiVersion: "node.healing.tke.cloud.tencent.com/v1"
  unhealthyConditions:
    - type: GPUXIDFatalError
      enableCheck: true
      enableHealing: true
      durationTime: "1m"
      priority: 100
    - type: GPUDeviceLost
      enableCheck: true
      enableHealing: true
      durationTime: "5m"
      priority: 80
  nodeHealingPolicy:
    maxHealConcurrency: 3
    minNodeCoolingMinutes: 30
```

```bash
kubectl apply -f gpu-node-healer.yaml
# expected: nodehealer.node.healing.tke.cloud.tencent.com/gpu-cluster-healer configured
```

### 为 GPU 节点添加自愈标签

```bash
# 批量为所有原生 GPU 节点添加标签
kubectl label node \
  -l node.tke.cloud.tencent.com/accelerator-type=gpu \
  node-healing.tke.cloud.tencent.com/belong-to=gpu-cluster-healer
# expected: nodes labeled
```

### GPU 故障检测类型

| 故障类型 | 说明 | 自愈策略 |
|---------|------|---------|
| `GPUDeviceLost` | GPU 设备丢失 | 完整自愈：隔离→排空→重启→解封 |
| `GPUDoubleBitEccError` | 双位 ECC 错误 | 仅隔离排空 |
| `GPUHighSingleBitEccError` | 单位 ECC 错误超阈值 | 仅隔离排空 |
| `GPUInfoROMCorrupted` | InfoROM 损坏 | 仅隔离排空 |
| `GPUNvlinkError` | NVLink 通信错误 | 完整自愈 |
| `GPUPCIeError` | PCIe 接口异常 | 完整自愈 |
| `GPUPendingRetiredPages` | 待退役内存页 | 完整自愈 |
| `GPUPowerError` | 电源问题 | 仅隔离排空 |
| `GPURemappingRowsError` | 内存行映射错误 | 仅隔离排空 |
| `GPUTempError` | 温度过高 | 仅隔离排空 |
| `GPUXIDFatalError` | 致命 XID 错误 | 仅隔离排空 |
| `GPUXIDWarningError` | 警告 XID 错误 | 完整自愈 |

### unhealthyConditions 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `type` | string | 故障类型 |
| `enableCheck` | bool | 是否启用检测 |
| `enableHealing` | bool | 是否启用自愈 |
| `durationTime` | string | 故障持续多久后触发自愈（如 `1m`、`5m`） |
| `priority` | int | 优先级（0-100） |

### nodeHealingPolicy 限流参数

| 参数 | 说明 | 建议值 |
|------|------|--------|
| `maxHealNodesPerDay` | 每天最大自愈节点数 | 10-20 |
| `maxHealNodesPerHour` | 每小时最大自愈节点数 | 3-5 |
| `maxHealConcurrency` | 最大并发自愈数 | 2-3 |
| `minNodeCoolingMinutes` | 同一节点两次自愈最小间隔（分钟） | 30-60 |

## 验证

```bash
kubectl get nodehealer
kubectl get ds -n kube-system | grep node-problem-detector
kubectl get healingtask -n kube-system
# expected: NodeHealer 存在，NPD DaemonSet 运行中
```

```text
NAME  STATUS  AGE
...
```

## 清理

```bash
kubectl delete nodehealer gpu-cluster-healer
# 移除自愈标签
kubectl label node -l node.tke.cloud.tencent.com/accelerator-type=gpu \
  node-healing.tke.cloud.tencent.com/belong-to-
```

## 排障

| 现象 | 根因 | 修复 |
|------|------|------|
| GPU 故障检测不生效 | NPD 组件未运行 | `kubectl get ds -n kube-system \| grep node-problem-detector` |
| 自愈任务未触发 | 节点标签与 NodeHealer 配置不匹配 | 检查 `matchLabels` 和节点标签是否一致 |
| 自愈任务未触发 | 节点非原生节点 | 确认节点有 `cloud.tencent.com/provider=tencentcloud` 标签 |

## 下一步

- [qGPU 概述](../GPU 共享/qGPU概述/tccli 操作.md)
- [GPU 监控指标获取](../GPU 监控指标获取/tccli 操作.md)

## 控制台替代

[TKE 控制台 → 集群详情 → 组件管理](https://console.cloud.tencent.com/tke2/cluster)：新建 NPDPlus 和 Node-Healing 组件。
