# 合理利用节点资源概述（tccli）

> 对照官方：[合理利用节点资源概述](https://cloud.tencent.com/document/product/457/45633) · page_id `45633`

## 概述

在 Kubernetes 集群中，节点资源（CPU、内存、GPU、临时存储等）是固定且有限的。合理利用节点资源是保障集群稳定性、控制成本、提升资源利用效率的核心前提。本文从概念层面介绍节点资源管理的核心机制，帮助你在部署工作负载前理解资源分配的基本原理。

> **kubectl 数据面不可达**：CAM 拒绝公网端点（strategyId:240463971），内网需 VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

### 核心概念

**Request 与 Limit**：Kubernetes 用两个维度描述容器资源需求：
- `requests`：容器**保证**能获取的最低资源量。调度器（kube-scheduler）依据所有容器的 requests 之和判断节点是否有足够资源放置 Pod。
- `limits`：容器**最多**能使用的资源上限。当容器实际用量超过 limits 时，CPU 会被限流（throttle），内存会被 OOMKill。

合理设置 request/limit 能平衡资源保障与超卖。request 设置过大导致资源浪费；设置过小则影响稳定性。limit 设置过大增加超出节点物理资源的 OOM 风险；设置过小则限制容器性能。

**QoS 类（Quality of Service）**：Kubernetes 根据 request/limit 的配置自动为 Pod 分配 QoS 类，共三类：

| QoS 类 | 条件 | OOM 优先级 | 驱逐优先级 | 适用场景 |
|--------|------|:--------:|:--------:|---------|
| **Guaranteed** | 每个容器的 request == limit，且均设置了 CPU 和内存 | 最低（最后被 Kill） | 最低 | 核心业务、数据库、有状态服务 |
| **Burstable** | 至少一个容器设置了 request 或 limit，但不满足 Guaranteed 条件 | 中等 | 中等 | 普通 Web 服务、API 服务 |
| **BestEffort** | 没有任何容器设置 request 或 limit | 最高（最先被 Kill） | 最高 | 离线任务、测试环境 |

当节点资源耗尽时，kubelet 按 BestEffort → Burstable → Guaranteed 的顺序驱逐 Pod。因此生产环境中建议至少使用 Burstable，核心服务使用 Guaranteed。

**节点资源状态**：
- `Capacity`：节点物理硬件资源总量。
- `Allocatable`：Capacity 扣除系统预留（system-reserved、kube-reserved、eviction-threshold）后可供 Pod 使用的资源量。调度器以此为准。

**资源碎片化**：当节点上 Pod 被频繁创建、销毁时，可能出现大量零碎可用资源（如 0.5 核 CPU + 300Mi 内存），虽然总可分配资源充足，但没有单个节点能容纳一个 2 核 4Gi 的 Pod。这就是资源碎片化问题——表面上集群资源充足，但实际上无法调度较大规格的 Pod。

**Bin-Packing（装箱优化）**：将 Pod 尽量集中调度到少量节点，最大化单节点资源利用率。优点是可减少闲置节点数量，降低费用；缺点是单点故障影响面更大。适用于成本敏感的场景。

**资源管理的重要性**：
- **稳定性**：合理设置 request/limit 避免节点资源过载导致 Pod 被 OOMKill。
- **成本**：提高资源利用率可减少节点数量，降低 CVM 费用。
- **效率**：避免资源碎片化，保证大规格 Pod 能被正常调度。
- **可预测性**：QoS 类决定了 Pod 在资源紧张时的行为，便于容量规划。

## 前置条件

- [环境准备](../../../../环境准备.md)：`tccli` 已配置，`kubectl` >= v1.28
- 已获取集群 kubeconfig（VPN/IOA 环境方可使用 kubectl）：

```bash
tccli tke DescribeClusterKubeconfig --region <Region> --ClusterId CLUSTER_ID --filter "Kubeconfig" > kubeconfig.yaml
export KUBECONFIG=$(pwd)/kubeconfig.yaml
# expected: 生成 kubeconfig.yaml
```

```json
{
  "Kubeconfig": "<Kubeconfig>",
  "RequestId": "<RequestId>"
}
```

- 需要 CAM 权限：`tke:DescribeClusters`、`tke:DescribeClusterKubeconfig`

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId、secretKey、region 均已配置，region 为目标地域

# 3. 确认目标集群存在且状态正常
tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterStatus "Running"
```

```json
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "CLUSTER_ID",
            "ClusterName": "MANAGED_CLUSTER",
            "ClusterStatus": "Running",
            "ClusterVersion": "1.30.0",
            "ClusterType": "MANAGED_CLUSTER"
        }
    ],
    "RequestId": "00000000-0000-0000-0000-000000000000"
}
```

- K8s 版本：1.30.0
- 集群类型：MANAGED_CLUSTER
- 地域：ap-guangzhou

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 查看集群信息 | `tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'` | 是 |
| 查看集群节点 | `tccli tke DescribeClusterInstances --region <Region> --ClusterId CLUSTER_ID` | 是 |
| 查看节点资源（Capacity/Allocatable） | `kubectl describe node NODE_ID`（需 VPN/IOA） | 是 |
| 查看节点实际用量 | `kubectl top node`（需 VPN/IOA） | 是 |
| 查看 Pod 资源描述 | `kubectl get pod -o json`（需 VPN/IOA） | 是 |
| 查看 QoS 类 | `kubectl describe pod`（需 VPN/IOA） | 是 |
| 查看 LimitRange | `kubectl describe limitrange -n NAMESPACE`（需 VPN/IOA） | 是 |
| 查看 ResourceQuota | `kubectl describe resourcequota -n NAMESPACE`（需 VPN/IOA） | 是 |

## 操作步骤

本页为概念概述页，不涉及具体的资源创建操作。"操作步骤"重点介绍资源管理决策框架和核心概念。

### 概念解析

#### 1. Request 与 Limit 的作用

在 Kubernetes 中，调度决策和运行限制由两组值控制：

```
          ┌─────────────────────┐
          │        Node         │
          │  Capacity: 4C 8Gi   │
          │                     │
          │  ┌───────────────┐  │
          │  │  Allocatable  │  │
          │  │   3.5C  7Gi   │  │  ← system-reserved 扣除后
          │  └──┬─────────┬──┘  │
          │     │         │     │
          │  ┌──▼──┐  ┌───▼──┐ │
          │  │Pod A│  │Pod B │ │
          │  │req: │  │req:  │ │
          │  │1C 1Gi│  │0.5C  │ │
          │  │lim: │  │0.5Gi │ │
          │  │2C 2Gi│  │lim:  │ │
          │  │     │  │1C 1Gi│ │
          │  └─────┘  └──────┘ │
          │                     │
          │  已调度: 1.5C 1.5Gi │
          │  剩余可调度: 2C 5.5Gi│
          └─────────────────────┘
```

- 调度决策只看 `requests`：3.5C - 1.5C = 2C 剩余，还可调度请求 2 核的 Pod。
- 运行限制只看 `limits`：Pod A 最高可用 2C 2Gi，Pod B 最高可用 1C 1Gi。
- Pod A 和 Pod B 的用量总和最高可达 3C 3Gi，但节点物理 CPU 为 4C——这就是超卖（oversubscription），是合理的。

#### 2. QoS 类判定规则

通过 `kubectl describe pod` 或 `kubectl get pod -o json` 查看 QoS 类：

```bash
kubectl get pod -o json -n NAMESPACE | jq '.items[].status.qosClass'
# expected: 输出每个 Pod 的 QoS 类
```

```text
"Guaranteed"
"Burstable"
"Burstable"
"BestEffort"
```

**Guaranteed（最高优先级）**— 所有容器必须同时满足：
- 设置 CPU request，且 request == limit
- 设置内存 request，且 request == limit

```yaml
spec:
  containers:
  - name: app
    resources:
      requests:
        cpu: 500m
        memory: 1Gi
      limits:
        cpu: 500m        # 与 request 相同
        memory: 1Gi      # 与 request 相同
```

**Burstable（中等优先级）**— 满足以下任一条件：
- 至少一个容器设置了 request（不要求 request == limit）
- 部分容器未设置 request 或 limit

```yaml
spec:
  containers:
  - name: app
    resources:
      requests:
        cpu: 200m        # 只设 request
        memory: 256Mi
```

**BestEffort（最低优先级）**— 没有任何容器设置 request 或 limit：

```yaml
spec:
  containers:
  - name: app
    # 未设置 resources
```

#### 3. 节点 Capacity vs Allocatable

```bash
kubectl describe node NODE_ID | grep -A 5 "Capacity"
# expected: 显示 Capacity 和 Allocatable
```

```text
Capacity:
  cpu:                4
  ephemeral-storage:  82451056Ki
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             8172224Ki
  pods:               64
Allocatable:
  cpu:                3800m
  ephemeral-storage:  75985356Ki
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             7143424Ki
  pods:               64
```

- CPU Capacity: 4 核（4000m），Allocatable: 3800m（系统预留 200m）
- 内存 Capacity: 8172224Ki（约 7.8Gi），Allocatable: 7143424Ki（约 6.8Gi，系统预留约 1Gi）
- Pods 上限: 64 个

Allocatable 才是调度器实际使用的可用资源量。所有容器的 requests 之和不可超过 Allocatable。

### 选择依据

不同业务类型应使用不同的资源策略：

| 业务类型 | 推荐 QoS | request:limit 比例 | 资源策略 |
|---------|:-------:|:-----------------:|---------|
| 数据库、核心 API | Guaranteed | 1:1 | 保证资源，避免被驱逐 |
| Web 服务（有弹性需求） | Burstable | 1:2 ~ 1:4 | request 保证基线，limit 允许峰值 |
| 定时任务、批处理 | Burstable | 1:1 ~ 1:2 | 设置合理的 request，避免 endless Pending |
| 离线任务、测试服务 | Burstable/BestEffort | — | 允许被优先驱逐，最大化资源利用 |

### 决策框架

#### 第 1 步：确定节点规模

```bash
# 查看集群当前节点数及规格
tccli tke DescribeClusterInstances --region <Region> --ClusterId CLUSTER_ID
# expected: InstanceSet 列出所有节点及其规格
```

```json
{
  "TotalCount": "<TotalCount>",
  "InstanceSet": "<InstanceSet>",
  "InstanceId": "<InstanceId>",
  "InstanceRole": "<InstanceRole>",
  "FailedReason": "<FailedReason>",
  "InstanceState": "<InstanceState>"
}
```

评估是否满足总资源需求：总 Allocatable >= 所有应用 requests 之和 + 20% 冗余。

#### 第 2 步：统计当前资源使用

```bash
kubectl top nodes
# expected: 显示每个节点的 CPU/内存实际使用率
```

```text
NAME             CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
NODE_1           1200m        31%    3400Mi          48%
NODE_2           2600m        68%    5200Mi          74%
NODE_3           800m         21%    1800Mi          25%
```

- NODE_2 已接近 75% 内存使用，高负载风险，不建议再调度内存密集型 Pod。
- NODE_3 利用率低，考虑是否可合并工作负载以减少节点数量。

#### 第 3 步：检测资源碎片化

```bash
kubectl describe nodes | grep -E "Name:|Allocatable:|Allocated"
# expected: 显示每个节点的可分配和已分配资源
```

```text
Name:               NODE_1
Allocatable:
  cpu:                3800m
  memory:             7143424Ki
Allocated resources:
  cpu:                3500m (92%)   ← 剩余 300m
  memory:             6800Mi (95%)  ← 剩余约 350Mi
Name:               NODE_2
Allocatable:
  cpu:                3800m
  memory:             7143424Ki
Allocated resources:
  cpu:                800m (21%)    ← 剩余 3000m
  memory:             1200Mi (16%)  ← 剩余约 5.8Gi
```

NODE_1 仅剩余 300m CPU 和 350Mi 内存，虽然 NODE_2 资源充裕，但无法迁出一个现有 Pod 到 NODE_2 以释放 NODE_1。这是典型的资源碎片化表现——需要审视 Pod 的 requests 设置是否过于零碎。

#### 第 4 步：选择合适的资源管控机制

根据集群治理需求选择：

| 管控层级 | 机制 | 适用范围 | CLI 操作 |
|---------|------|---------|---------|
| 容器级 | `resources.requests/limits` | 单个容器 | Deployment/Pod YAML |
| 命名空间级 | LimitRange | 设置默认 request/limit 边界 | `kubectl apply -f limitrange.yaml` |
| 命名空间级 | ResourceQuota | 限制总 request/limit 总量 | `kubectl apply -f resourcequota.yaml` |
| 节点级 | Taint/Toleration + NodeSelector | 将特定工作负载调度到专用节点 | `kubectl taint node`, Pod `tolerations` |
| 集群级 | Cluster Autoscaler / HPA | 自动扩缩节点/副本数 | 弹性伸缩相关配置 |

> 小型集群可直接使用容器级设置。多团队共享集群建议叠加命名空间级 ResourceQuota + LimitRange。

## 验证

### 控制面（tccli）

```bash
# 验证集群状态正常
tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterStatus "Running"

# 验证节点状态正常
tccli tke DescribeClusterInstances --region <Region> --ClusterId CLUSTER_ID
# expected: 所有节点 InstanceState "running"
```

```json
{
    "TotalCount": 3,
    "InstanceSet": [
        {
            "InstanceId": "INS_ID_1",
            "InstanceRole": "WORKER",
            "InstanceState": "running",
            "LanIP": "10.0.0.10"
        },
        {
            "InstanceId": "INS_ID_2",
            "InstanceRole": "WORKER",
            "InstanceState": "running",
            "LanIP": "10.0.0.11"
        },
        {
            "InstanceId": "INS_ID_3",
            "InstanceRole": "WORKER",
            "InstanceState": "running",
            "LanIP": "10.0.0.12"
        }
    ],
    "RequestId": "00000000-0000-0000-0000-000000000000"
}
```

### 数据面（需 VPN/IOA）

```bash
# 验证节点实际资源用量
kubectl top nodes
# expected: 每个节点显示 CPU% 和 MEMORY%，处于合理范围
```

```text
NAME             CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
NODE_1           1200m        31%    3400Mi          48%
NODE_2           1000m        26%    2800Mi          39%
NODE_3           800m         21%    2200Mi          31%
```

```bash
# 验证每个节点的可分配资源
kubectl describe nodes | grep -A 5 "Allocatable"
# expected: 每个节点显示 Allocatable 资源量
```

```text
Allocatable:
  cpu:                3800m
  ephemeral-storage:  75985356Ki
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             7143424Ki
  pods:               64
--
Allocatable:
  cpu:                3800m
  ...
--
Allocatable:
  cpu:                3800m
  ...
```

```bash
# 验证 Pod QoS 类分布
kubectl get pods --all-namespaces -o json | jq -r '
  .items[] | "\(.metadata.namespace)/\(.metadata.name): \(.status.qosClass)"
' | sort
# expected: 列出所有 Pod 及其 QoS 类
```

```text
NAMESPACE/pod-name-1: Burstable
NAMESPACE/pod-name-2: Guaranteed
NAMESPACE/pod-name-3: Burstable
NAMESPACE/pod-name-4: Guaranteed
```

```bash
# 验证命名空间级 ResourceQuota（如果已配置）
kubectl get resourcequota -n NAMESPACE
# expected: 列出 ResourceQuota 及使用情况
```

```text
NAME            AGE   REQUEST                      LIMIT
compute-quota   15d   requests.cpu: 1200m/4000m    limits.cpu: 2500m/8000m
                      requests.memory: 2Gi/8Gi    limits.memory: 4Gi/16Gi
```

## 清理

本页为概念概述，不涉及资源创建。以下为对应页面创建的资源清理指引：

| 相关内容 | 清理指引 |
|---------|---------|
| 设置 Request 与 Limit 页面创建的 LimitRange / ResourceQuota | `kubectl delete limitrange <name> -n <namespace>`<br>`kubectl delete resourcequota <name> -n <namespace>` |
| 资源合理分配页面创建的 Taint / Toleration | `kubectl taint node <node-name> <key>=<value>:<effect>-` |
| 弹性伸缩页面创建的 HPA | `kubectl delete hpa <name> -n <namespace>` |

无需额外清理操作。

## 排障

### 资源不足类

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Pod 状态 `Pending`，`kubectl describe pod` 显示 `Insufficient cpu` 或 `Insufficient memory` | `kubectl top nodes` + `kubectl describe node NODE_ID | grep -A 10 "Allocated resources"` | 所有节点可分配资源不足以满足 Pod 的 requests | 1) 降低 Pod requests；2) 扩增节点；3) 驱逐低优先级 Pod 腾空间 `kubectl drain NODE_ID --ignore-daemonsets` |
| Pod 状态 `Pending`，`kubectl describe pod` 显示 `no nodes available to schedule` 但 `kubectl top nodes` 显示资源充足 | `kubectl describe node NODE_ID \| grep -A 15 "Allocated resources"`，对比各节点剩余碎片 | 资源碎片化：各节点剩余资源分散，单个节点无法满足 Pod 的 request | 1) 调整 Pod requests 降低单副本要求；2) 驱逐零星 Pod 重新调度 `kubectl drain NODE_ID`；3) 启用 descheduler 做二次调度优化 |
| Pod 频繁重启，`kubectl describe pod` 显示 `Last State: Terminated`, `Exit Code: 137`（OOMKilled） | `kubectl top pod` 查看内存趋势 | 容器内存使用超出 limit，被 kubelet OOMKill | 1) 增大 `limits.memory`；2) 排查应用内存泄漏；3) 添加 `requests.memory` 防止调度到资源不足节点 |

### QoS 类相关问题

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Pod 被意外驱逐（Evicted），日志显示 `The node was low on resource` | `kubectl describe pod` 查看 `Status` 和 `QoS Class`，确认 QoS 类 | Pod 为 BestEffort 或 Burstable，节点资源紧张时被优先驱逐 | 升级为 Guaranteed：确保所有容器 request == limit，且 CPU 和内存都设置 |
| 预期 Guaranteed 的 Pod 实际为 Burstable | `kubectl get pod -o json \| jq '.spec.containers[].resources'` 检查每个容器的 resources 配置 | 某个容器未同时设置 CPU、内存的 request 和 limit，或 init container 未满足条件 | 修复所有容器（含 init containers）的 resources 设置，确保 CPU、内存 request == limit |

### 节点资源类

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Allocatable 远小于 Capacity | `kubectl describe node NODE_ID \| grep -A 5 "Capacity"` 对比 Capacity 和 Allocatable | 系统预留（system-reserved、kube-reserved）过大 | TKE 托管集群由平台管理，默认预留合理；非托管集群需调整 kubelet 参数 `--system-reserved` 和 `--kube-reserved` |
| 节点 Allocated 百分比接近 100%，但 `kubectl top node` 显示实际用量低（如 20%） | `kubectl describe node NODE_ID \| grep -A 10 "Allocated resources"` | Pod requests 设置过高，大量预留资源未被实际使用 | 降低 over-provisioned Pod 的 requests 值；利用 Vertical Pod Autoscaler（VPA）获取推荐值 |

## 下一步

- [设置 Request 与 Limit](../设置%20Request%20与%20Limit/tccli%20操作.md) — 在容器级和命名空间级设置资源约束
- [资源合理分配](../资源合理分配/tccli%20操作.md) — 通过调度策略将工作负载分配到合适节点
- [弹性伸缩](../弹性伸缩/tccli%20操作.md) — 基于资源利用率自动扩缩副本数和节点数
- [工作负载平滑升级](../../工作负载平滑升级/tccli%20操作.md) — 发布更新时控制新老版本并行度
- [应用高可用部署](../../应用高可用部署/tccli%20操作.md) — 跨节点/可用区分布 Pod 保障容灾

## 控制台替代

通过 [TKE 控制台 - 集群](https://console.cloud.tencent.com/tke2/cluster) 查看节点资源状态：节点管理 → 节点详情 → 资源使用率。通过 [TKE 控制台 - 工作负载](https://console.cloud.tencent.com/tke2/cluster/sub/list/resource/deployment) 管理容器的资源 Request/Limit 配置。
