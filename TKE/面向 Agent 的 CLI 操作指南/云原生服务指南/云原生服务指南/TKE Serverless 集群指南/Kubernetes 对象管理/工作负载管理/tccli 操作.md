# 工作负载管理（tccli）

> 对照官方：[工作负载管理](https://cloud.tencent.com/document/product/457/39816) · page_id `39816`

## 概述

在 TKE Serverless 集群中管理工作负载（Deployment、StatefulSet、Job、CronJob），通过 `kubectl` 命令行进行创建、查看、更新和删除操作。与标准集群不同，Serverless 集群无物理节点，工作负载创建时会为每个 Pod 分配实际的云资源。

> **重要提示：** Serverless 容器服务已升级为 TKE 标准集群 + 超级节点模式，新建 Serverless 集群入口已关闭。存量用户集群不受影响。

> **注意：** kubectl 不可达（公网端点 CAM 拒绝 strategyId:240463971）。以下 `kubectl` 命令是正确的 CLI 操作方式，实际执行需解决 CAM 权限或使用 Cloud Shell 连接。

## 前置条件

- [环境准备](../../环境准备.md)
- 已创建 Serverless 集群且状态为 `Running`，参见[创建集群](../../TKE%20Serverless%20集群管理/创建集群/tccli%20操作.md)
- 已[连接集群](../../TKE%20Serverless%20集群管理/连接集群/tccli%20操作.md)，`kubectl` 可用
- 集群中存在 `Active` 状态的命名空间

## 控制台与 CLI 参数映射

| 控制台操作 | kubectl 命令 | 幂等 |
|-----------|-------------|------|
| 创建 Deployment | `kubectl create deployment` 或 `kubectl apply -f <yaml>` | 否（apply 是幂等） |
| 查看 Deployment | `kubectl get deployments` | 是 |
| 查看 Deployment 详情 | `kubectl describe deployment <name>` | 是 |
| 更新 Deployment | `kubectl set image` 或 `kubectl edit deployment` | 否 |
| 扩缩容 | `kubectl scale deployment <name> --replicas=<N>` | 否（重复相同值幂等） |
| 删除 Deployment | `kubectl delete deployment <name>` | 是 |
| 创建 StatefulSet | `kubectl apply -f <yaml>` | 是 |
| 创建 Job | `kubectl create job <name> --image=<image>` 或 `kubectl apply -f <yaml>` | 否 |
| 创建 CronJob | `kubectl apply -f <yaml>` | 是 |
| 查看 Pod 日志 | `kubectl logs <pod-name>` | 是 |

## 操作步骤

### 1. Deployment（无状态应用）

#### 创建 Deployment

**方式一：直接命令**

```bash
kubectl create deployment <deployment-name> \
    --image=<image> \
    --replicas=<N> \
    --namespace=<namespace>
```

示例：

```bash
kubectl create deployment nginx-serverless \
    --image=nginx:1.25 \
    --replicas=3 \
    --namespace=default
```

**方式二：YAML 文件（推荐，支持 Serverless 特有 Annotation）**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-serverless
  namespace: default
  labels:
    app: nginx
spec:
  replicas: 3
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
        resources:
          requests:
            cpu: "1"
            memory: "2Gi"
          limits:
            cpu: "1"
            memory: "2Gi"
```

```bash
kubectl apply -f deployment-nginx.yaml
```

```text
# command executed successfully
```

#### 查看 Deployment

```bash
kubectl get deployments --namespace=default
kubectl describe deployment nginx-serverless --namespace=default
```

示例输出（`kubectl get deployments`）：

```text
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-serverless   3/3     3            3           2m
```

#### 更新 Deployment

```bash
# 更新镜像
kubectl set image deployment/nginx-serverless nginx=nginx:1.26 --namespace=default

# 查看滚动更新状态
kubectl rollout status deployment/nginx-serverless --namespace=default
```

#### 扩缩容

```bash
kubectl scale deployment nginx-serverless --replicas=5 --namespace=default
```

#### 删除 Deployment

```bash
kubectl delete deployment nginx-serverless --namespace=default
```

### 2. StatefulSet（有状态应用）

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql-serverless
  namespace: default
spec:
  serviceName: mysql
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
      annotations:
        eks.tke.cloud.tencent.com/cpu: "2"
        eks.tke.cloud.tencent.com/mem: "4Gi"
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: password
        ports:
        - containerPort: 3306
        resources:
          requests:
            cpu: "2"
            memory: "4Gi"
          limits:
            cpu: "2"
            memory: "4Gi"
```

```bash
kubectl apply -f statefulset-mysql.yaml
kubectl get statefulsets --namespace=default
kubectl describe statefulset mysql-serverless --namespace=default
```

> **建议：** 若无持久化标识需求，优先使用 Deployment 而非 StatefulSet。

### 3. Job（一次性任务）

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi-calculation
  namespace: default
spec:
  completions: 1
  parallelism: 1
  template:
    metadata:
      annotations:
        eks.tke.cloud.tencent.com/cpu: "1"
        eks.tke.cloud.tencent.com/mem: "1Gi"
    spec:
      containers:
      - name: pi
        image: perl:5.34
        command: ["perl", "-Mbignum=bpi", "-wle", "print bpi(2000)"]
        resources:
          requests:
            cpu: "1"
            memory: "1Gi"
          limits:
            cpu: "1"
            memory: "1Gi"
      restartPolicy: Never
```

```bash
kubectl apply -f job-pi.yaml
kubectl get jobs --namespace=default
kubectl logs job/pi-calculation --namespace=default
```

> **注意：** Job 完成后底层计算资源会被释放，日志将无法通过控制台查看。建议在 Job 运行期间及时查看日志。

### 4. CronJob（定时任务）

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: daily-cleanup
  namespace: default
spec:
  schedule: "0 2 * * *"
  jobTemplate:
    spec:
      template:
        metadata:
          annotations:
            eks.tke.cloud.tencent.com/cpu: "1"
            eks.tke.cloud.tencent.com/mem: "1Gi"
        spec:
          containers:
          - name: cleanup
            image: busybox:1.36
            command: ["sh", "-c", "echo 'Cleanup job running at $(date)'"]
            resources:
              requests:
                cpu: "1"
                memory: "1Gi"
              limits:
                cpu: "1"
                memory: "1Gi"
          restartPolicy: OnFailure
```

```bash
kubectl apply -f cronjob-cleanup.yaml
kubectl get cronjobs --namespace=default
```

```text
# command executed successfully
```

### 通过 tccli 查看容器实例（Pod 的底层资源）

```bash
# 查看所有容器实例（对应集群中的 Pod）
tccli tke DescribeEKSContainerInstances \
    --region ap-guangzhou \
    --output json
```

```json
{
  "TotalCount": 0,
  "EksCis": [],
  "AutoCreatedEipId": "<AutoCreatedEipId>",
  "CamRoleName": "<CamRoleName>",
  "Containers": [],
  "Image": "<Image>",
  "Name": "<Name>",
  "Args": []
}
```

## 验证

| 验证项 | 命令 | 期望结果 |
|--------|------|---------|
| Deployment 已创建 | `kubectl get deployments -n default` | `nginx-serverless` 在列表中，READY 列显示副本数 |
| Pod 正在运行 | `kubectl get pods -n default` | 所有 Pod 状态为 `Running` |
| 容器实例对应 Pod | `tke DescribeEKSContainerInstances --region ap-guangzhou` | 返回 `EksCiSet` 列表 |
| Job 已完成 | `kubectl get jobs -n default` | `COMPLETIONS` 为 `1/1` |
| CronJob 已调度 | `kubectl get cronjobs -n default` | `daily-cleanup` 在列表中 |

## 清理

```bash
# 删除 Deployment
kubectl delete deployment nginx-serverless --namespace=default

# 删除 StatefulSet
kubectl delete statefulset mysql-serverless --namespace=default

# 删除 Job
kubectl delete job pi-calculation --namespace=default

# 删除 CronJob
kubectl delete cronjob daily-cleanup --namespace=default
```

> **注意：** 删除工作负载后底层容器实例资源会被释放。删除 Deployment 会同时删除对应的 Pod。

## 排障

| 现象 | 原因 | 处理 |
|------|------|------|
| Pod 一直 `Pending` | 子网 IP 不足或超级节点不可调度 | 检查子网可用 IP 数，检查 `kubectl describe pod` 查看 Events |
| Pod 创建后立即 `Failed` | 镜像拉取失败或启动命令错误 | `kubectl logs <pod-name>` 查看错误日志，检查镜像名称与 tag |
| `kubectl` 命令不可用 | 未获取 kubeconfig 或端点未开启 | 参见 [连接集群](../../TKE%20Serverless%20集群管理/连接集群/tccli%20操作.md) |
| Job 日志无法查看 | Job 完成后资源释放 | 在 Job 运行期间使用 `kubectl logs -f` 实时查看 |
| CronJob 未按预期触发 | Schedule 表达式错误 | Cron 格式为 `分 时 日 月 周`，检查是否填写错误 |

## 下一步

- [服务管理](../服务管理/tccli%20操作.md) — 为工作负载创建 Service 暴露入口
- [其他资源管理](../其他资源管理/tccli%20操作.md) — 管理 ConfigMap、Secret 等资源

## 控制台替代

[容器服务控制台 → 集群 → 目标集群 → 工作负载](https://console.cloud.tencent.com/tke2/cluster) — 在控制台通过可视化界面创建工作负载，选择类型后填写参数。
