# Namespaces（tccli）

> 对照官方：[Namespaces](https://cloud.tencent.com/document/product/457/31701) · page_id `31701`

## 概述

Namespaces 是 Kubernetes 在同一个集群中进行逻辑环境划分的对象，可以通过 Namespaces 管理多个团队或多个项目的划分。每个 Namespace 下，Kubernetes 对象的名称必须唯一。可以通过资源配额分配可用资源，还可以进行不同 Namespaces 网络的访问控制。

控制面（tccli）用于获取集群信息；数据面（kubectl）完成 Namespace 的创建、资源配额（ResourceQuota）和网络策略（NetworkPolicy）配置。

## 前置条件

- [环境准备](../../../环境准备.md)：`tccli` 已配置
- 已获取集群 kubeconfig：

```bash
# 检查集群状态
tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'
# expected: ClusterStatus "Running"

# 获取 kubeconfig
tccli tke DescribeClusterKubeconfig --region <Region> --ClusterId <ClusterId> \
    | jq -r '.Kubeconfig' > kubeconfig.yaml
export KUBECONFIG=$(pwd)/kubeconfig.yaml
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

> **注意**：kubectl 数据面当前因公网端点被 CAM 策略 strategyId:240463971 拒绝而不可达。以下 kubectl 命令为参考格式，需在内网/VPN 环境执行。

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 查看集群 | `tccli tke DescribeClusters` | 是 |
| 查看命名空间列表 | `kubectl get namespaces` | 是 |
| 创建命名空间 | `kubectl create namespace <name>` | 否（同名报错） |
| 设置资源配额（配额管理） | `kubectl apply -f resourcequota.yaml` | 是 |
| 设置默认资源限制（LimitRange） | `kubectl apply -f limitrange.yaml` | 是 |
| 删除命名空间 | `kubectl delete namespace <name>` | 是（不存在不报错） |

## 操作步骤

### 使用方法

#### kubectl 创建 Namespace

**1. 确认集群信息**

```bash
tccli tke DescribeClusters --region ap-guangzhou --output json
```

```json
{
  "TotalCount": 1,
  "Clusters": [
    {
      "ClusterId": "cls-example",
      "ClusterName": "my-cluster",
      "ClusterStatus": "Running",
      "ClusterVersion": "1.28"
    }
  ]
}
```

**2. 创建 Namespace**

```bash
kubectl create namespace my-namespace
```

```
namespace/my-namespace created
```

**3. 查看 Namespace 列表**

```bash
kubectl get namespaces
```

```
NAME              STATUS   AGE
default           Active   30d
kube-node-lease   Active   30d
kube-public       Active   30d
kube-system       Active   30d
my-namespace      Active   5s
```

**4. 删除 Namespace**

```bash
kubectl delete namespace my-namespace
```

```
namespace "my-namespace" deleted
```

> 删除 Namespace 后，该 Namespace 下的资源对象也会被删除。

### 通过 ResourceQuota 设置 Namespaces 资源的使用配额

一个命名空间下可以拥有多个 ResourceQuota 资源，每个 ResourceQuota 可以设置每个 Namespace 资源的使用约束。可以设置 Namespaces 资源的使用约束如下：
- 计算资源的配额，例如 CPU、内存。
- 存储资源的配额，例如请求存储的总存储。
- Kubernetes 对象的计数，例如 Deployment 个数配额。

不同的 Kubernetes 版本，ResourceQuota 支持的配额设置略有差异，详情请参见 [Kubernetes ResourceQuota 官方文档](https://kubernetes.io/docs/concepts/policy/resource-quotas/)。

**ResourceQuota 示例：**

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: object-counts
  namespace: default
spec:
  hard:
    configmaps: "10"
    replicationcontrollers: "20"
    secrets: "10"
    services: "10"
    services.loadbalancers: "2"
    cpu: "1000"
    memory: 200Gi
```

字段说明：

| 配额项 | 含义 |
|--------|------|
| `configmaps: "10"` | 最多 10 个 ConfigMap |
| `replicationcontrollers: "20"` | 最多 20 个 ReplicationController |
| `secrets: "10"` | 最多 10 个 Secret |
| `services: "10"` | 最多 10 个 Service |
| `services.loadbalancers: "2"` | 最多 2 个 LoadBalancer 模式的 Service |
| `cpu: "1000"` | 该 Namespace 下最多使用 1000 个 CPU 的资源 |
| `memory: 200Gi` | 该 Namespace 下最多使用 200Gi 的内存 |

```bash
kubectl apply -f resourcequota.yaml
kubectl describe resourcequota -n default
```

### 设置 LimitRange（默认资源限制）

对命名空间设置 CPU/内存 limit/request 配额后，创建工作负载时，必须指定 CPU/内存 limit/request，或为命名空间配置 LimitRange。

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: my-namespace
spec:
  limits:
  - default:
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:
      cpu: "250m"
      memory: "256Mi"
    type: Container
```

```bash
kubectl apply -f limitrange.yaml
kubectl describe limitrange -n my-namespace
```

### 通过 NetworkPolicy 设置 Namespaces 网络的访问控制

Network Policy 是 Kubernetes 提供的一种资源，用于定义基于 Pod 的网络隔离策略。不仅可以限制 Namespaces，还可以控制 Pod 与 Pod 之间的网络访问控制。

**拒绝所有入站流量示例：**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: my-namespace
spec:
  podSelector: {}
  policyTypes:
  - Ingress
```

```bash
kubectl apply -f networkpolicy.yaml
```

```text
# command executed successfully
```

更多详情请参见 [使用 Network Policy 进行网络访问控制](https://cloud.tencent.com/document/product/457/19793)。

## 验证

### 数据面（kubectl）

```bash
kubectl get namespaces
kubectl describe namespace my-namespace
kubectl get resourcequota -n my-namespace
```

```text
NAME  STATUS  AGE
...
```

## 清理

### 数据面（kubectl）

```bash
kubectl delete namespace my-namespace  # 会同时删除该 Namespace 下所有资源
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `Unable to connect to the server` | `kubectl cluster-info` | 公网端点被 CAM 策略 strategyId:240463971 以 `tke:clusterExtranetEndpoint=true` 条件拒绝 | 通过 IOA/VPN 连接内网，或使用同 VPC CVM 执行 kubectl |
| 删除 Namespace 卡住（Terminating） | `kubectl get namespace <name> -o json \| jq '.spec.finalizers'` | 存在 Finalizer 阻止删除 | 手动清除 Finalizer 或排查依赖资源 |
| 创建工作负载时提示资源配额不足 | `kubectl describe resourcequota -n <ns>` | ResourceQuota 限制已达上限 | 调整 ResourceQuota 配额或清理无用资源 |
| 新建命名空间后镜像拉取失败 | `kubectl describe pod` | 新建命名空间不再自动下发镜像访问凭证 | 前往 Secret 页面创建镜像仓库密钥并下发至命名空间内 |

## 下一步

- [ConfigMap](../ConfigMap/tccli%20操作.md)
- [Secret](../Secret/tccli%20操作.md)
- [使用 Network Policy 进行网络访问控制](https://cloud.tencent.com/document/product/457/19793)

## 控制台替代

[控制台 → 集群 → 命名空间](https://console.cloud.tencent.com/tke2/cluster) 创建 / 配额管理 / 删除。
