# 内存精细调度（tccli）

> 对照官方：[内存精细调度](https://cloud.tencent.com/document/product/457/79778) · page_id `79778`

## 概述

内存精细调度（Memory Fine-Grained Scheduling）是 QoS 感知调度中的一项特性，TKE 通过 Crane PodQOS 和 NodeQOS CRD 实现 Pod/节点级内存精细控制。支持 Memcg OOM 优先级、内存水位控制、冷热内存页面回收等策略，保障高优业务内存供给。

**工作原理**：
- **NodeQOS**：节点级策略，定义内存可用水位阈值。当节点内存低于阈值时，触发 QoS 干预。
- **PodQOS**：Pod 级策略，通过 `memoryQOS.memPriority` 指定 OOM 优先级。`0` 为最高优先级（最后被 OOM kill），值越大优先级越低（优先被 kill）。
- **QoSAgent 执行**：qos-agent 采集节点内存指标，当触发阈值时，按优先级回收低优 Pod 内存或触发 OOM kill。

**tccli 能力范围**：内存精细调度为 CRD/kubectl 配置型特性，tccli 主要负责检查集群状态和 QoSAgent 组件状态。以下 kubectl 命令为数据面参考操作。

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
| 创建节点级内存策略 | `kubectl apply -f nodeqos-mem.yaml`（NodeQOS CRD） | 是 |
| 创建 Pod 级内存策略 | `kubectl apply -f podqos-mem.yaml`（PodQOS CRD） | 是 |
| 查看所有 QoS 策略 | `kubectl get podqos,nodeqos` | 是 |
| 验证 OOM 优先级 | 触发内存压力后观察 Pod OOM 顺序 | 是 |

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

### 步骤 2：创建 NodeQOS 节点级内存策略（需 kubectl 可达环境）

NodeQOS 定义节点内存可用水位探针规则和 Memcg OOM 策略。

`nodeqos-mem.yaml`：

```yaml
apiVersion: scheduling.crane.io/v1alpha1
kind: NodeQOS
metadata:
  name: mem-control
spec:
  nodeQualityProbe:
    rules:
    - name: memory-available
      nodeLocalGet:
        localCacheTTL: 60s
  resourceQOS:
    memoryQOS:
      memcgOOM:
        enable: true
        priority: 0
```

```bash
# 需 kubectl 可达环境
kubectl apply -f nodeqos-mem.yaml
# expected: nodeqos.scheduling.crane.io/mem-control created
```

### 步骤 3：创建 PodQOS Pod 级内存策略（需 kubectl 可达环境）

PodQOS 通过 `memoryQOS.memPriority` 为不同业务 Pod 设置 OOM 优先级。

`podqos-mem.yaml`：

```yaml
apiVersion: scheduling.crane.io/v1alpha1
kind: PodQOS
metadata:
  name: mem-high-priority
spec:
  resourceQOS:
    memoryQOS:
      memPriority: 0
  selector:
    matchLabels:
      mem-priority: high
---
apiVersion: scheduling.crane.io/v1alpha1
kind: PodQOS
metadata:
  name: mem-low-priority
spec:
  resourceQOS:
    memoryQOS:
      memPriority: 3
  selector:
    matchLabels:
      mem-priority: low
```

```bash
# 需 kubectl 可达环境
kubectl apply -f podqos-mem.yaml
# expected:
#   podqos.scheduling.crane.io/mem-high-priority created
#   podqos.scheduling.crane.io/mem-low-priority created
```

### 步骤 4：创建测试 Pod 验证优先级（需 kubectl 可达环境）

`mem-priority-pods.yaml`：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: high-priority-app
  labels:
    mem-priority: high
spec:
  containers:
  - name: app
    image: nginx:alpine
    resources:
      requests:
        memory: "128Mi"
      limits:
        memory: "256Mi"
---
apiVersion: v1
kind: Pod
metadata:
  name: low-priority-app
  labels:
    mem-priority: low
spec:
  containers:
  - name: app
    image: nginx:alpine
    resources:
      requests:
        memory: "128Mi"
      limits:
        memory: "256Mi"
```

```bash
# 需 kubectl 可达环境
kubectl apply -f mem-priority-pods.yaml
# expected:
#   pod/high-priority-app created
#   pod/low-priority-app created
```

### 内存 OOM 优先级等级

| memPriority 值 | 行为 |
|:-------------:|------|
| 0 | 最高优先级，OOM 时最后被 kill。适用于核心业务 Pod |
| 1 | 高优先级 |
| 2 | 中优先级 |
| 3 | 最低优先级，内存压力时首先被 OOM kill。适用于低优/批处理 Pod |

### 工作原理

1. **NodeQOS 定义阈值**：`nodeQualityProbe` 定义内存可用水位探针规则，craned 据此生成节点内存可用曲线。
2. **策略下发**：craned 将 PodQOS/NodeQOS 策略下发给各节点 qos-agent。
3. **QoSAgent 监控**：qos-agent 持续采集节点内存可用量，对比 NodeQOS 阈值。
4. **触发干预**：当节点内存低于阈值时，qos-agent 按 memPriority 从低到高对 Pod 执行内存回收或 OOM kill。
5. **Memcg OOM**：NodeQOS `memcgOOM.enable: true` 启用 cgroup 级别的 OOM 控制，可按优先级精确 kill Pod 内特定容器。

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
kubectl get podqos
# expected: mem-high-priority, mem-low-priority

# 确认 NodeQOS 策略已创建
kubectl get nodeqos
# expected: mem-control

# 确认 Pod 已匹配对应策略
kubectl get pod -l mem-priority=high
# expected: high-priority-app   1/1   Running

kubectl get pod -l mem-priority=low
# expected: low-priority-app    1/1   Running

# 查看 PodQOS 策略详情
kubectl get podqos mem-high-priority -o yaml | grep memPriority
# expected: memPriority: 0
```

```text
NAME  STATUS  AGE
...
```

## 清理

### 数据面（需 kubectl 可达环境）

```bash
# 删除测试 Pod
kubectl delete pod high-priority-app low-priority-app
# expected: pod "high-priority-app" deleted / pod "low-priority-app" deleted

# 删除 PodQOS 策略
kubectl delete podqos mem-high-priority mem-low-priority
# expected: podqos.scheduling.crane.io "mem-high-priority" deleted
#           podqos.scheduling.crane.io "mem-low-priority" deleted

# 删除 NodeQOS 策略
kubectl delete nodeqos mem-control
# expected: nodeqos.scheduling.crane.io "mem-control" deleted
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl apply -f nodeqos-mem.yaml` 返回 "no matches for kind NodeQOS" | `tccli tke DescribeAddon --region ap-guangzhou --ClusterId cls-xxxxxxxx --AddonName QoSAgent` | QoSAgent 未安装，NodeQOS CRD 未注册 | 先参见 [QoSAgent](../QoSAgent/tccli%20操作.md) 安装组件 |
| `kubectl apply -f podqos-mem.yaml` 返回 "no matches for kind PodQOS" | 同上 | QoSAgent 未安装，PodQOS CRD 未注册 | 同上 |

### 内存优先级不生效

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 低优 Pod 未被优先 OOM kill | 检查节点内存是否达到阈值；`kubectl top node` | 节点内存尚未达到触发 QoS 干预的阈值（此为正常行为） | 在低优 Pod 中产生内存压力后重新观察 |
| PodQOS selector 未匹配到 Pod | `kubectl get pod --show-labels` 对比 PodQOS `matchLabels` | Pod label 与 selector 不匹配 | 修正 Pod label |
| NodeQOS 策略未在所有节点生效 | `kubectl get pods -n crane-system -l app=qos-agent -o wide` | 部分节点 qos-agent 未运行 | 排查节点污点和 DaemonSet 调度状态 |

## 下一步

- [磁盘 IO 精细调度](../磁盘%20IO%20精细调度/tccli%20操作.md) — 通过 PodQOS 实现磁盘 IOPS/BPS 限制
- [网络精细调度](../网络精细调度/tccli%20操作.md) — 通过 CNRP CRD 实现网络带宽限制
- [CPU 使用优先级](../CPU%20使用优先级/tccli%20操作.md) — 通过 PodQOS 设置 CPU 使用优先级
- [QoSAgent 组件介绍](../组件介绍/QoSAgent/tccli%20操作.md) — 了解 QoSAgent 架构与工作原理

## 控制台替代

无独立控制台界面。内存精细调度通过 CRD 配置，在集群中 [安装 QoSAgent](https://console.cloud.tencent.com/tke2/cluster) 后，通过 kubectl 管理 PodQOS/NodeQOS 资源。
