# 基于云原生 API 网关监控指标的水平伸缩（tccli）

> 对照官方：[基于云原生 API 网关监控指标的水平伸缩](https://cloud.tencent.com/document/product/457/106562) · page_id `106562`

## 概述

通过 KEDA 通用的 HTTP Scaler 或 Prometheus Scaler，基于云原生 API 网关（TSE 云原生网关 / 自建 Nginx/Kong/APISIX）的监控指标（如请求 QPS、延迟 p99、错误率）触发后端服务的水平伸缩。无需改造应用代码，网关层指标即可驱动弹性。

> **kubectl 数据面不可达**：CAM拒绝公网端点(strategyId:240463971)，内网需VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

### 方案选择

基于 API 网关指标的弹性伸缩有多种实现方式：

| 方式 | 适用场景 | 指标来源 | 配置复杂度 |
|------|---------|---------|:--:|
| **KEDA Prometheus Scaler + 网关 Prometheus 指标** | 网关已对接 Prometheus，指标丰富 | 网关 Prometheus endpoint | 中 |
| **KEDA HTTP Addon (pendingRequests)** | 基于网关待处理请求数，无需 Prometheus | KEDA HTTP Addon 拦截器 | 低 |
| **KEDA CPU/Memory Scaler** | 通用方案，非网关专属 | metrics-server | 低 |

本页以 **Prometheus Scaler + 网关指标** 为例，因该方式覆盖最广泛的网关指标场景（QPS、延迟分布、错误率），且不依赖 KEDA HTTP Addon 额外组件。

## 前置条件

- [环境准备](../../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置，region 为目标地域

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:DescribeClusters
# 验证：执行 DescribeClusters 确认权限
tccli tke DescribeClusters --region <Region>
# expected: exit 0，返回集群列表（可为空）
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
# 4. 确认目标集群存在且状态正常
tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterStatus 为 "Running"

# 5. 检查 KEDA 组件是否已安装
tccli tke DescribeAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName keda
# expected: 返回 AddonStatus（如未安装则需先安装 KEDA）

# 6. 确认 Prometheus 已部署（如使用 Prometheus Scaler 方式）
tccli tke DescribeAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName prometheus
# expected: 确认是否有 Prometheus 可采集网关指标
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
|-----------|-----|------|
| 查看集群列表 | `tccli tke DescribeClusters --region ap-guangzhou` | 是 |
| 安装 Addon 组件 | `tccli tke InstallAddon --region ap-guangzhou` | 否 |
| 查看 Addon 状态 | `tccli tke DescribeAddon --region ap-guangzhou` | 是 |

## 关键字段说明

API 网关弹性伸缩 ScaledObject 中的核心参数：

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `serverAddress` | String | 是 | Prometheus server 地址，需从 KEDA operator 可达 | 地址不可达 → ScaledObject `ACTIVE` 为 `False` |
| `query` | String | 是 | PromQL 查询语句，针对网关指标编写。如 `rate(envoy_http_downstream_rq_total{envoy_http_conn_manager_prefix="ingress_http"}[2m])` | 语法错误 → Scaler 返回错误；标签不匹配 → 查询结果为空 |
| `threshold` | String | 是 | 触发扩容的 QPS 阈值，如 `"100"`（100 请求/秒） | 阈值不合理 → 频繁扩缩或永不触发 |
| `metricName` | String | 是 | KEDA 内部指标名，如 `gateway_qps` | 用于 HPA 指标追踪，可自定义 |
| `minReplicaCount` | Integer | 是 | 最小后端副本数，建议 ≥ 2 | 设为 0 → 冷启动时无法服务请求 |

## 操作步骤

### 步骤 1：验证集群状态

```bash
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]'
# expected: exit 0, ClusterStatus 为 "Running"
```

**预期输出**：

```json
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "cls-example",
            "ClusterName": "CLUSTER_NAME",
            "ClusterStatus": "Running",
            "ClusterVersion": "1.30.0",
            "ClusterNetworkSettings": {
                "VpcId": "vpc-example"
            }
        }
    ],
    "RequestId": "..."
}
```

### 步骤 2：部署后端服务（数据面）

部署一个简单的 HTTP 后端服务作为 API 网关的后端目标。

`backend-service.yaml`：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: DEPLOYMENT_NAME
  namespace: NAMESPACE
  labels:
    app: api-backend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api-backend
  template:
    metadata:
      labels:
        app: api-backend
    spec:
      containers:
        - name: backend
          image: nginx:latest
          ports:
            - containerPort: 80
              name: http
          resources:
            requests:
              cpu: 200m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 256Mi
---
apiVersion: v1
kind: Service
metadata:
  name: api-backend-svc
  namespace: NAMESPACE
  labels:
    app: api-backend
spec:
  selector:
    app: api-backend
  ports:
    - port: 80
      targetPort: 80
      name: http
  type: ClusterIP
```

```bash
kubectl apply -f backend-service.yaml
# expected: deployment.apps/DEPLOYMENT_NAME created, service/api-backend-svc created
```

**预期输出**：

```text
deployment.apps/api-backend created
service/api-backend-svc created
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `DEPLOYMENT_NAME` | 后端部署名称 | 长度 1-63 字符 | 自定义，如 `api-backend` |
| `NAMESPACE` | 命名空间 | 需已存在 | `kubectl get ns` 确认 |

### 步骤 3：部署 API 网关/代理层（数据面）

部署一个轻量级 API 网关代理（以 Nginx 为例），将请求转发到后端服务。

`api-gateway.yaml`：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: gateway-config
  namespace: NAMESPACE
data:
  nginx.conf: |
    events {
        worker_connections 1024;
    }
    http {
        server {
            listen 80;
            location /api/ {
                proxy_pass http://api-backend-svc:80;
                proxy_set_header Host $host;
            }
            location /stub_status {
                stub_status;
            }
        }
    }
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-gateway
  namespace: NAMESPACE
  labels:
    app: api-gateway
spec:
  replicas: 1
  selector:
    matchLabels:
      app: api-gateway
  template:
    metadata:
      labels:
        app: api-gateway
    spec:
      containers:
        - name: gateway
          image: nginx:latest
          ports:
            - containerPort: 80
              name: http
          volumeMounts:
            - name: config
              mountPath: /etc/nginx/nginx.conf
              subPath: nginx.conf
          resources:
            requests:
              cpu: 200m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 256Mi
      volumes:
        - name: config
          configMap:
            name: gateway-config
---
apiVersion: v1
kind: Service
metadata:
  name: api-gateway-svc
  namespace: NAMESPACE
  labels:
    app: api-gateway
spec:
  selector:
    app: api-gateway
  ports:
    - port: 80
      targetPort: 80
      name: http
  type: ClusterIP
```

```bash
kubectl apply -f api-gateway.yaml
# expected: configmap/gateway-config created, deployment.apps/api-gateway created, service/api-gateway-svc created
```

**预期输出**：

```text
configmap/gateway-config created
deployment.apps/api-gateway created
service/api-gateway-svc created
```

### 步骤 4：配置 KEDA ScaledObject 监控网关指标（数据面）

#### 选择依据

使用 Prometheus Scaler 监控网关指标的关键决策：

- **指标选择**：常见网关指标包括请求速率（QPS）、错误率（5xx 比例）、延迟分布（p99）。选择对业务最敏感的指标。本例以请求速率为目标。
- **`query`**：PromQL 表达式需包含网关 Pod 的标签选择器。如 `sum(rate(nginx_http_requests_total{app="api-gateway"}[2m]))` 表示近 2 分钟内所有网关 Pod 的请求速率之和。
- **`threshold`**：QPS 阈值设为 `"100"`。当所有网关 Pod 的总 QPS 超过 100 时，触发后端扩容。注意区分"单 Pod QPS"与"总 QPS"—后者更能反映整体负载。
- **Prometheus 来源**：若使用 Nginx Prometheus Exporter 暴露指标，需确保网关已配置 `prometheus.io/scrape: "true"` 注解。

`scaledobject-gateway.yaml`：

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: gateway-scaledobject
  namespace: NAMESPACE
spec:
  scaleTargetRef:
    name: DEPLOYMENT_NAME
  minReplicaCount: 2
  maxReplicaCount: 20
  cooldownPeriod: 300
  pollingInterval: 15
  triggers:
    - type: prometheus
      metadata:
        serverAddress: http://prometheus.monitoring:9090
        metricName: gateway_requests_per_second
        threshold: "100"
        query: sum(rate(nginx_http_requests_total{app="api-gateway",namespace="NAMESPACE"}[2m]))
```

```bash
kubectl apply -f scaledobject-gateway.yaml
# expected: scaledobject.keda.sh/gateway-scaledobject created
```

**预期输出**：

```text
scaledobject.keda.sh/gateway-scaledobject created
```

| 参数字段 | 说明 | 推荐值 | 调整建议 |
|---------|------|--------|---------|
| `minReplicaCount` | 最小后端副本数 | `2` | 保证至少 2 副本高可用 |
| `maxReplicaCount` | 最大后端副本数 | `20` | 根据集群资源、网关并发连接数上限设定 |
| `cooldownPeriod` | 缩容冷却时间（秒） | `300` | 防止流量瞬时波动导致频繁缩扩容 |
| `pollingInterval` | 指标拉取间隔（秒） | `15` | 流量高峰期间可能需要缩短至 5-10s |
| `query` | PromQL 查询 | 业务自定义 | 可用 `sum(rate(...[2m]))` 聚合所有网关 Pod |
| `threshold` | QPS 阈值 | `"100"` | 配合后端单 Pod QPS 上限设定 |

### 步骤 5：测试负载生成（数据面）

通过负载生成器向网关发送请求，模拟高流量场景以触发后端扩容。

```bash
# 从集群内启动负载生成器
kubectl run -it --rm load-generator \
    --image=busybox --restart=Never -n NAMESPACE \
    -- /bin/sh -c "
  while true; do
    for i in \$(seq 1 50); do
      wget -q -O- http://api-gateway-svc/api/ &
    done
    wait
    sleep 1
  done
"
# expected: 持续向网关发送 HTTP 请求，约 50 QPS
```

**预期输出**：

```text
...（持续发送请求，无错误输出为正常）
```

## 验证

### 控制面（tccli）

```bash
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]'
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
# 1. 确认后端服务运行
kubectl get pods -n NAMESPACE -l app=api-backend
# expected: 至少 2 个 Pod Running
```

**预期输出**：

```text
NAME                           READY   STATUS    RESTARTS   AGE
api-backend-5d8f7b9c6-dk3j9   1/1     Running   0          10m
api-backend-5d8f7b9c6-f7m2k   1/1     Running   0          10m
```

```bash
# 2. 确认网关 Pod 运行
kubectl get pods -n NAMESPACE -l app=api-gateway
# expected: 至少 1 个 Pod Running
```

```text
NAME  STATUS  AGE
...
```

```bash
# 3. 确认 ScaledObject 就绪
kubectl get scaledobject -n NAMESPACE
# expected: READY True
```

**预期输出**：

```text
NAME                     SCALETARGETKIND      SCALETARGETNAME   MIN   MAX   TRIGGERS      AUTHENTICATION   READY   ACTIVE   AGE
gateway-scaledobject     apps/v1.Deployment   api-backend       2     20    prometheus                      True    True     5m
```

```bash
# 4. 确认 HPA 自动生成
kubectl get hpa -n NAMESPACE
# expected: 存在 keda-hpa- 前缀的 HPA
```

```text
NAME  STATUS  AGE
...
```

```bash
# 5. 观察负载下的 Pod 扩容
kubectl get pods -n NAMESPACE -l app=api-backend -w
# expected: 副本数随 QPS 增加
```

```text
NAME  STATUS  AGE
...
```

```bash
# 6. 验证网关请求转发正常
kubectl exec -it -n NAMESPACE \
    $(kubectl get pod -n NAMESPACE -l app=api-gateway -o jsonpath='{.items[0].metadata.name}') \
    -- curl -s http://localhost/api/
# expected: 返回后端 nginx 默认页面或 200 状态码
```

```text
NAME  STATUS  AGE
...
```

| 验证维度 | 命令 | 预期 |
|---------|------|------|
| 后端 Pod Running | `kubectl get pods -n NAMESPACE -l app=api-backend` | 副本数 ≥ `minReplicaCount`，状态 Running |
| 网关 Pod Running | `kubectl get pods -n NAMESPACE -l app=api-gateway` | 至少 1 个 Running |
| ScaledObject 就绪 | `kubectl get scaledobject -n NAMESPACE` | `READY: True` |
| HPA 生成 | `kubectl get hpa -n NAMESPACE` | 存在 `keda-hpa-gateway-scaledobject` |
| 负载触发扩容 | 启动负载生成器后 `kubectl get pods -n NAMESPACE -l app=api-backend -w` | 副本数从 2 增长 |
| 网关转发正常 | 在网关 Pod 内 `curl` 后端接口 | 返回 HTTP 200 |

## 清理

> **警告**：删除 ScaledObject 会**同时删除关联的 HPA**，后端服务将保持当前副本数不再伸缩。网关 ConfigMap 包含路由配置，删除后将无法恢复。如有生产流量通过网关，请先切换流量再清理。

### 数据面清理（先执行）

```bash
# 1. 停止负载生成器
kubectl delete pod load-generator -n NAMESPACE --ignore-not-found
# expected: pod "load-generator" deleted

# 2. 删除 ScaledObject
kubectl delete scaledobject gateway-scaledobject -n NAMESPACE
# expected: scaledobject.keda.sh "gateway-scaledobject" deleted

# 3. 删除网关部署
kubectl delete -f api-gateway.yaml
# expected: configmap "gateway-config" deleted, deployment.apps "api-gateway" deleted, service "api-gateway-svc" deleted

# 4. 删除后端服务
kubectl delete -f backend-service.yaml
# expected: deployment.apps "api-backend" deleted, service "api-backend-svc" deleted
```

### 验证数据面清理

```bash
kubectl get deployment,svc,scaledobject -n NAMESPACE \
    -l 'app in (api-backend,api-gateway)'
# expected: No resources found
```

```text
NAME  STATUS  AGE
...
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl apply` 返回 `error: unable to recognize "scaledobject.yaml"` | `kubectl api-resources \| grep scaledobjects` 检查 CRD 是否存在 | KEDA 未安装，ScaledObject CRD 未注册 | 先在集群中安装 KEDA Addon：`tccli tke InstallAddon --region <Region> --ClusterId CLUSTER_ID --AddonName keda --AddonVersion KEDA_VERSION` |
| ScaledObject 创建后 `kubectl get hpa -n NAMESPACE` 无 HPA | `kubectl logs -n keda deployment/keda-operator` 查看 operator 日志 | KEDA operator 未运行或 ScaledObject 验证失败 | 确认 KEDA Addon 状态：`tccli tke DescribeAddon --region <Region> --ClusterId CLUSTER_ID --AddonName keda`；如状态异常，重装 KEDA |
| `kubectl apply` gateway YAML 返回验证错误 | `kubectl apply --dry-run=client -f api-gateway.yaml` 检查 YAML | ConfigMap 缩进错误或 Deployment 端口不匹配 | 逐段排查 YAML 语法；特别检查 nginx.conf 的缩进和 `subPath` 配置 |

### 操作提交成功但异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| ScaledObject `READY` 为 `False` | `kubectl describe scaledobject gateway-scaledobject -n NAMESPACE` 查看 Events | Prometheus 不可达或 PromQL 语法错误 | 验证 Prometheus 服务地址：`kubectl run -it --rm debug --image=busybox --restart=Never -n NAMESPACE -- wget -q -O- http://prometheus.monitoring:9090/-/healthy`；修正 `query` 中的标签选择器，确保与网关 Pod 标签一致 |
| ScaledObject `ACTIVE` 始终为 `False`（无流量时正常） | `kubectl describe scaledobject gateway-scaledobject -n NAMESPACE` | 网关指标未暴露给 Prometheus 或 QPS 低于阈值 | 确认网关 Pod 有 `prometheus.io/scrape: "true"` 注解；手动访问 Prometheus 查询网关指标：`kubectl port-forward -n monitoring svc/prometheus 9090:9090`，在 Prometheus UI 中执行 `sum(rate(nginx_http_requests_total{app="api-gateway"}[2m]))` 确认有数据 |
| 流量增长但后端未扩容 | `kubectl get hpa -n NAMESPACE` 查看 TARGETS；`kubectl describe hpa keda-hpa-gateway-scaledobject -n NAMESPACE` | HPA 冷启动期未过，或 QPS 阈值单位不匹配（如阈值设为 100 但实际 QPS 变化为 0.1/s 级别） | 检查 `pollingInterval` 和 `cooldownPeriod`，首次扩容需等待至少 1 个 polling 周期；确认 `query` 使用 `sum(rate(...[2m]))` 而非 `sum(...)`（rate 函数计算每秒速率）；检查阈值单位是否与 rate 返回值匹配 |
| 后端扩容到 `maxReplicaCount` 仍无法降低延迟 | `kubectl top pods -n NAMESPACE -l app=api-backend` | 单 Pod 处理能力已达上限，或网关本身成为瓶颈 | 增大 `maxReplicaCount`；优化后端应用性能；检查网关 Pod 资源使用：`kubectl top pods -n NAMESPACE -l app=api-gateway`；考虑对网关层也进行弹性伸缩 |
| 缩容后网关请求出现 502 错误 | `kubectl logs -n NAMESPACE -l app=api-gateway \| grep "upstream" \| tail -20` 检查网关日志 | 缩容过快，剩余后端 Pod 未及时更新路由表或网关 upstream 列表 | 增大 `cooldownPeriod`（如 600s）；在网关配置中添加健康检查和重试机制；确保 Service endpoints 更新及时 |

## 下一步

- [认识 KEDA](https://cloud.tencent.com/document/product/457/106143) -- page_id `106143`
- [在 TKE 上部署 KEDA](https://cloud.tencent.com/document/product/457/106144) -- page_id `106144`
- [基于 Prometheus 自定义指标的弹性伸缩](https://cloud.tencent.com/document/product/457/107998) -- page_id `107998`
- [多级服务同步水平伸缩（Workload 触发器）](https://cloud.tencent.com/document/product/457/106150) -- page_id `106150`
- [TSE 云原生 API 网关](https://cloud.tencent.com/document/product/1364) -- 腾讯云托管 API 网关产品文档

## 控制台替代

[TKE 控制台 → 集群 → 工作负载](https://console.cloud.tencent.com/tke2/cluster)：在 KEDA 控制台中可视化创建和管理 API 网关 ScaledObject；[TSE API 网关控制台](https://console.cloud.tencent.com/tse/cnapigw)：管理云原生 API 网关实例、路由和监控。
