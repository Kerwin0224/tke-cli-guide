# CPU 超线程隔离（tccli）

> 对照官方：[CPU 超线程隔离](https://cloud.tencent.com/document/product/457/79777) · page_id `79777`

## 概述

CPU 超线程隔离（CPU Hyper-Threading Isolation）是 QoS 感知调度中的一项特性，将 CPU 核心的交错超线程绑定到不同优先级业务，避免低优先级业务占用高优先级业务的超线程资源。TKE 通过 Crane PodQOS CRD 的 `cpuQOS.cpuHyperThreadIsolate` 字段配置。

**工作原理**：物理 CPU 核心通过超线程（HT）技术呈现为 2 个逻辑 CPU。默认情况下，高/低优先级 Pod 可能共享同一物理核心的超线程对，导致资源竞争。启用超线程隔离后，QoSAgent 将不同优先级的 Pod 绑定到不同的超线程组，确保高优业务独占物理核心资源。

**tccli 能力范围**：CPU 超线程隔离为 CRD/kubectl 配置型特性，tccli 主要负责检查集群状态和 QoSAgent 组件状态。以下 kubectl 命令为数据面参考操作。

## 前置条件

- [环境准备](../../../环境准备.md)
- 集群中已安装 [QoSAgent](../QoSAgent/tccli%20操作.md) 组件（依赖 craned）
- 节点 CPU 需支持超线程（HT）且已启用
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
| 启用超线程隔离（CRD） | `kubectl apply -f podqos-ht.yaml`（PodQOS `cpuQOS.cpuHyperThreadIsolate: true`） | 是 |
| 验证策略（数据面） | `kubectl describe podqos` | 是 |
| 为 Pod 指定优先级 label | `kubectl label pod`（匹配 PodQOS selector） | 是 |

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

### 步骤 2：创建超线程隔离策略（需 kubectl 可达环境）

超线程隔离策略通过 PodQOS CRD 定义。`cpuHyperThreadIsolate: true` 启用超线程隔离。

`podqos-ht.yaml`：

```yaml
apiVersion: scheduling.crane.io/v1alpha1
kind: PodQOS
metadata:
  name: ht-isolate-policy
spec:
  resourceQOS:
    cpuQOS:
      cpuHyperThreadIsolate: true
  selector:
    matchLabels:
      ht-isolate: enabled
```

```bash
# 需 kubectl 可达环境
kubectl apply -f podqos-ht.yaml
# expected: podqos.scheduling.crane.io/ht-isolate-policy created
```

### 步骤 3：标记 Pod 应用策略（需 kubectl 可达环境）

为需要超线程隔离的 Pod 添加与 PodQOS selector 匹配的 label。

```bash
# 需 kubectl 可达环境
kubectl label pod <pod-name> ht-isolate=enabled
# expected: pod/<pod-name> labeled
```

也可以在 Pod YAML 中创建时直接指定 label：

`ht-pod.yaml`：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ht-isolated-app
  labels:
    ht-isolate: enabled
  annotations:
    qos.crane.io/cpu-priority: high
spec:
  containers:
  - name: app
    image: nginx:alpine
    resources:
      requests:
        cpu: 500m
      limits:
        cpu: 1000m
```

```bash
# 需 kubectl 可达环境
kubectl apply -f ht-pod.yaml
# expected: pod/ht-isolated-app created
```

### 工作原理

1. **策略下发**：craned 检测到 PodQOS 资源变更后，将超线程隔离策略下发给各节点 QoSAgent。
2. **CPU 分组**：QoSAgent 根据 `cpuPriority` 注解将高优先级和低优先级 Pod 分配到不同的超线程组。
3. **cgroup 绑定**：通过 cgroup cpuset 将高优先级 Pod 绑定到独占的物理核心（关闭超线程共享），低优先级 Pod 使用剩余核心。
4. **动态调整**：当 Pod 的优先级变化或 Pod 增加/删除时，QoSAgent 实时更新 cgroup 绑定。

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

### 数据面（需 kubectl 可达环境）

```bash
# 确认 PodQOS 策略已创建
kubectl get podqos ht-isolate-policy -o yaml | grep cpuHyperThreadIsolate
# expected: cpuHyperThreadIsolate: true

# 确认 Pod 已匹配策略
kubectl get pod -l ht-isolate=enabled
# expected: ht-isolated-app   1/1   Running

# 在节点上确认 cgroup cpuset 绑定（需 ssh 到节点）
cat /sys/fs/cgroup/cpuset/kubepods.slice/.../cpuset.cpus
# 高优 Pod 的 cpuset 应为互斥核心（如 0,2,4,6），不包含超线程对
```

```text
NAME  STATUS  AGE
...
```

## 清理

### 数据面（需 kubectl 可达环境）

```bash
# 删除测试 Pod
kubectl delete pod ht-isolated-app
# expected: pod "ht-isolated-app" deleted

# 删除超线程隔离策略
kubectl delete podqos ht-isolate-policy
# expected: podqos.scheduling.crane.io "ht-isolate-policy" deleted
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl apply -f podqos-ht.yaml` 返回 "no matches for kind PodQOS" | `tccli tke DescribeAddon --region ap-guangzhou --ClusterId cls-xxxxxxxx --AddonName QoSAgent` 确认组件状态 | QoSAgent 组件未安装，PodQOS CRD 未注册 | 先参见 [QoSAgent](../QoSAgent/tccli%20操作.md) 安装组件 |

### 超线程隔离不生效

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 策略已创建但高优 Pod CPU 仍受干扰 | 节点 `lscpu \| grep Thread` 确认超线程启用 | 节点 BIOS 未启用 HT | 启用节点 BIOS 超线程 |
| 部分节点 qos-agent 未运行 | `kubectl get pods -n crane-system -l app=qos-agent -o wide` 确认 DaemonSet 覆盖 | 节点存在污点（Taint）阻止调度 | 调整节点污点或 DaemonSet tolerations |
| PodQOS selector 未匹配到 Pod | `kubectl get pod --show-labels` 对比 PodQOS `matchLabels` | Pod label 键/值与 selector 不匹配 | 修正 Pod label 使其与 selector 一致 |

## 下一步

- [CPU 使用优先级](../CPU%20使用优先级/tccli%20操作.md) — 通过 PodQOS 设置 CPU 优先级
- [内存精细调度](../内存精细调度/tccli%20操作.md) — 通过 PodQOS 实现内存水位控制
- [QoSAgent 组件介绍](../组件介绍/QoSAgent/tccli%20操作.md) — 了解 QoSAgent 架构与工作原理

## 控制台替代

无独立控制台界面。CPU 超线程隔离通过 CRD 配置，在集群中 [安装 QoSAgent](https://console.cloud.tencent.com/tke2/cluster) 后，通过 kubectl 管理 PodQOS 资源。
