# 自定义资源优先级（tccli）

> 对照官方：[自定义资源优先级](https://cloud.tencent.com/document/product/457/118259) · page_id `118259`

## 概述

Kubernetes 通过 PriorityClass 定义工作负载优先级。当集群资源紧张时，调度器优先调度高优先级 Pod，并可抢占（preempt）低优先级 Pod 为其让出资源。TKE 托管集群自带两个系统级 PriorityClass（`system-cluster-critical`、`system-node-critical`），用户可创建自定义 PriorityClass 实现业务优先级保障。

| 场景 | 推荐 PriorityClass | value 范围 | 说明 |
|------|-------------------|-----------|------|
| 核心基础设施 | `system-cluster-critical` | 2000000000 | 系统组件，不应抢占 |
| 生产关键服务 | 自建 `production-high` | 1000000 | 高优业务，可抢占所有低优 Pod |
| 常规业务 | 自建 `production-medium` | 100000 | 普通优先级 |
| 批处理/离线任务 | 自建 `batch-low` | 10000 | 低优任务，可被抢占 |
| 测试/开发 | `global-default`（系统默认） | 0 | 无显式指定时的默认优先级 |

## 前置条件

- [环境准备](../../../环境准备.md)

演示集群信息：**cls-xxxxxxxx**（ap-guangzhou，v1.30.0，Running，3 节点）。注意：kubectl 因 CAM 策略限制不可达，需通过 VPN/IOA 内网或数据面集群操作。以下 kubectl 命令为参考格式。

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置

# 3. 检查 kubectl 版本（数据面操作需要，当前集群 kubectl 不可达）
kubectl version --client
# expected: Client Version >= 1.30.0

# 4. 检查 CAM 权限 — 需要以下 Action 名
#    tke:DescribeClusters, tke:DescribeClusterKubeconfig
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
# 5. 确认目标集群存在
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
# 6. 获取 kubeconfig（需 VPN/IOA 内网使 kubectl 可达）
tccli tke DescribeClusterKubeconfig --region ap-guangzhou --ClusterId cls-xxxxxxxx \
    --output json | jq -r '.Kubeconfig' > ~/.kube/config-cls-xxxxxxxx
export KUBECONFIG=~/.kube/config-cls-xxxxxxxx
kubectl cluster-info
# expected: Kubernetes control plane is running
```

**预期输出**：

```text
Kubernetes control plane is running at https://cls-xxxxxxxx.ccs.tencent-cloud.com
```

```bash
# 7. 查看现有 PriorityClass
kubectl get priorityclass
# expected: 至少返回 system-cluster-critical 和 system-node-critical
```

**预期输出**：

```text
NAME                      VALUE        GLOBAL-DEFAULT   AGE
system-cluster-critical   2000000000   false            1d
system-node-critical      2000001000   false            1d
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 / kubectl 命令 | 幂等 |
|-----------|------------------------|:--:|
| 查看集群列表 | `tccli tke DescribeClusters --region ap-guangzhou` | 是 |
| 查看集群详情 | `tccli tke DescribeClusters --ClusterIds '["cls-xxxxxxxx"]'` | 是 |
| 获取 kubeconfig | `tccli tke DescribeClusterKubeconfig --ClusterId cls-xxxxxxxx` | 是 |
| 创建 PriorityClass | `kubectl apply -f priorityclass.yaml` | 否 |
| 查看 PriorityClass | `kubectl get priorityclass` | 是 |
| 为 Pod 指定优先级 | `kubectl apply -f deploy.yaml`（含 priorityClassName） | 否 |
| 删除 PriorityClass | `kubectl delete priorityclass NAME` | 否 |

## 操作步骤

### 步骤 1：创建自定义 PriorityClass

#### 选择依据

- **value**：数值越大优先级越高。建议按业务等级分档：核心服务 > 10 亿，生产高优 > 100 万，常规业务 > 10 万，批处理 < 1 万。避免使用 >= 10 亿的数值（系统保留范围）。
- **globalDefault**：设 `true` 则该 PriorityClass 成为集群默认优先级，所有未指定 `priorityClassName` 的 Pod 自动继承。通常保留 `global-default`（value=0）为系统默认，不覆盖。
- **preemptionPolicy**：`PreemptLowerPriority`（默认）允许抢占低优先级 Pod；`Never` 禁止抢占。生产关键服务应允许抢占，批处理任务建议 `Never`。

#### 最小创建

```bash
# 需 kubectl 可达环境 — 创建生产高优 PriorityClass
cat > priority-production-high.yaml <<'EOF'
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: production-high
value: 1000000
globalDefault: false
description: "Production critical services - high priority"
EOF

kubectl apply -f priority-production-high.yaml
# expected: priorityclass.scheduling.k8s.io/production-high created
```

**预期输出**：

```text
priorityclass.scheduling.k8s.io/production-high created
```

#### 增强配置（多种优先级分层）

```bash
# 需 kubectl 可达环境 — 创建中专和低优先级别
cat > priority-all.yaml <<'EOF'
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: production-medium
value: 100000
globalDefault: false
description: "Standard production workloads"
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: batch-low
value: 10000
globalDefault: false
description: "Batch and offline jobs - can be preempted"
preemptionPolicy: Never
EOF

kubectl apply -f priority-all.yaml
# expected: priorityclass.scheduling.k8s.io/production-medium created
#          priorityclass.scheduling.k8s.io/batch-low created
```

**预期输出**：

```text
priorityclass.scheduling.k8s.io/production-medium created
priorityclass.scheduling.k8s.io/batch-low created
```

### 步骤 2：为工作负载分配优先级

#### 选择依据

- **priorityClassName**：在 Pod spec 中指定 PriorityClass 名称。如未指定，继承 `globalDefault` PriorityClass（或系统默认 `global-default`，value=0）。
- 建议为集群中所有关键 Deployment/StatefulSet 显式指定优先级，避免因资源竞争导致核心服务中断。

```bash
# 需 kubectl 可达环境 — 创建高优先级 Deployment
cat > deploy-high.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api-service
  template:
    metadata:
      labels:
        app: api-service
    spec:
      priorityClassName: production-high
      containers:
      - name: nginx
        image: nginx:1.25
        resources:
          requests:
            cpu: 500m
            memory: 512Mi
          limits:
            cpu: 1000m
            memory: 1Gi
EOF

kubectl apply -f deploy-high.yaml
# expected: deployment.apps/api-service created
```

**预期输出**：

```text
deployment.apps/api-service created
```

```bash
# 需 kubectl 可达环境 — 创建低优先级批处理 Job
cat > job-batch.yaml <<'EOF'
apiVersion: batch/v1
kind: Job
metadata:
  name: data-processing
spec:
  template:
    spec:
      priorityClassName: batch-low
      containers:
      - name: processor
        image: busybox:1.36
        command: ["sleep", "300"]
      restartPolicy: Never
EOF

kubectl apply -f job-batch.yaml
# expected: job.batch/data-processing created
```

**预期输出**：

```text
job.batch/data-processing created
```

### 步骤 3：验证抢占行为（可选）

当集群资源不足时，高优先级 Pod 可抢占低优先级 Pod：

```bash
# 需 kubectl 可达环境 — 查看抢占事件
kubectl get events --all-namespaces --field-selector reason=Preempted \
    --sort-by='.lastTimestamp' | tail -10
# expected: 如有抢占事件，显示 Preempted 记录
```

```text
NAME  STATUS  AGE
...
```

## 验证

### 控制面（tccli）

```bash
# 维度 1：确认集群状态正常
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

### 数据面

```bash
# 需 kubectl 可达环境 — 维度 2：验证 PriorityClass 已创建
kubectl get priorityclass
# expected: production-high、production-medium、batch-low 在列表中
```

**预期输出**：

```text
NAME                      VALUE        GLOBAL-DEFAULT   AGE
production-high           1000000      false            5m
production-medium         100000       false            5m
batch-low                 10000        false            5m
system-cluster-critical   2000000000   false            1d
system-node-critical      2000001000   false            1d
```

```bash
# 需 kubectl 可达环境 — 维度 3：验证 Pod 优先级分配
kubectl get pod -l app=api-service -o jsonpath='{.items[0].spec.priorityClassName}'
# expected: production-high
```

**预期输出**：

```text
production-high
```

```bash
# 需 kubectl 可达环境 — 维度 4：查看 Pod 优先级数值
kubectl get pod -l app=api-service -o jsonpath='{.items[0].spec.priority}'
# expected: 1000000
```

**预期输出**：

```text
1000000
```

```bash
# 需 kubectl 可达环境 — 维度 5：验证低优先级 Job 运行状态
kubectl get job data-processing
# expected: COMPLETIONS 和 DURATION 按预期
```

**预期输出**：

```text
NAME              COMPLETIONS   DURATION   AGE
data-processing   1/1           5m         5m
```

| 维度 | 命令 | 预期 |
|------|------|------|
| 集群状态 | `DescribeClusters --ClusterIds '["cls-xxxxxxxx"]'` | `ClusterStatus: "Running"` |
| PriorityClass | `kubectl get priorityclass` | 自定义优先级在列表中 |
| 优先级类名 | `kubectl get pod -l app=api-service -o jsonpath` | `production-high` |
| 优先级数值 | 同上 `spec.priority` | `1000000` |
| Job 状态 | `kubectl get job data-processing` | 正常运行/完成 |

## 清理

> **警告**：删除 PriorityClass 后，已绑定该优先级的现有 Pod 不受影响（优先级在 Pod 创建时固化），但新创建的 Pod 将无法使用该 PriorityClass。删除 Deployment/Job 会级联删除其管理的 Pod。生产环境删除前请确认无业务依赖。

### 数据面

数据面资源清理在控制面之前。

```bash
# 需 kubectl 可达环境 — 清理前确认目标资源
kubectl get deploy api-service
kubectl get job data-processing
# 确认是待删除的目标工作负载
```

```text
NAME  STATUS  AGE
...
```

```bash
# 需 kubectl 可达环境 — 删除 Deployment
kubectl delete deploy api-service
# expected: deployment.apps "api-service" deleted

# 需 kubectl 可达环境 — 删除 Job
kubectl delete job data-processing
# expected: job.batch "data-processing" deleted

# 需 kubectl 可达环境 — 删除 PriorityClass
kubectl delete priorityclass production-high production-medium batch-low
# expected: priorityclass.scheduling.k8s.io "production-high" deleted
#          priorityclass.scheduling.k8s.io "production-medium" deleted
#          priorityclass.scheduling.k8s.io "batch-low" deleted
```

```bash
# 需 kubectl 可达环境 — 验证已删除
kubectl get priorityclass production-high
# expected: Error from server (NotFound)
```

**预期输出**：

```text
Error from server (NotFound): priorityclasses.scheduling.k8s.io "production-high" not found
```

```bash
kubectl get deploy api-service
# expected: Error from server (NotFound)
```

**预期输出**：

```text
Error from server (NotFound): deployments.apps "api-service" not found
```

```bash
kubectl get job data-processing
# expected: Error from server (NotFound)
```

**预期输出**：

```text
Error from server (NotFound): jobs.batch "data-processing" not found
```

清理确认：无需清理控制面资源（PriorityClass 和 Deployment/Job 均为数据面资源）。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl apply -f` PriorityClass 返回 `already exists` | `kubectl get priorityclass NAME` 确认 | PriorityClass 名称冲突 | 使用不同名称，或先 `kubectl delete priorityclass NAME` 后重建 |
| Pod 创建失败，Events 显示 `priority class "NAME" not found` | `kubectl get priorityclass` 确认列表 | Deployment 指定的 `priorityClassName` 不存在 | 先创建对应的 PriorityClass，或修正 Deployment 的 `priorityClassName` |
| Pod 状态长时间 Pending | `kubectl describe pod POD_NAME` 查看 Events | 集群资源不足且 PriorityClass 设为 `preemptionPolicy: Never`，无法抢占 | 扩容集群节点；或将 `preemptionPolicy` 改为 `PreemptLowerPriority`，允许抢占低优 Pod |
| kubectl 不可达 | 检查 VPN/IOA 连接 | CAM 策略限制或网络不通 | 通过 VPN/IOA 内网连接集群，或在数据面集群上执行 kubectl 命令 |

### 优先级配置后未生效

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 高优 Pod 被低优 Pod 抢占 | `kubectl get pod HIGH_POD -o jsonpath='{.spec.priority}'` 对比两者数值 | 高优 PriorityClass 的 value 数值实际上小于低优 PriorityClass | 检查各 PriorityClass 的 `value` 字段：`kubectl get priorityclass -o custom-columns=NAME:.metadata.name,VALUE:.value` |
| `globalDefault: true` 未生效 | `kubectl describe priorityclass NAME \| grep GlobalDefault` | 默认 PriorityClass 只影响**新创建**的未指定 `priorityClassName` 的 Pod（已有 Pod 不受影响） | 重建受影响 Pod；或为 Deployment 显式设置 `priorityClassName` |
| 所有 Pod 优先级相同 | `kubectl get pods -A -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.priority}{"\n"}{end}'` | 工作负载未指定 `priorityClassName`，全部继承默认 PriorityClass | 在 Deployment/StatefulSet 的 `spec.template.spec` 中增加 `priorityClassName` 字段 |

## 下一步

- [基于 Kueue 实现批调度和队列管理](../基于%20Kueue%20实现批调度和队列管理/tccli%20操作.md) — 更高级的队列化调度
- [可抢占式 Job](../可抢占式%20Job/tccli%20操作.md) — 利用优先级实现可抢占任务
- [Crane 调度器介绍](../../调度组件概述/Crane%20调度器/Crane%20调度器介绍/tccli%20操作.md) — 配合 Crane 实现 QoS 感知调度
- [Default-scheduler 调度策略配置](../../调度组件概述/原生调度器/Default-scheduler%20调度策略配置/tccli%20操作.md) — 原生调度策略

## 控制台替代

登录 [容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) → 选择集群 **cls-xxxxxxxx** → **节点管理** → **PriorityClass** → **新建** → 填写名称、优先级数值、是否设为全局默认、抢占策略 → **创建 PriorityClass**。
