# 节点放大（tccli）

> 对照官方：[节点放大](https://cloud.tencent.com/document/product/457/79032) · page_id `79032`

## 概述

节点放大（Node Amplification）是 TKE Crane 调度器提供的资源超售能力。通过调整 craned 组件的 `nodeResourceAmplificationRatio` 参数，虚增 kubelet 上报的 Allocatable 资源量（CPU 和 Memory），使调度器感知到比物理资源更多的可用资源，从而在"看起来资源充足"的情况下调度更多 Pod，提升集群装箱率。节点放大不改变实际物理资源，配合 QoS 策略（CPU 优先级、内存 OOM 优先级等）可安全地实现资源超售。

## 前置条件

- 演示集群 `cls-xxxxxxxx`（ap-guangzhou, v1.30.0, Running），kubectl 因 CAM 策略限制不可达，需通过 VPN/IOA 内网或数据面集群操作
- 已安装 [Crane 调度器](../../调度组件概述/tccli%20操作.md)（craned 组件）
- `tke-eni-ip-webhook` 组件需更新至支持版本（如节点使用 ENI 网络模式）
- 建议配合 [QoS 策略](../../Qos%20感知调度/tccli%20操作.md) 使用，在超售的同时保障高优业务资源
- 已阅读 [资源利用率优化调度概述](../tccli%20操作.md) — page_id `122378`

### 环境检查

```bash
# 1. 确认集群状态
tccli tke DescribeClusters --ClusterIds '["cls-xxxxxxxx"]' --region ap-guangzhou --output json
```

```json
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "cls-xxxxxxxx",
            "ClusterName": "demo-cluster",
            "ClusterType": "MANAGED_CLUSTER",
            "ClusterStatus": "Running",
            "ClusterVersion": "1.30.0",
            "ClusterNodeNum": 3,
            "ClusterOs": "tlinux3.1",
            "VpcId": "vpc-xxxxxxxx",
            "CreatedTime": "2024-06-15T10:30:00Z"
        }
    ],
    "RequestId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
}
```

```bash
# 2. 确认 Crane 调度器已安装及当前配置
tccli tke DescribeAddon --ClusterId cls-xxxxxxxx --AddonName craned --region ap-guangzhou --output json
```

```json
{
    "AddonName": "craned",
    "AddonVersion": "v1.2.0",
    "Status": "Succeeded",
    "AddonConfig": {
        "NodeResourceAmplificationRatio": 1.0
    },
    "RequestId": "b2c3d4e5-f6a7-8901-bcde-f12345678901"
}
```

```bash
# 3. 记录节点当前 Allocatable 资源作为基线（需 VPN/IOA 内网环境）
kubectl describe node <node-name> | grep -A5 "Allocatable"
```

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 查看集群状态 | `tccli tke DescribeClusters --ClusterIds '["cls-xxxxxxxx"]' --region ap-guangzhou` | 是 |
| 查看组件当前配置 | `tccli tke DescribeAddon --ClusterId cls-xxxxxxxx --AddonName craned --region ap-guangzhou` | 是 |
| 启用节点放大（设置放大系数） | `tccli tke UpdateAddon --ClusterId cls-xxxxxxxx --AddonName craned --AddonVersion v1.2.0 --RawValues '{"nodeResourceAmplificationRatio":1.5}' --region ap-guangzhou` | 否 |
| 修改放大系数 | `tccli tke UpdateAddon` 重设 `nodeResourceAmplificationRatio` | 否 |
| 查看放大后节点 Allocatable | `kubectl describe node <node-name> \| grep -A5 "Allocatable"` | 是 |
| 恢复默认系数（关闭放大） | `tccli tke UpdateAddon --ClusterId cls-xxxxxxxx --AddonName craned --AddonVersion v1.2.0 --RawValues '{"nodeResourceAmplificationRatio":1.0}' --region ap-guangzhou` | 否 |

## 操作步骤

### 工作原理

```
┌─────────────────────────────────────────────────┐
│ 物理资源（不变）                                   │
│   CPU: 4 cores      Memory: 8192Mi               │
│                                                   │
│ craned 组件拦截 kubelet 上报                       │
│   Allocatable = 物理资源 × nodeResourceAmplificationRatio│
│                                                   │
│ 放大系数 1.5 → 调度器感知                          │
│   CPU: 6 cores      Memory: 12288Mi               │
│                                                   │
│ 调度器基于虚拟 Allocatable 做决策                    │
│   → 更多 Pod 被调度（装箱率提升 50%）               │
└─────────────────────────────────────────────────┘
```

**关键限制**：节点放大不修改 cgroup 或内核资源限制，实际物理资源不变。当节点上 Pod 的实际资源需求超过物理资源时，会触发资源争用（CPU Throttling、OOM）。因此必须配合 QoS 策略使用。

### 步骤 1：评估放大系数

根据集群工作负载特征选择合适的放大系数：

| 放大系数 | Allocatable 变化（4C8G 节点） | 适用场景 | 风险 |
|---------|----------------------------|---------|------|
| 1.0（默认） | CPU=4, Mem=8192Mi | 默认，不启用放大 | 无风险 |
| 1.2 | CPU=4.8, Mem=9830Mi | 轻度超售，负载波动小 | 低风险 |
| 1.5 | CPU=6, Mem=12288Mi | 中度超售，在线离线混部 | 中风险，需 QoS 保障 |
| 2.0 | CPU=8, Mem=16384Mi | 高度超售，大部分 Pod 空闲 | 高风险，不推荐 |

**选择建议**：从 1.2 开始，观察 1-2 周节点实际利用率后再上调。推荐范围 1.2-1.5。

### 步骤 2：设置放大系数

```bash
tccli tke UpdateAddon --region ap-guangzhou \
    --ClusterId cls-xxxxxxxx \
    --AddonName craned \
    --AddonVersion v1.2.0 \
    --RawValues '{"nodeResourceAmplificationRatio":1.5}'
```

```json
{
    "RequestId": "e5f6a7b8-c9d0-1234-ef01-234567890123"
}
```

> **幂等说明**：`UpdateAddon` 非幂等——每次调用返回新的 RequestId 并触发组件更新。但设置相同 `nodeResourceAmplificationRatio` 值的效果是幂等的。

### 步骤 3：验证放大效果

等待组件更新生效后，观察节点 Allocatable 变化。

放大前（系数 1.0）：

```bash
kubectl describe node <node-name> | grep -A5 "Allocatable"
```

```text
Allocatable:
  cpu:                4
  ephemeral-storage:  50783456Ki
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             8192Mi
```

放大后（系数 1.5）：

```bash
kubectl describe node <node-name> | grep -A5 "Allocatable"
```

```text
Allocatable:
  cpu:                6
  ephemeral-storage:  50783456Ki
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             12288Mi
```

### 步骤 4：配合 QoS 策略（推荐）

为防止节点放大导致资源过载影响核心业务，建议同时配置 QoS 策略：

**CPU 优先级保障**（高优服务避免 CPU Throttling）：

```yaml
apiVersion: scheduling.crane.io/v1alpha1
kind: PodQOS
metadata:
  name: cpu-high-priority
spec:
  resourceQOS:
    cpuQOS:
      cpuPriority: 0         # 最高 CPU 优先级
  selector:
    matchLabels:
      qos-tier: critical
```

**内存 OOM 优先级**（高优服务 OOM 时最后被 Kill）：

```yaml
apiVersion: scheduling.crane.io/v1alpha1
kind: PodQOS
metadata:
  name: mem-high-priority
spec:
  resourceQOS:
    memoryQOS:
      memPriority: 0         # 最高内存优先级
  selector:
    matchLabels:
      qos-tier: critical
```

```bash
kubectl apply -f podqos-cpu-priority.yaml
kubectl apply -f podqos-mem-priority.yaml
```

为重点保障的 Pod 添加标签：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: critical-service
  labels:
    qos-tier: critical
spec:
  containers:
  - name: app
    image: nginx:latest
```

### 步骤 5：监控节点实际负载

放大后需持续监控节点实际资源使用情况，确保物理资源未被过度占用。

```bash
# 查看节点实际 CPU 和内存使用
kubectl top node <node-name>
```

```text
NAME          CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
node-001      3200m        80%    6800Mi          83%
```

> **告警阈值建议**：节点物理 CPU 使用率 > 85% 或内存使用率 > 90% 时考虑降低放大系数或配合自动规整迁移 Pod。

## 验证

### 控制面

```bash
# 维度 1：确认集群状态
tccli tke DescribeClusters --ClusterIds '["cls-xxxxxxxx"]' --region ap-guangzhou --output json
```

```json
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "cls-xxxxxxxx",
            "ClusterName": "demo-cluster",
            "ClusterType": "MANAGED_CLUSTER",
            "ClusterStatus": "Running",
            "ClusterVersion": "1.30.0",
            "ClusterNodeNum": 3,
            "ClusterOs": "tlinux3.1",
            "VpcId": "vpc-xxxxxxxx",
            "CreatedTime": "2024-06-15T10:30:00Z"
        }
    ],
    "RequestId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
}
```

```bash
# 维度 2：确认组件配置已更新
tccli tke DescribeAddon --ClusterId cls-xxxxxxxx --AddonName craned --region ap-guangzhou --output json
```

```json
{
    "AddonName": "craned",
    "AddonVersion": "v1.2.0",
    "Status": "Succeeded",
    "AddonConfig": {
        "NodeResourceAmplificationRatio": 1.5
    },
    "RequestId": "d4e5f6a7-b8c9-0123-def0-123456789012"
}
```

### 数据面（需 VPN/IOA 内网环境）

```bash
# 维度 3：确认节点 Allocatable 已按系数放大
kubectl describe node <node-name> | grep -A5 "Allocatable"
```

```text
Allocatable:
  cpu:                6
  ephemeral-storage:  50783456Ki
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             12288Mi
```

```bash
# 维度 4：确认 QoSAgent 运行正常
kubectl get pods -n crane-system -l app=qos-agent
```

```text
NAME              READY   STATUS    RESTARTS   AGE
qos-agent-xxxxx   1/1     Running   0          1d
qos-agent-yyyyy   1/1     Running   0          1d
qos-agent-zzzzz   1/1     Running   0          1d
```

| 维度 | 命令 | 预期 |
|------|------|------|
| 集群状态 | `DescribeClusters --ClusterIds '["cls-xxxxxxxx"]'` | `ClusterStatus: "Running"` |
| 组件配置 | `DescribeAddon --ClusterId cls-xxxxxxxx --AddonName craned` | `NodeResourceAmplificationRatio: 1.5` |
| 节点 Allocatable | `kubectl describe node` | `cpu` 和 `memory` 为物理资源的 1.5 倍 |
| QoSAgent Pod | `kubectl get pods -n crane-system -l app=qos-agent` | 全部 Running |

## 清理

恢复默认放大系数（关闭节点放大）：

```bash
tccli tke UpdateAddon --region ap-guangzhou \
    --ClusterId cls-xxxxxxxx \
    --AddonName craned \
    --AddonVersion v1.2.0 \
    --RawValues '{"nodeResourceAmplificationRatio":1.0}'
```

```json
{
    "RequestId": "f6a7b8c9-d0e1-2345-f012-345678901234"
}
```

验证恢复：

```bash
tccli tke DescribeAddon --ClusterId cls-xxxxxxxx --AddonName craned --region ap-guangzhou --output json
```

```json
{
    "AddonName": "craned",
    "AddonVersion": "v1.2.0",
    "Status": "Succeeded",
    "AddonConfig": {
        "NodeResourceAmplificationRatio": 1.0
    },
    "RequestId": "a7b8c9d0-e1f2-3456-0123-456789012345"
}
```

> **警告**：恢复默认系数后，已调度的 Pod 不会自动驱逐。如果恢复前节点实际使用率超过 100% 物理资源，恢复后新 Pod 将无法调度到该节点（因 kubectl 上报的 Allocatable 恢复正常，已用资源超过可分配资源）。建议先配合自动规整迁移部分 Pod 再恢复系数。

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 节点放大设置后未生效 | `tccli tke DescribeAddon --ClusterId cls-xxxxxxxx --AddonName craned --region ap-guangzhou` 查看配置 | 组件更新失败或版本过低 | 确认 `Status: "Succeeded"`；确认 AddonVersion >= v1.1.0 支持放大特性 |
| 放大后节点 CPU 飙升 | `kubectl top node` 查看实际 CPU 使用率 | 放大系数过高，物理 CPU 已满载，容器 CPU Throttling 严重 | 降低放大系数至 1.2-1.3；为关键服务配置 `cpuQOS.cpuPriority: 0` |
| 放大后节点频繁 OOM | `kubectl describe node \| grep "OOM"`；`kubectl get events --field-selector reason=OOMKilling` | 内存超售过度，物理内存不足 | 降低放大系数；配置 `memoryQOS.memPriority` 保护关键服务 |
| 某些 Pod 始终无法调度（放大后） | `kubectl describe pod <pod>` 查看调度事件 | 虽然 Allocatable 变大，但实际物理资源已被占满 | 降低放大系数或扩容节点 |
| UpdateAddon 返回 FailedOperation | `tccli tke DescribeAddon --ClusterId cls-xxxxxxxx --AddonName craned --region ap-guangzhou` | 组件当前状态异常或 RawValues JSON 格式错误 | 确认 RawValues 是合法 JSON 字符串；检查 AddonVersion 存在 |
| 放大与自动规整冲突 | `kubectl get events \| grep Descheduled` 频繁驱逐 | 放大后节点利用率始终高于规整阈值 | 调高规整策略 `targetThresholds` 或关闭自动规整 |
| ENI 网络模式下放大无效 | `kubectl describe node \| grep "eni"` | `tke-eni-ip-webhook` 组件版本不兼容 | 升级 `tke-eni-ip-webhook` 至最新版本 |

## 下一步

- [资源利用率优化调度概述](../tccli%20操作.md) — page_id `122378`（节点放大与自动规整的策略组合建议）
- [自动规整集群资源](../自动规整集群资源/tccli%20操作.md) — DeScheduler 安装与策略配置（释放低利用率节点）
- [Qos 感知调度](../../Qos%20感知调度/tccli%20操作.md) — page_id `79775`（节点放大的 QoS 保障前提）
  - [CPU 使用优先级](../../Qos%20感知调度/CPU%20使用优先级/tccli%20操作.md) — 保障放大后高优服务 CPU
  - [内存精细调度](../../Qos%20感知调度/内存精细调度/tccli%20操作.md) — 保障放大后内存 OOM 优先级
- [调度组件概述](../../调度组件概述/tccli%20操作.md) — page_id `111862`

## 控制台替代

登录 [容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) -> 选择集群 `cls-xxxxxxxx` -> **组件管理** -> 点击 `craned` 组件 -> **更新配置** -> 在参数配置中设置 `nodeResourceAmplificationRatio` 值 -> 确认更新。
