# 基于 Kueue 实现批调度和队列管理（tccli）

> 对照官方：[基于 Kueue 实现批调度和队列管理](https://cloud.tencent.com/document/product/457/126173) · page_id `126173`

## 概述

TKE 通过集成 Kueue 组件（AddonName=`Kueue`）实现批调度和队列管理。Kueue 是 Kubernetes 原生的资源配额与队列管理项目，支持 ResourceFlavor（资源池）、ClusterQueue（集群队列）、LocalQueue（命名空间队列）三层模型，实现工作负载的排队、配额管控和公平调度。

## 前置条件

- [环境准备](../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置

# 3. 检查 kubectl 版本
kubectl version --client
# expected: Client Version >= 1.30.0

# 4. 检查 CAM 权限 — 需要以下 Action 名
#    tke:DescribeClusters, tke:InstallAddon, tke:DescribeAddon
#    tke:DeleteAddon, tke:DescribeClusterKubeconfig
# 验证：执行 DescribeClusters 确认权限
tccli tke DescribeClusters --region ap-guangzhou
# expected: exit 0，返回集群列表
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
            "ClusterStatus": "Running"
        }
    ]
}
```

```bash
# 6. 获取 kubeconfig 并验证 kubectl 可达
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
# 7. 确认集群有足够节点（至少 1 个）
kubectl get nodes
# expected: 至少 1 个节点 STATUS 为 Ready
```

**预期输出**：

```text
NAME          STATUS   ROLES    AGE   VERSION
node-ex-001   Ready    <none>   1d    v1.30.0
```

```bash
# 8. 检查现有组件，确认未重复安装
tccli tke DescribeAddon --region ap-guangzhou \
    --ClusterId cls-xxxxxxxx \
    --AddonName Kueue
# expected: 如已安装返回组件详情；未安装返回 ResourceNotFound 或空
```

**预期输出**（未安装时）：

```json
{
    "Error": {
        "Code": "ResourceNotFound",
        "Message": "Addon not found"
    }
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 安装 Kueue | `InstallAddon --AddonName Kueue` | 是 |
| 查看组件状态 | `DescribeAddon --AddonName Kueue` | 是 |
| 升级组件版本 | `UpdateAddon --AddonName Kueue` | 否 |
| 卸载组件 | `DeleteAddon --AddonName Kueue` | 否 |

## Key parameters

`InstallAddon` 的主要参数：

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `ClusterId` | String | 是 | 目标集群 ID，格式 `cls-xxxxxxxx` | 集群不存在 → `InvalidParameter.ClusterId` |
| `AddonName` | String | 是 | `Kueue`，大小写敏感 | 名称错误 → `InvalidParameter.AddonName` |
| `AddonVersion` | String | 否 | 组件版本号。不指定则安装最新版 | 版本不存在 → `InvalidParameter.AddonVersion` |

## 操作步骤

### 步骤 1：安装 Kueue 组件

#### 选择依据

- **Kueue** 适用于批量工作负载（Job、JobSet、MPIJob 等）的资源排队场景，通过队列配额控制多租户资源竞争。
- 与传统 PriorityClass 的区别：PriorityClass 实现 Pod 级别的抢占，Kueue 实现工作负载级别的排队调度。两者可配合使用。
- **安装时机**：建议在集群创建后、部署批处理框架前安装。

#### 最小安装（仅必填字段）

```bash
tccli tke InstallAddon --region ap-guangzhou \
    --ClusterId cls-xxxxxxxx \
    --AddonName Kueue
# expected: exit 0，返回 RequestId，组件开始安装
```

**预期输出**：

```json
{
    "RequestId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
}
```

#### 增强配置（指定版本）

```bash
tccli tke InstallAddon --region ap-guangzhou \
    --ClusterId cls-xxxxxxxx \
    --AddonName Kueue \
    --AddonVersion ADDON_VERSION
# expected: exit 0，返回 RequestId
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `ap-guangzhou` | 地域 | 如 `ap-guangzhou` | `tccli configure list` |
| `cls-xxxxxxxx` | 目标集群 ID | 格式 `cls-xxxxxxxx` | `tccli tke DescribeClusters --region ap-guangzhou` |
| `ADDON_VERSION` | 组件版本 | 如 `v0.6.0` | `tccli tke DescribeAddonValues --region ap-guangzhou --ClusterId cls-xxxxxxxx --AddonName Kueue` |

### 步骤 2：轮询组件安装状态

```bash
tccli tke DescribeAddon --region ap-guangzhou \
    --ClusterId cls-xxxxxxxx \
    --AddonName Kueue
# expected: Status: "Succeeded"
```

**预期输出**：

```json
{
    "AddonName": "Kueue",
    "AddonVersion": "v0.6.0",
    "Status": "Succeeded",
    "RequestId": "b2c3d4e5-f6a7-8901-bcde-f12345678901"
}
```

### 步骤 3：创建 ResourceFlavor

#### 选择依据

- **ResourceFlavor** 定义一组可分配资源的节点标签集合。一个 ClusterQueue 可引用多个 ResourceFlavor，实现"借用"机制。
- 如集群节点无特殊标签，创建一个无 nodeLabels 的 ResourceFlavor 表示"所有节点资源"。

```bash
# 需 kubectl 可达环境 — 确认 Kueue Pod 运行
kubectl get pods -n kueue-system
# expected: kueue-controller-manager-xxx Pod 状态为 Running
```

**预期输出**：

```text
NAME                                        READY   STATUS    RESTARTS   AGE
kueue-controller-manager-xxxxxxxxxx-xxxxx    2/2     Running   0          2m
```

#### 最小创建

```bash
# 需 kubectl 可达环境 — 创建覆盖所有节点的 ResourceFlavor
cat > resource-flavor-default.yaml <<'EOF'
apiVersion: kueue.x-k8s.io/v1beta1
kind: ResourceFlavor
metadata:
  name: default-flavor
EOF

kubectl apply -f resource-flavor-default.yaml
# expected: resourceflavor.kueue.x-k8s.io/default-flavor created
```

**预期输出**：

```text
resourceflavor.kueue.x-k8s.io/default-flavor created
```

### 步骤 4：创建 ClusterQueue

#### 选择依据

- **ClusterQueue** 是集群级资源池，分配 CPU、内存、GPU 等配额给一组 ResourceFlavor。
- `spec.resourceGroups` 中 `nominalQuota` 定义该队列的可借用上限，`borrowingLimit` 限制可从其他队列借用的量。
- 多个 ClusterQueue 间按 `.spec.preemption` 配置抢占规则，实现跨队列优先级。

#### 最小创建

```bash
# 需 kubectl 可达环境 — 创建小配额的 ClusterQueue
cat > cluster-queue-dev.yaml <<'EOF'
apiVersion: kueue.x-k8s.io/v1beta1
kind: ClusterQueue
metadata:
  name: cluster-queue-dev
spec:
  namespaceSelector: {}
  resourceGroups:
  - coveredResources: ["cpu", "memory"]
    flavors:
    - name: default-flavor
      resources:
      - name: cpu
        nominalQuota: 4
      - name: memory
        nominalQuota: 8Gi
EOF

kubectl apply -f cluster-queue-dev.yaml
# expected: clusterqueue.kueue.x-k8s.io/cluster-queue-dev created
```

**预期输出**：

```text
clusterqueue.kueue.x-k8s.io/cluster-queue-dev created
```

#### 增强配置（多队列层级）

```bash
# 需 kubectl 可达环境 — 创建生产级大配额 ClusterQueue
cat > cluster-queue-prod.yaml <<'EOF'
apiVersion: kueue.x-k8s.io/v1beta1
kind: ClusterQueue
metadata:
  name: cluster-queue-prod
spec:
  namespaceSelector: {}
  preemption:
    withinClusterQueue: LowerPriority
    reclaimWithinCohort: LowerPriority
  resourceGroups:
  - coveredResources: ["cpu", "memory"]
    flavors:
    - name: default-flavor
      resources:
      - name: cpu
        nominalQuota: 16
      - name: memory
        nominalQuota: 32Gi
EOF

kubectl apply -f cluster-queue-prod.yaml
# expected: clusterqueue.kueue.x-k8s.io/cluster-queue-prod created
```

### 步骤 5：创建 LocalQueue

#### 选择依据

- **LocalQueue** 是命名空间级队列，用户实际提交工作的入口。一个 LocalQueue 必须属于一个 ClusterQueue。
- 建议每个团队或命名空间一个 LocalQueue，实现租户隔离。

```bash
# 需 kubectl 可达环境 — 创建 LocalQueue 绑定到 cluster-queue-dev
cat > local-queue-dev.yaml <<'EOF'
apiVersion: kueue.x-k8s.io/v1beta1
kind: LocalQueue
metadata:
  name: user-queue
  namespace: default
spec:
  clusterQueue: cluster-queue-dev
EOF

kubectl apply -f local-queue-dev.yaml
# expected: localqueue.kueue.x-k8s.io/user-queue created
```

**预期输出**：

```text
localqueue.kueue.x-k8s.io/user-queue created
```

### 步骤 6：提交 Job 到队列

```bash
# 需 kubectl 可达环境 — 创建带 Kueue 标签的 Job
cat > kueue-job.yaml <<'EOF'
apiVersion: batch/v1
kind: Job
metadata:
  name: sample-job
  namespace: default
  labels:
    kueue.x-k8s.io/queue-name: user-queue
spec:
  parallelism: 3
  completions: 3
  suspend: true
  template:
    spec:
      containers:
      - name: sample
        image: busybox:1.36
        command: ["sh", "-c", "echo 'Processing...' && sleep 10"]
      restartPolicy: Never
EOF

kubectl apply -f kueue-job.yaml
# expected: job.batch/sample-job created
```

**预期输出**：

```text
job.batch/sample-job created
```

| 关键配置 | 说明 |
|----------|------|
| `labels: kueue.x-k8s.io/queue-name` | 指定目标 LocalQueue，Kueue 通过此标签识别 |
| `suspend: true` | Job 初始暂停，由 Kueue 控制器根据配额决定何时恢复 |

## 验证

### 控制面（tccli）

```bash
# 维度 1：确认组件状态
tccli tke DescribeAddon --region ap-guangzhou \
    --ClusterId cls-xxxxxxxx \
    --AddonName Kueue
# expected: Status: "Succeeded"
```

**预期输出**：

```json
{
    "AddonName": "Kueue",
    "Status": "Succeeded"
}
```

```bash
# 维度 2：确认集群状态正常
tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]'
# expected: ClusterStatus: "Running"
```

**预期输出**：

```json
{
    "ClusterId": "cls-xxxxxxxx",
    "ClusterStatus": "Running"
}
```

### 数据面

```bash
# 需 kubectl 可达环境 — 维度 3：确认 Kueue Pod 运行
kubectl get pods -n kueue-system
# expected: kueue-controller-manager-xxx 状态 Running
```

**预期输出**：

```text
NAME                                        READY   STATUS    RESTARTS   AGE
kueue-controller-manager-xxxxxxxxxx-xxxxx    2/2     Running   0          2m
```

```bash
# 需 kubectl 可达环境 — 维度 4：验证 ResourceFlavor
kubectl get resourceflavor
# expected: default-flavor 在列表中
```

**预期输出**：

```text
NAME              AGE
default-flavor    5m
```

```bash
# 需 kubectl 可达环境 — 维度 5：验证 ClusterQueue
kubectl get clusterqueue
# expected: cluster-queue-dev、cluster-queue-prod 在列表中，状态 Active
```

**预期输出**：

```text
NAME                  AGE
cluster-queue-dev     5m
cluster-queue-prod    5m
```

```bash
# 需 kubectl 可达环境 — 维度 6：验证 LocalQueue
kubectl get localqueue -A
# expected: user-queue 在列表中，PENDING WORKLOADS 显示待处理数
```

**预期输出**：

```text
NAMESPACE   NAME         CLUSTERQUEUE         PENDING WORKLOADS
default     user-queue   cluster-queue-dev    0
```

```bash
# 需 kubectl 可达环境 — 维度 7：验证 Job 状态
kubectl get job sample-job -n default
# expected: Job 已由 Kueue 恢复运行，COMPLETIONS 按预期
```

**预期输出**：

```text
NAME         COMPLETIONS   DURATION   AGE
sample-job   3/3           30s        2m
```

| 维度 | 命令 | 预期 |
|------|------|------|
| 组件状态 | `DescribeAddon --AddonName Kueue` | `Status: "Succeeded"` |
| 集群状态 | `DescribeClusters` | `ClusterStatus: "Running"` |
| Pod 运行 | `kubectl get pods -n kueue-system` | Running |
| ResourceFlavor | `kubectl get resourceflavor` | default-flavor 存在 |
| ClusterQueue | `kubectl get clusterqueue` | Active |
| LocalQueue | `kubectl get localqueue -A` | 队列就绪 |
| Job | `kubectl get job sample-job -n default` | 正常完成/运行 |

## 清理

> **警告**：卸载 Kueue 后将失去队列管理和配额控制能力。所有在队列中等待的工作负载将直接以普通 Kubernetes Job 方式调度（不受配额限制）。卸载后 CRD（ResourceFlavor、ClusterQueue、LocalQueue）不会自动删除，需手动清理。

### 数据面

数据面资源清理在控制面之前。

```bash
# 需 kubectl 可达环境 — 清理前确认目标资源
kubectl get job sample-job -n default
kubectl get localqueue -A
kubectl get clusterqueue
kubectl get resourceflavor
# 确认是待删除的目标资源
```

```text
NAME  STATUS  AGE
...
```

```bash
# 需 kubectl 可达环境 — 删除 Job
kubectl delete job sample-job -n default
# expected: job.batch "sample-job" deleted

# 需 kubectl 可达环境 — 删除 LocalQueue
kubectl delete localqueue user-queue -n default
# expected: localqueue.kueue.x-k8s.io "user-queue" deleted

# 需 kubectl 可达环境 — 删除 ClusterQueue
kubectl delete clusterqueue cluster-queue-dev cluster-queue-prod
# expected: clusterqueue.kueue.x-k8s.io "cluster-queue-dev" deleted
#          clusterqueue.kueue.x-k8s.io "cluster-queue-prod" deleted

# 需 kubectl 可达环境 — 删除 ResourceFlavor
kubectl delete resourceflavor default-flavor
# expected: resourceflavor.kueue.x-k8s.io "default-flavor" deleted
```

```bash
# 需 kubectl 可达环境 — 验证已删除
kubectl get localqueue -A
# expected: No resources found
```

**预期输出**：

```text
No resources found
```

```bash
kubectl get clusterqueue
# expected: No resources found
```

**预期输出**：

```text
No resources found
```

```bash
kubectl get resourceflavor
# expected: No resources found
```

**预期输出**：

```text
No resources found
```

### 控制面（tccli）

```bash
# 清理前状态检查
tccli tke DescribeAddon --region ap-guangzhou \
    --ClusterId cls-xxxxxxxx \
    --AddonName Kueue
# expected: 确认组件当前状态
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

```bash
# 卸载组件
tccli tke DeleteAddon --region ap-guangzhou \
    --ClusterId cls-xxxxxxxx \
    --AddonName Kueue
# expected: exit 0，返回 RequestId
```

```bash
# 验证已卸载
tccli tke DescribeAddon --region ap-guangzhou \
    --ClusterId cls-xxxxxxxx \
    --AddonName Kueue
# expected: ResourceNotFound 或 AddonName 不在列表中
```

**预期输出**：

```json
{
    "Error": {
        "Code": "ResourceNotFound",
        "Message": "Addon not found"
    }
}
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `InstallAddon` 返回 `InvalidParameter.AddonName` | 检查 AddonName 大小写：`Kueue` 而非 `kueue` | AddonName 大小写不匹配 | 使用精确名称 `Kueue`（K 大写） |
| `InstallAddon` 返回 `FailedOperation.AddonAlreadyExists` | `tccli tke DescribeAddon --region ap-guangzhou --ClusterId cls-xxxxxxxx --AddonName Kueue` | 组件已安装 | 如状态异常，先 `DeleteAddon` 卸载后重新安装 |
| `kubectl apply -f` ResourceFlavor/ClusterQueue 返回资源类型未找到 | `kubectl get crd \| grep kueue` | Kueue CRD 未注册，组件安装未完成 | 等待安装完成（`DescribeAddon` 状态为 `Succeeded`）后重试 |
| Job 提交后一直 `suspend: true` 未恢复 | `kubectl describe localqueue user-queue -n default` 查看队列状态 | ClusterQueue 配额不足，或有更高优先级任务在排队 | 增加 ClusterQueue `nominalQuota`；或等待当前任务完成释放配额 |

### 安装成功但功能异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| ClusterQueue 状态为 `Inactive` | `kubectl describe clusterqueue CLUSTER_QUEUE_NAME` 查看 Conditions | 引用的 ResourceFlavor 不存在或配置错误 | 确认 ResourceFlavor 已创建且名称正确 |
| LocalQueue 无法提交工作 | `kubectl describe localqueue LOCAL_QUEUE_NAME -n NAMESPACE` | LocalQueue 绑定的 ClusterQueue 不存在或容量已满 | 检查 ClusterQueue：`kubectl get clusterqueue`；如有待处理 workload：`kubectl get workload -A` |
| Kueue controller Pod CrashLoopBackOff | `kubectl logs -n kueue-system -l control-plane=controller-manager --tail=50` | 启动配置错误或依赖资源不可用 | 根据日志修复；如无法定位 → 卸载后重新安装 |
| Job 完成但资源未释放 | `kubectl get workload -A \| grep JOB_NAME` | Job 对应的 Workload 对象未自动清理 | 手动删除 Workload：`kubectl delete workload WORKLOAD_NAME -n NAMESPACE` |

## 下一步

- [自定义资源优先级](../自定义资源优先级/tccli%20操作.md) — PriorityClass 优先级抢占调度
- [可抢占式 Job](../可抢占式%20Job/tccli%20操作.md) — 利用优先级实现可抢占批处理
- [Crane 调度器介绍](../../调度组件概述/Crane%20调度器/Crane%20调度器介绍/tccli%20操作.md) — 配合 Crane 实现智能调度
- [自动规整集群资源](../../资源利用率优化调度/自动规整集群资源/tccli%20操作.md) — DeScheduler 资源整理

## 控制台替代

登录 [容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) → 选择集群 → **组件管理** → **新建** → 搜索 `Kueue` → 选择版本并安装 → 通过 **CRD 管理** 创建 ResourceFlavor/ClusterQueue/LocalQueue。
