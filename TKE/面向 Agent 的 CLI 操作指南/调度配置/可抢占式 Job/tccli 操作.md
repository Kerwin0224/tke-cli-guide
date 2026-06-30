# 可抢占式 Job（tccli）

> 对照官方：[可抢占式 Job](https://cloud.tencent.com/document/product/457/81751) · page_id `81751`

## 概述

可抢占式 Job（Preemptible Job）是 TKE 原生节点提供的调度优先级抢占能力，允许低优先级的批处理 / 离线任务（Job/CronJob）在集群资源紧张时被高优先级在线业务抢占，从而实现资源的动态共享和成本优化。

核心价值：
- **在离线混部**：在线业务（高优先级）和离线任务（低优先级）共享同一节点池，离线任务在资源闲置时运行，在线业务需要时被抢占释放资源
- **降本增效**：利用在线业务波谷期的空闲资源运行批处理任务，无需为离线任务单独购买节点
- **弹性回收**：被抢占的 Job Pod 按重试策略自动重新调度，无需人工干预

## 前置条件

- [环境准备](../../环境准备.md)
- 熟悉 Kubernetes 调度优先级机制（`PriorityClass`、Pod Priority、Pod Preemption）
- 了解 Job/CronJob 的重启策略（`restartPolicy`、`backoffLimit`）
- **演示集群**：`cls-xxxxxxxx`（ap-guangzhou, v1.30.0, Running, 3 节点, tlinux3.1），**kubectl 因 CAM 策略限制不可达**（strategyId: 240463971），数据面操作（提交 Job、配置 PriorityClass）需通过 VPN/IOA 内网或数据面集群执行

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region = ap-guangzhou

# 3. 确认演示集群存在且为 Running
tccli tke DescribeClusters --ClusterIds '["cls-xxxxxxxx"]' --region ap-guangzhou --output json
# expected: TotalCount >= 1，ClusterStatus = "Running"
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
            "VpcId": "vpc-a1b2c3d4",
            "ProjectId": 0,
            "CreatedTime": "2024-06-15T10:30:00Z"
        }
    ],
    "RequestId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看集群基本信息 | `tccli tke DescribeClusters --ClusterIds '["cls-xxxxxxxx"]' --region ap-guangzhou` | 是 |
| 查看集群状态 | `tccli tke DescribeClusterStatus --ClusterIds '["cls-xxxxxxxx"]' --region ap-guangzhou` | 是 |
| 查看调度策略 | `tccli tke DescribeClusterSchedulerPolicy --ClusterId cls-xxxxxxxx --region ap-guangzhou` | 是 |
| 查看节点列表 | `tccli tke DescribeClusterInstances --ClusterId cls-xxxxxxxx --region ap-guangzhou` | 是 |
| 查看资源使用概况 | `tccli tke DescribeResourceUsage --ClusterId cls-xxxxxxxx --region ap-guangzhou` | 是 |
| 创建/配置 PriorityClass | `kubectl apply` PriorityClass YAML（需 VPN/IOA 内网） | 否 |
| 提交可抢占式 Job | `kubectl apply` Job YAML（需 VPN/IOA 内网） | 否 |
| 查看 Pod 抢占事件 | `kubectl describe pod` / `kubectl get events`（需 VPN/IOA 内网） | 是 |

## 操作步骤

### 步骤 1：查看集群调度策略（确认抢占功能状态）

可抢占式 Job 依赖 Kubernetes 原生的 Pod Priority Preemption 机制，该机制在 v1.30.0 中默认启用：

```bash
tccli tke DescribeClusterSchedulerPolicy \
    --ClusterId cls-xxxxxxxx \
    --region ap-guangzhou --output json
# expected: 确认调度策略未禁用抢占功能
```

**预期输出**：

```json
{
    "SchedulerPolicy": {
        "PolicyName": "balanced",
        "Enabled": true
    },
    "RequestId": "e5f6a7b8-c9d0-1234-ef56-7890abcdef12"
}
```

### 步骤 2：理解可抢占式 Job 工作原理

**抢占决策流程图**：

```
新 Pod 到达（高优先级，PriorityClass=high）
  │
  ▼
调度器检查所有节点
  │
  ├── 有足够空闲资源？ ──── YES ──→ 直接调度
  │
  └── NO
      │
      ▼
  遍历节点上运行的低优先级 Pod
      │
      ▼
  存在优先级更低的 Pod？ ──→ 驱逐低优先级 Pod → 调度高优先级 Pod
      │
      └── NO ──→ 新 Pod 进入 Pending 等待
```

**PriorityClass 定义示例**（需通过 kubectl 创建，VPN/IOA 内网）：

```yaml
# 高优先级在线业务
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
description: "高优先级在线业务，不容抢占"
---
# 低优先级离线任务（可抢占）
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: low-priority
value: 100
globalDefault: false
description: "低优先级批处理任务，可被抢占"
```

> `value` 数值越大优先级越高。Kubernetes 内置优先级：`system-cluster-critical`（2000000000）、`system-node-critical`（2000001000）。业务 PriorityClass 建议 `value` 在 1 ~ 1000000 区间。

**可抢占式 Job 示例**（需通过 kubectl 提交，VPN/IOA 内网）：

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: preemptible-batch-job
spec:
  backoffLimit: 3          # 失败重试次数（注意：抢占视为失败触发重试）
  completions: 1
  template:
    spec:
      priorityClassName: low-priority  # 引用低优先级 PriorityClass
      restartPolicy: Never             # 抢占后不重启同一 Pod，由 Job 控制器重建
      containers:
        - name: batch
          image: busybox:1.35
          command: ["sh", "-c", "echo 'Processing...' && sleep 600"]
          resources:
            requests:
              cpu: "500m"
              memory: "256Mi"
            limits:
              cpu: "1"
              memory: "512Mi"
```

**关键设计要点**：

| 机制 | 说明 |
|------|------|
| PriorityClass `value` | 优先级数值，差值越大越优先抢占；建议在线业务与离线任务之间至少差 2 个数量级 |
| Pod 驱逐 | 低优先级 Pod 被优雅终止（`terminationGracePeriodSeconds` 默认 30s） |
| Job 重试 | Job 的 `backoffLimit` 参数控制抢占后重新调度的重试上限 |
| 优雅终止 | 容器接收 SIGTERM 信号后可执行清理逻辑；需确保批处理任务支持断点续传或幂等重试 |
| 不可抢占资源 | DaemonSet、kube-system namespace 下的 Pod 不可被抢占 |

### 步骤 3：适用场景

| 场景 | 说明 | 示例 |
|------|------|------|
| 大数据分析 | 批处理任务可接受中断和重试 | Spark/Flink 离线 ETL |
| AI 训练/推理 | 训练 Job 可接受 Checkpoint 恢复 | PyTorch/TensorFlow 分布式训练 |
| 报表生成 | 定时任务利用波谷资源 | 每日报表计算 |
| CPU/GPU 混部 | 在线推理（GPU）+ 离线训练（GPU）共享 GPU 节点 | vLLM 推理 + LLaMA-Factory 微调 |
| 渲染任务 | 高并发短暂任务可被重调度 | 视频转码、3D 渲染 |

## 验证

```bash
# 1. 确认集群状态为 Running
tccli tke DescribeClusterStatus \
    --ClusterIds '["cls-xxxxxxxx"]' \
    --region ap-guangzhou --output json \
    | jq '.ClusterStatusSet[0].ClusterState'
# expected: "Running"

# 2. 确认集群版本支持抢占功能
tccli tke DescribeClusters \
    --ClusterIds '["cls-xxxxxxxx"]' \
    --region ap-guangzhou --output json \
    | jq '.Clusters[0].ClusterVersion'
# expected: >= "1.15.0"（演示集群 v1.30.0，满足要求）

# 3. 确认集群资源使用概况（作为抢占是否触发的参考）
tccli tke DescribeResourceUsage \
    --ClusterId cls-xxxxxxxx \
    --region ap-guangzhou --output json \
    | jq '.ResourceUsage | {TotalCPU, AllocatedCPU, TotalMemory, AllocatedMemory}'
# expected: 对比 Total 和 Allocated，若 Allocated 接近 Total，抢占更容易触发
```

**预期输出**：

```json
{
    "TotalCPU": 12.0,
    "AllocatedCPU": 8.5,
    "TotalMemory": 24.0,
    "AllocatedMemory": 18.0
}
```

> 抢占行为是否实际发生需在内网通过 `kubectl get events --all-namespaces | grep Preempted` 确认。

## 清理

本页为功能说明与集群状态查询，未创建新资源，无需清理。若在数据面集群中创建了测试用的 PriorityClass 和 Job，需通过 kubectl 删除：

```bash
# 内网环境下执行（参考格式）
kubectl delete job preemptible-batch-job
kubectl delete priorityclass low-priority high-priority
```

## 排障

### 功能未生效

| 现象 | 诊断步骤 | 根因 | 修复 |
|------|---------|------|------|
| 低优先级 Job 未被抢占 | 1. 确认有高优先级 Pod 到达且节点资源不足；2. `kubectl describe pod <high-priority-pod>` 查看 Events；3. 对比两个 PriorityClass 的 `value` 值 | 优先级差值不足，或集群有空闲节点 | 调大 PriorityClass value 差值（建议 > 100 倍），确保节点资源紧张 |
| 抢占后 Job 永久失败 | 1. `kubectl describe job <job-name>` 查看状态；2. 检查 `backoffLimit` 配置 | `backoffLimit` 过小，抢占导致的失败消耗了重试配额 | 增大 `backoffLimit`（建议 >= 5），或使用 CronJob 替代 Job |
| Job 抢占后数据丢失 | 检查容器是否在 SIGTERM 时间内完成数据保存 | 批处理任务不支持断点续传 | 增加 `terminationGracePeriodSeconds`，实现 Checkpoint + 重试机制 |

### 配置问题

| 现象 | 诊断步骤 | 根因 | 修复 |
|------|---------|------|------|
| PriorityClass 无法创建 | `kubectl apply` 时权限报错 | RBAC 权限不足 | 确认当前 kubeconfig 用户有 PriorityClass 的 create 权限 |
| `priorityClassName` 引用无效 | `kubectl describe pod` 查看 Events | PriorityClass 名称拼写错误或未创建 | 确认 PriorityClass 已创建且名称匹配 |
| DaemonSet 被抢占 | 检查抢占日志 | DaemonSet 不应被抢占，此为异常行为 | [提交工单](https://console.cloud.tencent.com/workorder) 排查 |

## 下一步

- [业务优先级保障调度](../业务优先级保障调度/tccli%20操作.md) -- page_id `--`
- [自定义资源优先级](../业务优先级保障调度/自定义资源优先级/tccli%20操作.md) -- page_id `--`
- [基于 Kueue 实现批调度和队列管理](../业务优先级保障调度/基于%20Kueue%20实现批调度和队列管理/tccli%20操作.md) -- page_id `--`
- [调度组件概述](../调度组件概述/tccli%20操作.md) -- page_id `111862`
- [TKE API 概览](https://cloud.tencent.com/document/product/457/31853)

## 控制台替代

[TKE 控制台 → 集群 `cls-xxxxxxxx` → 工作负载 → Job](https://console.cloud.tencent.com/tke2/cluster?rid=1) 提供 Job/CronJob 的可视化创建界面，支持指定 PriorityClass。控制台还提供 Pod 事件查看（含 Preempted 事件）、资源使用仪表盘和节点负载监控。CLI 可用于查询集群状态、资源概况和调度策略，但 PriorityClass 和 Job 的创建/管理需通过 kubectl 操作。
