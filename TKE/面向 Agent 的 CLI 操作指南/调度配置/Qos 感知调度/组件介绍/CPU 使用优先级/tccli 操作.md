# CPU 使用优先级（tccli）

> 对照官方：[CPU 使用优先级](https://cloud.tencent.com/document/product/457/79775) · page_id `79775`

## 概述

CPU 使用优先级（CPU Usage Priority）是 QoS 感知调度体系中 PodQOS CRD 的核心字段之一，用于为不同业务 Pod 设置 CPU 使用优先级。当节点 CPU 资源紧张时，QoSAgent 根据优先级压制低优 Pod 的 CPU 使用，保障高优业务的 CPU 供给。

**优先级语义**：通过 PodQOS `cpuQOS.cpuPriority` 字段指定，值越小优先级越高：
- `0`：独占级（Exclusive），CPU 资源保证不受干扰
- `1`：高优先级（High）
- `2`：中优先级（Medium）
- `3`：低优先级（Low），CPU 紧张时首先被限流

**工作原理**：craned 将 PodQOS 策略下发给各节点 QoSAgent。qos-agent 通过 cgroup cpu 子系统（cpu.shares、cpu.cfs_quota_us）对低优先级 Pod 施加 CPU 限制，将释放的 CPU 资源分配给高优先级 Pod。整个过程无须人工干预，由 QoSAgent 实时动态调整。

**本文档定位**：本文为组件介绍文档，说明 CPU 使用优先级的语义、配置模型和架构原理。关于完整的操作步骤（创建 CPUPriorityClass、为 Pod 添加注解等），请参见 [CPU 使用优先级操作指南](../../CPU%20使用优先级/tccli%20操作.md)。

## 前置条件

- [环境准备](../../../../环境准备.md)
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

说明：CPU 使用优先级通过 PodQOS CRD 的 `cpuPriority` 字段配置，无独立 tccli 命令。控制台无直接操作界面。

## 操作步骤

本章节说明 CPU 使用优先级的配置模型、字段语义和工作原理。

### PodQOS cpuPriority 字段说明

PodQOS CRD 中 `cpuQOS.cpuPriority` 是核心字段，定义如下：

| 字段 | 类型 | 必填 | 取值范围 | 说明 |
|------|------|:--:|------|------|
| `cpuPriority` | int | 否 | 0~3 | CPU 使用优先级。0=独占，1=高，2=中，3=低。值越小优先级越高 |

完整 PodQOS 配置示例：

```yaml
apiVersion: scheduling.crane.io/v1alpha1
kind: PodQOS
metadata:
  name: cpu-priority-policy
spec:
  resourceQOS:
    cpuQOS:
      cpuPriority: 0       # 独占级：CPU 不受低优先级 Pod 干扰
      cpuBurst: true       # 可组合 CPU Burst 特性
  selector:
    matchLabels:
      cpu-priority: high   # 匹配带有此 label 的 Pod
```

### 优先级等级与行为

| cpuPriority | 名称 | 行为描述 |
|:----------:|------|------|
| 0 | 独占（Exclusive） | CPU 资源不受低优先级 Pod 干扰。QoSAgent 通过 cgroup cpuset 将高优 Pod 绑定到独占核心，禁用超线程共享 |
| 1 | 高优先级（High） | 在 CPU 争抢时优先获得资源，可充分利用空闲 CPU |
| 2 | 中优先级（Medium） | 默认级别。CPU 充足时不受限制，CPU 紧张时被部分限流 |
| 3 | 低优先级（Low） | CPU 紧张时优先被限流。其 CPU 配额被压缩，释放资源给高优先级 Pod |

### 优先级生效机制

```text
                  craned (中心调度器)
                     │
                     │ 下发 PodQOS 策略
                     v
    ┌─────────────────────────────────┐
    │         QoSAgent (节点代理)       │
    │                                  │
    │  1. 读取 Pod label → 匹配 PodQOS  │
    │  2. 按 cpuPriority 排序 Pod 列表  │
    │  3. 监控 node CPU 使用率          │
    │  4. CPU 紧张时：                  │
    │     - 高优 Pod: 维持 cpu.shares   │
    │     - 低优 Pod: 缩减 cpu.shares   │
    │     - cgroup cpuset 隔离核心      │
    └─────────────────────────────────┘
                     │
                     │ cgroup cpu 子系统
                     v
               Linux Kernel (CFS)
```

### 与其他 QoS 特性的组合

CPU 使用优先级可与同 PodQOS 中的其他 QoS 特性组合使用：

| 组合特性 | 字段 | 效果 |
|---------|------|------|
| CPU Burst | `cpuQOS.cpuBurst` | 高优 Pod 空闲积累 credit，突发时可短时突破 Limit |
| 超线程隔离 | `cpuQOS.cpuHyperThreadIsolate` | 独占级 Pod 绑定到互斥超线程组，避免交叉干扰 |
| 内存优先级 | `memoryQOS.memPriority` | CPU 和内存优先级联合保障，全方位资源隔离 |

### 配置方式

CPU 使用优先级有两种配置方式：

**方式一：PodQOS + label**（推荐）

通过 PodQOS selector 和 Pod label 匹配实现声明式优先级管理。Pod 无须修改 spec，只需携带对应 label：

```bash
# 需 kubectl 可达环境
kubectl label pod <pod-name> cpu-priority=high
```

**方式二：CPUPriorityClass + annotation**

创建 CPUPriorityClass 资源，Pod 通过 `qos.crane.io/cpu-priority` 注解引用。详见 [CPU 使用优先级操作指南](../../CPU%20使用优先级/tccli%20操作.md)。

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
    "RequestId": "c3d4e5f6-a7b8-9012-cdef-123456789012"
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
    "RequestId": "d4e5f6a7-b8c9-0123-def0-234567890123"
}
```

### 数据面（需 kubectl 可达环境）

```bash
# 确认 CPU 优先级策略已创建
kubectl get podqos -o custom-columns=NAME:.metadata.name,PRIORITY:.spec.resourceQOS.cpuQOS.cpuPriority
# expected: cpu-priority-policy   0

# 确认 Pod 已匹配策略
kubectl get pod -l cpu-priority=high
# expected: <high-priority-pod>   1/1   Running

# 确认 cgroup cpu.shares 已按优先级调整（需进入节点）
cat /sys/fs/cgroup/cpu/kubepods.slice/.../cpu.shares
# 高优 Pod cpu.shares > 低优 Pod cpu.shares
```

```text
NAME  STATUS  AGE
...
```

## 清理

CPU 使用优先级的清理即删除对应的 PodQOS 资源。参见 [CPU 使用优先级操作指南 - Cleanup](../../CPU%20使用优先级/tccli%20操作.md#清理)。

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| PodQOS CRD 未注册 | `kubectl get crd podqos.scheduling.crane.io` | QoSAgent 未安装 | 先参见 [QoSAgent](../QoSAgent/tccli%20操作.md) 安装 |
| Pod 已标记但未被限流 | `kubectl top node` 检查 CPU 使用率 | 节点 CPU 尚未达阈值（正常行为） | 在低优 Pod 中产生 CPU 压力后观察 |
| 两种配置方式冲突 | Pod 同时有 label 和 annotation | label 优先级高于 annotation | 选择一种方式，移除另一种 |

## 下一步

- [CPU 使用优先级操作指南](../../CPU%20使用优先级/tccli%20操作.md) — 完整的操作步骤（创建策略、标记 Pod、验证效果）
- [CPU Burst](../../CPU%20Burst/tccli%20操作.md) — 允许容器短时间窗口内突破 CPU limits
- [CPU 超线程隔离](../../CPU%20超线程隔离/tccli%20操作.md) — 超线程隔离策略
- [QoSAgent 组件介绍](../QoSAgent/tccli%20操作.md) — 了解 QoSAgent 架构与工作原理

## 控制台替代

无独立控制台界面。CPU 使用优先级通过 CRD 配置，在集群中 [安装 QoSAgent](https://console.cloud.tencent.com/tke2/cluster) 后，通过 kubectl 管理 PodQOS 资源。
