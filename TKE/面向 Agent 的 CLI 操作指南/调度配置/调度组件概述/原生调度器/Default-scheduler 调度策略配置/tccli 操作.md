# Default-scheduler 调度策略配置（tccli）

> 对照官方：[Default-scheduler 调度策略配置](https://cloud.tencent.com/document/product/457/111862) · page_id `111862`

## 概述

TKE 托管集群使用 Kubernetes 原生 kube-scheduler（Default-scheduler）进行 Pod 调度。通过配置 nodeSelector、亲和性（affinity）、污点与容忍（taints/tolerations）、拓扑分布约束（topologySpreadConstraints）四种策略，可实现灵活的工作负载调度控制。

| 策略 | 作用 | 典型场景 |
|------|------|----------|
| `nodeSelector` | 按节点标签简单匹配 | 将 GPU 工作负载调度到 GPU 节点 |
| `nodeAffinity` | 按节点标签高级匹配（硬/软约束） | 优先调度到 SSD 节点，无 SSD 时可 fallback |
| `podAffinity/podAntiAffinity` | 按 Pod 标签聚合或分散 | 同一服务的 Pod 分散到不同节点 |
| `taints/tolerations` | 节点排斥 + Pod 容忍 | 专用节点池，仅特定工作负载可调度 |
| `topologySpreadConstraints` | 按拓扑域均匀分布 | 跨可用区均匀分布 Pod |

## 前置条件

- [环境准备](../../../../环境准备.md)

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
```

**预期输出**：

```text
Client Version: v1.30.0
Kustomize Version: v5.0.4-0.20230601165947-6ce0bf390ce3
```

```bash
# 4. 检查 CAM 权限 — 需要以下 Action 名
#    tke:DescribeClusters, tke:DescribeClusterKubeconfig
# 验证：执行 DescribeClusters 确认权限
tccli tke DescribeClusters --region ap-guangzhou
# expected: exit 0，返回集群列表（可为空）
```

**预期输出**：

```json
{
    "TotalCount": 0,
    "Clusters": [],
    "RequestId": "..."
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
            "ClusterName": "CLUSTER_NAME",
            "ClusterStatus": "Running",
            "ClusterVersion": "1.30.0"
        }
    ],
    "RequestId": "..."
}
```

```bash
tccli tke DescribeClusterKubeconfig --region ap-guangzhou --ClusterId cls-xxxxxxxx \
    --output json | jq -r '.Kubeconfig' > ~/.kube/config-cls-xxxxxxxx
# expected: 文件创建成功，非空

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
CoreDNS is running at https://cls-xxxxxxxx.ccs.tencent-cloud.com/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
```

```bash
# 7. 检查集群节点列表
kubectl get nodes
# expected: 返回节点列表，STATUS 均为 Ready
```

**预期输出**：

```text
NAME          STATUS   ROLES    AGE   VERSION
node-ex-001   Ready    <none>   1d    v1.30.0
node-ex-002   Ready    <none>   1d    v1.30.0
node-ex-003   Ready    <none>   1d    v1.30.0
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看集群列表 | `DescribeClusters` | 是 |
| 获取 kubeconfig | `DescribeClusterKubeconfig --ClusterId cls-xxxxxxxx` | 是 |
| 配置调度策略 | kubectl — 为工作负载添加 nodeSelector/affinity/taints/topologySpreadConstraints | — |

## 操作步骤

### 步骤 1：nodeSelector — 按节点标签简单匹配

#### 选择依据

- **nodeSelector** 是最简单的调度约束，仅将 Pod 调度到具有匹配标签的节点。适用于节点类型明确的场景（如 GPU 节点 vs CPU 节点）。
- 与 nodeAffinity 相比：nodeSelector 只有"必须匹配"语义，不支持"优先匹配"（软约束）。若只需硬约束，nodeSelector 更简洁。

```bash
# 需 kubectl 可达环境 — 查看节点标签
kubectl get nodes --show-labels
# expected: 每个节点列表行含 LABELS 列
```

**预期输出**：

```text
NAME           STATUS   ROLES    AGE   VERSION   LABELS
node-ex-001    Ready    <none>   1d    v1.30.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/hostname=node-ex-001
node-ex-002    Ready    <none>   1d    v1.30.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,node.kubernetes.io/instance-type=S5.MEDIUM4
```

#### 最小示例：按 disktype 标签调度

```bash
# 需 kubectl 可达环境 — 为目标节点打标签
kubectl label node NODE_NAME disktype=ssd
# expected: node/NODE_NAME labeled

# 创建带 nodeSelector 的 Deployment
cat > node-selector-deploy.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-ssd
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-ssd
  template:
    metadata:
      labels:
        app: nginx-ssd
    spec:
      nodeSelector:
        disktype: ssd
      containers:
      - name: nginx
        image: nginx:1.25
EOF

kubectl apply -f node-selector-deploy.yaml
# expected: deployment.apps/nginx-ssd created
```

#### 增强配置：多标签组合 + affinity

对于更复杂的场景，使用 nodeAffinity 替代 nodeSelector（参见步骤 2）。

### 步骤 2：nodeAffinity — 高级节点亲和性

#### 选择依据

- **nodeAffinity** 支持 `requiredDuringSchedulingIgnoredDuringExecution`（硬约束）和 `preferredDuringSchedulingIgnoredDuringExecution`（软约束）。
- 选择软约束的场景：期望调度到 SSD 节点，但无 SSD 节点时允许调度到普通节点，避免 Pod 无法调度。
- `matchExpressions` 支持 `In`、`NotIn`、`Exists`、`DoesNotExist`、`Gt`、`Lt` 运算符，比 nodeSelector 更灵活。

```bash
# 需 kubectl 可达环境 — 创建带 nodeAffinity 的 Deployment
cat > affinity-deploy.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-affinity
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx-affinity
  template:
    metadata:
      labels:
        app: nginx-affinity
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: kubernetes.io/arch
                operator: In
                values:
                - amd64
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 80
            preference:
              matchExpressions:
              - key: disktype
                operator: In
                values:
                - ssd
      containers:
      - name: nginx
        image: nginx:1.25
EOF

kubectl apply -f affinity-deploy.yaml
# expected: deployment.apps/nginx-affinity created
```

### 步骤 3：taints/tolerations — 节点排斥与 Pod 容忍

#### 选择依据

- **taints** 在节点侧设置排斥规则，**tolerations** 在 Pod 侧声明容忍。
- Taint effect 选择：
  - `NoSchedule`：不调度不耐受的 Pod（推荐用于资源预留）。
  - `PreferNoSchedule`：尽量不调度（软约束）。
  - `NoExecute`：驱逐已有不耐受 Pod（最强），用于节点维护。
- 与 nodeSelector 的关键区别：taints 可以**驱逐**已运行的 Pod（NoExecute），而 nodeSelector/nodeAffinity 仅影响调度。

```bash
# 需 kubectl 可达环境 — 为节点添加污点
kubectl taint nodes NODE_NAME dedicated=gpu:NoSchedule
# expected: node/NODE_NAME tainted

# 创建带 tolerations 的 Pod
cat > toleration-pod.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: gpu-workload
spec:
  tolerations:
  - key: dedicated
    operator: Equal
    value: gpu
    effect: NoSchedule
  containers:
  - name: cuda-app
    image: nvidia/cuda:12.0-base
EOF

kubectl apply -f toleration-pod.yaml
# expected: pod/gpu-workload created
```

### 步骤 4：topologySpreadConstraints — 拓扑分布约束

#### 选择依据

- **topologySpreadConstraints** 用于控制 Pod 在拓扑域（可用区、节点、地域）中的分布均匀度。
- `maxSkew`：任意两拓扑域的 Pod 数量最大差值。`maxSkew: 1` 意味着相邻域的 Pod 数量最多差 1。
- `whenUnsatisfiable`：
  - `DoNotSchedule`：不满足时不调度（硬约束，谨慎使用）。
  - `ScheduleAnyway`：不满足时仍然调度（软约束，推荐生产环境）。
- 当集群跨多可用区时，建议对所有生产 Deployment 添加此约束。

```bash
# 需 kubectl 可达环境 — 创建带拓扑分布约束的 Deployment
cat > spread-deploy.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-ha
spec:
  replicas: 6
  selector:
    matchLabels:
      app: nginx-ha
  template:
    metadata:
      labels:
        app: nginx-ha
    spec:
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: ScheduleAnyway
        labelSelector:
          matchLabels:
            app: nginx-ha
      - maxSkew: 1
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: nginx-ha
      containers:
      - name: nginx
        image: nginx:1.25
EOF

kubectl apply -f spread-deploy.yaml
# expected: deployment.apps/nginx-ha created
```

## 验证

### 控制面（tccli）

```bash
# 确认集群状态正常
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
# 需 kubectl 可达环境 — 维度 1：验证 nodeSelector 生效
kubectl get pods -l app=nginx-ssd -o wide
# expected: 所有 Pod 的 NODE 列为带 disktype=ssd 标签的节点
```

**预期输出**：

```text
NAME                         READY   STATUS    RESTARTS   AGE   NODE
nginx-ssd-xxxxxxxxxx-xxxxx   1/1     Running   0          1m    node-ex-001
nginx-ssd-xxxxxxxxxx-yyyyy   1/1     Running   0          1m    node-ex-001
```

```bash
# 需 kubectl 可达环境 — 维度 2：验证 nodeAffinity 调度分布
kubectl get pods -l app=nginx-affinity -o wide
# expected: Pod 分布在 amd64 架构节点上，优先 SSD 节点
```

**预期输出**：

```text
NAME                              READY   STATUS    RESTARTS   AGE   NODE
nginx-affinity-xxxxxxxxxx-aaaaa   1/1     Running   0          1m    node-ex-001
nginx-affinity-xxxxxxxxxx-bbbbb   1/1     Running   0          1m    node-ex-002
nginx-affinity-xxxxxxxxxx-ccccc   1/1     Running   0          1m    node-ex-001
```

```bash
# 需 kubectl 可达环境 — 维度 3：验证 taints/tolerations
kubectl describe node NODE_NAME | grep Taints
# expected: Taints: dedicated=gpu:NoSchedule
```

**预期输出**：

```text
Taints:             dedicated=gpu:NoSchedule
```

```bash
kubectl get pod gpu-workload -o jsonpath='{.spec.tolerations}'
# expected: 含 dedicated=gpu 的 toleration
```

**预期输出**：

```text
[{"effect":"NoSchedule","key":"dedicated","operator":"Equal","value":"gpu"}]
```

```bash
# 需 kubectl 可达环境 — 维度 4：验证拓扑分布
kubectl get pods -l app=nginx-ha -o wide --sort-by='.spec.nodeName'
# expected: Pod 在不同节点/可用区间均匀分布，maxSkew <= 1
```

**预期输出**：

```text
NAME                        READY   STATUS    RESTARTS   AGE   NODE
nginx-ha-xxxxxxxxxx-11111   1/1     Running   0          1m    node-ex-001
nginx-ha-xxxxxxxxxx-22222   1/1     Running   0          1m    node-ex-001
nginx-ha-xxxxxxxxxx-33333   1/1     Running   0          1m    node-ex-002
nginx-ha-xxxxxxxxxx-44444   1/1     Running   0          1m    node-ex-002
nginx-ha-xxxxxxxxxx-55555   1/1     Running   0          1m    node-ex-003
nginx-ha-xxxxxxxxxx-66666   1/1     Running   0          1m    node-ex-003
```

| 维度 | 命令 | 预期 |
|------|------|------|
| 策略生效 | `kubectl get pods -l app=nginx-ssd -o wide` | Pod 在匹配节点运行 |
| 亲和性分布 | `kubectl get pods -l app=nginx-affinity -o wide` | 硬约束满足，优先 SSD |
| 污点与容忍 | `kubectl describe node NODE_NAME` + `kubectl get pod gpu-workload -o jsonpath` | Taint 在节点，Pod 有匹配 toleration |
| 拓扑均匀 | `kubectl get pods -l app=nginx-ha -o wide` | 跨可用区/节点均匀分布 |

## 清理

> **警告**：删除 Deployment 会级联删除其管理的 Pod。删除节点标签或污点可能影响其他工作负载的调度行为。建议在清理前确认前述资源未被其他业务使用。

### 数据面

数据面资源清理在控制面之前。

```bash
# 需 kubectl 可达环境 — 清理前确认目标资源
kubectl get deploy nginx-ssd nginx-affinity nginx-ha
# 确认是待删除的目标 Deployment
```

**预期输出**：

```text
NAME          READY   UP-TO-DATE   AVAILABLE   AGE
nginx-ssd     2/2     2            2           10m
nginx-affinity   3/3     3            3           10m
nginx-ha      6/6     6            6           10m
```

```bash
kubectl get pod gpu-workload
# 确认是待删除的目标 Pod
```

**预期输出**：

```text
NAME           READY   STATUS    RESTARTS   AGE
gpu-workload   1/1     Running   0          10m
```

删除工作负载：

```bash
# 需 kubectl 可达环境 — 删除 Deployments
kubectl delete deploy nginx-ssd nginx-affinity nginx-ha
# expected: deployment.apps "nginx-ssd" deleted, ...

# 需 kubectl 可达环境 — 删除 Pod
kubectl delete pod gpu-workload
# expected: pod "gpu-workload" deleted
```

清理节点标签和污点：

```bash
# 需 kubectl 可达环境 — 移除节点标签
kubectl label node NODE_NAME disktype-
# expected: node/NODE_NAME unlabeled

# 需 kubectl 可达环境 — 移除节点污点
kubectl taint nodes NODE_NAME dedicated=gpu:NoSchedule-
# expected: node/NODE_NAME untainted
```

验证已删除：

```bash
# 需 kubectl 可达环境
kubectl get deploy nginx-ssd nginx-affinity nginx-ha
# expected: Error from server (NotFound)
```

**预期输出**：

```text
Error from server (NotFound): deployments.apps "nginx-ssd" not found
Error from server (NotFound): deployments.apps "nginx-affinity" not found
Error from server (NotFound): deployments.apps "nginx-ha" not found
```

```bash
kubectl get pod gpu-workload
# expected: Error from server (NotFound)
```

**预期输出**：

```text
Error from server (NotFound): pods "gpu-workload" not found
```

```bash
kubectl describe node NODE_NAME | grep -E "Labels:|Taints:"
# expected: 已无 disktype 标签和 dedicated 污点
```

**预期输出**：

```text
Labels:             beta.kubernetes.io/arch=amd64
Taints:             <none>
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Pod 状态 Pending，`kubectl describe pod POD_NAME` 显示 `0/xxx nodes are available: xxx node(s) didn't match node selector` | 检查节点标签：`kubectl get nodes --show-labels \| grep disktype` | 节点无匹配标签，nodeSelector 无法匹配 | 为目标节点打标签：`kubectl label node NODE_NAME disktype=ssd`，或修改 nodeSelector 为已有标签 |
| Pod 状态 Pending，`kubectl describe pod POD_NAME` 显示 `0/xxx nodes are available: xxx node(s) had untolerated taint` | 检查节点污点：`kubectl describe node NODE_NAME \| grep Taints` | Pod spec 缺少对应 toleration | 为 Pod 添加匹配的 toleration，或移除节点污点：`kubectl taint nodes NODE_NAME TAINT_KEY:NoSchedule-` |
| `kubectl describe pod POD_NAME` 显示 `0/xxx nodes are available: xxx node(s) didn't match pod affinity/anti-affinity rules` | `kubectl get pods -o wide` 对比各节点已有 Pod 标签 | podAffinity/podAntiAffinity 规则无法满足 | 检查 `matchLabels` 是否与节点上已有 Pod 匹配；调整规则或增加节点 |
| `kubectl describe pod POD_NAME` 显示 `0/xxx nodes are available: xxx node(s) didn't satisfy topology spread constraints` | `kubectl get nodes -L topology.kubernetes.io/zone` 检查拓扑域节点数 | 拓扑域节点不足，无法满足 maxSkew | 将 `whenUnsatisfiable` 改为 `ScheduleAnyway`，或增加节点跨拓扑域分布 |

### 调度策略配置后集群行为异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 新 Pod 全部 Pending | `kubectl get nodes --show-labels` 检查节点标签变化；`kubectl describe node NODE_NAME` 检查污点 | 错误的标签或污点导致所有节点不匹配 | 回滚错误的节点标签：`kubectl label node NODE_NAME WRONG_LABEL-`；移除错误污点 |
| 某些节点 Pod 密度过高 | `kubectl get pods -o wide --all-namespaces \| awk '{print $7}' \| sort \| uniq -c` 统计每节点 Pod 数 | topologySpreadConstraints 未覆盖所有工作负载或 maxSkew 过大 | 为高密度节点上的 Deployment 添加 topologySpreadConstraints |
| NoExecute 污点导致 Pod 被意外驱逐 | `kubectl get events --all-namespaces --field-selector reason=NodeControllerEviction \| tail -20` | 节点被添加了 NoExecute 污点，且 Pod 无对应 toleration | 为关键 Pod 添加 toleration，或移除 NoExecute 污点 |

## 下一步

- [创建集群](https://cloud.tencent.com/document/product/457/103982) — 通过 CLI 创建托管集群
- [节点池管理](https://cloud.tencent.com/document/product/457/48556) — 为不同策略创建专用节点池
- [Crane 调度器介绍](../../Crane%20调度器/Crane%20调度器介绍/tccli%20操作.md) — 智能调度增强
- [自定义资源优先级](../../../业务优先级保障调度/自定义资源优先级/tccli%20操作.md) — PriorityClass 优先级调度

## 控制台替代

登录 [容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) → 选择集群 → **工作负载** → **Deployment** → **新建** → 在「高级设置」中配置节点调度策略（nodeSelector/亲和性/容忍/拓扑分布约束）。
