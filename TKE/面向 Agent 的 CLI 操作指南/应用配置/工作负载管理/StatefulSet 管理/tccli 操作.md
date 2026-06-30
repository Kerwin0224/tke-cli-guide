# StatefulSet 管理（tccli）

> 对照官方：[StatefulSet 管理](https://cloud.tencent.com/document/product/457/31707) · page_id `31707`

## 概述

StatefulSet 是 Kubernetes 控制器，管理有状态应用的 Pod 声明式配置。与 Deployment 不同，StatefulSet 为每个 Pod 提供持久化标识（有序命名、稳定网络标识、持久存储），适用于数据库、消息队列等有状态服务。通过 kubectl 创建、扩缩、更新和回滚 StatefulSet。

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
| 创建 StatefulSet | `kubectl apply -f statefulset.yaml` | 是 |
| 查看 StatefulSet 列表 | `kubectl get statefulset` | 是 |
| 查看详情 | `kubectl describe statefulset/<name>` | 是 |
| 更新镜像 | `kubectl set image statefulset/<name> <container>=<image>` | 否 |
| 调整副本数 | `kubectl scale statefulset/<name> --replicas=<N>` | 是 |
| 滚动更新状态 | `kubectl rollout status statefulset/<name>` | 是 |
| 版本回滚 | `kubectl rollout undo statefulset/<name>` | 否 |
| 删除 StatefulSet | `kubectl delete statefulset/<name>` | 是 |

## 操作步骤

### 步骤 1：创建 Headless Service 和 StatefulSet

StatefulSet 需要 Headless Service 提供稳定网络标识。YAML 清单：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: statefulset-svc
  namespace: default
spec:
  ports:
    - port: 80
      name: web
  clusterIP: None
  selector:
    app: statefulset-example
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: statefulset-example
  namespace: default
spec:
  serviceName: "statefulset-svc"
  replicas: 3
  selector:
    matchLabels:
      app: statefulset-example
  template:
    metadata:
      labels:
        app: statefulset-example
    spec:
      containers:
        - name: nginx
          image: nginx:latest
          ports:
            - containerPort: 80
          volumeMounts:
            - name: data
              mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 1Gi
```

```bash
kubectl apply -f statefulset.yaml
# expected: statefulset.apps/statefulset-example created
```

**预期输出**：

```text
service/statefulset-svc created
statefulset.apps/statefulset-example created
```

### 步骤 2：查看 StatefulSet 状态

```bash
kubectl get statefulset statefulset-example
# expected: READY 列显示期望副本数
```

```text
NAME  STATUS  AGE
...
```

```bash
kubectl get pods -l app=statefulset-example
# expected: Pod 按序号命名（statefulset-example-0, -1, -2）
```

```text
NAME  STATUS  AGE
...
```

```bash
kubectl describe statefulset/statefulset-example
# expected: 显示 StatefulSet 详情和事件
```

```text
NAME  STATUS  AGE
...
```

### 步骤 3：扩缩容

```bash
kubectl scale statefulset/statefulset-example --replicas=5
# expected: statefulset.apps/statefulset-example scaled
```

### 步骤 4：滚动更新与回滚

```bash
kubectl set image statefulset/statefulset-example nginx=nginx:1.25
# expected: statefulset.apps/statefulset-example image updated

kubectl rollout status statefulset/statefulset-example
# expected: statefulset rolling update successfully rolled out

kubectl rollout undo statefulset/statefulset-example
# expected: statefulset.apps/statefulset-example rolled back
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
kubectl get statefulset statefulset-example
# expected: READY 3/3

kubectl get pods -l app=statefulset-example
# expected: 关联 Pod Running，有序命名

kubectl get pvc -l app=statefulset-example
# expected: 每个 Pod 对应一个 PVC
```

```text
NAME  STATUS  AGE
...
```

## 清理

```bash
kubectl delete statefulset statefulset-example
# expected: statefulset.apps "statefulset-example" deleted

kubectl delete service statefulset-svc
# expected: service "statefulset-svc" deleted
```

> **注意**：StatefulSet 删除后 PVC 不会被自动删除，需手动清理：

```bash
kubectl delete pvc -l app=statefulset-example
# expected: PVC 已删除
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `Unable to connect to the server` | `kubectl cluster-info` 检查连通性 | 公网端点被 CAM 策略 strategyId:240463971 以 `tke:clusterExtranetEndpoint=true` 条件拒绝 | 通过 IOA/VPN 连接内网，或使用同 VPC CVM 执行 kubectl 命令 |
| Pod `ImagePullBackOff` | `kubectl describe pod <pod>` 查看事件 | 镜像拉取失败 | 检查镜像地址和访问凭证 |
| Pod `Pending` | `kubectl describe pod <pod>` 检查 Events | 资源不足或无可用 PV | 检查节点资源、StorageClass 和 PV 供给 |

### 资源创建成功但状态异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Pod `CrashLoopBackOff` | `kubectl logs <pod>` 查看容器日志 | 容器启动崩溃 | 检查应用启动命令和配置 |
| Pod 卡在 `ContainerCreating` | `kubectl describe pod <pod>` 查看 Events | PVC 绑定失败 | 确认 StorageClass 存在且有可用 PV |

## 下一步

- [Deployment 管理](../Deployment%20管理/tccli%20操作.md) — page_id `31705`
- [设置工作负载的健康检查](../设置工作负载的健康检查/tccli%20操作.md) — page_id `32815`

## 控制台替代

[容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) → 集群 → 工作负载 → StatefulSet → 新建。
