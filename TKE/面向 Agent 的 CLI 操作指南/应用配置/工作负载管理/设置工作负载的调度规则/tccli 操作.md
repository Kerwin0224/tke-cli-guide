# 设置工作负载的调度规则（tccli）

> 对照官方：[设置工作负载的调度规则](https://cloud.tencent.com/document/product/457/32814) · page_id `32814`

## 概述

通过 Kubernetes 调度策略控制 Pod 在集群节点上的分布，支持 nodeSelector、nodeAffinity、podAffinity/podAntiAffinity、taints/tolerations 等机制。

## 前置条件

- [环境准备](../../../环境准备.md)：`tccli` 已配置，`kubectl` >= v1.28
- 需要 CAM 权限：`tke:DescribeClusters`、`tke:DescribeClusterKubeconfig`

```bash
# 检查集群状态
tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'
# expected: ClusterStatus "Running"

# 获取 kubeconfig
tccli tke DescribeClusterKubeconfig --region <Region> --ClusterId <ClusterId> \
    | jq -r '.Kubeconfig' > kubeconfig.yaml
export KUBECONFIG=$(pwd)/kubeconfig.yaml
```

> **注意**：kubectl 数据面当前因公网端点被组织级 CAM 策略拒绝（strategyId:240463971）而不可达。外网端点被 `tke:clusterExtranetEndpoint=true` 条件拦截，内网端点需通过 IOA/VPN 或同 VPC 内 CVM 访问。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 获取集群凭证 | `tccli tke DescribeClusterKubeconfig --region <Region> --ClusterId <ClusterId>` | 是 |
| 设置节点选择器 | `kubectl apply -f deployment-with-nodeselector.yaml` | 是 |
| 设置亲和性 | `kubectl apply -f deployment-with-affinity.yaml` | 是 |
| 设置污点容忍 | `kubectl apply -f deployment-with-tolerations.yaml` | 是 |
| 查看 Pod 调度节点 | `kubectl get pods -o wide` | 是 |
| 给节点加污点 | `kubectl taint nodes <node> key=value:NoSchedule` | 否 |
| 给节点加标签 | `kubectl label nodes <node> key=value` | 否 |

## 操作步骤

### 步骤 1：nodeSelector（简单节点选择）

YAML 清单：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: scheduled-demo
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: scheduled-demo
  template:
    metadata:
      labels:
        app: scheduled-demo
    spec:
      nodeSelector:
        node.kubernetes.io/instance-type: SA2.MEDIUM4
      containers:
        - name: nginx
          image: nginx:alpine
```

```bash
kubectl apply -f deployment-nodeselector.yaml
# expected: deployment.apps/scheduled-demo created
```

### 步骤 2：nodeAffinity（节点亲和性）

支持 `requiredDuringSchedulingIgnoredDuringExecution`（硬约束）和 `preferredDuringSchedulingIgnoredDuringExecution`（软约束）：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: affinity-demo
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: affinity-demo
  template:
    metadata:
      labels:
        app: affinity-demo
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: topology.kubernetes.io/zone
                    operator: In
                    values:
                      - ap-guangzhou-3
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              preference:
                matchExpressions:
                  - key: disktype
                    operator: In
                    values:
                      - ssd
      containers:
        - name: nginx
          image: nginx:alpine
```

```bash
kubectl apply -f deployment-affinity.yaml
# expected: deployment.apps/affinity-demo created
```

### 步骤 3：podAntiAffinity（Pod 反亲和性，分散部署）

将同一应用的 Pod 分散到不同节点，提高可用性：

```yaml
spec:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchExpressions:
              - key: app
                operator: In
                values:
                  - affinity-demo
          topologyKey: kubernetes.io/hostname
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          podAffinityTerm:
            labelSelector:
              matchExpressions:
                - key: app
                  operator: In
                  values:
                    - affinity-demo
            topologyKey: topology.kubernetes.io/zone
```

### 步骤 4：Taints 与 Tolerations

给节点添加污点，使默认 Pod 不被调度到该节点：

```bash
# 给节点添加污点
kubectl taint nodes <node-name> dedicated=special:NoSchedule
# expected: node/<node-name> tainted

# 验证污点
kubectl describe node <node-name> | grep Taints
# expected: dedicated=special:NoSchedule
```

为 Pod 添加容忍，使其可调度到带污点的节点：

```yaml
spec:
  tolerations:
    - key: "dedicated"
      operator: "Equal"
      value: "special"
      effect: "NoSchedule"
```

## 验证

### 控制面（tccli）

```bash
tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'
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

### 数据面（kubectl）

```bash
kubectl get pods -o wide
# expected: 显示 Pod 调度的节点名和 IP

kubectl describe node <node-name> | grep -A5 "Taints\|Allocated"
# expected: 节点污点和资源分配
```

```text
NAME  STATUS  AGE
...
```

## 清理

```bash
kubectl delete deployment scheduled-demo affinity-demo
# expected: deployment.apps "scheduled-demo" deleted

# 移除污点
kubectl taint nodes <node-name> dedicated:NoSchedule-
# expected: node/<node-name> untainted
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `Unable to connect to the server` | `kubectl cluster-info` 检查连通性 | 公网端点被 CAM 策略 strategyId:240463971 以 `tke:clusterExtranetEndpoint=true` 条件拒绝 | 通过 IOA/VPN 连接内网，或使用同 VPC CVM 执行 kubectl 命令 |

### 资源创建成功但状态异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Pod `Pending`（调度失败） | `kubectl describe pod <pod>` 查看 Events | 无节点匹配 nodeSelector/Affinity | 检查节点标签和调度约束 |
| Pod 集中在少数节点 | `kubectl get pods -o wide` 查看分布 | 未配置 podAntiAffinity | 添加 podAntiAffinity 分散部署 |
| 污点后 Pod 被驱逐 | `kubectl describe pod <pod>` 查看 Events | Pod 无对应 toleration | 添加 toleration 或移除 taint |

## 下一步

- [设置工作负载的健康检查](../设置工作负载的健康检查/tccli%20操作.md) — page_id `32815`
- [设置工作负载的资源限制](../设置工作负载的资源限制/tccli%20操作.md) — page_id `32813`

## 控制台替代

[容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) → 集群 → 工作负载 → 新建 → 高级设置 → 调度策略。
