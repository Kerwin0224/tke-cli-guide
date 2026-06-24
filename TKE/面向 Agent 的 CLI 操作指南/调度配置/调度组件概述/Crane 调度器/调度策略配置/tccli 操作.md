# 调度策略配置（tccli）

> 对照官方：[调度策略配置](https://cloud.tencent.com/document/product/457/110349) · page_id `110349`

## 概述

安装 CraneScheduler 后，通过 `SchedulingPolicy` CRD 定义调度策略，再为节点打标签、为工作负载加注解建立绑定关系，实现 QoS 感知调度、CPU Burst、内存精细管理等调度优化。

## 前置条件

- [环境准备](../../../../环境准备.md)
- 已安装 [Crane 调度器](../Crane%20调度器介绍/tccli%20操作.md)（`DescribeAddon --AddonName CraneScheduler` 状态为 `Succeeded`）

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
#    tke:DescribeClusters, tke:DescribeClusterKubeconfig, tke:DescribeAddon
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
# 6. 确认 CraneScheduler 已安装
tccli tke DescribeAddon --region ap-guangzhou \
    --ClusterId cls-xxxxxxxx \
    --AddonName CraneScheduler
# expected: Status: "Succeeded"
```

**预期输出**：

```json
{
    "AddonName": "CraneScheduler",
    "Status": "Succeeded"
}
```

```bash
# 7. 获取 kubeconfig 并验证 kubectl 可达
tccli tke DescribeClusterKubeconfig --region ap-guangzhou --ClusterId cls-xxxxxxxx \
    --output json | jq -r '.Kubeconfig' > ~/.kube/config-cls-xxxxxxxx
export KUBECONFIG=~/.kube/config-cls-xxxxxxxx
```

```json
{
  "Kubeconfig": "<Kubeconfig>",
  "RequestId": "<RequestId>"
}
```

```bash
kubectl cluster-info
# expected: Kubernetes control plane is running
```

**预期输出**：

```text
Kubernetes control plane is running at https://cls-xxxxxxxx.ccs.tencent-cloud.com
```

```bash
# 8. 确认 CRD 可用
kubectl get crd schedulingpolicies.scheduling.crane.io
# expected: CREATED AT 时间
```

**预期输出**：

```text
NAME                                         CREATED AT
schedulingpolicies.scheduling.crane.io       2024-01-01T00:00:00Z
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 安装 CraneScheduler | `InstallAddon --AddonName CraneScheduler` | 是 |
| 查看组件状态 | `DescribeAddon --AddonName CraneScheduler` | 是 |
| 配置调度策略 | kubectl — 创建 SchedulingPolicy CRD 实例 | — |

## 操作步骤

### 步骤 1：创建 SchedulingPolicy

#### 选择依据

- **SchedulingPolicy** 是 Crane 调度器的核心 CRD，定义节点筛选规则和调度参数。一个 SchedulingPolicy 可被多个命名空间的工作负载复用。
- `spec.schedulePolicy` 中的 `nodeSelector`、`tolerations` 规则类似于原生 K8s 调度器，但由 Crane 调度器二次裁决。
- 策略粒度：建议为每类场景（如高优在线、低优离线）创建独立 SchedulingPolicy，避免规则混淆。

```bash
# 需 kubectl 可达环境 — 创建 SchedulingPolicy
cat > scheduling-policy-high.yaml <<'EOF'
apiVersion: scheduling.crane.io/v1alpha1
kind: SchedulingPolicy
metadata:
  name: high-priority-policy
spec:
  schedulePolicy:
    nodeSelector:
    - key: workload-type
      operator: In
      values:
      - online
    tolerations:
    - key: high-priority
      operator: Exists
      effect: NoSchedule
EOF

kubectl apply -f scheduling-policy-high.yaml
# expected: schedulingpolicy.scheduling.crane.io/high-priority-policy created
```

**预期输出**：

```text
schedulingpolicy.scheduling.crane.io/high-priority-policy created
```

### 步骤 2：为节点打标签

```bash
# 需 kubectl 可达环境 — 为目标节点打标签
kubectl label node NODE_NAME_1 workload-type=online
# expected: node/NODE_NAME_1 labeled

kubectl label node NODE_NAME_2 workload-type=online
# expected: node/NODE_NAME_2 labeled
```

```bash
# 需 kubectl 可达环境 — 可选：为专用节点添加污点
kubectl taint nodes NODE_NAME_1 high-priority=true:NoSchedule
# expected: node/NODE_NAME_1 tainted
```

### 步骤 3：为工作负载添加调度策略注解

#### 选择依据

- 通过 `annotations` 将 SchedulingPolicy 绑定到 Deployment/StatefulSet，格式为 `scheduling.crane.io/policy: POLICY_NAME`。
- **注解 vs 标签**：调度策略绑定使用注解而非标签，因为注解不被选择器（selector）匹配，不影响 Pod 标签管理。

```bash
# 需 kubectl 可达环境 — 创建带策略注解的 Deployment
cat > deploy-with-policy.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: online-service
  annotations:
    scheduling.crane.io/policy: high-priority-policy
spec:
  replicas: 3
  selector:
    matchLabels:
      app: online-service
  template:
    metadata:
      labels:
        app: online-service
    spec:
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

kubectl apply -f deploy-with-policy.yaml
# expected: deployment.apps/online-service created
```

**预期输出**：

```text
deployment.apps/online-service created
```

### 步骤 4：配置 PodQOS 实现 QoS 感知调度（增强）

#### 选择依据

- **PodQOS** CRD 用于定义工作负载的 QoS 等级，配合 SchedulingPolicy 实现基于节点实际资源使用量的调度优化。
- 启用 QoS 感知调度后，Crane 调度器不仅按标签筛选节点，还会根据节点实时资源使用数据（由 QoSAgent 采集）动态调整调度决策。

```bash
# 需 kubectl 可达环境 — 创建 PodQOS
cat > podqos-high.yaml <<'EOF'
apiVersion: scheduling.crane.io/v1alpha1
kind: PodQOS
metadata:
  name: high-priority-qos
spec:
  resourceQOS:
    cpuQOS:
      cpuPriority: 0
    memoryQOS:
      memoryPriority: 0
EOF

kubectl apply -f podqos-high.yaml
# expected: podqos.scheduling.crane.io/high-priority-qos created
```

**预期输出**：

```text
podqos.scheduling.crane.io/high-priority-qos created
```

| 字段 | 说明 | 推荐值 |
|------|------|--------|
| `cpuQOS.cpuPriority` | CPU 优先级，值越小优先级越高 | `0`（高优），`2`（低优） |
| `memoryQOS.memoryPriority` | 内存优先级，值越小优先级越高 | `0`（高优），`2`（低优） |

在 Deployment 中通过注解关联 PodQOS：

```bash
# 需 kubectl 可达环境 — 更新 Deployment 添加 PodQOS 注解
kubectl annotate deployment online-service \
    scheduling.crane.io/qos=high-priority-qos
# expected: deployment.apps/online-service annotated
```

## 验证

### 控制面（tccli）

```bash
# 维度 1：确认组件状态正常
tccli tke DescribeAddon --region ap-guangzhou \
    --ClusterId cls-xxxxxxxx \
    --AddonName CraneScheduler
# expected: Status: "Succeeded"
```

**预期输出**：

```json
{
    "AddonName": "CraneScheduler",
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
# 需 kubectl 可达环境 — 维度 3：验证 SchedulingPolicy 生效
kubectl get schedulingpolicy high-priority-policy -o yaml
# expected: spec 中包含定义的 nodeSelector 和 tolerations
```

**预期输出**：

```yaml
apiVersion: scheduling.crane.io/v1alpha1
kind: SchedulingPolicy
metadata:
  name: high-priority-policy
spec:
  schedulePolicy:
    nodeSelector:
    - key: workload-type
      operator: In
      values:
      - online
    tolerations:
    - effect: NoSchedule
      key: high-priority
      operator: Exists
```

```bash
# 需 kubectl 可达环境 — 维度 4：验证 Pod 按策略调度到正确节点
kubectl get pods -l app=online-service -o wide
# expected: Pod 的 NODE 列全部为带 workload-type=online 标签的节点
```

**预期输出**：

```text
NAME                             READY   STATUS    RESTARTS   AGE   NODE
online-service-xxxxxxxxxx-aaaa   1/1     Running   0          1m    node-ex-001
online-service-xxxxxxxxxx-bbbb   1/1     Running   0          1m    node-ex-002
online-service-xxxxxxxxxx-cccc   1/1     Running   0          1m    node-ex-001
```

```bash
# 需 kubectl 可达环境 — 维度 5：验证注解绑定
kubectl get deployment online-service -o jsonpath='{.metadata.annotations}'
# expected: 含 scheduling.crane.io/policy: high-priority-policy
```

**预期输出**：

```text
{"deployment.kubernetes.io/revision":"1","scheduling.crane.io/policy":"high-priority-policy"}
```

```bash
# 需 kubectl 可达环境 — 维度 6：验证 PodQOS 生效
kubectl get podqos high-priority-qos -o yaml
# expected: spec.resourceQOS 按预期配置
```

**预期输出**：

```yaml
apiVersion: scheduling.crane.io/v1alpha1
kind: PodQOS
metadata:
  name: high-priority-qos
spec:
  resourceQOS:
    cpuQOS:
      cpuPriority: 0
    memoryQOS:
      memoryPriority: 0
```

| 维度 | 命令 | 预期 |
|------|------|------|
| 组件状态 | `DescribeAddon --AddonName CraneScheduler` | `Status: "Succeeded"` |
| SchedulingPolicy | `kubectl get schedulingpolicy high-priority-policy` | 策略已创建 |
| Pod 分布 | `kubectl get pods -l app=online-service -o wide` | 在匹配节点运行 |
| 注解绑定 | `kubectl get deploy online-service -o jsonpath` | 注解正确 |
| PodQOS | `kubectl get podqos high-priority-qos` | CRD 实例存在 |

## 清理

> **警告**：删除 SchedulingPolicy 将导致依赖该策略的工作负载回退到默认调度行为。Pod 不会被驱逐，但新扩缩的 Pod 不再按 Crane 策略调度。删除节点标签可能影响其他工作负载。

### 数据面

数据面资源清理在控制面之前。

```bash
# 需 kubectl 可达环境 — 清理前确认目标资源
kubectl get deploy online-service
kubectl get schedulingpolicy high-priority-policy
kubectl get podqos high-priority-qos
# 确认是待删除的目标资源
```

```text
NAME  STATUS  AGE
...
```

```bash
# 需 kubectl 可达环境 — 删除 Deployment
kubectl delete deploy online-service
# expected: deployment.apps "online-service" deleted

# 需 kubectl 可达环境 — 删除 PodQOS
kubectl delete podqos high-priority-qos
# expected: podqos.scheduling.crane.io "high-priority-qos" deleted

# 需 kubectl 可达环境 — 删除 SchedulingPolicy
kubectl delete schedulingpolicy high-priority-policy
# expected: schedulingpolicy.scheduling.crane.io "high-priority-policy" deleted
```

```bash
# 需 kubectl 可达环境 — 移除节点标签和污点
kubectl label node NODE_NAME_1 workload-type-
kubectl label node NODE_NAME_2 workload-type-
# expected: node/NODE_NAME_1 unlabeled

kubectl taint nodes NODE_NAME_1 high-priority=true:NoSchedule-
# expected: node/NODE_NAME_1 untainted
```

```bash
# 需 kubectl 可达环境 — 验证已删除
kubectl get schedulingpolicy high-priority-policy
# expected: Error from server (NotFound)
```

**预期输出**：

```text
Error from server (NotFound): schedulingpolicies.scheduling.crane.io "high-priority-policy" not found
```

```bash
kubectl get podqos high-priority-qos
# expected: Error from server (NotFound)
```

**预期输出**：

```text
Error from server (NotFound): podqos.scheduling.crane.io "high-priority-qos" not found
```

```bash
kubectl get deploy online-service
# expected: Error from server (NotFound)
```

**预期输出**：

```text
Error from server (NotFound): deployments.apps "online-service" not found
```

### 清理说明

CraneScheduler 组件本身无需卸载，保留可继续为其他工作负载提供调度服务。如需完整清理，参见 [Crane 调度器介绍](../Crane%20调度器介绍/tccli%20操作.md#清理)。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl apply -f` SchedulingPolicy 返回 `error: resource mapping not found` | `kubectl get crd schedulingpolicies.scheduling.crane.io` 确认 CRD 存在 | CraneScheduler 未正确安装或 CRD 未注册 | 检查组件状态：`tccli tke DescribeAddon --region ap-guangzhou --ClusterId cls-xxxxxxxx --AddonName CraneScheduler`；如 `Status` 非 `Succeeded`，等待安装完成或重新安装 |
| Pod 创建后一直 Pending | `kubectl describe pod POD_NAME` 查看 Events，关注 SchedulingPolicy 相关事件 | SchedulingPolicy 的 nodeSelector 无法匹配任何节点 | 检查节点标签：`kubectl get nodes --show-labels \| grep workload-type`；为节点打标签或在策略中添加备选规则 |
| `kubectl annotate` 返回 `NotFound` | `kubectl get deploy DEPLOY_NAME` 确认 Deployment 存在 | Deployment 名称错误或不在当前命名空间 | 检查命名空间：`kubectl get deploy -A \| grep DEPLOY_NAME` |

### 调度策略配置后未生效

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Deployment 注解正确但 Pod 未按策略调度 | `kubectl logs -n crane-system -l app=crane-scheduler --tail=20` 查看调度器日志 | SchedulingPolicy 名称拼写错误或注解格式不对 | 确认注解 key 为 `scheduling.crane.io/policy`，值为策略名称（非 namespace 限定） |
| PodQOS 配置后无效果 | `kubectl describe podqos PODQOS_NAME` 确认配置；`kubectl get pods -n crane-system \| grep qos-agent` 确认 QoSAgent 运行 | QoSAgent 未安装或未运行 | 安装 QoSAgent：参见 [QoSAgent](../../../Qos%20感知调度/QoSAgent/tccli%20操作.md) |
| 部分节点上 Pod 调度失败 | `kubectl describe node NODE_NAME \| grep -E "Taints:|Labels:"` | 节点带有意外污点或标签不匹配 | 移除无关污点或补充调度策略中的 tolerations |

## 下一步

- [QoSAgent](../../../Qos%20感知调度/QoSAgent/tccli%20操作.md) — 安装节点资源采集组件
- [CPU Burst](../../../Qos%20感知调度/CPU%20Burst/tccli%20操作.md) — 启用 CPU 突发
- [内存精细调度](../../../Qos%20感知调度/内存精细调度/tccli%20操作.md) — 内存 NUMA 感知调度
- [自定义资源优先级](../../../业务优先级保障调度/自定义资源优先级/tccli%20操作.md) — PriorityClass 优先级调度

## 控制台替代

登录 [容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) → 选择集群 → **组件管理** → **CraneScheduler** → **调度策略** 页签 → 新建 SchedulingPolicy → 为工作负载绑定策略。
