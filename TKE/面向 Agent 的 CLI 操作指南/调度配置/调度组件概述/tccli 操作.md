# 调度组件概述（tccli）

> 对照官方：[调度组件概述](https://cloud.tencent.com/document/product/457/111862) · page_id `111862`

## 概述

TKE 提供原生节点调度增强能力，通过一系列调度组件实现精细化资源管理和调度策略优化。核心组件包括 Crane 调度器（基于真实负载的智能调度）、Dynamic Scheduler（动态调度决策）、DeScheduler（重调度自动规整）、HPC（定时水平伸缩）和 QoSAgent（QoS 感知代理）。这些组件覆盖资源利用率优化、QoS 保障、业务优先级和高性能计算等场景。

## 前置条件

演示集群信息：**cls-xxxxxxxx**（ap-guangzhou，v1.30.0，Running，3 节点）。注意：kubectl 因 CAM 策略限制不可达，需通过 VPN/IOA 内网或数据面集群操作。以下 kubectl 命令为参考格式。

- 熟悉 Kubernetes 调度概念（Scheduler、Node、Pod 亲和性/反亲和性、Taint/Toleration 等）
- 已了解 TKE 集群原生节点调度能力

### 环境检查

```bash
# 1. 检查 tccli 版本和凭据
tccli --version
# expected: tccli version >= 1.0.0

tccli configure list
# expected: secretId, secretKey, region 均已配置

# 2. 检查 CAM 权限 — 需要以下 Action 名
#    tke:DescribeClusters, tke:DescribeAddon, tke:InstallAddon
tccli tke DescribeClusters --region ap-guangzhou
# expected: exit 0，返回集群列表
```

**预期输出**：

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

### 资源检查

```bash
# 3. 确认目标集群存在且状态正常
tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]'
# expected: TotalCount >= 1，ClusterStatus: "Running"
```

**预期输出**：

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
# 4. 查看已安装的调度组件
tccli tke DescribeAddon --region ap-guangzhou \
    --ClusterId cls-xxxxxxxx \
    --AddonName craned
# expected: 查看 Crane 调度器安装状态
```

**预期输出**（已安装时）：

```json
{
    "Addons": [
        {
            "AddonName": "craned",
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
| 查看集群状态 | `tccli tke DescribeClusters --ClusterIds '["cls-xxxxxxxx"]' --region ap-guangzhou` | 是 |
| 查看已安装组件列表 | `tccli tke DescribeAddon --ClusterId cls-xxxxxxxx --region ap-guangzhou` | 是 |
| 安装 Crane 调度器（含 QoSAgent） | `tccli tke InstallAddon --ClusterId cls-xxxxxxxx --AddonName craned --region ap-guangzhou` | 是 |
| 安装 Dynamic Scheduler | `tccli tke InstallAddon --ClusterId cls-xxxxxxxx --AddonName DynamicScheduler --region ap-guangzhou` | 是 |
| 安装 DeScheduler | `tccli tke InstallAddon --ClusterId cls-xxxxxxxx --AddonName DeScheduler --region ap-guangzhou` | 是 |
| 安装 HPC | `tccli tke InstallAddon --ClusterId cls-xxxxxxxx --AddonName HPC --region ap-guangzhou` | 是 |
| 升级组件版本 | `tccli tke UpdateAddon --ClusterId cls-xxxxxxxx --AddonName <name> --AddonVersion <version> --region ap-guangzhou` | 否 |
| 卸载组件 | `tccli tke DeleteAddon --ClusterId cls-xxxxxxxx --AddonName <name> --region ap-guangzhou` | 否 |
| 查看组件 DaemonSet/Deployment | `kubectl get ds,deploy -n crane-system`（需 kubectl 可达） | 是 |
| 查看调度策略 CRD | `kubectl get podqos,nodeqos,deschedulerpolicy`（需 kubectl 可达） | 是 |

## 操作步骤

### 步骤 1：理解调度组件体系

TKE 调度增强组件按功能划分为调度器、辅助组件和运维组件三类：

| 组件 | 类型 | 功能定位 | 运行形态 | AddonName |
|------|------|---------|---------|-----------|
| **Crane 调度器** | 调度器 | 基于真实负载的智能调度，避免资源热点，支持 NUMA 感知 | Deployment | `craned` |
| **Dynamic Scheduler** | 调度器 | 动态调度决策，支持自定义调度策略和节点打分 | Deployment | `DynamicScheduler` |
| **DeScheduler** | 运维组件 | 重调度自动规整，将 Pod 从低效/碎片化节点驱逐重新调度 | Deployment | `DeScheduler` |
| **HPC** | 运维组件 | 定时水平伸缩（HorizontalPodCron），基于时间策略的弹性伸缩 | Deployment | `HPC` |
| **QoSAgent** | 辅助组件 | QoS 感知数据采集与策略执行代理 | DaemonSet | 随 `craned` 安装 |

### 步骤 2：调度能力总览矩阵

| 能力域 | 涉及组件 | 下游子功能 | 典型场景 |
|--------|---------|-----------|---------|
| **资源利用率优化** | Crane, DeScheduler | 节点放大、自动规整、CPU/内存精细调度 | 提高装箱率，降低资源碎片 |
| **QoS 保障** | Crane, QoSAgent | CPU 使用优先级、CPU Burst、CPU 超线程隔离、内存精细调度、磁盘 IO 精细调度、网络精细调度 | 在线离线混部，保障核心服务稳定性 |
| **业务优先级** | Crane, Kueue | PriorityClass、可抢占式 Job、批调度与队列管理 | 高优业务优先调度，低优批任务在空闲时执行 |
| **高性能计算** | Crane, HPC | 磁盘 IO 精细调度、网络精细调度、定时弹性伸缩 | AI 训练、大数据计算 |

### 步骤 3：了解组件间关系

```
Crane 调度器 (craned)
├── Crane Scheduler (Deployment)  ← 智能调度决策
├── QoSAgent (DaemonSet)          ← QoS 数据采集与策略执行
│   ├── CPU 使用优先级
│   ├── CPU Burst
│   ├── CPU 超线程隔离
│   ├── 内存精细调度 (NodeQOS + PodQOS)
│   ├── 磁盘 IO 精细调度
│   └── 网络精细调度
├── DeScheduler (独立组件)         ← 重调度自动规整
├── Dynamic Scheduler (独立组件)   ← 动态调度策略
├── HPC (独立组件)                 ← 定时水平伸缩
└── Kueue (独立组件)               ← 批调度与队列管理
```

### 步骤 4：安装 Crane 调度器（调度增强入口）

Crane 调度器是 TKE 调度增强的核心组件，安装后自动部署 Crane Scheduler 和 QoSAgent。

```bash
tccli tke InstallAddon --region ap-guangzhou \
    --ClusterId cls-xxxxxxxx \
    --AddonName craned
# expected: exit 0，返回 RequestId
```

**预期输出**：

```json
{
    "RequestId": "c3d4e5f6-a7b8-9012-cdef-123456789012"
}
```

安装后确认：

```bash
tccli tke DescribeAddon --region ap-guangzhou \
    --ClusterId cls-xxxxxxxx \
    --AddonName craned
# expected: Status: "Succeeded"
```

**预期输出**：

```json
{
    "Addons": [
        {
            "AddonName": "craned",
            "AddonVersion": "v1.2.0",
            "Status": "Succeeded",
            "Phase": "Running"
        }
    ],
    "RequestId": "d4e5f6a7-b8c9-0123-def0-123456789012"
}
```

## 验证

```bash
# 维度 1：确认集群状态
tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]'
# expected: ClusterStatus: "Running"
```

**预期输出**：

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
tccli tke DescribeAddon --region ap-guangzhou \
    --ClusterId cls-xxxxxxxx \
    --AddonName craned
# expected: Status: "Succeeded"
```

**预期输出**：

```json
{
    "Addons": [
        {
            "AddonName": "craned",
            "AddonVersion": "v1.2.0",
            "Status": "Succeeded",
            "Phase": "Running"
        }
    ],
    "RequestId": "d4e5f6a7-b8c9-0123-def0-123456789012"
}
```

```bash
# 维度 3（需 VPN/IOA 内网使 kubectl 可达）：确认 QoSAgent DaemonSet 运行
kubectl get ds -n crane-system qos-agent
# expected: DESIRED 3, READY 3/3
```

**预期输出**：

```text
NAME         DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
qos-agent    3         3         3       3            3           <none>          1d
```

| 维度 | 命令 | 预期 |
|------|------|------|
| 集群状态 | `DescribeClusters --ClusterIds '["cls-xxxxxxxx"]'` | `ClusterStatus: "Running"` |
| Crane 组件 | `DescribeAddon --ClusterId cls-xxxxxxxx --AddonName craned` | `Status: "Succeeded"` |
| QoSAgent | `kubectl get ds -n crane-system qos-agent` | `READY 3/3` |

## 清理

本文档为调度组件概述页面，无需清理操作。如需卸载已安装的调度组件，参考对应组件操作文档的 Cleanup 章节。

卸载 Crane 调度器的参考命令：

```bash
tccli tke DeleteAddon --region ap-guangzhou \
    --ClusterId cls-xxxxxxxx \
    --AddonName craned
# expected: exit 0，返回 RequestId
```

**预期输出**：

```json
{
    "RequestId": "a7b8c9d0-e1f2-3456-abcd-567890123456"
}
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 组件安装失败 | `tccli tke DescribeAddon --region ap-guangzhou --ClusterId cls-xxxxxxxx --AddonName craned` 查看 Status | 集群资源不足或版本不兼容 | 检查集群状态为 `Running`；升级集群版本 |
| QoSAgent 未全部 Ready | `kubectl get pods -n crane-system -l app=qos-agent`（需 kubectl 可达） | 节点资源不足或镜像拉取失败 | `kubectl describe pod -n crane-system <qos-agent-pod>` 查看事件 |
| 调度策略未生效 | `kubectl get podqos,nodeqos`（需 kubectl 可达） | CRD 未创建或 selector 未匹配 | 检查 CRD YAML 的 `selector.matchLabels` 与目标 Pod/Node 的 labels 一致 |
| 组件版本过低不支持某功能 | `tccli tke DescribeAddon --region ap-guangzhou --ClusterId cls-xxxxxxxx --AddonName craned` 查看 AddonVersion | 当前版本较低 | `tccli tke UpdateAddon` 升级到最新版本 |
| kubectl 不可达 | 检查 VPN/IOA 连接 | CAM 策略限制或网络不通 | 通过 VPN/IOA 内网连接集群，或在数据面集群上执行 kubectl 命令 |

## 下一步

各调度功能模块的操作页面：

- **QoS 保障**：[QoS 感知调度](../Qos%20感知调度/tccli%20操作.md)
  - [QoSAgent 安装与配置](../Qos%20感知调度/QoSAgent/tccli%20操作.md) — page_id `79774`
  - [CPU Burst](../Qos%20感知调度/CPU%20Burst/tccli%20操作.md) — page_id `79776`
  - [应用启动时 CPU 突增](../Qos%20感知调度/应用启动时%20CPU%20突增/tccli%20操作.md) — page_id `100618`
  - [内存精细调度](../Qos%20感知调度/内存精细调度/tccli%20操作.md) — page_id `79778`
- **业务优先级**：[业务优先级保障调度](../业务优先级保障调度/tccli%20操作.md)
  - [自定义资源优先级](../业务优先级保障调度/自定义资源优先级/tccli%20操作.md) — page_id `118259`
  - [可抢占式 Job](../业务优先级保障调度/可抢占式%20Job/tccli%20操作.md) — page_id `81751`
- **资源利用率**：[资源利用率优化调度](../资源利用率优化调度/tccli%20操作.md)
  - [节点放大](../资源利用率优化调度/节点放大/tccli%20操作.md) — page_id `79032`
- **调度器详情**：[Crane 调度器介绍](Crane%20调度器介绍/tccli%20操作.md) — page_id `75472`

## 控制台替代

登录 [容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) → 选择集群 **cls-xxxxxxxx** → **组件管理** → 搜索 `craned` 安装与管理 → 各调度子功能在集群内通过组件配置和调度策略界面操作。
