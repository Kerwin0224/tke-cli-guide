# 多级服务同步水平伸缩（Workload 触发器）（tccli）

> 对照官方：[多级服务同步水平伸缩（Workload 触发器）](https://cloud.tencent.com/document/product/457/106148) · page_id `106148`

## 概述

通过 KEDA Workload 触发器（`kubernetes-workload`）实现多级服务联动伸缩——一个工作负载（如前端 Nginx）的副本数变化自动驱动另一个工作负载（如后端 API）按比例扩缩容。无需外部事件源或 Prometheus，直接以集群内另一个工作负载的副本数作为伸缩指标。

常见场景：微服务架构中前端扩容后，后端需按比例扩容以承接增加的请求量；或一组批处理 Worker 的副本数根据主调度器的 Pod 数量自动调整。

> **kubectl 数据面不可达**：CAM拒绝公网端点(strategyId:240463971)，内网需VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

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

# 3. 检查 kubectl 版本
kubectl version --client
# expected: Client Version >= 1.16.0

# 4. 检查 CAM 权限 — 需要以下 Action 名
#    tke:DescribeClusters, tke:DescribeClusterKubeconfig
# 验证：执行 DescribeClusters 确认权限
tccli tke DescribeClusters --region <Region>
# expected: exit 0，返回集群列表
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
# 5. 确认目标集群存在
tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
# expected: TotalCount >= 1，ClusterStatus: "Running"
```

**预期输出**：

```json
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "cls-example",
            "ClusterStatus": "Running",
            "ClusterVersion": "1.30.0",
            "ClusterNodeNum": 2
        }
    ],
    "RequestId": "..."
}
```

```bash
# 6. 确认 KEDA 已安装
tccli tke DescribeAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName keda
# expected: Status: "Succeeded"
```

**预期输出**：

```json
{
    "AddonName": "keda",
    "AddonVersion": "2.14.0",
    "Status": "Succeeded"
}
```

```bash
# 7. 获取 kubeconfig
tccli tke DescribeClusterKubeconfig --region <Region> \
    --ClusterId CLUSTER_ID \
    | jq -r '.Kubeconfig' > ~/.kube/config-CLUSTER_ID
export KUBECONFIG=~/.kube/config-CLUSTER_ID
kubectl get ns
# expected: 返回 default 等系统命名空间
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli / kubectl 命令 | 幂等 |
|-----------|---------------------|:--:|
| 查看集群状态 | `tccli tke DescribeClusters --ClusterIds '["CLUSTER_ID"]'` | 是 |
| 查看 KEDA 组件 | `tccli tke DescribeAddon --AddonName keda` | 是 |
| 创建 Workload 联动 ScaledObject | `kubectl apply -f cascade-scaledobject.yaml`（需 VPN/IOA） | 是 |
| 查看级联伸缩状态 | `kubectl describe scaledobject SCALED_OBJECT_NAME -n NAMESPACE`（需 VPN/IOA） | 是 |
| 手动触发前端伸缩验证联动 | `kubectl scale deployment DEPLOYMENT_NAME -n NAMESPACE --replicas N`（需 VPN/IOA） | 否 |

## 关键字段说明

KEDA Workload 触发器（`kubernetes-workload`）的 `triggers[].metadata` 字段：

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `podSelector` | String | 是 | K8s Label Selector，用于匹配目标工作负载的 Pod，如 `app=frontend`。支持 `=` 和 `!=` 操作符 | Selector 不匹配 → Pod 数为 0，目标工作负载始终缩容至最小 |
| `namespace` | String | 否 | 被观测工作负载所在的命名空间。默认为 ScaledObject 所在命名空间 | 指定错误的命名空间 → 找不到目标 Pod |
| `value` | String | 否 | 伸缩比例的分母，默认 `"1"`。后端副本数 = 前端副本数 / value。**值类型为字符串** | 设为 0 → 除以零错误 |
| `name` | String | 否 | 自定义指标名称，用于识别 HPA 中的外部指标。不指定则自动生成 | — |

**伸缩比例公式**：

```
目标副本数 = 被观测工作负载 Pod 数 / value
```

例如：前端 Pod 数为 6，`value: "2"`，则后端副本数目标为 3。KEDA 会确保结果在 `minReplicaCount` 和 `maxReplicaCount` 之间。

## 操作步骤

### 步骤 1：部署前端和后端工作负载

#### 选择依据

- **前端（frontend）**：作为伸缩触发源，使用 nginx 镜像模拟。初始副本数 2。
- **后端（backend）**：作为伸缩目标，受 KEDA Workload 触发器控制。初始副本数 2。
- **命名空间**：前端和后端可在同一命名空间或不同命名空间。跨命名空间时需在 `metadata.namespace` 中指定被观测端的命名空间。

#### 创建前端 Deployment

```bash
cat > frontend-deploy.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: NAMESPACE
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "200m"
            memory: "256Mi"
EOF

kubectl apply -f frontend-deploy.yaml
# expected: deployment.apps/frontend created
```

**预期输出**：

```text
deployment.apps/frontend created
```

#### 创建后端 Deployment

```bash
cat > backend-deploy.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: NAMESPACE
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: api
        image: nginx:1.25
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "200m"
            memory: "256Mi"
EOF

kubectl apply -f backend-deploy.yaml
# expected: deployment.apps/backend created
```

**预期输出**：

```text
deployment.apps/backend created
```

```bash
# 确认两个 Deployment 的 Pod 均已运行
kubectl get pods -n NAMESPACE
# expected: frontend Pod x2 Running, backend Pod x2 Running
```

**预期输出**：

```text
NAME                        READY   STATUS    RESTARTS   AGE
frontend-xxxxxxxxxx-aaaa    1/1     Running   0          30s
frontend-xxxxxxxxxx-bbbb    1/1     Running   0          30s
backend-xxxxxxxxxx-cccc     1/1     Running   0          20s
backend-xxxxxxxxxx-dddd     1/1     Running   0          20s
```

### 步骤 2：创建 Workload 触发 ScaledObject（最小配置）

#### 选择依据

- **触发器类型**：`kubernetes-workload` 直接读取 `podSelector` 匹配的 Pod 数量，无需外部事件源或凭据。
- **默认比例 (1:1)**：`value` 默认 `"1"`，即后端副本数 = 前端 Pod 数。最小配置不指定 `value` 即使用默认 1:1。
- **minReplicaCount**：设为 1 保证至少 1 个后端 Pod 始终运行。如需节省成本可设为 0（但会导致首次请求延迟）。
- **maxReplicaCount**：根据集群容量设置上限，防止前端异常扩容导致后端耗尽资源。

```bash
cat > cascade-scaledobject.yaml <<'EOF'
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: backend-follow-frontend
  namespace: NAMESPACE
spec:
  scaleTargetRef:
    name: backend
  minReplicaCount: 1
  maxReplicaCount: 20
  triggers:
  - type: kubernetes-workload
    metadata:
      podSelector: "app=frontend"
      namespace: NAMESPACE
EOF

kubectl apply -f cascade-scaledobject.yaml
# expected: scaledobject.keda.sh/backend-follow-frontend created
```

**预期输出**：

```text
scaledobject.keda.sh/backend-follow-frontend created
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `NAMESPACE` | K8s 命名空间 | 已存在的命名空间 | `kubectl get ns` |

### 步骤 3：增强配置——设置伸缩比例

#### 选择依据

- **为什么要设置比例**：实际微服务架构中，前后端通常不是 1:1 关系。例如一个前端请求会导致后端多次调用，后端需要更多副本。此处设置为 1:2（前端 1 个 Pod 对应后端 2 个 Pod），即 `value: "0.5"`（后端副本数 = 前端 Pod 数 / 0.5 = 前端 Pod 数 x 2）。
- **value 计算**：`value` 是分母，后端目标副本数 = 前端 Pod 数 / value。想要后端是前端的 2 倍 → value = 0.5。
- **多触发器组合**：同时加入 CPU 触发器作为兜底——即使前端副本数少，后端 CPU 使用率过高时也可独立扩容。

```bash
cat > cascade-enhanced-scaledobject.yaml <<'EOF'
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: backend-follow-frontend-ratio
  namespace: NAMESPACE
spec:
  scaleTargetRef:
    name: backend
  minReplicaCount: 1
  maxReplicaCount: 30
  triggers:
  - type: kubernetes-workload
    metadata:
      podSelector: "app=frontend"
      namespace: NAMESPACE
      value: "0.5"
  - type: cpu
    metricType: Utilization
    metadata:
      value: "70"
EOF

kubectl apply -f cascade-enhanced-scaledobject.yaml
# expected: scaledobject.keda.sh/backend-follow-frontend-ratio created
```

**预期输出**：

```text
scaledobject.keda.sh/backend-follow-frontend-ratio created
```

### 步骤 4：验证级联伸缩

#### 触发前端扩容

```bash
# 将前端从 2 副本扩容至 6 副本
kubectl scale deployment frontend -n NAMESPACE --replicas=6
# expected: deployment.apps/frontend scaled
```

**预期输出**：

```text
deployment.apps/frontend scaled
```

```bash
# 等待 KEDA 响应（约 15-60 秒）
# 检查后端副本数是否跟随
kubectl get pods -n NAMESPACE -l app=backend
# expected: 后端 Pod 数量随前端变化。若 value=0.5，6/0.5=12，capped by maxReplicaCount
```

**预期输出**（value=0.5，前端 6 Pod）：

```text
NAME                        READY   STATUS    RESTARTS   AGE
backend-xxxxxxxxxx-cccc     1/1     Running   0          10m
backend-xxxxxxxxxx-dddd     1/1     Running   0          10m
backend-xxxxxxxxxx-eeee     1/1     Running   0          30s
backend-xxxxxxxxxx-ffff     1/1     Running   0          30s
backend-xxxxxxxxxx-gggg     1/1     Running   0          30s
backend-xxxxxxxxxx-hhhh     1/1     Running   0          30s
backend-xxxxxxxxxx-iiii     1/1     Running   0          30s
backend-xxxxxxxxxx-jjjj     1/1     Running   0          30s
backend-xxxxxxxxxx-kkkk     1/1     Running   0          30s
backend-xxxxxxxxxx-llll     1/1     Running   0          30s
backend-xxxxxxxxxx-mmmm     1/1     Running   0          30s
backend-xxxxxxxxxx-nnnn     1/1     Running   0          30s
```

#### 触发前端缩容

```bash
# 将前端缩至 1 副本
kubectl scale deployment frontend -n NAMESPACE --replicas=1
# expected: deployment.apps/frontend scaled
```

```bash
# 等待后，后端副本数应降至 2（1 / 0.5 = 2）
kubectl get pods -n NAMESPACE -l app=backend
# expected: 后端 Pod 数降至约 2
```

**预期输出**：

```text
NAME                        READY   STATUS    RESTARTS   AGE
backend-xxxxxxxxxx-cccc     1/1     Running   0          15m
backend-xxxxxxxxxx-dddd     1/1     Running   0          15m
```

## 验证

### 控制面（tccli）

```bash
# 维度 1：确认集群正常
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterStatus: "Running"
```

**预期输出**：

```json
{
    "ClusterId": "cls-example",
    "ClusterStatus": "Running"
}
```

```bash
# 维度 2：确认 KEDA 组件正常
tccli tke DescribeAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName keda
# expected: Status: "Succeeded"
```

```json
{
  "Addons": "<Addons>",
  "AddonName": "<AddonName>",
  "AddonVersion": "<AddonVersion>",
  "RawValues": "<RawValues>",
  "Phase": "<Phase>",
  "Reason": "<Reason>"
}
```

### 数据面

```bash
# 维度 3：确认 ScaledObject 就绪
kubectl get scaledobject -n NAMESPACE
# expected: backend-follow-frontend 和 backend-follow-frontend-ratio READY: True
```

**预期输出**：

```text
NAME                              SCALETARGETKIND    SCALETARGETNAME   MIN   MAX   TRIGGERS                   READY   ACTIVE   AGE
backend-follow-frontend           apps/v1.Deployment backend            1    20   kubernetes-workload        True    True     5m
backend-follow-frontend-ratio     apps/v1.Deployment backend            1    30   kubernetes-workload,cpu   True    True     2m
```

```bash
# 维度 4：确认 HPA 已创建
kubectl get hpa -n NAMESPACE
# expected: 每个 ScaledObject 对应一个 keda-hpa-* HPA
```

**预期输出**：

```text
NAME                                          REFERENCE          TARGETS            MINPODS   MAXPODS   REPLICAS   AGE
keda-hpa-backend-follow-frontend              Deployment/backend 2/1 (avg)          1         20        2          5m
keda-hpa-backend-follow-frontend-ratio        Deployment/backend 4/2 (avg)          1         30        4          2m
```

```bash
# 维度 5：查看 Workload 触发器详情
kubectl describe scaledobject backend-follow-frontend-ratio -n NAMESPACE | grep -A 15 "Triggers:"
# expected: 显示 kubernetes-workload 触发器 podSelector 和 value
```

```text
NAME  STATUS  AGE
...
```

```bash
# 维度 6：对比前后端副本数验证比例关系
echo "前端 Pod 数:"
kubectl get pods -n NAMESPACE -l app=frontend --no-headers | wc -l
echo "后端 Pod 数:"
kubectl get pods -n NAMESPACE -l app=backend --no-headers | wc -l
# expected: 后端副本数 = ceil(前端Pod数 / value)
```

**预期输出**：

```text
前端 Pod 数:
6
后端 Pod 数:
12
```

| 维度 | 命令 | 预期 |
|------|------|------|
| 集群状态 | `DescribeClusters --ClusterIds '["CLUSTER_ID"]'` | `ClusterStatus: "Running"` |
| KEDA 状态 | `DescribeAddon --AddonName keda` | `Status: "Succeeded"` |
| ScaledObject | `kubectl get scaledobject -n NAMESPACE` | READY: True |
| HPA | `kubectl get hpa -n NAMESPACE` | keda-hpa-* 存在且 TARGETS 有效 |
| 比例关系 | 前后端 Pod 数对比 | 后端 = ceil(前端 / value) |
| KEDA 日志 | `kubectl logs -n keda -l app=keda-operator --tail=20` | 无 ERROR |

## 清理

> **警告**：删除 ScaledObject 后，KEDA 自动删除关联的 HPA，后端副本数将**立即恢复为 Deployment 原始 `replicas` 值**。如果前端此时处于高副本状态，后端可能因副本不足导致服务降级。建议先缩容前端至正常水平，再执行清理。

### 数据面

数据面资源清理在控制面之前。

```bash
# 清理前确认所有 ScaledObject
kubectl get scaledobject -n NAMESPACE
# 确认目标 ScaledObject
```

```text
NAME  STATUS  AGE
...
```

```bash
# 删除 ScaledObject（清理关联 HPA）
kubectl delete scaledobject backend-follow-frontend -n NAMESPACE
kubectl delete scaledobject backend-follow-frontend-ratio -n NAMESPACE
# expected: scaledobject.keda.sh "backend-follow-frontend" deleted
```

**预期输出**：

```text
scaledobject.keda.sh "backend-follow-frontend" deleted
scaledobject.keda.sh "backend-follow-frontend-ratio" deleted
```

```bash
# 确认 HPA 已自动清理
kubectl get hpa -n NAMESPACE
# expected: No resources found（或不再含 keda-hpa 前缀的条目）
```

```text
NAME  STATUS  AGE
...
```

```bash
# 删除测试 Deployment
kubectl delete deployment frontend -n NAMESPACE
kubectl delete deployment backend -n NAMESPACE
# expected: deployment.apps "frontend" deleted
```

**预期输出**：

```text
deployment.apps "frontend" deleted
deployment.apps "backend" deleted
```

```bash
# 确认所有 Pod 已终止
kubectl get pods -n NAMESPACE
# expected: No resources found
```

```text
NAME  STATUS  AGE
...
```

### 控制面（tccli）

Workload 联动伸缩的资源全部在集群内部，无 tccli 控制面资源需要清理。

```bash
# 确认 KEDA 组件正常（如不再需要可卸载，参见部署页）
tccli tke DescribeAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName keda
# expected: Status: "Succeeded"
```

```json
{
  "Addons": "<Addons>",
  "AddonName": "<AddonName>",
  "AddonVersion": "<AddonVersion>",
  "RawValues": "<RawValues>",
  "Phase": "<Phase>",
  "Reason": "<Reason>"
}
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl apply -f` 返回 `no matches for kind "ScaledObject"` | `kubectl get crd \| grep keda.sh` | KEDA 未安装或 CRD 未注册 | 安装 KEDA（参见 [在 TKE 上部署 KEDA](../在%20TKE%20上部署%20KEDA/tccli%20操作.md)） |
| `kubectl apply -f` 返回 `field is immutable` | `kubectl get scaledobject SCALED_OBJECT_NAME -n NAMESPACE -o yaml` 对比变更 | 修改了不可变字段（如 `spec.scaleTargetRef.name`） | `kubectl delete scaledobject` 后重新 apply |
| `kubectl scale` 返回 `deployment ... not found` | `kubectl get deployments -n NAMESPACE` | Deployment 名称或命名空间错误 | 确认 Deployment 名称和 `-n` 参数正确 |

### 创建成功但级联未生效

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| ScaledObject READY=True 但后端副本数不跟随 | `kubectl describe scaledobject SCALED_OBJECT_NAME -n NAMESPACE` 查看 Events | `podSelector` 未匹配到任何前端 Pod | 确认 `podSelector: "app=frontend"` 与被观测 Pod 的 labels 完全一致；`kubectl get pods -n NAMESPACE --show-labels` 验证标签 |
| 后端副本数总是等于前端副本数（未按比例） | `kubectl get scaledobject SCALED_OBJECT_NAME -n NAMESPACE -o yaml \| grep value` | `value` 字段缺失或为 `"1"`（默认） | 设置 `value` 为期望分母值（字符串类型），如 `"0.5"` 表示后端=前端x2 |
| 后端副本数远超预期 | `kubectl get hpa keda-hpa-SCALED_OBJECT_NAME -n NAMESPACE -o yaml` 查看当前指标 | 多触发器（如 workload + cpu）取最大值，CPU 触发器推动了额外扩容 | 确认比例值是否准确；如 CPU 触发器不必要，从 triggers 中移除 |
| 后端副本数卡在 minReplicaCount 不变 | `kubectl describe scaledobject SCALED_OBJECT_NAME -n NAMESPACE` | `podSelector` 指向了不存在的 Deployment 标签，Pod 数为 0；0 / value = 0 < minReplicaCount | 验证 `kubectl get pods -n NAMESPACE -l app=frontend` 是否有输出 |
| 跨命名空间级联不生效 | `kubectl describe scaledobject SCALED_OBJECT_NAME -n NAMESPACE` 查看 Events | 未在 metadata 中指定 `namespace` 字段，或 KEDA Operator RBAC 无权读取目标命名空间 | 在 triggers metadata 中添加 `namespace: TARGET_NAMESPACE`；确认 KEDA Operator 的 ClusterRole 包含跨命名空间的 Pod 读取权限 |

## 下一步

- [认识 KEDA](../认识%20KEDA/tccli%20操作.md) — KEDA 概念与 Scaler 类型概览
- [定时水平伸缩（Cron 触发器）](../定时水平伸缩%20(Cron%20触发器)/tccli%20操作.md) — 基于时间表的伸缩配置
- [基于 Prometheus 自定义指标的弹性伸缩](../基于%20Prometheus%20自定义指标的弹性伸缩/tccli%20操作.md) — 使用 Prometheus 指标驱动伸缩
- [在 TKE 上部署 KEDA](../在%20TKE%20上部署%20KEDA/tccli%20操作.md) — KEDA 安装与卸载

## 控制台替代

登录 [容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) → 选择目标集群 → **工作负载** → **Deployment** → 选择目标部署（如 backend）→ **弹性伸缩** → **添加触发策略** → 选择指标类型为 **Workload** → 指定观测的目标工作负载和命名空间 → 设置比例。
