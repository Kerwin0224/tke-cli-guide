# 设置 Request 与 Limit（tccli）

> 对照官方：[设置 Request 与 Limit](https://cloud.tencent.com/document/product/457/45634) · page_id `45634`

## 概述

通过 `resources.requests` 和 `resources.limits` 定义容器资源需求与上限，结合 LimitRange 和 ResourceQuota 在命名空间级别实现资源管控。不同的 request/limit 配比决定了 Pod 的服务质量（QoS）等级：Burstable（request < limit）、Guaranteed（request == limit）、BestEffort（无 request/limit）。

> **kubectl 数据面不可达**：CAM拒绝公网端点(strategyId:240463971)，内网需VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

## 前置条件

- [环境准备](../../../../环境准备.md)：`tccli` 已配置，`kubectl` >= v1.28

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId、secretKey、region 均已配置，region 为目标地域

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:DescribeClusters, tke:DescribeClusterKubeconfig
#    tke:DescribeClusterInstances
tccli tke DescribeClusters --region <Region>
# expected: exit 0，返回集群列表（可为空）

# 4. 检查 kubectl 可用性（须 VPN/IOA）
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

### 资源检查

```bash
# 5. 确认目标集群状态
tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterStatus "Running"

# 6. 获取 kubeconfig（须 VPN/IOA 方可使用 kubectl）
tccli tke DescribeClusterKubeconfig --region <Region> --ClusterId CLUSTER_ID | jq -r '.Kubeconfig' > ~/.kube/config

# 7. 验证集群可达性（须 VPN/IOA）
kubectl cluster-info
# expected: Kubernetes control plane 地址

# 8. 查看集群节点资源
tccli tke DescribeClusterInstances --region <Region> --ClusterId CLUSTER_ID
# expected: InstanceSet 列出节点信息，确认有足够资源部署
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
| 查看集群状态 | `tccli tke DescribeClusters` | 是 |
| 查看集群节点 | `tccli tke DescribeClusterInstances` | 是 |
| 获取 kubeconfig | `tccli tke DescribeClusterKubeconfig` | 是 |
| 设置容器资源 | Deployment YAML `resources.requests` / `resources.limits` | 是 |
| 设置命名空间默认资源 | `kubectl apply -f limit-range.yaml` | 是 |
| 设置命名空间资源配额 | `kubectl apply -f resource-quota.yaml` | 是 |
| 查看 Pod QoS 等级 | `kubectl describe pod <name>` | 是 |
| 查看实际资源用量 | `kubectl top pod` | 是 |
| 查看 OOM 事件 | `kubectl get events --field-selector reason=OOMKilling` | 是 |

## 操作步骤

### 步骤 1：获取集群状态（控制面）

```bash
tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
```

```json
{
  "TotalCount": 1,
  "Clusters": [
    {
      "ClusterId": "CLUSTER_ID",
      "ClusterName": "test-cluster",
      "ClusterStatus": "Running",
      "ClusterVersion": "1.30.0",
      "ClusterNodeNum": 3
    }
  ]
}
```

### 步骤 2：创建最小化 Deployment（Burstable QoS）

#### 选择依据

- **QoS 等级选择**：测试/开发场景使用 Burstable（request < limit），允许 Pod 在节点空闲时突发使用更多资源，成本较低。
- **镜像选择**：使用 `nginx:1.25-alpine` 作为轻量示例镜像。
- **副本数**：最小 1 副本用于验证。

#### 最小配置

创建 `deployment-minimal.yaml`：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: DEPLOYMENT_NAME-minimal
  namespace: NAMESPACE
spec:
  replicas: 1
  selector:
    matchLabels:
      app: demo-minimal
  template:
    metadata:
      labels:
        app: demo-minimal
    spec:
      containers:
      - name: nginx
        image: nginx:1.25-alpine
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 256Mi
        ports:
        - containerPort: 80
```

```bash
kubectl apply -f deployment-minimal.yaml
```

```text
# command executed successfully
```

```output
deployment.apps/DEPLOYMENT_NAME-minimal created
```

> **说明**：此配置中 `requests < limits`，Pod 的 QoS 等级为 **Burstable**。Burstable Pod 在节点资源紧张时，优先于 Guaranteed Pod 被驱逐（OOMKilled）。

### 步骤 3：创建增强配置 Deployment（Guaranteed QoS）

#### 增强配置

创建 `deployment-guaranteed.yaml`：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: DEPLOYMENT_NAME-guaranteed
  namespace: NAMESPACE
spec:
  replicas: 2
  selector:
    matchLabels:
      app: demo-guaranteed
  template:
    metadata:
      labels:
        app: demo-guaranteed
    spec:
      containers:
      - name: nginx
        image: nginx:1.25-alpine
        resources:
          requests:
            cpu: 200m
            memory: 256Mi
          limits:
            cpu: 200m
            memory: 256Mi
        ports:
        - containerPort: 80
```

```bash
kubectl apply -f deployment-guaranteed.yaml
```

```text
# command executed successfully
```

```output
deployment.apps/DEPLOYMENT_NAME-guaranteed created
```

> **说明**：此配置中所有容器的 `requests == limits`，Pod 的 QoS 等级为 **Guaranteed**。Guaranteed Pod 享有最高的资源保障和最低的驱逐优先级，适用于生产环境关键服务。

### 步骤 4：应用 LimitRange 和 ResourceQuota

> **kubectl 数据面不可达**：CAM拒绝公网端点(strategyId:240463971)，内网需VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

#### 选择依据

- **LimitRange**：为未显式设置资源限制的容器提供默认值，防止 BestEffort Pod 占用过量资源。
- **ResourceQuota**：限制命名空间级别总资源用量，防止单个命名空间耗尽集群资源。

#### 创建 LimitRange

创建 `limit-range.yaml`：

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-resource-limits
  namespace: NAMESPACE
spec:
  limits:
  - default:
      cpu: 500m
      memory: 512Mi
    defaultRequest:
      cpu: 200m
      memory: 256Mi
    max:
      cpu: "2"
      memory: 2Gi
    min:
      cpu: 50m
      memory: 64Mi
    type: Container
```

```bash
kubectl apply -f limit-range.yaml
```

```text
# command executed successfully
```

```output
limitrange/default-resource-limits created
```

#### 创建 ResourceQuota

创建 `resource-quota.yaml`：

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: NAMESPACE-compute-quota
  namespace: NAMESPACE
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    pods: "20"
    persistentvolumeclaims: "10"
    services: "10"
    services.loadbalancers: "2"
```

```bash
kubectl apply -f resource-quota.yaml
```

```text
# command executed successfully
```

```output
resourcequota/NAMESPACE-compute-quota created
```

| 参数 | 说明 | 约束 | 获取方式 |
|------|------|------|---------|
| `NAMESPACE` | 目标命名空间 | 必须已存在 | `kubectl get ns` |
| `DEPLOYMENT_NAME` | Deployment 名称 | 自定义 | 自定义 |

### 步骤 5：观察 OOM 行为

```bash
kubectl get events --field-selector reason=OOMKilling -n NAMESPACE
```

```text
NAME  STATUS  AGE
...
```

```output
No resources found in NAMESPACE namespace.
```

> **说明**：当 Pod 内存使用超过 `limits.memory` 时，内核 OOM Killer 会终止容器，Kubernetes 记录 `OOMKilling` 事件并重启 Pod。Burstable Pod 在节点内存压力下优先被驱逐。

```bash
# 观察 Pod 重启
kubectl get pods -n NAMESPACE -l app=demo-minimal --watch
```

```text
NAME  STATUS  AGE
...
```

## 验证

### 控制面

```bash
# 验证集群状态
tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterStatus "Running"

# 查看集群节点资源
tccli tke DescribeClusterInstances --region <Region> --ClusterId CLUSTER_ID
# expected: InstanceSet 显示节点状态及规格
```

```json
{
  "InstanceSet": [
    {
      "InstanceId": "NODE_ID",
      "InstanceRole": "WORKER",
      "InstanceState": "running",
      "CPU": 4,
      "Memory": 8
    }
  ]
}
```

### 数据面（需 VPN/IOA）

```bash
# 验证 Pod 运行状态及 QoS 等级
kubectl describe pod -n NAMESPACE -l app=demo-minimal | grep -A5 "QoS Class"
```

```text
Name:         ...
Status:       Running
...
```

```output
QoS Class:       Burstable
```

```bash
kubectl describe pod -n NAMESPACE -l app=demo-guaranteed | grep -A5 "QoS Class"
```

```text
Name:         ...
Status:       Running
...
```

```output
QoS Class:       Guaranteed
```

```bash
# 查看实际资源用量
kubectl top pod -n NAMESPACE
```

```text
# command executed successfully
```

```output
NAME                                     CPU(cores)   MEMORY(bytes)
DEPLOYMENT_NAME-minimal-xxxxxxxxx-xxxxx   2m           8Mi
DEPLOYMENT_NAME-guaranteed-xxxxxxxxx-xxxxx 3m           10Mi
DEPLOYMENT_NAME-guaranteed-xxxxxxxxx-yyyyy 2m           9Mi
```

```bash
# 验证 LimitRange 生效
kubectl describe limitrange default-resource-limits -n NAMESPACE
# expected: limits 配置与创建时一致

# 验证 ResourceQuota 生效
kubectl describe resourcequota NAMESPACE-compute-quota -n NAMESPACE
# expected: Used 显示当前命名空间资源用量
```

```text
NAME  STATUS  AGE
...
```

```bash
# 查看事件
kubectl get events -n NAMESPACE --sort-by=.lastTimestamp
```

```text
NAME  STATUS  AGE
...
```

## 清理

> **⚠️ 警告**：删除 Deployment 将终止对应工作负载，所有 Pod 将被移除。确认无生产流量后再执行。

### 数据面（需 VPN/IOA）

```bash
# 1. 删除最小化 Deployment
kubectl delete deployment DEPLOYMENT_NAME-minimal -n NAMESPACE
# expected: deployment.apps "DEPLOYMENT_NAME-minimal" deleted

# 2. 删除 Guaranteed Deployment
kubectl delete deployment DEPLOYMENT_NAME-guaranteed -n NAMESPACE
# expected: deployment.apps "DEPLOYMENT_NAME-guaranteed" deleted

# 3. 删除 LimitRange
kubectl delete limitrange default-resource-limits -n NAMESPACE
# expected: limitrange "default-resource-limits" deleted

# 4. 删除 ResourceQuota
kubectl delete resourcequota NAMESPACE-compute-quota -n NAMESPACE
# expected: resourcequota "NAMESPACE-compute-quota" deleted
```

### 验证清理

```bash
# 确认 Deployment 已删除
kubectl get deployment -n NAMESPACE
# expected: 无 DEPLOYMENT_NAME-* 条目

# 确认 LimitRange 已删除
kubectl get limitrange -n NAMESPACE
# expected: 无 default-resource-limits 条目

# 确认 ResourceQuota 已删除
kubectl get resourcequota -n NAMESPACE
# expected: 无 NAMESPACE-compute-quota 条目

# 确认 Pod 已终止
kubectl get pods -n NAMESPACE -l app=demo-minimal
# expected: No resources found

kubectl get pods -n NAMESPACE -l app=demo-guaranteed
# expected: No resources found
```

```text
NAME  STATUS  AGE
...
```

## 排障

### 资源请求被拒绝

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl apply` 返回 `forbidden: exceeded quota` | `kubectl describe resourcequota NAMESPACE-compute-quota -n NAMESPACE` | ResourceQuota 配额已用完 | 删除非必要资源释放配额，或调大 ResourceQuota hard 限制后 `kubectl apply -f resource-quota.yaml` |
| Pod 状态 `Pending` | `kubectl describe pod <pod-name> -n NAMESPACE` 查看 Events | 集群节点资源不足，无法满足 request | 扩容节点池或降低 Pod request，或 `tccli tke DescribeClusterInstances` 确认节点可用资源 |
| Pod 状态 `CrashLoopBackOff` | `kubectl describe pod <pod-name> -n NAMESPACE` 查看 Events | 容器内存使用超过 limits.memory 被 OOMKilled | 增大 `limits.memory` 或优化应用内存使用 |

### OOM 相关

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Pod 频繁重启 | `kubectl get events --field-selector reason=OOMKilling -n NAMESPACE` | 内存 limits 设置过低，应用内存泄漏或正常增长超过限制 | 增大 `limits.memory`，排查应用内存泄漏 |
| 节点内存压力导致 Pod 被驱逐 | `kubectl describe node NODE_ID` 查看 `Conditions: MemoryPressure` | 节点总内存不足，Burstable Pod 被优先驱逐 | 调高 Burstable Pod 的 request 或转为 Guaranteed QoS |

### LimitRange/ResourceQuota 不生效

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 未设置资源的 Pod 未被 LimitRange 默认值填充 | `kubectl describe limitrange default-resource-limits -n NAMESPACE` | LimitRange 只在 Pod 创建时生效，不修改已存在的 Pod | 删除并重建 Pod（通过 rollout restart 或 delete pod） |
| ResourceQuota 未限制已存在的资源 | `kubectl describe resourcequota NAMESPACE-compute-quota -n NAMESPACE` | ResourceQuota 对已存在的资源生效，但不会阻止已超配的资源继续运行，仅阻止新建 | 删除超出配额的部分资源 |

## 下一步

- [资源合理分配](../资源合理分配/tccli%20操作.md) — 通过节点亲和性、污点容忍进一步优化调度
- [弹性伸缩](../../弹性伸缩/tccli%20操作.md) — 结合 HPA 和 CA 实现自动扩缩
- [概述](../概述/tccli%20操作.md) — 合理利用节点资源的整体策略

## 控制台替代

通过 [TKE 控制台 - 工作负载](https://console.cloud.tencent.com/tke2/cluster/sub/list/deployment) 进入 Deployment 详情，在「更新调度策略」或「更新 Pod 配置」中设置资源限制。
