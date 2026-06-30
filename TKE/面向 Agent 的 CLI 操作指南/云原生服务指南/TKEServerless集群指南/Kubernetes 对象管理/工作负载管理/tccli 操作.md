# 工作负载管理

> 对照官方：[工作负载管理](https://cloud.tencent.com/document/product/457/39816) · page_id `39816`

## 概述

TKE Serverless 集群支持通过 kubectl 管理 Kubernetes 工作负载，包括 Deployment、StatefulSet、Job、CronJob 等。与标准 TKE 集群不同，Serverless 集群中 Pod 无需指定节点，平台自动将 Pod 调度到弹性容器实例上运行。Pod 的 CPU 和内存资源通过 annotations 声明，而不是传统的 `resources.requests/limits`，这使得 Pod 可以按实际使用量精准计费。

**重要**：Serverless 集群中 Pod 的 CPU/内存规格通过 `eks.tke.cloud.tencent.com/cpu` 和 `eks.tke.cloud.tencent.com/mem` annotation 指定，单位为核和 Gi。

## 前置条件

- [环境准备](../../../../环境准备.md)

### 集群可达性检查（Before you begin）

工作负载操作为 kubectl-only 操作。确认集群连接正常后方可继续。

```bash
# 1. 确认集群处于 Running 状态
tccli tke DescribeEKSClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterStatus: Running

# 2. 验证 kubectl 连接
kubectl --kubeconfig ~/.kube/config-eks-CLUSTER_ID cluster-info
# expected: Kubernetes control plane is running at https://...

# 3. 确认 kubectl 版本兼容
kubectl version --client --short
# expected: 版本与集群 Kubernetes 版本偏差不超过 1 个小版本
```

## 控制台与 CLI 参数映射

| 控制台操作 | kubectl 命令 | 幂等 |
|-----------|-----------|:--:|
| 创建 Deployment | `kubectl apply -f deployment.yaml` | 是 |
| 查看工作负载 | `kubectl get deployments -n NAMESPACE` | 是 |
| 扩缩容 | `kubectl scale deployment/NAME --replicas=N -n NAMESPACE` | 否 |
| 更新镜像 | `kubectl set image deployment/NAME CONTAINER=IMAGE -n NAMESPACE` | 是 |
| 回滚 | `kubectl rollout undo deployment/NAME -n NAMESPACE` | 否 |
| 删除工作负载 | `kubectl delete deployment/NAME -n NAMESPACE` | 是 |

## 操作步骤

### 步骤 1: 创建 Deployment

**选择依据**：
- **replicas**：测试环境 1-2 副本，生产环境按负载需求设定
- **cpu annotation**：Pod 的 vCPU 核数，可选 0.25/0.5/1/2/4/8/16 等
- **mem annotation**：Pod 的内存 Gi 数，需与 CPU 规格匹配（1C 对应 1-8Gi）
- **image**：选择与架构匹配的容器镜像

`nginx-deployment.yaml`：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: NAMESPACE
  labels:
    app: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
      annotations:
        eks.tke.cloud.tencent.com/cpu: "1"
        eks.tke.cloud.tencent.com/mem: "2Gi"
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
```

```bash
kubectl --kubeconfig ~/.kube/config-eks-CLUSTER_ID apply -f nginx-deployment.yaml
# expected: deployment.apps/nginx-deployment created
```

### 步骤 2: 查看工作负载状态

```bash
kubectl --kubeconfig ~/.kube/config-eks-CLUSTER_ID get deployments -n NAMESPACE
# expected: nginx-deployment 显示 READY 2/2

kubectl --kubeconfig ~/.kube/config-eks-CLUSTER_ID get pods -n NAMESPACE -o wide
# expected: 2 个 Pod 状态为 Running
```

预期输出：

```
NAME               READY   STATUS    RESTARTS   AGE
nginx-deployment-xxxx-xxxxx   1/1     Running   0          30s
nginx-deployment-xxxx-yyyyy   1/1     Running   0          30s
```

### 步骤 3: 扩缩容

```bash
# 扩容到 3 个副本
kubectl --kubeconfig ~/.kube/config-eks-CLUSTER_ID scale deployment/nginx-deployment \
    --replicas=3 -n NAMESPACE
# expected: deployment.apps/nginx-deployment scaled

# 验证
kubectl --kubeconfig ~/.kube/config-eks-CLUSTER_ID get pods -n NAMESPACE
# expected: 3 个 Pod 状态为 Running
```

### 步骤 4: 更新工作负载

```bash
# 更新镜像
kubectl --kubeconfig ~/.kube/config-eks-CLUSTER_ID set image deployment/nginx-deployment \
    nginx=nginx:1.26 -n NAMESPACE
# expected: deployment.apps/nginx-deployment image updated

# 查看滚动更新状态
kubectl --kubeconfig ~/.kube/config-eks-CLUSTER_ID rollout status deployment/nginx-deployment -n NAMESPACE
# expected: deployment "nginx-deployment" successfully rolled out
```

### 步骤 5: 创建 StatefulSet

Serverless 集群也支持 StatefulSet，适用于有状态应用：

`redis-statefulset.yaml`：

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
  namespace: NAMESPACE
spec:
  serviceName: redis
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
      annotations:
        eks.tke.cloud.tencent.com/cpu: "1"
        eks.tke.cloud.tencent.com/mem: "2Gi"
    spec:
      containers:
      - name: redis
        image: redis:7-alpine
        ports:
        - containerPort: 6379
```

```bash
kubectl --kubeconfig ~/.kube/config-eks-CLUSTER_ID apply -f redis-statefulset.yaml
# expected: statefulset.apps/redis created
```

### 步骤 6: 创建 CronJob

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hello-cron
  namespace: NAMESPACE
spec:
  schedule: "*/5 * * * *"
  jobTemplate:
    spec:
      template:
        metadata:
          annotations:
            eks.tke.cloud.tencent.com/cpu: "0.25"
            eks.tke.cloud.tencent.com/mem: "0.5Gi"
        spec:
          restartPolicy: OnFailure
          containers:
          - name: hello
            image: busybox:1.36
            command: ["echo", "Hello from Serverless CronJob"]
```

```bash
kubectl --kubeconfig ~/.kube/config-eks-CLUSTER_ID apply -f hello-cronjob.yaml
# expected: cronjob.batch/hello-cron created
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `NAMESPACE` | Kubernetes 命名空间 | 需预先存在，或设为 `default` | `kubectl get ns` |
| `CLUSTER_ID` | 集群 ID | `cls-xxxxxxxx` 格式 | `tccli tke DescribeEKSClusters --region <Region>` |
| `REGION` | 地域 | 如 `ap-guangzhou` | `tccli configure list` |

## 验证

| 维度 | 命令 | 预期 |
|------|------|------|
| Deployment 状态 | `kubectl get deploy -n NAMESPACE` | READY 列显示期望副本数全部就绪 |
| Pod 运行 | `kubectl get pods -n NAMESPACE` | 所有 Pod STATUS 为 Running |
| Pod 规格 | `kubectl describe pod POD_NAME -n NAMESPACE \| grep eks.tke` | cpu/mem annotation 与定义一致 |
| 滚动更新 | `kubectl rollout history deployment/nginx-deployment -n NAMESPACE` | 显示版本历史 |
| 日志正常 | `kubectl logs deployment/nginx-deployment -n NAMESPACE` | 无错误日志 |

```bash
# 综合验证
kubectl --kubeconfig ~/.kube/config-eks-CLUSTER_ID get all -n NAMESPACE
# expected: 列出 Deployment、ReplicaSet、Pod，状态均为健康
```

## 清理

> **警告**：删除工作负载会终止所有关联 Pod。StatefulSet 的 PVC（如有）默认不会被自动删除，需手动清理。

```bash
# 删除 Deployment
kubectl --kubeconfig ~/.kube/config-eks-CLUSTER_ID delete deployment/nginx-deployment -n NAMESPACE
# expected: deployment.apps "nginx-deployment" deleted

# 删除 StatefulSet
kubectl --kubeconfig ~/.kube/config-eks-CLUSTER_ID delete statefulset/redis -n NAMESPACE
# expected: statefulset.apps "redis" deleted

# 删除 CronJob
kubectl --kubeconfig ~/.kube/config-eks-CLUSTER_ID delete cronjob/hello-cron -n NAMESPACE
# expected: cronjob.batch "hello-cron" deleted

# 验证清理
kubectl --kubeconfig ~/.kube/config-eks-CLUSTER_ID get pods -n NAMESPACE
# expected: No resources found 或仅剩系统 Pod
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Pod 一直 `Pending` | `kubectl describe pod POD_NAME -n NAMESPACE` 查看 Events | 资源配置不合理或底层资源不足 | 检查 CPU/Mem annotation 是否在允许范围内（如 0.25C/0.5Gi 起步） |
| Pod 报 `ImagePullBackOff` | `kubectl describe pod POD_NAME -n NAMESPACE \| grep -A5 Events` | 镜像不存在或无拉取权限 | 确认镜像名称正确；私有镜像需配置 imagePullSecrets |
| Pod 报 `CrashLoopBackOff` | `kubectl logs POD_NAME -n NAMESPACE` 查看日志 | 容器启动后立即退出 | 检查容器启动命令和应用日志定位原因 |
| `kubectl apply` 返回 `error: the server doesn't have a resource type` | `kubectl api-resources` 检查 API | 集群版本不支持该资源类型 | 确认资源类型与集群 K8s 版本兼容 |
| 扩容失败 `insufficient quota` | `kubectl get resourcequota -n NAMESPACE` 检查配额 | 命名空间 ResourceQuota 限制 | 调整 ResourceQuota 或减少副本数 |

### 操作成功但异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Pod 频繁重启 | `kubectl describe pod POD_NAME -n NAMESPACE` 查看 Restart Count | OOMKilled 或应用自身崩溃 | 提高 mem annotation 规格，或修复应用 bug |
| 滚动更新卡住 | `kubectl rollout status deployment/NAME -n NAMESPACE` | 新 Pod 无法变为 Ready | `kubectl describe pod` 排查新 Pod 启动问题 |
| Pod 启动慢 | `kubectl get events -n NAMESPACE --field-selector involvedObject.name=POD_NAME` | 首次调度需创建底层容器实例 | 通常 30s-2min 为正常范围，持续慢则检查镜像大小 |

## 下一步

- [服务管理](../服务管理/tccli%20操作.md) — 通过 Service 暴露工作负载
- [其他资源管理](../其他资源管理/tccli%20操作.md) — 管理 ConfigMap、Secret 等资源
- [监控和告警](../../运维中心/监控和告警/tccli%20操作.md) — 为工作负载配置 HPA 和监控

## 控制台替代

控制台：容器服务控制台 → 集群 → 选择目标 Serverless 集群 → 工作负载 → Deployment/StatefulSet/Job/CronJob → 新建。
