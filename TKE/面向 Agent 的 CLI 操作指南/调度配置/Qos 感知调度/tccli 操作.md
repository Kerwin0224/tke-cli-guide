# Qos 感知调度（tccli）

> 对照官方：[Qos 感知调度](https://cloud.tencent.com/document/product/457/79775) · page_id `79775`

## 概述

QoS 感知调度是 TKE 基于 Crane 调度器套件实现的调度增强能力，通过 QoSAgent 采集节点 QoS 指标（CPU 限流率、内存压力、IO 延迟、网络延迟等），在调度决策时自动过滤 QoS 劣化节点，优先将 Pod 调度到 QoS 健康的节点。覆盖 CPU 使用优先级、CPU Burst、CPU 超线程隔离、内存精细调度、磁盘 IO 精细调度、网络精细调度等子功能。

## 前置条件

- 演示集群 `cls-xxxxxxxx`（ap-guangzhou, v1.30.0, Running），kubectl 因 CAM 策略限制不可达，需通过 VPN/IOA 内网或数据面集群操作
- 已安装 [Crane 调度器](../调度组件概述/tccli%20操作.md)（含 QoSAgent）
- 节点内核版本 >= 3.10（CFS Burst 等特性依赖内核版本）

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

```bash
# 3. 确认 QoSAgent DaemonSet 运行（需 VPN/IOA 内网）
kubectl get ds -n crane-system qos-agent
```

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 查看集群状态 | `tccli tke DescribeClusters --ClusterIds '["cls-xxxxxxxx"]' --region ap-guangzhou` | 是 |
| 查看已安装组件 | `tccli tke DescribeAddon --ClusterId cls-xxxxxxxx --AddonName craned --region ap-guangzhou` | 是 |
| CPU 使用优先级配置 | `kubectl apply -f` (PodQOS `cpuQOS.cpuPriority`) | 是 |
| CPU Burst 启用 | `kubectl apply -f` (PodQOS `cpuQOS.cpuBurst: true`) | 是 |
| CPU 超线程隔离 | `kubectl apply -f` (PodQOS `cpuQOS.cpuHTIsolation`) | 是 |
| 内存精细调度（水位控制） | `kubectl apply -f` (NodeQOS `memoryQOS.memcgWaterMark`) | 是 |
| 内存精细调度（页面回收） | `kubectl apply -f` (NodeQOS `memoryQOS.memPageCacheLimit`) | 是 |
| 内存 OOM 优先级 | `kubectl apply -f` (PodQOS `memoryQOS.memPriority`) | 是 |
| 磁盘 IO 精细调度 | `kubectl apply -f` (PodQOS `diskIOQOS` 配置) | 是 |
| 网络精细调度 | `kubectl apply -f` (PodQOS `netIOQOS` 配置) | 是 |
| 查看所有 QoS 策略 | `kubectl get podqos,nodeqos` | 是 |
| 删除 QoS 策略 | `kubectl delete podqos <name> && kubectl delete nodeqos <name>` | 否 |

## 操作步骤

### QoS 指标采集

QoSAgent 以 DaemonSet 形态运行在每个节点上，持续采集以下 QoS 指标：

| 指标 | 采集方式 | 影响 | 调度决策 |
|------|---------|------|---------|
| CPU 限流率 | cgroup cpu.stat 中的 nr_throttled | 限流严重时影响响应延迟 | 避免调度到 CPU 限流率高的节点 |
| 内存压力 | /proc/meminfo（MemAvailable、PageCache） | 高内存压力可能导致 OOM | 避免调度到内存水位超标的节点 |
| IO 延迟 | blkio cgroup 和 disk stats | 影响 IO 密集型应用吞吐 | 避免调度到 IO 拥堵的节点 |
| 网络延迟 | netns 统计和 eBPF 采集 | 影响网络密集型应用 RTT | 避免调度到网络带宽瓶颈的节点 |

### 工作原理

```
1. QoSAgent (DaemonSet) 在每个节点持续采集 QoS 指标
2. Crane Scheduler 在调度 Pod 时查询各节点 QoS 状态
3. 过滤 QoS 劣化节点（指标超过阈值）
4. 在剩余健康节点中按常规调度策略打分和优选
5. 支持热迁移：节点 QoS 恶化后，可触发 Pod 重新调度
```

### 子功能能力矩阵

| 子功能 | 实现方式 | CRD/配置 | 典型场景 |
|--------|---------|----------|---------|
| **CPU 使用优先级** | PodQOS `cpuQOS.cpuPriority` | PodQOS CRD | 在线服务高优先级，离线任务低优先级，CPU 争抢时高优优先 |
| **CPU Burst** | PodQOS `cpuQOS.cpuBurst: true` | PodQOS CRD | Web 服务突发流量，空闲积累 CPU credit，突发时短时超越 Limit |
| **CPU 超线程隔离** | PodQOS `cpuQOS.cpuHTIsolation` | PodQOS CRD | 延迟敏感应用独占物理核，避免 SMT 资源共享干扰 |
| **内存精细调度** | NodeQOS `memoryQOS.*` + PodQOS `memoryQOS.memPriority` | NodeQOS、PodQOS CRD | 内存超售场景下的水位控制和 OOM 优先级 |
| **磁盘 IO 精细调度** | PodQOS `diskIOQOS` | PodQOS CRD | 数据库等高 IO 负载，设置 IO 权重和带宽限制 |
| **网络精细调度** | PodQOS `netIOQOS` | PodQOS CRD | 流媒体、大数据传输，设置网络带宽上限 |

### QoS 策略生效流程

```yaml
# 步骤 1：创建 PodQOS 策略（定义 QoS 规则）
apiVersion: scheduling.crane.io/v1alpha1
kind: PodQOS
metadata:
  name: high-qos
spec:
  resourceQOS:
    cpuQOS:
      cpuPriority: 0            # CPU 最高优先级
      cpuBurst: true            # 允许 CPU Burst
    memoryQOS:
      memPriority: 0            # 内存最高优先级（OOM 最后被 Kill）
  selector:
    matchLabels:
      qos-tier: high
---
# 步骤 2：创建 NodeQOS 策略（定义节点级 QoS 阈值）
apiVersion: scheduling.crane.io/v1alpha1
kind: NodeQOS
metadata:
  name: qos-thresholds
spec:
  nodeQualityProbe:
    rules:
    - name: cpu-throttling
      nodeLocalGet:
        localCacheTTL: 30s
    - name: memory-pressure
      nodeLocalGet:
        localCacheTTL: 30s
  resourceQOS:
    memoryQOS:
      memcgWaterMark:
        enable: true
        lowPercent: 80
        highPercent: 90
    cpuQOS:
      cpuBurstConfig:
        enable: true
```

```bash
kubectl apply -f podqos-high.yaml
kubectl apply -f nodeqos-thresholds.yaml
```

步骤 3：Pod 通过 label 匹配策略：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: critical-service
  labels:
    qos-tier: high
spec:
  containers:
  - name: app
    image: nginx:latest
```

### CPU 优先级映射表

| cpuPriority | 内核 Nice 值 | 行为 | 适用负载 |
|------------|-------------|------|---------|
| 0 | -20 | 最高优先级，CPU 争抢时优先分配时间片 | 核心在线服务（API、数据库） |
| 1 | -10 | 次高优先级 | 重要离线任务 |
| 2 | 0 | 默认优先级 | 一般批处理任务 |
| 3 | 10 | 低优先级 | 可抢占的后台任务 |

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
# 维度 3：确认 QoSAgent 运行
kubectl get pods -n crane-system -l app=qos-agent
```

```text
NAME              READY   STATUS    RESTARTS   AGE
qos-agent-xxxxx   1/1     Running   0          30s
qos-agent-yyyyy   1/1     Running   0          30s
qos-agent-zzzzz   1/1     Running   0          30s
```

```bash
# 维度 4：确认 QoS 策略已创建
kubectl get podqos
kubectl get nodeqos
```

```text
NAME       AGE
high-qos   5m

NAME             AGE
qos-thresholds   5m
```

```bash
# 维度 5：查看 QoSAgent 采集日志（确认指标采集正常）
kubectl logs -n crane-system -l app=qos-agent --tail=10
```

```text
...log output...
```

| 维度 | 命令 | 预期 |
|------|------|------|
| 集群状态 | `DescribeClusters --ClusterIds '["cls-xxxxxxxx"]'` | `ClusterStatus: "Running"` |
| 组件状态 | `DescribeAddon --ClusterId cls-xxxxxxxx --AddonName craned` | `Status: "Succeeded"` |
| QoSAgent Pod | `kubectl get pods -n crane-system -l app=qos-agent` | 全部 Running |
| 策略就绪 | `kubectl get podqos,nodeqos` | 策略已创建 |

## 清理

无需清理（概述说明页）。如已创建测试策略：

```bash
kubectl delete podqos high-qos
kubectl delete nodeqos qos-thresholds
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| QoSAgent 未上报指标 | `kubectl logs -n crane-system -l app=qos-agent --tail=50` 搜索 "error" | 节点 cgroup 挂载异常或内核版本过低 | 确认内核 >= 3.10；`mount \| grep cgroup` 确认 cgroup v1 已启用 |
| CPU Burst 不生效 | `kubectl describe pod <name> \| grep "cpuBurst"` | 节点内核版本 < 4.18（CFS Burst 特性依赖） | 升级节点内核至 >= 4.18 |
| CPU 优先级未生效 | `cat /sys/fs/cgroup/cpu/<container>/cpu.shares` | PodQOS CRD 未创建或 selector 不匹配 | 确认 Pod 的 labels 与 PodQOS 的 `selector.matchLabels` 一致 |
| 内存 OOM 优先级顺序错误 | `cat /sys/fs/cgroup/memory/<container>/memory.oom_control` | memPriority 配置错误或内核 OOM killer 策略覆盖 | 确认 PodQOS `memoryQOS.memPriority` 值正确（0=最高，3=最低） |
| QoS 劣化节点未被过滤 | `kubectl get pod -o wide \| grep <node-name>` 检查 Pod 仍在劣化节点 | 调度策略未开启 QoS 感知过滤 | 确认 QoSAgent 健康；检查 `crane-scheduler` 配置中 `QoSFilter` 已启用 |
| 磁盘 IO 策略未生效 | `kubectl logs -n crane-system -l app=qos-agent \| grep "blkio"` | blkio cgroup 配置失败 | 确认节点内核支持 blkio cgroup v1；`ls /sys/fs/cgroup/blkio/` |

## 下一步

各 QoS 子功能的详细操作指南：

- **CPU 相关**：
  - [CPU 使用优先级](CPU%20使用优先级/tccli%20操作.md)
  - [CPU Burst](CPU%20Burst/tccli%20操作.md) — page_id `79776`
  - [CPU 超线程隔离](CPU%20超线程隔离/tccli%20操作.md)
  - [应用启动时 CPU 突增](应用启动时%20CPU%20突增/tccli%20操作.md)
- **内存相关**：
  - [内存精细调度](内存精细调度/tccli%20操作.md) — page_id `79778`
- **IO 相关**：
  - [磁盘 IO 精细调度](磁盘%20IO%20精细调度/tccli%20操作.md)
  - [网络精细调度](网络精细调度/tccli%20操作.md)
- **基础组件**：
  - [QoSAgent 安装与配置](QoSAgent/tccli%20操作.md) — page_id `79774`
  - [调度组件概述](../调度组件概述/tccli%20操作.md) — page_id `111862`

## 控制台替代

登录 [容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) -> 选择集群 `cls-xxxxxxxx` -> **组件管理** -> 安装 `craned`（含 QoSAgent） -> 各 QoS 子功能通过 **调度策略** 和 **工作负载** 页面配置。
