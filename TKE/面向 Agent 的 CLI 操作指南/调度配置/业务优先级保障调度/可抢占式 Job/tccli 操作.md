# 可抢占式 Job（tccli）

> 对照官方：[可抢占式 Job](https://cloud.tencent.com/document/product/457/81751) · page_id `81751`

## 概述

可抢占式 Job 是 TKE Crane 调度器提供的抢占式批处理任务能力。通过在 Job 的 Pod 模板中标注 `crane.io/preemptible: "true"` 并结合低优先级 PriorityClass，使批处理 Job 在集群资源紧张时能被高优先级业务抢占——调度器自动驱逐低优 Job Pod 释放资源，供高优 Pod 调度。Job 被驱逐后重新进入 Pending 状态，等待资源可用时自动重试，适合非实时的数据计算、模型训练和批处理任务。

## 前置条件

- 演示集群 `cls-xxxxxxxx`（ap-guangzhou, v1.30.0, Running），kubectl 因 CAM 策略限制不可达，需通过 VPN/IOA 内网或数据面集群操作
- 已安装 [Crane 调度器](../../调度组件概述/tccli%20操作.md)
- 已了解 Kubernetes Job 和 PriorityClass 概念
- 已阅读 [业务优先级保障调度概述](../tccli%20操作.md) — page_id `118259`

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
# 3. 检查 kubectl 版本（需 VPN/IOA 内网环境）
kubectl version --client
# expected: Client Version >= 1.30.0
```

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 查看集群状态 | `tccli tke DescribeClusters --ClusterIds '["cls-xxxxxxxx"]' --region ap-guangzhou` | 是 |
| 查看已安装组件 | `tccli tke DescribeAddon --ClusterId cls-xxxxxxxx --AddonName craned --region ap-guangzhou` | 是 |
| 创建 PriorityClass | `kubectl apply -f priority-class.yaml` | 是 |
| 查看 PriorityClass 列表 | `kubectl get priorityclass` | 是 |
| 创建可抢占 Job | `kubectl apply -f preemptible-job.yaml` | 是 |
| 查看 Job 状态 | `kubectl get jobs` | 是 |
| 查看 Job Pod 状态 | `kubectl get pods -l job-name=<job-name>` | 是 |
| 查看抢占事件 | `kubectl get events --all-namespaces --field-selector reason=Preempted` | 是 |
| 查看被驱逐 Pod 原因 | `kubectl describe pod <evicted-pod>` | 是 |
| 删除 Job | `kubectl delete job <name>` | 否 |
| 删除 PriorityClass | `kubectl delete priorityclass <name>` | 否 |

## 操作步骤

### 步骤 1：创建 PriorityClass

为高优先级业务和低优先级可抢占 Job 分别创建 PriorityClass。

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
preemptionPolicy: PreemptLowerPriority
description: "High priority for critical online workloads"
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: low-preemptible
value: 10
globalDefault: false
preemptionPolicy: PreemptLowerPriority
description: "Low priority preemptible batch jobs"
```

```bash
kubectl apply -f priority-class.yaml
```

```text
priorityclass.scheduling.k8s.io/high-priority created
priorityclass.scheduling.k8s.io/low-preemptible created
```

### 步骤 2：创建高优先级业务 Pod

先创建一个高优先级 Pod 作为抢占者（模拟关键业务）：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: critical-service
spec:
  priorityClassName: high-priority
  containers:
  - name: nginx
    image: nginx:latest
    resources:
      requests:
        cpu: "2"
        memory: "2Gi"
      limits:
        cpu: "4"
        memory: "4Gi"
```

### 步骤 3：创建可抢占 Job

创建标注了 `crane.io/preemptible: "true"` 的低优先级 Job。当集群资源紧张、高优 Pod 无法调度时，该 Job 的 Pod 会被自动驱逐。

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: preemptible-batch
spec:
  completions: 1
  parallelism: 1
  template:
    metadata:
      annotations:
        crane.io/preemptible: "true"
    spec:
      priorityClassName: low-preemptible
      containers:
      - name: worker
        image: busybox:latest
        command: ["sh", "-c", "echo 'Processing...' && sleep 600"]
        resources:
          requests:
            cpu: "500m"
            memory: "512Mi"
          limits:
            cpu: "1"
            memory: "1Gi"
      restartPolicy: Never
```

```bash
kubectl apply -f preemptible-job.yaml
```

```text
job.batch/preemptible-batch created
```

查看 Job 运行状态：

```bash
kubectl get jobs preemptible-batch
```

```text
NAME                COMPLETIONS   DURATION   AGE
preemptible-batch   0/1           30s        30s
```

```bash
kubectl get pods -l job-name=preemptible-batch
```

```text
NAME                      READY   STATUS    RESTARTS   AGE
preemptible-batch-xxxxx   1/1     Running   0          30s
```

### 步骤 4：模拟抢占场景

当高优先级 Pod 因资源不足无法调度时，Crane 调度器会自动识别低优先级的可抢占 Job Pod 并驱逐它们。

验证抢占行为：

```bash
# 查看抢占事件
kubectl get events --all-namespaces --field-selector reason=Preempted
```

```text
NAMESPACE   LAST SEEN   TYPE     REASON      OBJECT                         MESSAGE
default     10s         Normal   Preempted   pod/preemptible-batch-xxxxx    Preempted by critical-service
```

```bash
# 查看被驱逐 Pod 的详细信息
kubectl describe pod <evicted-pod-name>
```

### 抢占流程说明

```
1. 高优先级 Pod (critical-service) 创建，需要 2 CPU / 2Gi 内存
2. 调度器检查所有节点，发现资源不足
3. 调度器按优先级从低到高扫描可抢占 Pod：
   - low-preemptible PriorityClass (value: 10) 的 Pod 符合条件
   - Pod 标注了 crane.io/preemptible: "true"，允许抢占
4. 选中的低优 Job Pod 被优雅驱逐（terminationGracePeriodSeconds 默认 30s）
5. 高优先级 Pod 调度到释放资源的节点
6. 被驱逐的 Job Pod 重新进入 Pending，Job 控制器创建新 Pod 等待调度
7. 当资源再次充足时，Job Pod 被重新调度并继续执行
```

### 抢占行为约束

| 约束条件 | 说明 |
|---------|------|
| 仅标注 `crane.io/preemptible: "true"` 的 Pod 可被抢占 | 未标注的低优先级 Pod 不会被抢占 |
| 优先级数值越低的 PriorityClass 越先被抢占 | value: 10 比 value: 100 更容易被选中 |
| Job 的 `restartPolicy: Never` | 被抢占后 Job 控制器创建新 Pod 重试 |
| `terminationGracePeriodSeconds` | 被抢占 Pod 的优雅终止时间，默认 30s |
| 同一节点上的多个可抢占 Pod | 调度器会选择释放资源后刚好满足高优 Pod 需求的 Pod 组合 |

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
# 维度 3：确认 PriorityClass 已创建
kubectl get priorityclass
```

```text
NAME                      VALUE        GLOBAL-DEFAULT   AGE
high-priority             1000000      false            10m
low-preemptible           10           false            10m
```

```bash
# 维度 4：确认 Job 正在运行
kubectl get jobs preemptible-batch
```

```text
NAME                COMPLETIONS   DURATION   AGE
preemptible-batch   0/1           2m         2m
```

```bash
# 维度 5：确认 annotation 正确
kubectl get job preemptible-batch -o jsonpath='{.spec.template.metadata.annotations}'
```

```text
{"crane.io/preemptible":"true"}
```

| 维度 | 命令 | 预期 |
|------|------|------|
| 集群状态 | `DescribeClusters --ClusterIds '["cls-xxxxxxxx"]'` | `ClusterStatus: "Running"` |
| 组件状态 | `DescribeAddon --ClusterId cls-xxxxxxxx --AddonName craned` | `Status: "Succeeded"` |
| PriorityClass | `kubectl get priorityclass low-preemptible` | `VALUE: 10` |
| Job 运行 | `kubectl get jobs preemptible-batch` | 正常调度运行 |
| Annotation | `kubectl get job preemptible-batch -o jsonpath='{...}'` | `crane.io/preemptible: "true"` |

## 清理

```bash
# 删除 Job
kubectl delete job preemptible-batch
```

```text
job.batch "preemptible-batch" deleted
```

```bash
# 删除 PriorityClass
kubectl delete priorityclass low-preemptible high-priority
```

```text
priorityclass.scheduling.k8s.io "low-preemptible" deleted
priorityclass.scheduling.k8s.io "high-priority" deleted
```

```bash
# 验证已清理
kubectl get jobs preemptible-batch
```

```text
Error from server (NotFound): jobs.batch "preemptible-batch" not found
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 低优 Job Pod 未被抢占 | `kubectl describe pod <low-pod> \| grep Annotations` | 缺少 `crane.io/preemptible: "true"` annotation | 在 Job 模板中添加 annotation: `kubectl patch job preemptible-batch -p '{"spec":{"template":{"metadata":{"annotations":{"crane.io/preemptible":"true"}}}}}'` |
| 高优 Pod 抢占后低优 Job 卡住 | `kubectl get jobs preemptible-batch` | Job `backoffLimit` 已耗尽 | 增大 `spec.backoffLimit` 或删除 Job 重建 |
| 抢占后数据丢失 | -- | 被抢占 Pod 未持久化中间状态 | 为批处理 Job 设计幂等逻辑或将中间状态写入持久卷 |
| 抢占过于频繁 | `kubectl get events --field-selector reason=Preempted \| wc -l` | 高优 Pod 持续创建，低优 Job 反复被抢占 | 增大集群资源或调整 PriorityClass 值拉开差距；对低优 Job 发停止信号而非依赖抢占 |
| `crane.io/preemptible` annotation 不生效 | `kubectl logs -n crane-system -l app=crane-scheduler --tail=50` | Crane 调度器未启用抢占特性或版本过低 | 确认 craned 版本 >= v1.1.0；`DescribeAddon` 确认状态为 `Succeeded` |
| Job Pod 被抢占后不断重启 | `kubectl describe job preemptible-batch` | `restartPolicy: OnFailure`（而非 Never） | 被抢占属于终止而非失败，应使用 `restartPolicy: Never` 让 Job 控制器创建新 Pod |

## 下一步

- [业务优先级保障调度概述](../tccli%20操作.md) — page_id `118259`（PriorityClass 总体介绍和 Kueue 队列管理）
- [基于 Kueue 实现批调度和队列管理](../基于%20Kueue%20实现批调度和队列管理/tccli%20操作.md) — 高级队列调度方案
- [调度组件概述](../../调度组件概述/tccli%20操作.md) — page_id `111862`
- [Qos 感知调度](../../Qos%20感知调度/tccli%20操作.md) — page_id `79775`（配合 QoS 策略保障抢占后高优业务的稳定性）

## 控制台替代

无专属控制台界面。在 [容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) 中，选择集群 `cls-xxxxxxxx` -> **工作负载** -> **Job** -> 创建 Job 时手动添加 annotation `crane.io/preemptible: "true"` 和 PriorityClass `low-preemptible`。
