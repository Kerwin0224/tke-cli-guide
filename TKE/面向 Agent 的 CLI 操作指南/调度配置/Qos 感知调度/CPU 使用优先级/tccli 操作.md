# CPU 使用优先级（tccli）

> 对照官方：[CPU 使用优先级](https://cloud.tencent.com/document/product/457/79775) · page_id `79775`

## 概述

CPU 使用优先级（CPU Usage Priority）是 QoS 感知调度中的核心特性，TKE Crane 通过 PodQOS CRD 的 `cpuQOS.cpuPriority` 字段定义 CPU 使用优先级。高优先级业务（值越小的 Pod）在 CPU 争抢时获得更多资源，低优先级业务被压制。配合 QoSAgent 实现节点级 CPU 资源隔离。

**工作原理**：
- **PodQOS 策略**：定义 CPU 优先级（0=独占、1=高、2=中、3=低）和 Pod selector
- **Label 匹配**：Pod 携带与 PodQOS selector 匹配的 label，自动受策略管控
- **QoSAgent 执行**：通过 cgroup cpu 子系统（cpu.shares、cpuset）调整各优先级 Pod 的 CPU 配额

**tccli 能力范围**：CPU 使用优先级为 CRD/kubectl 配置型特性，tccli 主要负责检查集群状态和 QoSAgent 组件状态。以下 kubectl 命令为数据面参考操作。

**组件介绍**：参见 [CPU 使用优先级组件介绍](../组件介绍/CPU%20使用优先级/tccli%20操作.md) 了解字段语义和架构原理。

## 前置条件

- [环境准备](../../../环境准备.md)
- 集群中已安装 [QoSAgent](../QoSAgent/tccli%20操作.md) 组件（依赖 craned）
- 演示集群 `cls-xxxxxxxx`（ap-guangzhou，v1.30.0，Running），kubectl 因 CAM 策略限制不可达，需通过 VPN/IOA 内网或数据面集群操作。以下 kubectl 命令为参考格式。

### 控制面检查

```bash
# 确认集群状态正常
tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]'
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
# 确认 QoSAgent 组件已安装且运行正常
tccli tke DescribeAddon --region ap-guangzhou \
    --ClusterId cls-xxxxxxxx \
    --AddonName QoSAgent
```

```json
{
    "Addons": [
        {
            "AddonName": "QoSAgent",
            "AddonVersion": "v1.2.0",
            "Status": "Succeeded",
            "Phase": "Running"
        }
    ],
    "RequestId": "b2c3d4e5-f6a7-8901-bcde-f12345678901"
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 查看集群状态 | `tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]'` | 是 |
| 查看 QoSAgent 组件状态 | `tccli tke DescribeAddon --region ap-guangzhou --ClusterId cls-xxxxxxxx --AddonName QoSAgent` | 是 |
| 创建 PodQOS 策略 | `kubectl apply -f podqos-cpu.yaml` | 是 |
| 查看 PodQOS 策略 | `kubectl get podqos -o yaml` | 是 |
| 为 Pod 标记优先级 | `kubectl label pod <name> cpu-priority=high` | 是 |
| 验证优先级生效 | `kubectl describe pod <name>` 查看 label 和 cgroup 配置 | 是 |
| 创建 CPUPriorityClass（方式二） | `kubectl apply -f cpu-priority-class.yaml` | 是 |

## 操作步骤

### 步骤 1：确认控制面状态

```bash
tccli tke DescribeAddon --region ap-guangzhou \
    --ClusterId cls-xxxxxxxx \
    --AddonName QoSAgent
```

```json
{
    "Addons": [
        {
            "AddonName": "QoSAgent",
            "AddonVersion": "v1.2.0",
            "Status": "Succeeded",
            "Phase": "Running"
        }
    ],
    "RequestId": "c3d4e5f6-a7b8-9012-cdef-123456789012"
}
```

### 步骤 2：创建 CPU 优先级 PodQOS 策略（需 kubectl 可达环境）

**方式一：PodQOS + Label（推荐）**

`podqos-cpu.yaml`：

```yaml
apiVersion: scheduling.crane.io/v1alpha1
kind: PodQOS
metadata:
  name: cpu-priority-policy
spec:
  resourceQOS:
    cpuQOS:
      cpuPriority: 0
  selector:
    matchLabels:
      cpu-priority: high
```

```bash
# 需 kubectl 可达环境
kubectl apply -f podqos-cpu.yaml
# expected: podqos.scheduling.crane.io/cpu-priority-policy created
```

**方式二：CPUPriorityClass + Annotation**

创建 CPUPriorityClass 资源，Pod 通过 `qos.crane.io/cpu-priority` 注解引用。

`cpu-priority-class.yaml`：

```yaml
apiVersion: scheduling.crane.io/v1alpha1
kind: CPUPriorityClass
metadata:
  name: high-priority
spec:
  priority: 10
  description: "核心业务最高 CPU 优先级"
---
apiVersion: scheduling.crane.io/v1alpha1
kind: CPUPriorityClass
metadata:
  name: low-priority
spec:
  priority: 2
  description: "低优业务 CPU 优先级"
```

```bash
# 需 kubectl 可达环境
kubectl apply -f cpu-priority-class.yaml
# expected:
#   cpupriorityclass.scheduling.crane.io/high-priority created
#   cpupriorityclass.scheduling.crane.io/low-priority created
```

### 步骤 3：为 Pod 指定 CPU 优先级（需 kubectl 可达环境）

**方式一：Pod 携带 Label**

```bash
# 需 kubectl 可达环境
kubectl label pod <pod-name> cpu-priority=high
# expected: pod/<pod-name> labeled
```

或创建时在 YAML 中指定：

`high-priority-pod.yaml`：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: high-priority-app
  labels:
    cpu-priority: high
spec:
  containers:
  - name: app
    image: nginx:alpine
    resources:
      requests:
        cpu: "500m"
        memory: "256Mi"
      limits:
        cpu: "1000m"
        memory: "512Mi"
```

```bash
# 需 kubectl 可达环境
kubectl apply -f high-priority-pod.yaml
# expected: pod/high-priority-app created
```

**方式二：Pod 携带 Annotation**

```bash
# 需 kubectl 可达环境
kubectl annotate pod <pod-name> qos.crane.io/cpu-priority=high-priority
# expected: pod/<pod-name> annotated
```

或创建时在 YAML 中指定：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: high-priority-app
  annotations:
    qos.crane.io/cpu-priority: high-priority
spec:
  containers:
  - name: app
    image: nginx:alpine
```

### CPU 优先级等级

| cpuPriority | 名称 | PodQOS 方式 | CPUPriorityClass 方式 | 行为 |
|:----------:|------|------------|----------------------|------|
| 0 | 独占（Exclusive） | `cpuPriority: 0` | `priority: 10` | CPU 不受低优先级 Pod 干扰，绑定独占核心 |
| 1 | 高优先级（High） | `cpuPriority: 1` | `priority: 7-9` | CPU 争抢时优先获得资源 |
| 2 | 中优先级（Medium） | `cpuPriority: 2` | `priority: 4-6` | 默认级别，CPU 充足时不受限 |
| 3 | 低优先级（Low） | `cpuPriority: 3` | `priority: 1-3` | CPU 紧张时优先被限流 |

### 工作原理

1. **策略创建**：创建 PodQOS 或 CPUPriorityClass 资源，指定优先级和 selector。
2. **Pod 匹配**：QoSAgent 扫描节点上所有 Pod，匹配 PodQOS selector 或 CPUPriorityClass 注解。
3. **cgroup 调整**：QoSAgent 通过 cgroup cpu.shares 调整各 Pod 的 CPU 权重：
   - 高优 Pod `cpu.shares` 值大 -> CFS 分配更多 CPU 时间片
   - 低优 Pod `cpu.shares` 值小 -> CFS 限制 CPU 时间片
4. **动态生效**：节点 CPU 使用率变化时，QoSAgent 实时调整配额分配。

## 验证

### 控制面

```bash
tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]'
```

```json
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "cls-xxxxxxxx",
            "ClusterName": "demo-cluster",
            "ClusterStatus": "Running",
            "ClusterVersion": "1.30.0",
            "ClusterNodeNum": 3
        }
    ],
    "RequestId": "d4e5f6a7-b8c9-0123-def0-234567890123"
}
```

```bash
tccli tke DescribeAddon --region ap-guangzhou \
    --ClusterId cls-xxxxxxxx \
    --AddonName QoSAgent
```

```json
{
    "Addons": [
        {
            "AddonName": "QoSAgent",
            "AddonVersion": "v1.2.0",
            "Status": "Succeeded",
            "Phase": "Running"
        }
    ],
    "RequestId": "e5f6a7b8-c9d0-1234-ef01-345678901234"
}
```

### 数据面（需 kubectl 可达环境）

```bash
# 确认 PodQOS 策略已创建
kubectl get podqos cpu-priority-policy -o yaml | grep cpuPriority
# expected: cpuPriority: 0

# 或确认 CPUPriorityClass 已创建
kubectl get cpupriorityclass
# expected: high-priority, low-priority

# 确认 Pod 已匹配策略
kubectl get pod -l cpu-priority=high
# expected: high-priority-app   1/1   Running

# 或确认 Pod 注解已生效
kubectl describe pod <pod-name> | grep cpu-priority
# expected: qos.crane.io/cpu-priority=high-priority

# 查看 cgroup cpu.shares 确认优先级（需进入节点）
# 高优 Pod cpu.shares 应显著高于低优 Pod
```

```text
NAME  STATUS  AGE
...
```

## 清理

> **警告**：删除 PodQOS 或 CPUPriorityClass 后，已匹配的所有 Pod 将失去优先级保护。在 CPU 资源紧张时可能被其他高优 Pod 抢占。

### 步骤 1：清理前状态检查

```bash
# 需 kubectl 可达环境
kubectl get podqos
kubectl get cpupriorityclass
kubectl get pods --all-namespaces -o json | jq '.items[] | select(.metadata.labels["cpu-priority"] != null or .metadata.annotations["qos.crane.io/cpu-priority"] != null) | .metadata.name'
# 记录所有使用 CPU 优先级的 Pod
```

```text
NAME  STATUS  AGE
...
```

### 步骤 2：移除 Pod 优先级标记

```bash
# 需 kubectl 可达环境
# 移除 label
kubectl label pod <pod-name> cpu-priority-
# expected: pod/<pod-name> unlabeled

# 或移除 annotation
kubectl annotate pod <pod-name> qos.crane.io/cpu-priority-
# expected: pod/<pod-name> annotated
```

### 步骤 3：删除优先级策略

```bash
# 需 kubectl 可达环境
# 删除 PodQOS
kubectl delete podqos cpu-priority-policy
# expected: podqos.scheduling.crane.io "cpu-priority-policy" deleted

# 或删除 CPUPriorityClass
kubectl delete cpupriorityclass high-priority low-priority
# expected: cpupriorityclass.scheduling.crane.io "high-priority" deleted
#           cpupriorityclass.scheduling.crane.io "low-priority" deleted
```

### 步骤 4：验证已清理

```bash
# 需 kubectl 可达环境
kubectl get podqos
kubectl get cpupriorityclass
# expected: No resources found 或列表中不含已删除的资源
```

```text
NAME  STATUS  AGE
...
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl apply -f podqos-cpu.yaml` 返回 "no matches for kind PodQOS" | `tccli tke DescribeAddon --region ap-guangzhou --ClusterId cls-xxxxxxxx --AddonName QoSAgent` | QoSAgent 未安装，CRD 未注册 | 先参见 [QoSAgent](../QoSAgent/tccli%20操作.md) 安装 |
| `kubectl apply -f cpu-priority-class.yaml` 返回 "no matches for kind CPUPriorityClass" | 同上 | craned 未安装，CPUPriorityClass CRD 未注册 | 确认 craned 组件已安装 |
| `kubectl label pod` 返回 "NotFound" | `kubectl get pods` 确认 Pod 存在 | Pod 名称或命名空间错误 | 检查 Pod 名称和命名空间 |

### 优先级不生效

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Pod 已标记但 CPU 未被限流 | `kubectl top node` 检查 CPU 使用率 | 节点 CPU 未达阈值（正常行为） | 使用 stress 产生 CPU 压力后重新观察 |
| PodQOS selector 未匹配 Pod | `kubectl get pod --show-labels` | Pod label 键/值与 selector 不匹配 | 修正 Pod label |
| Part 节点 qos-agent 未运行 | `kubectl get pods -n crane-system -l app=qos-agent -o wide` | 节点污点阻止调度 | 调整 DaemonSet tolerations |
| 高优 Pod 仍出现 CPU 限流 | `kubectl describe node \| grep "Allocated resources"` | 节点 CPU 资源耗尽 | 扩容节点或减少同节点 Pod 数量 |

## 下一步

- [CPU 使用优先级组件介绍](../组件介绍/CPU%20使用优先级/tccli%20操作.md) — 了解 cpuPriority 字段语义、架构原理
- [CPU Burst](../CPU%20Burst/tccli%20操作.md) — 允许容器短时突破 CPU limits
- [CPU 超线程隔离](../CPU%20超线程隔离/tccli%20操作.md) — 超线程隔离策略
- [内存精细调度](../内存精细调度/tccli%20操作.md) — 通过 PodQOS 实现内存 OOM 优先级控制
- [可抢占式 Job](../../可抢占式%20Job/tccli%20操作.md) — 基于优先级抢占的批处理任务调度
- [QoSAgent 操作指南](../QoSAgent/tccli%20操作.md) — 安装和管理 QoSAgent 组件

## 控制台替代

无独立控制台界面。CPU 使用优先级通过 CRD 配置，在集群中 [安装 QoSAgent](https://console.cloud.tencent.com/tke2/cluster) 后，通过 kubectl 管理 PodQOS 资源。
