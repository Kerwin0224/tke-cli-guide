# 安装 CoScheduling 实现批调度（tccli）

> 对照官方：[安装 CoScheduling 实现批调度](https://cloud.tencent.com/document/product/457/106150) · page_id `106150`

## 概述

CoScheduling（Gang Scheduling）是一种批调度策略，确保一组关联 Pod 在满足所有成员资源需求后同时被调度执行，避免因部分 Pod 等待资源而其他 Pod 已占用资源导致的死锁和资源浪费。

典型场景：
- **分布式训练**：MPI/Horovod 分布式训练要求所有 Worker 同时启动（如 4 卡 × 8 节点的 All-Reduce 通信）
- **大数据批处理**：Spark/Flink 作业的 JobManager + TaskManager 需成组调度
- **推理服务**：模型加载 + API Server + Sidecar 需同时就绪

CoScheduling 提供两种调度策略：
- **Strict**：所有 Pod 必须同时满足资源才能调度（All-or-Nothing）
- **NonStrict**：最少 minMember 个 Pod 满足资源即可调度，剩余的排队等待

> **kubectl 数据面不可达**：CAM拒绝公网端点(strategyId:240463971)，内网需VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

## 前置条件

### 环境检查

```bash
tccli --version
# expected: tccli version >= 1.0.0

tccli configure list
# expected: secretId/secretKey 已配置，region: ap-guangzhou

tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]'
# expected: TotalCount >= 1, ClusterStatus "Running", ClusterVersion >= 1.20

# 检查集群是否已有 CoScheduling 组件
tccli tke DescribeAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName coscheduling
# expected: Status "Succeeded"（已安装）或 ResourceNotFound（未安装）
```

```json
{
  "TotalCount": "<TotalCount>",
  "Clusters": "<Clusters>",
  "ClusterId": "<ClusterId>",
  "ClusterName": "<ClusterName>",
  "ClusterDescription": "<ClusterDescription>",
  "ClusterVersion": "<ClusterVersion>"
}
```

### 资源检查

```bash
# 查看集群可调度资源
tccli tke DescribeClusterInstances --region <Region> \
    --ClusterId CLUSTER_ID \
    --Limit 50
# expected: InstanceSet 含节点实例信息

# 查看已安装组件列表
tccli tke DescribeAddon --region <Region> \
    --ClusterId CLUSTER_ID
# expected: Addons 列表，含各组件版本和状态
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

## 控制台与 CLI 参数映射

| 控制台操作 | tccli/kubectl 命令 | 幂等 |
|-----------|-------------------|:--:|
| 查看可用组件 | `tccli tke DescribeAddon --region <Region> --ClusterId CLUSTER_ID` | 是 |
| 查看组件详情 | `tccli tke DescribeAddon --region <Region> --ClusterId CLUSTER_ID --AddonName coscheduling` | 是 |
| 安装 CoScheduling | `tccli tke InstallAddon --region <Region> --cli-input-json file://install.json` | 否 |
| 升级组件 | `tccli tke UpdateAddon --region <Region> --AddonName coscheduling` | 否 |
| 创建 PodGroup | `kubectl apply -f podgroup.yaml`（需 VPN/IOA） | 是 |
| 查看 PodGroup 状态 | `kubectl describe podgroup PG_NAME -n NAMESPACE`（需 VPN/IOA） | 是 |
| 查看调度事件 | `kubectl get events -n NAMESPACE \| grep PG_NAME`（需 VPN/IOA） | 是 |
| 删除 PodGroup | `kubectl delete podgroup PG_NAME -n NAMESPACE`（需 VPN/IOA） | 否 |

## 操作步骤

### 步骤 1：安装 CoScheduling 组件

#### 选择依据

- **CoScheduling** vs **默认调度器**：默认调度器对同一 Deployment 的 Pod 逐个调度，先调度的 Pod 可能占住资源等待其他 Pod；CoScheduling 等待全组资源就绪后原子调度
- **Strict** vs **NonStrict 策略**：Strict 保证所有成员同时运行（适合 MPI 训练）；NonStrict 允许部分先运行（适合 Elastics 训练）
- **调度器名称**：安装后 Pod 需显式指定 `schedulerName: coscheduling` 才使用 Gang Scheduling

```bash
# 查看 CoScheduling 可用版本
tccli tke DescribeAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    | jq '.Addons[] | select(.AddonName=="coscheduling")'
# expected: AddonVersion 列表，选择最新版本

cat > install-coscheduling.json <<'EOF'
{
    "ClusterId": "CLUSTER_ID",
    "AddonName": "coscheduling",
    "AddonVersion": "1.2.0"
}
EOF
tccli tke InstallAddon --region <Region> \
    --cli-input-json file://install-coscheduling.json
# expected: exit 0（安装异步执行）
```

```json
{
  "Addons": "<Addons>",
  "AddonName": "<AddonName>",
  "AddonVersion": "<AddonVersion>",
  "RawValues": "<RawValues>",
  "Phase": "<Phase>",
  "Reason": "<Reason>"
}
```

#### 增强配置

```bash
# 等待组件安装完成（轮询）
for i in $(seq 1 30); do
    STATUS=$(tccli tke DescribeAddon --region <Region> \
        --ClusterId CLUSTER_ID \
        --AddonName coscheduling 2>/dev/null \
        | jq -r '.Status')
    echo "CoScheduling status: $STATUS"
    if [ "$STATUS" = "Succeeded" ]; then
        echo "CoScheduling installed successfully"
        break
    fi
    sleep 10
done
# expected: CoScheduling status: Succeeded
```

### 步骤 2：创建 PodGroup

#### 选择依据

- **minMember**：最小成员数，Strict 模式下等于期望 Pod 总数
- **minResources**：每个 Pod 的最低资源需求（聚合计算总需求）
- **scheduleTimeoutSeconds**：等待资源超时时间（超时后 Pod 会被打上 `Unschedulable` 标记）

```yaml
apiVersion: scheduling.sigs.k8s.io/v1alpha1
kind: PodGroup
metadata:
  name: mpi-training
  namespace: NAMESPACE
spec:
  minMember: 4
  scheduleTimeoutSeconds: 600
  minResources:
    cpu: "16"
    memory: "64Gi"
    nvidia.com/gpu: "4"
---
apiVersion: scheduling.sigs.k8s.io/v1alpha1
kind: PodGroup
metadata:
  name: spark-batch
  namespace: NAMESPACE
spec:
  minMember: 3
  scheduleTimeoutSeconds: 300
  minResources:
    cpu: "8"
    memory: "32Gi"
```

```bash
kubectl apply -f podgroup.yaml
# expected: podgroup.scheduling.sigs.k8s.io/mpi-training created
# expected: podgroup.scheduling.sigs.k8s.io/spark-batch created

kubectl get podgroup -n NAMESPACE
# expected: NAME STATUS MINMEMBER RUNNING AGE
```

### 步骤 3：创建使用 CoScheduling 的 Pod/Deployment

#### 最小创建（MPI 训练示例）

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: mpi-training-job
  namespace: NAMESPACE
  labels:
    pod-group.scheduling.sigs.k8s.io: mpi-training
spec:
  parallelism: 4
  completions: 4
  template:
    metadata:
      labels:
        pod-group.scheduling.sigs.k8s.io: mpi-training
    spec:
      schedulerName: coscheduling
      containers:
      - name: mpi-worker
        image: mpich/horovod:latest
        command: ["mpirun", "-np", "4", "python", "train.py"]
        resources:
          requests:
            cpu: "4"
            memory: "16Gi"
            nvidia.com/gpu: "1"
          limits:
            cpu: "4"
            memory: "16Gi"
            nvidia.com/gpu: "1"
      restartPolicy: Never
```

```bash
kubectl apply -f mpi-job.yaml
# expected: job.batch/mpi-training-job created
```

#### 增强配置（Spark 批处理示例）

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: spark-jobmanager
  namespace: NAMESPACE
  labels:
    pod-group.scheduling.sigs.k8s.io: spark-batch
spec:
  schedulerName: coscheduling
  containers:
  - name: spark-master
    image: spark:3.4.0
    ports:
    - containerPort: 7077
      name: spark-master
    - containerPort: 8080
      name: spark-ui
    resources:
      requests:
        cpu: "2"
        memory: "8Gi"
      limits:
        cpu: "4"
        memory: "16Gi"
---
apiVersion: v1
kind: Pod
metadata:
  name: spark-taskmanager-0
  namespace: NAMESPACE
  labels:
    pod-group.scheduling.sigs.k8s.io: spark-batch
spec:
  schedulerName: coscheduling
  containers:
  - name: spark-worker
    image: spark:3.4.0
    resources:
      requests:
        cpu: "3"
        memory: "12Gi"
      limits:
        cpu: "4"
        memory: "16Gi"
```

```bash
kubectl apply -f spark-pods.yaml
# expected: 3 个 Pod 同时进入调度队列，资源满足后同时启动
```

### 步骤 4：验证批调度效果

```bash
# 查看 PodGroup 调度详情
kubectl describe podgroup mpi-training -n NAMESPACE
# expected: ScheduleStartTime 记录调度启动时间
#          status.conditions 含 Scheduled True
#          status.scheduled >= minMember

# 查看调度事件时间线
kubectl get events -n NAMESPACE --sort-by='.lastTimestamp' \
    | grep -E "mpi-training|Schedu"
# expected: 所有 Pod 调度时间戳相近（秒级差异）

# 验证 Pod 同时 Running（而非逐个启动）
kubectl get pods -n NAMESPACE \
    -l pod-group.scheduling.sigs.k8s.io=mpi-training \
    -o custom-columns=NAME:.metadata.name,STATUS:.status.phase,AGE:.metadata.creationTimestamp
# expected: 所有 Pod 状态 Running，创建时间戳接近
```

```text
NAME  STATUS  AGE
...
```

## 验证

### 控制面（tccli）

```bash
# 验证组件状态
tccli tke DescribeAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName coscheduling
# expected: Status "Succeeded", AddonVersion 显示已安装版本
```

```json
{
  "Addons": "<Addons>",
  "AddonName": "<AddonName>",
  "AddonVersion": "<AddonVersion>",
  "RawValues": "<RawValues>",
  "Phase": "<Phase>",
  "Reason": "<Reason>"
}
```

### 数据面（需 VPN/IOA）

```bash
# 验证 PodGroup 调度成功
kubectl get podgroup mpi-training -n NAMESPACE -o yaml \
    | yq '.status'
# expected: phase: Running, scheduled: 4, succeeded: 4

# 验证所有 Pod 同时 Running
kubectl get pods -n NAMESPACE \
    -l pod-group.scheduling.sigs.k8s.io=mpi-training --no-headers \
    | awk '{print $3}' | sort | uniq -c
# expected: 4 Running

# 对比默认调度器的行为：
# 创建相同资源的 Job 但 schedulerName 使用默认 default-scheduler
kubectl get pods -n NAMESPACE -l pod-group.scheduling.sigs.k8s.io=mpi-training \
    -o jsonpath='{range .items[*]}{.metadata.name} {.status.startTime}{"\n"}{end}'
# expected: 所有 Pod startTime 差异 < 5 秒（CoScheduling）
```

```text
NAME  STATUS  AGE
...
```

## 清理

### 数据面（需 VPN/IOA）

```bash
# 删除 Job/Pod
kubectl delete job mpi-training-job -n NAMESPACE
# expected: job "mpi-training-job" deleted
# ⚠️ 警告：删除 Job 会级联删除关联 Pod，请确认训练已完成

kubectl delete pod spark-jobmanager spark-taskmanager-0 -n NAMESPACE
# expected: pods deleted

# 删除 PodGroup
kubectl delete podgroup mpi-training spark-batch -n NAMESPACE
# expected: podgroups deleted
# ⚠️ 警告：删除 PodGroup 后，使用该 PodGroup 的新 Pod 将无法调度
```

### 控制面（tccli）

```bash
# 卸载 CoScheduling 组件
tccli tke DeleteAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName coscheduling
# expected: exit 0
# ⚠️ 警告：删除后，使用 schedulerName: coscheduling 的已有 Pod 仍可正常运行，
# 但新调度请求将失败（调度器不存在）
```

```bash
# 验证卸载
tccli tke DescribeAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName coscheduling 2>&1
# expected: ResourceNotFound（组件已卸载）
```

```json
{
  "Addons": "<Addons>",
  "AddonName": "<AddonName>",
  "AddonVersion": "<AddonVersion>",
  "RawValues": "<RawValues>",
  "Phase": "<Phase>",
  "Reason": "<Reason>"
}
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| PodGroup 状态 Phase Pending 超时 | `kubectl describe podgroup PG_NAME -n NAMESPACE` 查看 events | 集群资源不足以满足 minMember 所有 Pod 的资源需求 | 降低 minMember、增加集群节点或调整 Pod resource requests |
| Pod 未使用 CoScheduling 调度 | `kubectl describe pod POD_NAME -n NAMESPACE \| grep "Scheduler Name"` | Pod spec 未指定 `schedulerName: coscheduling` | 在 Pod/Deployment/Job 模板中设置 `schedulerName: coscheduling` |
| Pod 事件 "podgroup not found" | `kubectl get podgroup -A` 搜索 PodGroup | PodGroup 未创建或 namespace 不匹配 | 确认 PodGroup 与 Pod 在同一 namespace，且 label 正确匹配 |
| 组件安装失败，状态 Error | `tccli tke DescribeAddon --AddonName coscheduling --ClusterId CLUSTER_ID` | 集群版本不兼容（需 K8s >= 1.20）或集群资源不足 | 升级集群版本或等待集群节点就绪后重试 |

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Pod 被 Partial 调度 | `kubectl describe podgroup PG_NAME -n NAMESPACE` 查看 scheduled vs minMember | minMember 设置小于实际 Pod 数，NonStrict 模式 | 设置 `minMember` 等于实际 Pod 数并确保 spec 使用 Strict 策略 |
| 调度后 Pod 立即被驱逐 | `kubectl describe pod POD_NAME -n NAMESPACE \| grep -A5 "Events"` | 节点资源预留不足，调度后 OOM 或 DiskPressure | 增加 Pod resource limits 匹配 nodes allocatable 余量 |
| PodGroup minResources 聚合计算错误 | `kubectl get podgroup PG_NAME -o yaml \| yq '.spec.minResources'` | minResources 为每个 Pod 需求而非总需求 | 设置 minResources 为单个 Pod 的需求；CoScheduling 自动乘以 minMember |
| 组件升级后旧 PodGroup 不生效 | `kubectl get podgroup PG_NAME -o yaml \| grep apiVersion` | API 版本不兼容（v1alpha1 → v1beta1） | 更新 PodGroup YAML 的 apiVersion 到新版并重建 |

## 下一步

- [原生节点提升集群装箱率](../原生节点提升集群装箱率/tccli%20操作.md) -- page_id `97925`
- [调度组件概述](../../../调度配置/调度组件概述/tccli%20操作.md) -- page_id `111862`
- [资源利用率优化调度](../../../调度配置/资源利用率优化调度/tccli%20操作.md) -- page_id `118259`

## 控制台替代

[容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) -> 选择集群 -> 组件管理 -> Addon 列表 -> 安装 `coscheduling`。
控制台支持组件安装和升级，PodGroup YAML 创建和执行需通过 kubectl 操作。
