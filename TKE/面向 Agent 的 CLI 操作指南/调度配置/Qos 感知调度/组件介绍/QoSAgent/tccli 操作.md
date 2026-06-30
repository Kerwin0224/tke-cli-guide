# QoSAgent（tccli）

> 对照官方：[QoSAgent](https://cloud.tencent.com/document/product/457/79774) · page_id `79774`

## 概述

QoSAgent 是 TKE Crane 调度器套件中的节点级 QoS 代理组件，以 DaemonSet 形式部署在集群每个节点上。它负责采集节点 CPU、内存、IO、网络等资源使用数据，并执行中心调度器 craned 下发的 QoS 策略（CPU Burst、CPU 优先级、内存回收、IO 隔离、超线程隔离等）。QoSAgent 是 QoS 感知调度体系中连接控制面（craned）与数据面（节点 cgroup）的关键组件。

**本文档定位**：本文为组件介绍文档，说明 QoSAgent 的架构设计、工作原理、模块组成和部署模型。关于安装/卸载等操作步骤，请参见 [QoSAgent 操作指南](../../QoSAgent/tccli%20操作.md)。

## 前置条件

- [环境准备](../../../../环境准备.md)
- 集群中需已安装 `craned`（Crane 调度器）组件。QoSAgent 依赖 craned 下发 QoS 策略
- 演示集群 `cls-xxxxxxxx`（ap-guangzhou，v1.30.0，Running），kubectl 因 CAM 策略限制不可达，需通过 VPN/IOA 内网或数据面集群操作。以下 kubectl 命令为参考格式。

### 控制面检查

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
| 查看 QoSAgent 可用版本 | `tccli tke DescribeAddonValues --region ap-guangzhou --ClusterId cls-xxxxxxxx --AddonName QoSAgent` | 是 |

说明：安装/卸载操作参见 [QoSAgent 操作指南](../../QoSAgent/tccli%20操作.md)。

## 操作步骤

本章节说明 QoSAgent 的架构设计、模块组成和工作原理。

### 整体架构

```text
┌──────────────────────────────────────────────────────────────────┐
│                    TKE 控制面 (tccli)                              │
│   InstallAddon / DescribeAddon / DeleteAddon / UpdateAddon        │
└─────────────────────────────────────┬────────────────────────────┘
                                      │ 集群内部署
                                      v
┌──────────────────────────────────────────────────────────────────┐
│                    craned (中心调度器)                              │
│   - 监听 PodQOS/NodeQOS CRD 变更                                   │
│   - 计算节点 QoS 策略                                              │
│   - 下发策略到各节点 QoSAgent                                       │
└─────────────────────────────────────┬────────────────────────────┘
                                      │ gRPC / HTTP
                    ┌─────────────────┼─────────────────┐
                    v                 v                 v
┌──────────────────────┐ ┌──────────────────────┐ ┌──────────────────────┐
│   QoSAgent (Node-1)  │ │   QoSAgent (Node-2)  │ │   QoSAgent (Node-3)  │
│                      │ │                      │ │                      │
│  ┌────────────────┐  │ │  ┌────────────────┐  │ │  ┌────────────────┐  │
│  │ 指标采集模块    │  │ │  │ 指标采集模块    │  │ │  │ 指标采集模块    │  │
│  │ (cAdvisor/eBPF)│  │ │  │ (cAdvisor/eBPF)│  │ │  │ (cAdvisor/eBPF)│  │
│  └───────┬────────┘  │ │  └───────┬────────┘  │ │  └───────┬────────┘  │
│          v           │ │          v           │ │          v           │
│  ┌────────────────┐  │ │  ┌────────────────┐  │ │  ┌────────────────┐  │
│  │ 策略执行模块    │  │ │  │ 策略执行模块    │  │ │  │ 策略执行模块    │  │
│  │ (cgroup 操作)   │  │ │  │ (cgroup 操作)   │  │ │  │ (cgroup 操作)   │  │
│  └────────────────┘  │ │  └────────────────┘  │ │  └────────────────┘  │
└──────────────────────┘ └──────────────────────┘ └──────────────────────┘
           │                         │                         │
           v                         v                         v
    ┌──────────┐              ┌──────────┐              ┌──────────┐
    │  cgroup   │              │  cgroup   │              │  cgroup   │
    │ cpu/mem/io│              │ cpu/mem/io│              │ cpu/mem/io│
    └──────────┘              └──────────┘              └──────────┘
```

### 核心模块

#### 1. 指标采集模块

QoSAgent 通过以下数据源采集节点和 Pod 实时指标：

| 数据源 | 采集内容 | 采集频率 |
|--------|---------|:-------:|
| cAdvisor | Pod CPU 使用率、内存使用量、IO 统计 | 10s |
| eBPF | 细粒度 CPU 调度延迟、内存页回收事件 | 实时 |
| /proc 文件系统 | 节点整体 CPU/内存/IO 使用率 | 5s |
| prometheus | 自定义应用指标（可选） | 15s |

采集的指标包括但不限于：CPU 使用率、内存 RSS/Cache、页回收速率、blkio IOPS/BPS、网络收发字节数等。

#### 2. 策略同步模块

qos-agent 通过以下机制与 craned 保持策略同步：

- **启动时**：从 craned 全量拉取所有 PodQOS/NodeQOS 策略
- **运行时**：通过 Watch 机制监听 CRD 增量变更
- **离线保护**：本地缓存策略文件，craned 不可达时继续按最近策略执行
- **心跳上报**：定期向 craned 上报节点状态和执行结果（默认间隔 30s）

#### 3. 策略执行模块

策略执行模块是 QoSAgent 的核心，负责将 QoS 策略转化为 cgroup 操作：

| QoS 策略 | cgroup 子系统 | 操作 |
|---------|--------------|------|
| CPU Burst | cpu.cfs_quota_us | 动态调整 CPU 配额，允许短时突破 |
| CPU 优先级 | cpu.shares / cpuset.cpus | 调整 CPU 权重占比和核心绑定 |
| 超线程隔离 | cpuset.cpus | 将不同优先级 Pod 绑定到互斥核心组 |
| 内存优先级 | memory.oom_control | 设置容器级 OOM kill 优先级 |
| 内存回收 | memory.limit_in_bytes | 动态压缩低优 Pod 内存上限 |
| 磁盘 IO 限制 | blkio.throttle.* | 设置 IOPS/BPS 读写限制 |
| 网络带宽限制 | net_cls.classid | 配合 tc（Traffic Control）限制网络带宽 |

### 部署模型

```bash
# 需 kubectl 可达环境
kubectl get ds -n crane-system qos-agent
```

QoSAgent 以 DaemonSet 部署，关键配置：

| 属性 | 值 |
|------|-----|
| 命名空间 | `crane-system` |
| DaemonSet 名称 | `qos-agent` |
| 容器名 | `qos-agent` |
| 部署节点 | 所有 worker 节点（含新增节点自动部署） |
| 资源请求 | 约 100m CPU / 128Mi 内存 |
| 特权模式 | 是（需要访问 cgroup 文件系统和 eBPF 程序加载） |
| 宿主路径挂载 | `/sys/fs/cgroup`、`/proc`、`/var/run` |

### 生命周期

```text
InstallAddon → QoSAgent 组件注册
    │
    v
DaemonSet 创建 → Pod 调度到每个节点
    │
    v
qos-agent 启动 → 连接 craned → 拉取策略 → 开始指标采集和策略执行
    │
    v
Running → 持续监控、执行策略、心跳上报
    │
    v
DeleteAddon → DaemonSet 删除 → Pod 终止 → cgroup 限制自动解除
```

### 健康检查

QoSAgent 提供以下健康检查端点：

| 检查项 | 方式 | 路径 |
|--------|------|------|
| 存活探针 | HTTP GET | `/healthz` |
| 就绪探针 | HTTP GET | `/readyz` |
| 指标采集状态 | HTTP GET | `/metrics/status` |
| craned 连接状态 | HTTP GET | `/craned/status` |

```bash
# 需 kubectl 可达环境
kubectl exec -n crane-system <qos-agent-pod> -- curl -s http://localhost:8080/healthz
# expected: OK
```

## 验证

### 控制面

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

### 数据面（需 kubectl 可达环境）

```bash
# 确认 DaemonSet 全部节点就绪
kubectl get ds -n crane-system qos-agent
# expected: DESIRED = CURRENT = READY = UP-TO-DATE = AVAILABLE = 节点数

# 确认所有 qos-agent Pod Running
kubectl get pods -n crane-system -l app=qos-agent
# expected: 每个节点一个 Pod，STATUS: Running，READY: 1/1

# 检查日志（应包含策略加载和指标采集信息）
kubectl logs -n crane-system -l app=qos-agent --tail=20
# expected: 无 ERROR/FATAL 日志，正常包含 "policy loaded"、"metric collected"

# 确认策略文件已同步到节点
kubectl exec -n crane-system <qos-agent-pod> -- cat /etc/qos-agent/policies.json
# expected: JSON 格式的当前生效策略列表
```

```text
NAME  STATUS  AGE
...
```

## 清理

QoSAgent 的卸载通过 `DeleteAddon` 完成。详见 [QoSAgent 操作指南 - Cleanup](../../QoSAgent/tccli%20操作.md#清理)。

## 排障

### 组件状态异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| DescribeAddon `Phase: "Failed"` | `kubectl describe ds -n crane-system qos-agent` | 镜像拉取失败或节点资源不足 | 检查镜像仓库连通性；确认节点有足够资源 |
| 部分节点无 qos-agent Pod | `kubectl describe ds -n crane-system qos-agent` 查看 Events | 节点存在污点（Taint）阻止调度 | 调整节点污点或 DaemonSet tolerations |
| qos-agent Pod CrashLoopBackOff | `kubectl logs -n crane-system <pod>` 查看退出原因 | 内核版本不兼容或 cgroup 挂载异常 | 确认节点 OS 和内核版本满足要求 |
| qos-agent 与 craned 断开连接 | `kubectl logs -n crane-system <pod> \| grep craned` | craned 服务不可达 | 确认 craned Pod Running；检查网络策略 |

### 策略执行异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| QoS 策略不生效 | 检查策略是否下发：`kubectl exec -n crane-system <qos-agent-pod> -- cat /etc/qos-agent/policies.json` | 策略同步失败或 Pod 未匹配 selector | 确认 PodQOS selector 匹配 Pod label；检查 qos-agent 日志 |
| 指标采集异常 | `kubectl logs -n crane-system -l app=qos-agent \| grep metric` | cAdvisor 或 eBPF 异常 | 确认节点 cAdvisor 运行正常；检查内核 eBPF 支持 |

## 下一步

- [QoSAgent 操作指南](../../QoSAgent/tccli%20操作.md) — 完整的安装/卸载操作步骤
- [CPU 使用优先级介绍](../CPU%20使用优先级/tccli%20操作.md) — 了解 PodQOS cpuPriority 字段语义
- [CPU Burst](../../CPU%20Burst/tccli%20操作.md) — 允许容器短时间窗口内突破 CPU limits
- [内存精细调度](../../内存精细调度/tccli%20操作.md) — 通过 PodQOS 实现内存水位控制
- [Crane 调度器介绍](https://cloud.tencent.com/document/product/457/75472) — 了解完整的 Crane 调度能力

## 控制台替代

[TKE 控制台 → 集群 → 组件管理](https://console.cloud.tencent.com/tke2/cluster)，在组件列表中查看 QoSAgent 状态、版本信息和运行状况。
