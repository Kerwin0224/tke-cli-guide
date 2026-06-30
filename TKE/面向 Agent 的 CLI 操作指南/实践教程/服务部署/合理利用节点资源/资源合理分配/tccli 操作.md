# 资源合理分配（tccli）

> 对照官方：[资源合理分配](https://cloud.tencent.com/document/product/457/45635) · page_id `45635`

## 概述

通过 nodeSelector、nodeAffinity/antiAffinity、taint/toleration、topologySpreadConstraints 等 Kubernetes 调度策略，实现工作负载在集群节点上的合理分配。核心策略包括：装箱调度（bin-packing）提升资源利用率、反亲和分散调度避免单点故障、专用节点池隔离不同业务类型，以及拓扑分布约束防止资源碎片化。

> **kubectl 数据面不可达**：CAM 拒绝公网端点（strategyId:240463971），内网需 VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

## 前置条件

- [环境准备](../../../../环境准备.md)
- 已获取集群 kubeconfig：

```bash
tccli tke DescribeClusterKubeconfig --region <Region> --ClusterId CLUSTER_ID \
    --filter "Kubeconfig" > kubeconfig.yaml
export KUBECONFIG=$(pwd)/kubeconfig.yaml
# expected: 生成 kubeconfig.yaml
```

```json
{
  "Kubeconfig": "<Kubeconfig>",
  "RequestId": "<RequestId>"
}
```

- 需要 CAM 权限：`tke:DescribeClusters`、`tke:DescribeClusterKubeconfig`、`tke:DescribeClusterInstances`
- 集群中已有多个工作节点，节点具备不同标签或资源规格
- kubectl >= v1.28（支持 topologySpreadConstraints `minDomains` 等字段）

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId、secretKey、region 均已配置，region 为目标地域

# 3. 验证控制面可达
tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterStatus "Running"

# 4. 检查 kubectl 版本（须 VPN/IOA）
kubectl version --client
# expected: Client Version v1.28+
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

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 获取集群凭证 | `tccli tke DescribeClusterKubeconfig` | 是 |
| 查看集群节点 | `tccli tke DescribeClusterInstances --region <Region> --ClusterId CLUSTER_ID` | 是 |
| 查看节点资源 | `kubectl get nodes -o custom-columns=NAME:.metadata.name,CPU:.status.allocatable.cpu,MEMORY:.status.allocatable.memory` | 是 |
| 节点选择器 | `nodeSelector`（Pod Spec） | 是 |
| 节点亲和性 | `nodeAffinity`（Pod Spec） | 是 |
| Pod 反亲和 | `podAntiAffinity`（Pod Spec） | 是 |
| 拓扑分布约束 | `topologySpreadConstraints`（Pod Spec） | 是 |
| 节点打污点 | `kubectl taint node NODE_ID key=value:Effect` | 是 |
| 容忍污点 | `tolerations`（Pod Spec） | 是 |
| 去污点 | `kubectl taint node NODE_ID key=value:Effect-` | 是 |

## 操作步骤

### 步骤 1：查看集群节点资源分布

#### 选择依据

在执行调度策略前，需先了解集群节点的资源分布和标签情况。控制面通过 tccli 查看节点池和实例列表，数据面通过 kubectl 查看节点可分配资源和已有标签。

**控制面查询：**

```bash
tccli tke DescribeClusterInstances --region <Region> --ClusterId CLUSTER_ID
```

```json
{
  "InstanceSet": [
    {
      "InstanceId": "ins-xxxxxxxx",
      "InstanceRole": "WORKER",
      "InstanceState": "running",
      "LanIP": "10.0.1.10",
      "NodePoolId": "np-xxxxxxxx"
    }
  ],
  "TotalCount": 4
}
```

**数据面查询（需 VPN/IOA）：**

```bash
kubectl get nodes -o custom-columns=NAME:.metadata.name,CPU:.status.allocatable.cpu,MEMORY:.status.allocatable.memory
```

```text
NAME           CPU   MEMORY
10.0.1.10      4     15861952Ki
10.0.1.11      8     31723904Ki
10.0.1.12      4     15861952Ki
10.0.1.13      16    63447808Ki
```

```bash
kubectl get nodes --show-labels
```

```text
NAME           LABELS
10.0.1.10      ... node.kubernetes.io/instance-type=S5.MEDIUM4 ...
10.0.1.11      ... node.kubernetes.io/instance-type=S5.LARGE8 ...
10.0.1.12      ... node.kubernetes.io/instance-type=S5.MEDIUM4 ...
10.0.1.13      ... node.kubernetes.io/instance-type=S5.2XLARGE16 ...
```

### 步骤 2：使用 nodeSelector 和 nodeAffinity 实现基本分配策略

#### 选择依据

- **nodeSelector**：最基础的硬性调度约束，Pod 只能调度到带有指定标签的节点。适用于「服务必须运行在特定节点」的简单场景。
- **nodeAffinity**：比 nodeSelector 更灵活，支持 In/NotIn/Exists/DoesNotExist/Gt/Lt 等操作符，支持 required（硬约束）和 preferred（软偏好）两种模式。适用于「优先调度到某类节点，但无可用节点时可降级」的场景。
- 基本策略：先使用 nodeSelector 快速验证调度方向，再用 nodeAffinity 实现更精细的控制。

#### 最小配置：nodeSelector

创建 `deployment-nodeselector.yaml`：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: DEPLOYMENT_NAME-basic
  namespace: NAMESPACE
spec:
  replicas: 2
  selector:
    matchLabels:
      app: basic-worker
  template:
    metadata:
      labels:
        app: basic-worker
    spec:
      nodeSelector:
        node-role.kubernetes.io/worker: ""
      containers:
        - name: nginx
          image: nginx:1.25-alpine
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 200m
              memory: 256Mi
            limits:
              cpu: 500m
              memory: 512Mi
```

```bash
kubectl apply -f deployment-nodeselector.yaml
# expected: deployment.apps/DEPLOYMENT_NAME-basic created
```

```bash
kubectl get pods -l app=basic-worker -o wide
# expected: 2 Pods 均调度到带有 node-role.kubernetes.io/worker 标签的节点
```

#### 增强配置：nodeAffinity

创建 `deployment-nodeaffinity.yaml`：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: DEPLOYMENT_NAME-affinity
  namespace: NAMESPACE
spec:
  replicas: 3
  selector:
    matchLabels:
      app: affinity-app
  template:
    metadata:
      labels:
        app: affinity-app
    spec:
      affinity:
        nodeAffinity:
          # 硬约束：必须调度到 worker 节点
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: node-role.kubernetes.io/worker
                    operator: In
                    values:
                      - ""
          # 软约束：优先调度到大内存节点（打分策略）
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              preference:
                matchExpressions:
                  - key: node.kubernetes.io/instance-type
                    operator: In
                    values:
                      - "S5.LARGE8"
                      - "S5.2XLARGE16"
      containers:
        - name: nginx
          image: nginx:1.25-alpine
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 500m
              memory: 512Mi
            limits:
              cpu: 1
              memory: 1Gi
```

```bash
kubectl apply -f deployment-nodeaffinity.yaml
# expected: deployment.apps/DEPLOYMENT_NAME-affinity created
```

```bash
kubectl get pods -l app=affinity-app -o wide
# expected: 3 Pods 调度到 worker 节点，优先分配到大内存实例类型
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `DEPLOYMENT_NAME` | Deployment 名称 | 自定义，需符合 K8s 命名规范 | 自定义 |
| `NAMESPACE` | K8s 命名空间 | 需已存在 | `kubectl get ns` |

### 步骤 3：使用 Pod 反亲和实现分散调度（避免单点故障）

#### 选择依据

Pod 反亲和（podAntiAffinity）确保同一应用的多个副本不会全部调度到同一节点，从而避免单节点故障导致服务整体不可用。

- **requiredDuringSchedulingIgnoredDuringExecution**：硬约束，如果找不到满足反亲和条件的节点，Pod 将无法调度（Pending）。
- **preferredDuringSchedulingIgnoredDuringExecution**：软约束，优先分散，但允许在节点不足时合并。
- 生产环境推荐 `requiredDuringSchedulingIgnoredDuringExecution` + `topologyKey: kubernetes.io/hostname`（以节点为拓扑域）。

#### 增强配置

创建 `deployment-anti-affinity.yaml`：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: DEPLOYMENT_NAME-spread
  namespace: NAMESPACE
spec:
  replicas: 3
  selector:
    matchLabels:
      app: spread-app
  template:
    metadata:
      labels:
        app: spread-app
    spec:
      affinity:
        podAntiAffinity:
          # 硬约束：同一节点上最多只能运行一个此应用的 Pod
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: app
                    operator: In
                    values:
                      - spread-app
              topologyKey: kubernetes.io/hostname
        # 同时使用节点亲和，优先大内存节点
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              preference:
                matchExpressions:
                  - key: node.kubernetes.io/instance-type
                    operator: In
                    values:
                      - "S5.LARGE8"
                      - "S5.2XLARGE16"
      containers:
        - name: nginx
          image: nginx:1.25-alpine
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 500m
              memory: 512Mi
            limits:
              cpu: 1
              memory: 1Gi
```

```bash
kubectl apply -f deployment-anti-affinity.yaml
# expected: deployment.apps/DEPLOYMENT_NAME-spread created
```

```bash
kubectl get pods -l app=spread-app -o wide
# expected: 3 Pods 分布在不同节点上，每个节点最多 1 个
```

**关键参数说明**：

| 参数 | 说明 | 取值 |
|------|------|------|
| `topologyKey` | 拓扑域标识 | `kubernetes.io/hostname`（节点级）、`topology.kubernetes.io/zone`（可用区级）、`topology.kubernetes.io/region`（地域级） |
| `requiredDuringSchedulingIgnoredDuringExecution` | 硬约束 | Pod 必须满足才能调度 |
| `preferredDuringSchedulingIgnoredDuringExecution` | 软约束 | 调度器尽量满足，不满足也可调度 |

### 步骤 4：使用 topologySpreadConstraints 实现均衡分布

#### 选择依据

`topologySpreadConstraints` 是比 podAntiAffinity 更精细的分布控制策略，适用于以下场景：

- 需要控制多个拓扑域（如同时按节点和可用区分布）
- 需要定义最大差异值（maxSkew），而非简单的「不允许共存」
- 需要同时约束多个不同 label 的 Pod 分布

与 podAntiAffinity 的区别：

| 特性 | podAntiAffinity | topologySpreadConstraints |
|------|----------------|---------------------------|
| 控制粒度 | 二元（允许/不允许共存） | 数值偏差（maxSkew） |
| 多拓扑域 | 需声明多条规则 | 单条规则可声明多个 topologyKey |
| 跨应用约束 | 仅限同一应用（通过 labelSelector） | 可跨不同应用 |
| 调度失败处理 | Pod Pending | 降低 maxSkew 可提高调度成功率 |

#### 增强配置

创建 `deployment-topology-spread.yaml`：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: DEPLOYMENT_NAME-balanced
  namespace: NAMESPACE
spec:
  replicas: 6
  selector:
    matchLabels:
      app: balanced-app
  template:
    metadata:
      labels:
        app: balanced-app
    spec:
      topologySpreadConstraints:
        # 按节点均衡分布
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: balanced-app
        # 按可用区均衡分布
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              app: balanced-app
      containers:
        - name: nginx
          image: nginx:1.25-alpine
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 200m
              memory: 256Mi
            limits:
              cpu: 500m
              memory: 512Mi
```

```bash
kubectl apply -f deployment-topology-spread.yaml
# expected: deployment.apps/DEPLOYMENT_NAME-balanced created
```

```bash
kubectl get pods -l app=balanced-app -o wide
# expected: 6 Pods 在各节点和各可用区之间均衡分布，maxSkew <= 1
```

```text
NAME  STATUS  AGE
...
```

**关键参数说明**：

| 参数 | 说明 | 推荐值 |
|------|------|--------|
| `maxSkew` | 任意两个拓扑域之间 Pod 数量差的最大允许值 | 1（严格均衡），2（允许一定偏差） |
| `whenUnsatisfiable` | 无法满足约束时的行为 | `DoNotSchedule`（不调度，严格保证分布）；`ScheduleAnyway`（尽量满足，优先保证可用） |
| `topologyKey` | 拓扑域的标签键 | `kubernetes.io/hostname`（节点级）、`topology.kubernetes.io/zone`（可用区级） |

### 步骤 5：使用 nodeAffinity + taints/tolerations 实现专用节点池

#### 选择依据

当集群中某些节点专用于特定业务（如 GPU 节点、高内存节点、数据库节点）时，需通过 taint（污点）+ toleration（容忍）机制防止其他业务 Pod 调度到这些节点，再通过 nodeAffinity 引导目标 Pod 精确调度。

- **taint 效果**：
  - `NoSchedule`：不允许新 Pod 调度到该节点，已运行的保持不变。
  - `PreferNoSchedule`：尽量不调度，但不强制。
  - `NoExecute`：不允许新 Pod 调度，且驱逐已有的不匹配 Pod。
- **适用场景**：GPU 推理节点池、高内存数据库节点池、IO 密集型专有节点池。

#### 增强配置

**第一步：给目标节点打污点**

```bash
kubectl taint node NODE_ID dedicated=high-memory:NoSchedule
```

```text
node/NODE_ID tainted
```

```bash
kubectl describe node NODE_ID | grep Taint
```

```text
Taints:             dedicated=high-memory:NoSchedule
```

**第二步：创建专用 Deployment（带 toleration + affinity）**

创建 `deployment-dedicated-node.yaml`：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: DEPLOYMENT_NAME-dedicated
  namespace: NAMESPACE
spec:
  replicas: 2
  selector:
    matchLabels:
      app: dedicated-app
  template:
    metadata:
      labels:
        app: dedicated-app
    spec:
      # 容忍污点，允许调度到专用节点
      tolerations:
        - key: dedicated
          operator: Equal
          value: high-memory
          effect: NoSchedule
      affinity:
        # 硬亲和：必须调度到专用节点
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: node.kubernetes.io/instance-type
                    operator: In
                    values:
                      - "S5.2XLARGE16"
      containers:
        - name: nginx
          image: nginx:1.25-alpine
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 2
              memory: 4Gi
            limits:
              cpu: 4
              memory: 8Gi
```

```bash
kubectl apply -f deployment-dedicated-node.yaml
# expected: deployment.apps/DEPLOYMENT_NAME-dedicated created
```

```bash
kubectl get pods -l app=dedicated-app -o wide
# expected: 2 Pods 调度到带有 dedicated=high-memory 污点的 S5.2XLARGE16 节点
```

```text
NAME  STATUS  AGE
...
```

**验证普通 Pod 无法调度到专用节点：**

```bash
# 创建一个不带 toleration 的测试 Pod，验证其无法调度到污点节点
kubectl run test-no-toleration --image=nginx:alpine --dry-run=client -o yaml | kubectl apply -f -
# expected: Pod 调度到非污点节点或 Pending（若无可用节点）
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `NODE_ID` | 目标节点 ID（kubectl 节点名） | 须为集群中已有节点 | `kubectl get nodes` |
| `DEPLOYMENT_NAME` | Deployment 名称 | 自定义 | 自定义 |
| `NAMESPACE` | K8s 命名空间 | 须已存在 | `kubectl get ns` |

### 步骤 6：多策略组合 — 软硬约束、打分 vs 约束条件对比

#### 选择依据

实际生产环境中，通常需要组合使用多种调度策略。以下是各策略的组合推荐和适用场景：

**调度策略组合总结**：

| 策略 | 类型 | 机制 | 适用场景 | 调度失败影响 |
|------|------|------|----------|-------------|
| `nodeSelector` | 硬约束 | 精确匹配标签 | 简单场景、快速验证 | Pod Pending（若无匹配节点） |
| `requiredDuringSchedulingIgnoredDuringExecution`（nodeAffinity） | 硬约束 | 标签表达式匹配 | 必须调度到特定节点类型 | Pod Pending（若无匹配节点） |
| `preferredDuringSchedulingIgnoredDuringExecution`（nodeAffinity） | 软约束/打分 | 调度器加权打分 | 优先某种节点但不强制 | 无影响，降级调度到其他节点 |
| `requiredDuringSchedulingIgnoredDuringExecution`（podAntiAffinity） | 硬约束 | 拒绝与指定 Pod 共存于同一拓扑域 | 高可用部署 | Pod Pending（若无足够节点） |
| `topologySpreadConstraints`（DoNotSchedule） | 硬约束 | 控制拓扑域内 Pod 数量偏差 | 均衡分布 | Pod Pending（若无法满足 maxSkew） |
| `topologySpreadConstraints`（ScheduleAnyway） | 软约束/打分 | 尽量满足偏差，优先调度 | 弹性伸缩场景 | 无影响，允许临时不均衡 |
| `taint + toleration + nodeAffinity` | 组合 | 排斥非目标 Pod + 亲和引导 | 专用节点池 | 两者缺一即 Pending |

**推荐组合策略**：

| 场景 | 组合策略 |
|------|----------|
| 普通 Web 应用高可用 | `podAntiAffinity`(required) + `topologySpreadConstraints`（按可用区，ScheduleAnyway） |
| GPU 推理服务 | `taint` + `toleration` + `nodeAffinity`(required) + `podAntiAffinity`(preferred) |
| 大数据批处理 | `nodeAffinity`(preferred，大内存) + `topologySpreadConstraints`（按节点，DoNotSchedule） |
| 微服务通用 | `podAntiAffinity`(preferred) + `topologySpreadConstraints`（按节点和可用区） |

**完整组合示例** — 混合使用软硬约束：

```yaml
spec:
  affinity:
    # 硬约束：必须分散到不同节点
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchLabels:
              app: combined-app
          topologyKey: kubernetes.io/hostname
    # 软约束：优先大内存节点（打分机制）
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          preference:
            matchExpressions:
              - key: node.kubernetes.io/instance-type
                operator: In
                values:
                  - "S5.LARGE8"
                  - "S5.2XLARGE16"
  # 拓扑分布：允许弹性伸缩时临时不均衡
  topologySpreadConstraints:
    - maxSkew: 1
      topologyKey: topology.kubernetes.io/zone
      whenUnsatisfiable: ScheduleAnyway
      labelSelector:
        matchLabels:
          app: combined-app
```

## 验证

### 控制面（tccli）

```bash
# 验证集群节点实例状态
tccli tke DescribeClusterInstances --region <Region> --ClusterId CLUSTER_ID
# expected: 返回所有节点实例，InstanceState 均为 "running"
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

```bash
# 查看集群整体状态
tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterStatus "Running"
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

### 数据面（需 VPN/IOA）

```bash
# 1. 验证所有 Deployment 的 Pod 分布
kubectl get pods -A -o wide --field-selector status.phase=Running
# expected: 各应用 Pod 分布在期望的节点上

# 2. 验证 nodeSelector 策略生效
kubectl get pods -l app=basic-worker -o custom-columns=NAME:.metadata.name,NODE:.spec.nodeName,NODE-SELECTOR:.spec.nodeSelector
# expected: 所有 Pod 调度到带有 node-role.kubernetes.io/worker 标签的节点

# 3. 验证反亲和策略 — Pod 分布在不同节点
kubectl get pods -l app=spread-app -o custom-columns=NAME:.metadata.name,NODE:.spec.nodeName
# expected: 每个 NODE 列的值不重复（每个节点至多 1 个 Pod）

# 4. 验证拓扑分布约束 — maxSkew 不超标
kubectl get pods -l app=balanced-app -o custom-columns=NODE:.spec.nodeName | sort | uniq -c | sort -n
# expected: 各节点 Pod 数量差 ≤ maxSkew

# 5. 验证污点和容忍 — 专用 Pod 在污点节点上
kubectl get pods -l app=dedicated-app -o custom-columns=NAME:.metadata.name,NODE:.spec.nodeName
# 对应查看节点污点
for node in $(kubectl get nodes -o name); do echo "$node: $(kubectl get $node -o jsonpath='{.spec.taints}')"; done
# expected: dedicated-app Pod 的节点带有 dedicated=high-memory:NoSchedule 污点

# 6. 验证普通 Pod 未调度到污点节点
kubectl get pods test-no-toleration -o wide
# expected: NODE 不是带有 dedicated=high-memory 污点的节点
```

```text
NAME  STATUS  AGE
...
```

## 清理

### 数据面（需 VPN/IOA）

```bash
# 1. 删除测试 Deployment
kubectl delete deployment DEPLOYMENT_NAME-basic
kubectl delete deployment DEPLOYMENT_NAME-affinity
kubectl delete deployment DEPLOYMENT_NAME-spread
kubectl delete deployment DEPLOYMENT_NAME-balanced
kubectl delete deployment DEPLOYMENT_NAME-dedicated
kubectl delete pod test-no-toleration --ignore-not-found
# expected: 各资源已删除

# 2. 移除节点污点（还原节点状态）
kubectl taint node NODE_ID dedicated=high-memory:NoSchedule-
# expected: node/NODE_ID untainted

# 3. 验证污点已移除
kubectl describe node NODE_ID | grep Taint
# expected: 不再显示 dedicated=high-memory:NoSchedule
```

```text
NAME  STATUS  AGE
...
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Pod 一直 Pending，Events 显示 `0/X nodes are available: X node(s) didn't match node selector` | `kubectl describe pod <pod-name>` 查看 Events | nodeSelector 标签与任何节点都不匹配 | 检查 `kubectl get nodes --show-labels`，确认目标节点带有指定的 label；或修正 nodeSelector 值 |
| Pod 一直 Pending，Events 显示 `X node(s) had taint {key: value}, that the pod didn't tolerate` | `kubectl describe pod <pod-name>` 查看 Events | Pod 缺少 toleration，无法调度到被 taint 的节点 | 给 Pod 添加匹配的 toleration，或确认是否需要移除节点的污点 |
| Pod 一直 Pending，Events 显示 `X node(s) didn't match pod anti-affinity rules` | `kubectl describe pod <pod-name>` 查看 Events | podAntiAffinity 硬约束导致无可用节点（如 replicas > 节点数） | 降级为 preferredDuringScheduling 或增加节点数，或降低 replicas |
| topologySpreadConstraints 不生效，Pod 分布不均 | `kubectl get pods -o wide` 统计各节点 Pod 数量 | `whenUnsatisfiable: ScheduleAnyway` 在节点不足时允许不均衡；或 `labelSelector` 未正确匹配 | 将 `whenUnsatisfiable` 改为 `DoNotSchedule`；或检查 labelSelector 是否匹配所有目标 Pod |
| nodeAffinity preferred 似乎未生效，Pod 未调度到期望节点 | `kubectl describe pod <pod-name>` 查看调度决策 | preferredDuringScheduling 只是打分偏好，其他因素（如资源不足、污点）权重可能更高 | 如果必须调度到特定节点，改用 requiredDuringScheduling；或检查节点资源是否充足 |
| 污点移除后 Pod 未被重新调度 | `kubectl get pods <pod-name>` 查看 Pod 是否已 Running | Pod 已经在其他节点运行，不会主动迁移（taint 只阻止新调度，不驱逐已运行的 Pod） | 删除 Pod 触发重建：`kubectl delete pod <pod-name>`，Deployment 将重新调度 |
| 多策略组合导致 Pod Pending，难以定位原因 | 依次注释各策略字段，逐步部署测试 | 多个硬约束叠加后，同时满足所有条件的节点不存在 | 将其中一个硬约束改为软约束（preferred/ScheduleAnyway），或扩容节点以满足全部约束 |
| `kubectl apply` 返回 `Unauthorized` 或 `connection refused` | `kubectl cluster-info` 检查连接状态 | CAM 策略拒绝公网端点（strategyId:240463971） | 通过 VPN/IOA 连接内网端点；或重新获取 kubeconfig：`tccli tke DescribeClusterKubeconfig --region <Region> --ClusterId CLUSTER_ID` |

## 下一步

- [设置 Request 与 Limit](../设置%20Request%20与%20Limit/tccli%20操作.md) -- page_id `45634`
- [应用高可用部署](../../应用高可用部署/tccli%20操作.md) -- page_id `40212`
- [弹性伸缩](../../弹性伸缩/tccli%20操作.md) -- page_id `45633`
- [设置工作负载的调度规则](../../../../应用配置/工作负载管理/设置工作负载的调度规则/tccli%20操作.md) -- page_id `32814`

## 控制台替代

通过 [TKE 控制台 - 工作负载](https://console.cloud.tencent.com/tke2/cluster/sub/list/deployment/deployment) 编辑 YAML，在「调度策略」中配置 nodeSelector、nodeAffinity、podAntiAffinity 和 topologySpreadConstraints；通过 [节点池管理](https://console.cloud.tencent.com/tke2/cluster/sub/list/nodePool/nodepool) 配置节点污点。
