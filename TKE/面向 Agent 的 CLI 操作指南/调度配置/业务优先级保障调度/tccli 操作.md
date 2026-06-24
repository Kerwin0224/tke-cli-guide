# 业务优先级保障调度（tccli）

> 对照官方：[业务优先级保障调度](https://cloud.tencent.com/document/product/457/118259) · page_id `118259`

## 概述

TKE 业务优先级保障调度通过 Kubernetes 原生 PriorityClass 和 Crane 调度器扩展，实现高优先级业务优先调度、低优先级可抢占式 Job 在被抢占时自动释放资源。同时支持 Kueue（基于 Kubernetes 的批调度与队列管理系统），提供任务队列、资源配额和公平共享等高级排队调度能力，适用于在线离线混部和批处理任务调度场景。

## 前置条件

- 演示集群 `cls-xxxxxxxx`（ap-guangzhou, v1.30.0, Running），kubectl 因 CAM 策略限制不可达，需通过 VPN/IOA 内网或数据面集群操作
- 熟悉 Kubernetes PriorityClass 概念
- 已安装 [Crane 调度器](../调度组件概述/tccli%20操作.md)
- 如需使用 Kueue 批调度，需单独安装 [Kueue 组件](#kueue-批调度与队列管理)

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
# 2. 确认 Crane 调度器已安装
tccli tke DescribeAddon --ClusterId cls-xxxxxxxx --AddonName craned --region ap-guangzhou --output json
```

```json
{
    "AddonName": "craned",
    "AddonVersion": "v1.2.0",
    "Status": "Succeeded",
    "RequestId": "b2c3d4e5-f6a7-8901-bcde-f12345678901"
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 查看集群状态 | `tccli tke DescribeClusters --ClusterIds '["cls-xxxxxxxx"]' --region ap-guangzhou` | 是 |
| 查看已安装组件 | `tccli tke DescribeAddon --ClusterId cls-xxxxxxxx --AddonName craned --region ap-guangzhou` | 是 |
| 创建 PriorityClass | `kubectl apply -f priority-class.yaml` | 是 |
| 查看 PriorityClass 列表 | `kubectl get priorityclass` | 是 |
| 创建可抢占 Job（标注 `crane.io/preemptible`） | `kubectl apply -f preemptible-job.yaml` | 是 |
| 查看抢占事件 | `kubectl get events --all-namespaces --field-selector reason=Preempted` | 是 |
| 安装 Kueue 组件 | `tccli tke InstallAddon --ClusterId cls-xxxxxxxx --AddonName Kueue --region ap-guangzhou` | 是 |
| 创建 ClusterQueue | `kubectl apply -f cluster-queue.yaml` | 是 |
| 创建 LocalQueue | `kubectl apply -f local-queue.yaml` | 是 |
| 创建 ResourceFlavor | `kubectl apply -f resource-flavor.yaml` | 是 |
| 删除 Kueue 队列 | `kubectl delete clusterqueue <name> && kubectl delete localqueue <name>` | 否 |

## 操作步骤

### 调度优先级体系概览

| 能力 | 实现方式 | 适用场景 |
|------|---------|---------|
| PriorityClass 优先级 | Kubernetes 原生 PriorityClass | 高优 Pod 优先调度，资源不足时低优 Pod 被驱逐 |
| 可抢占式 Job | Crane 调度器 + `crane.io/preemptible` annotation | 低优先级批处理任务，高优 Pod 出现时自动释放资源 |
| Kueue 批调度 | Kueue 组件 (ClusterQueue/LocalQueue/ResourceFlavor) | 多租户队列管理、资源配额、公平共享 |
| 队列管理 | Kueue WorkloadPriorityClass | 按工作负载类型分配不同优先级 |

### PriorityClass 优先级

Kubernetes 原生 PriorityClass 允许为 Pod 分配调度优先级，数值越大优先级越高（范围 -2147483648 到 10 亿）。高优先级 Pod 无法调度时，调度器会尝试抢占低优先级 Pod。

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
preemptionPolicy: PreemptLowerPriority
description: "高优先级业务，优先调度和资源保障"
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: medium-priority
value: 100000
globalDefault: false
preemptionPolicy: PreemptLowerPriority
description: "中等优先级业务"
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: low-priority
value: 10
globalDefault: false
preemptionPolicy: PreemptLowerPriority
description: "低优先级可抢占批处理任务"
```

```bash
kubectl apply -f priority-class.yaml
```

在 Pod 中引用：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: critical-app
spec:
  priorityClassName: high-priority
  containers:
  - name: app
    image: nginx:latest
```

**抢占流程**：

1. 高优先级 Pod 创建后，调度器尝试寻找满足条件的目标节点
2. 若资源不足，调度器按优先级从低到高寻找可被抢占的 Pod
3. 被选中的低优先级 Pod 被优雅驱逐（`terminationGracePeriodSeconds`）
4. 高优先级 Pod 调度到释放资源的节点
5. 被驱逐的 Pod 进入 Pending 状态，等待新的调度机会

### 可抢占式 Job

通过 Crane 调度器扩展的可抢占式 Job，允许批处理任务在资源紧张时被自动驱逐，适合离线计算和不紧急的数据处理任务。详细操作见 [可抢占式 Job](../业务优先级保障调度/可抢占式%20Job/tccli%20操作.md)。

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: batch-preemptible
spec:
  template:
    metadata:
      annotations:
        crane.io/preemptible: "true"
    spec:
      priorityClassName: low-priority
      containers:
      - name: worker
        image: busybox
        command: ["sh", "-c", "sleep 600"]
      restartPolicy: Never
```

### Kueue 批调度与队列管理

Kueue 是一个 Kubernetes 原生的批调度与队列管理系统，提供多租户队列隔离、资源配额控制和公平共享调度能力。

**核心概念**：

| 概念 | 说明 | CRD |
|------|------|-----|
| ClusterQueue | 集群级队列，定义资源配额（Quota）和公平共享策略 | `ClusterQueue` |
| LocalQueue | 命名空间级队列，用户提交任务的目标队列，指向 ClusterQueue | `LocalQueue` |
| ResourceFlavor | 资源类型定义（如 spot、on-demand、GPU），与节点 label 关联 | `ResourceFlavor` |
| Workload | 对 Job、Deployment 等工作负载的抽象封装，由 Kueue 自动管理 | `Workload` |

**Kueue 工作流程**：

```
1. 管理员创建 ResourceFlavor（定义不同资源类型）
2. 管理员创建 ClusterQueue（定义每个队列的配额和资源类型）
3. 用户在命名空间中创建 LocalQueue（指向某个 ClusterQueue）
4. 用户提交 Job，指定 LocalQueue
5. Kueue 根据配额和优先级决定何时准入 Job
6. 准入后，Kueue 自动将 Pod 的 nodeSelector 注入为 ResourceFlavor 对应节点
```

**安装 Kueue**：

```bash
tccli tke InstallAddon --region ap-guangzhou \
    --ClusterId cls-xxxxxxxx \
    --AddonName Kueue
```

```json
{
    "RequestId": "e5f6a7b8-c9d0-1234-ef01-234567890123"
}
```

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
# 维度 2：确认 Crane 组件状态
tccli tke DescribeAddon --ClusterId cls-xxxxxxxx --AddonName craned --region ap-guangzhou --output json
```

```json
{
    "AddonName": "craned",
    "AddonVersion": "v1.2.0",
    "Status": "Succeeded",
    "RequestId": "d4e5f6a7-b8c9-0123-def0-123456789012"
}
```

### 数据面（需 VPN/IOA 内网环境）

```bash
# 维度 3：检查 PriorityClass
kubectl get priorityclass
```

```text
NAME                      VALUE        GLOBAL-DEFAULT   AGE
high-priority             1000000      false            10m
medium-priority           100000       false            10m
low-priority              10           false            10m
system-cluster-critical   2000000000   false            30d
system-node-critical      2000001000   false            30d
```

```bash
# 维度 4：查看抢占事件
kubectl get events --all-namespaces --field-selector reason=Preempted
```

```text
NAME  STATUS  AGE
...
```

## 清理

无需清理（功能说明页）。如已创建测试资源：

```bash
kubectl delete job batch-preemptible
kubectl delete priorityclass low-priority medium-priority high-priority
tccli tke DeleteAddon --ClusterId cls-xxxxxxxx --AddonName Kueue --region ap-guangzhou
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 高优 Pod 未抢占低优 Pod | `kubectl describe pod <high-pod>` 查看调度事件 | `preemptionPolicy` 为 `Never` | 设置 PriorityClass 的 `preemptionPolicy: PreemptLowerPriority` |
| 低优 Job 未被标记为可抢占 | `kubectl describe job <name>` 查看 annotations | 缺少 `crane.io/preemptible: "true"` annotation | 在 Job 的 `spec.template.metadata.annotations` 中添加该标注 |
| 抢占后低优 Pod 一直 Pending | `kubectl describe pod <low-pod>` | 集群资源持续紧张，无可用节点 | 扩容集群或等待高优 Pod 释放资源后自动调度 |
| Kueue Job 一直 Pending | `kubectl describe workload <name>` | 队列配额耗尽或未指定 LocalQueue | 检查 ClusterQueue 资源使用量，增加配额或等待其他 Job 完成 |
| PriorityClass 冲突 | `kubectl get events \| grep "PriorityClass"` | 同 namespace 下 PriorityClass 名称冲突 | 使用唯一名称，避免与系统 PriorityClass 重名 |

## 下一步

- [可抢占式 Job 详细操作](../业务优先级保障调度/可抢占式%20Job/tccli%20操作.md) — page_id `81751`
- [基于 Kueue 实现批调度和队列管理](../业务优先级保障调度/基于%20Kueue%20实现批调度和队列管理/tccli%20操作.md)
- [调度组件概述](../调度组件概述/tccli%20操作.md) — page_id `111862`
- [Qos 感知调度](../Qos%20感知调度/tccli%20操作.md) — page_id `79775`

## 控制台替代

登录 [容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) -> 选择集群 `cls-xxxxxxxx` -> **组件管理** -> 安装 `craned` -> 在 **工作负载** 页面创建 Job 时指定 PriorityClass -> Kueue 队列在 **调度策略** 页面管理。
