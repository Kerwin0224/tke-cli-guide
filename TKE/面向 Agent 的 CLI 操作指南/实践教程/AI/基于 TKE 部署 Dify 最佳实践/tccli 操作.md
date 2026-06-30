
# 基于 TKE 部署 Dify 最佳实践（tccli）

> 对照官方：[基于 TKE 部署 Dify 最佳实践](https://cloud.tencent.com/document/product/457/118720) · page_id `118720`

## 概述

在 TKE 集群中部署 Dify 平台（LLM 应用开发平台），包含 API 服务、异步 Worker、Web 前端以及依赖的 PostgreSQL（元数据库）、Redis（缓存/队列）、Weaviate（向量数据库）等组件。

> **kubectl 数据面不可达**：CAM 拒绝公网端点（strategyId:240463971），内网需 VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

| 维度 | 外部 DB 模式 | 集群内 DB 模式（本页方案） |
|------|-----------|------------------------|
| 数据库 | 使用腾讯云 CDB PostgreSQL + Redis | 集群内 StatefulSet/Deployment 自建 |
| 运维复杂度 | 低，利用云服务 SLA | 高，需自行管理备份、扩缩容 |
| 成本 | CDB + Redis 云服务月费 | CVM/Pod 资源费（可复用已有节点） |
| 适用场景 | 生产环境 | 测试、POC、快速体验 |
| 高可用 | 云服务自带主备 | 需自行配置 StatefulSet 多副本 |

**选择依据**：本页演示集群内部署全部组件（自包含方案），适合快速体验和 POC 验证。生产环境建议将 PostgreSQL 和 Redis 替换为腾讯云 CDB/Redis 托管服务，参考「增强配置」中的外部数据库连接方式。

### 组件架构

```
┌────────────────────────────────────────────────┐
│                    TKE 集群                      │
│                                                 │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐  │
│  │Dify Web  │   │Dify API  │   │Dify Worker│  │
│  │ (前端)   │→  │ (主服务)  │←  │ (异步任务) │  │
│  └──────────┘   └────┬─────┘   └──────────┘  │
│                      │                         │
│         ┌────────────┼────────────┐            │
│         ↓            ↓            ↓            │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐      │
│  │PostgreSQL│ │  Redis   │ │ Weaviate │      │
│  │(元数据)  │ │(缓存/队列)│ │(向量存储) │      │
│  └──────────┘ └──────────┘ └──────────┘      │
│                                                 │
│  ┌──────────────────────────┐                  │
│  │       Ingress/LB          │                  │
│  └──────────────────────────┘                  │
└────────────────────────────────────────────────┘
```

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

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:DescribeClusters, tke:CreateCluster（如需新建集群）
#    vpc:DescribeVpcs, vpc:DescribeSubnets, cvm:DescribeInstances
#    cvm:DescribeInstanceConfigInfos, cvm:DescribeImages
# 验证：
tccli tke DescribeClusters --region <Region>
# expected: exit 0，返回集群列表

# 4. 检查 kubectl 和 helm
kubectl version --client
# expected: Client Version >= v1.28
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
# 5. 确认目标集群存在（或新建）且状态 Running
tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
# expected: TotalCount >= 1，ClusterStatus: "Running"

# 6. 检查可用存储类（用于 PVC）
kubectl get storageclass
# expected: 至少 1 个存储类（如 cbs 或 cloud-ssd）

# 7. 检查集群节点资源（Dify 全组件约需 8 核 32GB + 100GB 存储）
kubectl top nodes
# expected: allocatable cpu >= 8, memory >= 32Gi
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

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看集群列表 | `DescribeClusters` | 是 |
| 新建集群（若无） | `CreateCluster` | 否 |
| 获取 kubeconfig | `DescribeClusterKubeconfig` | 是 |

## 关键字段说明

如需新建集群部署 Dify，`CreateCluster` 的主要参数：

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `ClusterType` | String | 是 | `MANAGED_CLUSTER`（推荐托管集群） | 填错 → `InvalidParameter.ClusterType` |
| `ClusterVersion` | String | 是 | >= 1.28。`DescribeVersions` 查询可用版本 | 版本不存在 → `InvalidParameter.ClusterVersion` |
| `VpcId` | String | 是 | 格式 `vpc-xxxxxxxx`。`DescribeVpcs` 获取 | VPC 不存在 → `InvalidParameter.VpcId` |
| `SubnetId` | String | 是 | VPC 下的子网，需有足够 IP | 同上 |
| `ClusterCIDR` | String | 是 | Pod IP 范围，不能与 VPC CIDR 重叠 | CIDR 冲突 → `InvalidParameter.ClusterCIDRSettings` |
| `ServiceCIDR` | String | 是 | Service IP 范围，不能与其他 CIDR 重叠 | 同上 |

## 操作步骤

### 步骤 1：确认/创建 TKE 集群

#### 选择依据

如已有集群，直接确认可用。如无集群，创建托管集群（`MANAGED_CLUSTER`），选 `containerd` 运行时。Dify 对 GPU 无强制要求（LLM API 调用依赖外部模型提供商），普通 CVM 节点即可。

```bash
# 查现有集群
tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterStatus: "Running"
```

如需新建集群：

`cluster-dify-minimal.json`：

```json
{
  "ClusterType": "MANAGED_CLUSTER",
  "ClusterBasicSettings": {
    "ClusterName": "dify-cluster",
    "ClusterVersion": "1.30.0",
    "VpcId": "VPC_ID",
    "SubnetId": "SUBNET_ID"
  },
  "ClusterCIDRSettings": {
    "ClusterCIDR": "10.1.0.0/16",
    "ServiceCIDR": "10.2.0.0/20"
  }
}
```

```bash
tccli tke CreateCluster --region <Region> \
    --cli-input-json file://cluster-dify-minimal.json
# expected: exit 0，返回 ClusterId
```

等待集群 Running 后获取 kubeconfig：

```bash
tccli tke DescribeClusterKubeconfig --region <Region> \
    --ClusterId CLUSTER_ID | jq -r '.Kubeconfig' > ~/.kube/config
kubectl cluster-info
# expected: Kubernetes control plane is running
```

```json
{
  "Kubeconfig": "<Kubeconfig>",
  "RequestId": "<RequestId>"
}
```

### 步骤 2：创建 Namespace

```bash
kubectl create namespace dify
# expected: namespace/dify created

kubectl get ns dify
# expected: NAME: dify, STATUS: Active
```

### 步骤 3：部署 PostgreSQL（元数据库）

#### 选择依据

Dify 将用户、应用配置、对话历史等存储在 PostgreSQL 中。测试环境使用单副本 StatefulSet + PVC 即可。生产环境建议使用腾讯云 CDB PostgreSQL（设置 `DB_HOST` 为 CDB 内网地址）。

`postgres.yaml`：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: dify-postgres-secret
  namespace: dify
type: Opaque
stringData:
  postgres-password: "dify-postgres-pwd"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
  namespace: dify
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 20Gi
  storageClassName: cbs
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: dify
spec:
  serviceName: postgres
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:15-alpine
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_USER
          value: "dify"
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: dify-postgres-secret
              key: postgres-password
        - name: POSTGRES_DB
          value: "dify"
        - name: PGDATA
          value: "/var/lib/postgresql/data/pgdata"
        volumeMounts:
        - name: postgres-data
          mountPath: /var/lib/postgresql/data
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
          limits:
            cpu: 2
            memory: 4Gi
        livenessProbe:
          exec:
            command: ["pg_isready", "-U", "dify"]
          initialDelaySeconds: 30
          periodSeconds: 10
  volumeClaimTemplates:
  - metadata:
      name: postgres-data
    spec:
      accessModes: [ReadWriteOnce]
      resources:
        requests:
          storage: 20Gi
      storageClassName: cbs
---
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: dify
spec:
  selector:
    app: postgres
  ports:
  - port: 5432
    targetPort: 5432
```

```bash
kubectl apply -f postgres.yaml
# expected: secret/dify-postgres-secret created
#          persistentvolumeclaim/postgres-pvc created
#          statefulset.apps/postgres created
#          service/postgres created
```

等待就绪：

```bash
kubectl wait --for=condition=ready pod -l app=postgres -n dify --timeout=120s
# expected: pod/postgres-0 condition met
```

### 步骤 4：部署 Redis（缓存/消息队列）

`redis.yaml`：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: redis-config
  namespace: dify
data:
  redis.conf: |
    bind 0.0.0.0
    port 6379
    maxmemory 512mb
    maxmemory-policy allkeys-lru
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  namespace: dify
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:7-alpine
        command: ["redis-server", "/etc/redis/redis.conf"]
        ports:
        - containerPort: 6379
        volumeMounts:
        - name: config
          mountPath: /etc/redis
        resources:
          requests:
            cpu: 200m
            memory: 512Mi
          limits:
            cpu: 1
            memory: 1Gi
        livenessProbe:
          exec:
            command: ["redis-cli", "ping"]
          initialDelaySeconds: 10
      volumes:
      - name: config
        configMap:
          name: redis-config
---
apiVersion: v1
kind: Service
metadata:
  name: redis
  namespace: dify
spec:
  selector:
    app: redis
  ports:
  - port: 6379
    targetPort: 6379
```

```bash
kubectl apply -f redis.yaml
# expected: configmap/redis-config created
#          deployment.apps/redis created
#          service/redis created

kubectl wait --for=condition=ready pod -l app=redis -n dify --timeout=60s
# expected: pod/redis-xxxxxxxxxx-xxxxx condition met
```

### 步骤 5：部署 Weaviate（向量数据库）

`weaviate.yaml`：

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: weaviate-pvc
  namespace: dify
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 30Gi
  storageClassName: cbs
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: weaviate
  namespace: dify
spec:
  serviceName: weaviate
  replicas: 1
  selector:
    matchLabels:
      app: weaviate
  template:
    metadata:
      labels:
        app: weaviate
    spec:
      containers:
      - name: weaviate
        image: semitechnologies/weaviate:1.24.1
        ports:
        - containerPort: 8080
        env:
        - name: AUTHENTICATION_ANONYMOUS_ACCESS_ENABLED
          value: "true"
        - name: PERSISTENCE_DATA_PATH
          value: "/var/lib/weaviate"
        - name: QUERY_DEFAULTS_LIMIT
          value: "25"
        - name: DEFAULT_VECTORIZER_MODULE
          value: "none"
        - name: ENABLE_MODULES
          value: ""
        - name: CLUSTER_HOSTNAME
          value: "node1"
        volumeMounts:
        - name: weaviate-data
          mountPath: /var/lib/weaviate
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
          limits:
            cpu: 2
            memory: 4Gi
        livenessProbe:
          httpGet:
            path: /v1/.well-known/ready
            port: 8080
          initialDelaySeconds: 30
  volumeClaimTemplates:
  - metadata:
      name: weaviate-data
    spec:
      accessModes: [ReadWriteOnce]
      resources:
        requests:
          storage: 30Gi
      storageClassName: cbs
---
apiVersion: v1
kind: Service
metadata:
  name: weaviate
  namespace: dify
spec:
  selector:
    app: weaviate
  ports:
  - port: 8080
    targetPort: 8080
```

```bash
kubectl apply -f weaviate.yaml
# expected: persistentvolumeclaim/weaviate-pvc created
#          statefulset.apps/weaviate created
#          service/weaviate created

kubectl wait --for=condition=ready pod -l app=weaviate -n dify --timeout=120s
# expected: pod/weaviate-0 condition met
```

### 步骤 6：创建 Dify 配置

#### 选择依据

Dify API 通过环境变量配置所有依赖服务地址。必须配置的关键变量：`DB_HOST`/`DB_PASSWORD`（PostgreSQL）、`REDIS_HOST`（Redis）、`VECTOR_STORE`/`WEAVIATE_ENDPOINT`（向量数据库）、LLM 模型提供商的 API Key（如 OpenAI、腾讯混元）。Secret Key 放入 Secret 中保护。

`dify-config.yaml`：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: dify-secrets
  namespace: dify
type: Opaque
stringData:
  secret-key: "dify-secret-key-change-in-production"
  db-password: "dify-postgres-pwd"
  openai-api-key: "sk-YOUR_OPENAI_API_KEY"
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: dify-config
  namespace: dify
data:
  # 运行模式
  MODE: "api"
  LOG_LEVEL: "INFO"
  # 数据库
  DB_USERNAME: "dify"
  DB_HOST: "postgres.dify.svc.cluster.local"
  DB_PORT: "5432"
  DB_DATABASE: "dify"
  # Redis
  REDIS_HOST: "redis.dify.svc.cluster.local"
  REDIS_PORT: "6379"
  REDIS_USE_SSL: "false"
  # 向量存储
  VECTOR_STORE: "weaviate"
  WEAVIATE_ENDPOINT: "http://weaviate.dify.svc.cluster.local:8080"
  WEAVIATE_API_KEY: ""
  # Celery Broker
  CELERY_BROKER_URL: "redis://redis.dify.svc.cluster.local:6379/1"
  # 存储（本地文件系统）
  STORAGE_TYPE: "local"
  # 模型提供商（至少配置一个）
  OPENAI_API_BASE: "https://api.openai.com/v1"
```

```bash
kubectl apply -f dify-config.yaml
# expected: secret/dify-secrets created
#          configmap/dify-config created

kubectl get configmap dify-config -n dify
# expected: NAME: dify-config, DATA: 14
```

> **注意**：替换 `sk-YOUR_OPENAI_API_KEY` 为实际 API Key。支持多种模型提供商：`OPENAI_API_KEY`、`AZURE_OPENAI_API_KEY`、`HUNYUAN_SECRET_ID`/`HUNYUAN_SECRET_KEY`（腾讯混元）等。

### 步骤 7：初始化 Dify 数据库

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: dify-init-db
  namespace: dify
spec:
  ttlSecondsAfterFinished: 60
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: init-db
        image: langgenius/dify-api:1.0.0
        command: ["flask", "db", "upgrade"]
        envFrom:
        - configMapRef:
            name: dify-config
        - secretRef:
            name: dify-secrets
        env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: dify-secrets
              key: db-password
```

```bash
kubectl apply -f - <<'EOF'
apiVersion: batch/v1
kind: Job
metadata:
  name: dify-init-db
  namespace: dify
spec:
  ttlSecondsAfterFinished: 60
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: init-db
        image: langgenius/dify-api:1.0.0
        command: ["flask", "db", "upgrade"]
        envFrom:
        - configMapRef:
            name: dify-config
        - secretRef:
            name: dify-secrets
        env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: dify-secrets
              key: db-password
EOF
# expected: job.batch/dify-init-db created

kubectl wait --for=condition=complete job/dify-init-db -n dify --timeout=120s
# expected: job.batch/dify-init-db condition met
```

验证初始化：

```bash
kubectl logs job/dify-init-db -n dify | tail -5
# expected: 无 ERROR 级别日志，最后一行显示数据库迁移完成
```

### 步骤 8：部署 Dify API + Worker + Web

#### 选择依据

Dify 由三个组件组成：API 服务（处理 HTTP 请求）、Worker（处理 Celery 异步任务）、Web（React 前端）。API 和 Worker 共享相同镜像和大部分配置，区别在于启动命令（`gunicorn` vs `celery worker`）。Web 通过 `NEXT_PUBLIC_API_PREFIX` 指向 API Service。

`dify-api.yaml`：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dify-api
  namespace: dify
spec:
  replicas: 2
  selector:
    matchLabels:
      app: dify-api
  template:
    metadata:
      labels:
        app: dify-api
    spec:
      containers:
      - name: api
        image: langgenius/dify-api:1.0.0
        command: ["gunicorn", "app:app", "-b", "0.0.0.0:5001", "-w", "4", "-k", "gevent", "--timeout", "360"]
        ports:
        - containerPort: 5001
        envFrom:
        - configMapRef:
            name: dify-config
        - secretRef:
            name: dify-secrets
        env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: dify-secrets
              key: db-password
        - name: SECRET_KEY
          valueFrom:
            secretKeyRef:
              name: dify-secrets
              key: secret-key
        - name: OPENAI_API_KEY
          valueFrom:
            secretKeyRef:
              name: dify-secrets
              key: openai-api-key
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
          limits:
            cpu: 2
            memory: 4Gi
        livenessProbe:
          httpGet:
            path: /health
            port: 5001
          initialDelaySeconds: 60
        readinessProbe:
          httpGet:
            path: /health
            port: 5001
          initialDelaySeconds: 30
---
apiVersion: v1
kind: Service
metadata:
  name: dify-api
  namespace: dify
spec:
  selector:
    app: dify-api
  ports:
  - port: 5001
    targetPort: 5001
```

`dify-worker.yaml`：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dify-worker
  namespace: dify
spec:
  replicas: 1
  selector:
    matchLabels:
      app: dify-worker
  template:
    metadata:
      labels:
        app: dify-worker
    spec:
      containers:
      - name: worker
        image: langgenius/dify-api:1.0.0
        command: ["celery", "-A", "app.celery", "worker", "-P", "gevent", "-c", "1", "--loglevel", "INFO", "-Q", "dataset,generation,mail,ops_trace"]
        envFrom:
        - configMapRef:
            name: dify-config
        - secretRef:
            name: dify-secrets
        env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: dify-secrets
              key: db-password
        - name: SECRET_KEY
          valueFrom:
            secretKeyRef:
              name: dify-secrets
              key: secret-key
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
          limits:
            cpu: 2
            memory: 4Gi
```

`dify-web.yaml`：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dify-web
  namespace: dify
spec:
  replicas: 1
  selector:
    matchLabels:
      app: dify-web
  template:
    metadata:
      labels:
        app: dify-web
    spec:
      containers:
      - name: web
        image: langgenius/dify-web:1.0.0
        ports:
        - containerPort: 3000
        env:
        - name: NEXT_PUBLIC_API_PREFIX
          value: "http://dify-api.dify.svc.cluster.local:5001"
        - name: NEXT_PUBLIC_PUBLIC_API_PREFIX
          value: "http://dify-api.dify.svc.cluster.local:5001"
        resources:
          requests:
            cpu: 200m
            memory: 256Mi
          limits:
            cpu: 1
            memory: 512Mi
---
apiVersion: v1
kind: Service
metadata:
  name: dify-web
  namespace: dify
spec:
  selector:
    app: dify-web
  ports:
  - port: 3000
    targetPort: 3000
```

部署所有 Dify 组件：

```bash
kubectl apply -f dify-api.yaml
# expected: deployment.apps/dify-api created, service/dify-api created

kubectl apply -f dify-worker.yaml
# expected: deployment.apps/dify-worker created

kubectl apply -f dify-web.yaml
# expected: deployment.apps/dify-web created, service/dify-web created
```

等待所有组件就绪：

```bash
kubectl get pods -n dify -l 'app in (dify-api,dify-worker,dify-web)'
# expected: 4 个 Pod（2 API + 1 Worker + 1 Web），全部 Running
```

**预期输出**：

```text
NAME                            READY   STATUS    RESTARTS   AGE
dify-api-xxxxxxxxxx-xxxxx       1/1     Running   0          1m
dify-api-xxxxxxxxxx-yyyyy       1/1     Running   0          1m
dify-worker-xxxxxxxxxx-zzzzz    1/1     Running   0          1m
dify-web-xxxxxxxxxx-wwwww       1/1     Running   0          1m
```

### 步骤 9：暴露服务并访问 Dify UI

#### 选择依据

测试环境使用 LoadBalancer 直接暴露 Web 服务（将分配公网 IP）。生产环境建议使用 Ingress + Nginx Ingress Controller，配置 HTTPS 和自定义域名。

```bash
# 方式 1：LoadBalancer（快速测试）
kubectl expose deploy dify-web -n dify \
    --type=LoadBalancer --port=80 --target-port=3000 --name=dify-web-lb
# expected: service/dify-web-lb exposed

# 查询外部 IP
kubectl get svc dify-web-lb -n dify
# expected: EXTERNAL-IP 分配后通过 http://<EXTERNAL-IP> 访问 Dify UI
```

**预期输出**：

```text
NAME           TYPE           CLUSTER-IP       EXTERNAL-IP      PORT(S)        AGE
dify-web-lb    LoadBalancer   10.2.xxx.xxx     43.xxx.xxx.xxx   80:xxxx/TCP    1m
```

配置 Ingress（HTTPS 生产推荐）：

`dify-ingress.yaml`：

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: dify-ingress
  namespace: dify
  annotations:
    kubernetes.io/ingress.class: "nginx"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  tls:
  - hosts:
    - dify.example.com
    secretName: dify-tls
  rules:
  - host: dify.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: dify-web
            port:
              number: 3000
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: dify-api
            port:
              number: 5001
```

```bash
kubectl apply -f dify-ingress.yaml
# expected: ingress.networking.k8s.io/dify-ingress created
```

## 验证

### 控制面（tccli）

```bash
tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterStatus: "Running"
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
# 验证所有 Dify 组件状态
kubectl get pods,svc -n dify
# expected: 所有 Pod Running，Service 均有 ClusterIP

# 验证依赖服务健康
kubectl exec -n dify deploy/dify-api -- curl -s http://postgres:5432 2>&1 | head -1
# expected: PostgreSQL 连接正常（含版本信息或空响应）

kubectl exec -n dify deploy/redis -- redis-cli ping
# expected: PONG

kubectl exec -n dify statefulset/weaviate -- wget -qO- http://localhost:8080/v1/.well-known/ready
# expected: 返回 Weaviate ready 信息

# 验证 Dify API 健康端点
kubectl exec -n dify deploy/dify-api -- curl -s http://localhost:5001/health
# expected: {"status": "ok"}

# 验证 Web 外部访问（在可访问公网的机器上）
curl -s -o /dev/null -w "%{http_code}" http://<EXTERNAL-IP>
# expected: 200
```

```text
NAME  STATUS  AGE
...
```

| 验证维度 | 命令 | 预期 |
|---------|------|------|
| 集群状态 | `DescribeClusters` | `Running` |
| 基础设施组件（PG/Redis/Weaviate） | `kubectl get pods -n dify` | 3 个 Pod Running |
| Dify 核心组件（API/Worker/Web） | `kubectl get pods -n dify -l app` | 4 个 Pod Running |
| API 健康检查 | `curl /health` | `{"status": "ok"}` |
| DB 连接 | `kubectl logs deploy/dify-api` | 无数据库连接错误 |
| Web 外部访问 | 浏览器访问 LoadBalancer IP | Dify 初始化设置页面 |

## 清理

> **计费警告**：LoadBalancer Service 会分配公网 CLB，即使无流量也产生 CLB 实例费和数据流量费。PVC 持久卷存储持续计费。
> **副作用警告**：删除 Namespace 会级联删除其下所有资源（Deployment、StatefulSet、Service、PVC、Secret、ConfigMap），不可恢复。请确认数据已备份。

### 数据面（需 VPN/IOA）

```bash
# 1. 删除 LoadBalancer（先释放 CLB）
kubectl delete svc dify-web-lb -n dify
# expected: service "dify-web-lb" deleted

# 2. 删除 Ingress（若有）
kubectl delete ingress dify-ingress -n dify
# expected: ingress.networking.k8s.io "dify-ingress" deleted

# 3. 清理前状态检查 — 列出所有资源确认
kubectl get all,configmap,secret,pvc,ingress -n dify
# 确认所有资源为待删除对象，记录重要数据

# 4. 删除 Namespace（级联删除全部资源）
kubectl delete namespace dify
# expected: namespace "dify" deleted
# 这将删除：所有 Deployment、StatefulSet、Pod、Service、PVC、Secret、ConfigMap
```

```text
NAME  STATUS  AGE
...
```

### 控制面（tccli）

如创建了专用集群且不再需要：

```bash
# 清理前检查
tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
# 确认是待删除的 Dify 测试集群

# 删除集群（级联删除所有节点）
tccli tke DeleteCluster --region <Region> \
    --ClusterId CLUSTER_ID \
    --InstanceDeleteMode terminate
# 此操作将删除集群所有关联 CVM 实例及磁盘
```

```json
{
  "TotalCount": 0,
  "Clusters": [],
  "ClusterId": "<ClusterId>",
  "ClusterName": "<ClusterName>",
  "ClusterDescription": "<ClusterDescription>",
  "ClusterVersion": "<ClusterVersion>",
  "ClusterOs": "<ClusterOs>",
  "ClusterType": "<ClusterType>"
}
```

验证清理：

```bash
kubectl get ns dify 2>/dev/null
# expected: Error from server (NotFound): namespaces "dify" not found

tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]' 2>&1
# expected: ResourceNotFound（如已删集群）
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

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `CreateCluster` 返回 `InvalidParameter.ClusterCIDRSettings` | 对比 VPC CIDR：`tccli vpc DescribeVpcs --region <Region> --VpcIds '["VPC_ID"]'` | Pod/Service CIDR 与 VPC CIDR 或已有集群 CIDR 冲突 | 修改 `ClusterCIDR` 或 `ServiceCIDR` 为不重叠网段 |
| `CreateCluster` 返回 `LimitExceeded` | `tccli tke DescribeClusters --region <Region>` 统计集群数量 | 集群数量达配额上限（此为环境限制，非命令错误） | 删除不再使用的集群后重试，或提交配额提升申请 |
| `kubectl apply` 返回 `connection refused` | `kubectl cluster-info` 检查连通性 | kubeconfig 未配置或集群 API Server 不可达 | `tccli tke DescribeClusterKubeconfig ...` 重新获取 kubeconfig；确认在内网/VPN 环境 |

### 部署成功但 Dify 不可用

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `dify-init-db` Job 失败 | `kubectl logs job/dify-init-db -n dify` 查看日志 | PostgreSQL 未就绪或密码错误 | 确认 `postgres` Service 可达：`kubectl exec deploy/dify-api -n dify -- nc -zv postgres 5432`；检查 `DB_PASSWORD` 是否匹配 `dify-postgres-secret` |
| `dify-api` Pod CrashLoopBackOff | `kubectl logs deploy/dify-api -n dify --tail=50` | 数据库连接失败或配置缺失 | 检查 `DB_HOST`、`REDIS_HOST`、`WEAVIATE_ENDPOINT` 配置；确认所有依赖 Service 已创建且 ClusterIP 可达 |
| Dify Web 页面空白或 API 错误 | 浏览器 DevTools Network 选项卡查看 API 请求 | `NEXT_PUBLIC_API_PREFIX` 配置错误或 Web 无法访问 API | 确认 `NEXT_PUBLIC_API_PREFIX` 指向 `http://dify-api.dify.svc.cluster.local:5001`（集群内 Service 名） |
| LLM 调用返回 `401 Unauthorized` | `kubectl exec deploy/dify-api -n dify -- env \| grep API_KEY` 确认密钥注入 | OpenAI API Key 未配置或无效 | 更新 `dify-secrets` Secret 中的 `openai-api-key`，重启 API Deployment：`kubectl rollout restart deploy/dify-api -n dify` |
| Weaviate 连接超时 | `kubectl logs deploy/weaviate -n dify --tail=20` | Weaviate 未完全启动或内存不足 | 等待 Weaviate Ready（http health check 返回 200）；增加 `resources.limits.memory`；确认 PVC 存储已挂载 |
| Worker 任务不执行 | `kubectl logs deploy/dify-worker -n dify --tail=30` | Celery 无法连接 Redis Broker | 确认 `CELERY_BROKER_URL` 中 Redis 地址正确；`kubectl exec deploy/redis -n dify -- redis-cli ping` 验证 Redis 可达 |

## 下一步

- [Dify 官方文档](https://docs.dify.ai/) — Dify 应用开发指南
- [使用 TKE 部署 AI 大模型](https://cloud.tencent.com/document/product/457/103983) — LLM 推理部署方案
- [在 TKE 上使用 RDMA 加速 AI 推理和训练](https://cloud.tencent.com/document/product/457/116720) — RDMA 网络加速
- [腾讯云 CDB PostgreSQL](https://cloud.tencent.com/document/product/409) — 生产环境推荐使用托管数据库
- [腾讯云 Redis](https://cloud.tencent.com/document/product/239) — 生产环境推荐使用托管缓存

## 控制台替代

[TKE 控制台 → 集群 → 工作负载](https://console.cloud.tencent.com/tke2/cluster)：通过 YAML 逐个部署 PostgreSQL、Redis、Weaviate、Dify API/Worker/Web 工作负载，使用 Service/Ingress 暴露 Web 前端。
