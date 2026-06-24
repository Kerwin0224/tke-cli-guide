# 在 TKE 上部署 KEDA（tccli）

> 对照官方：[在 TKE 上部署 KEDA](https://cloud.tencent.com/document/product/457/106146) · page_id `106146`

## 概述

通过 TKE 组件市场安装 KEDA（Kubernetes Event-Driven Autoscaling），为集群添加事件驱动自动伸缩能力。安装后集群将支持 ScaledObject/ScaledJob CRD，可通过 Cron、Prometheus、Kafka 等 50+ 种事件源驱动 Pod 副本伸缩。

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
#    tke:DescribeClusters, tke:InstallAddon, tke:DescribeAddon
#    tke:DescribeClusterKubeconfig, tke:DeleteAddon
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
# 5. 确认目标集群存在且状态为 Running
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
            "ClusterName": "CLUSTER_NAME",
            "ClusterType": "MANAGED_CLUSTER",
            "ClusterStatus": "Running",
            "ClusterVersion": "1.30.0",
            "ClusterNodeNum": 2
        }
    ],
    "RequestId": "..."
}
```

```bash
# 6. 获取 kubeconfig 以验证集群连通性
tccli tke DescribeClusterKubeconfig --region <Region> \
    --ClusterId CLUSTER_ID \
    | jq -r '.Kubeconfig' > ~/.kube/config-CLUSTER_ID
export KUBECONFIG=~/.kube/config-CLUSTER_ID
kubectl cluster-info
# expected: Kubernetes control plane 运行中
kubectl get nodes
# expected: 至少 1 个节点状态 Ready
```

**预期输出**：

```text
NAME          STATUS   ROLES    AGE   VERSION
node-ex-001   Ready    <none>   1d    v1.30.0
node-ex-002   Ready    <none>   1d    v1.30.0
```

```bash
# 7. 检查 KEDA 是否已安装（避免重复安装）
tccli tke DescribeAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName keda
# expected: 如未安装返回 ResourceNotFound 或错误
```

**预期输出**（未安装时）：

```json
{
    "Error": {
        "Code": "ResourceNotFound",
        "Message": "Addon not found"
    }
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 安装 KEDA 组件 | `InstallAddon --AddonName keda` | 否 |
| 查看组件状态 | `DescribeAddon --AddonName keda` | 是 |
| 卸载组件 | `DeleteAddon --AddonName keda` | 否 |

## 关键字段说明

`InstallAddon` 安装 KEDA 的主要参数：

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `ClusterId` | String | 是 | 目标集群 ID，格式 `CLUSTER_ID` | 集群不存在 → `InvalidParameter.ClusterId` |
| `AddonName` | String | 是 | `keda`，大小写敏感（全小写） | 名称错误 → `InvalidParameter.AddonName` |
| `AddonVersion` | String | 否 | 组件版本号。不指定则安装最新版。可通过 `DescribeAddonValues` 查询可用版本 | 版本不存在 → `InvalidParameter.AddonVersion` |

## 操作步骤

### 步骤 1：安装 KEDA 组件

#### 选择依据

- **安装方式**：TKE 提供了两种方式——`InstallAddon`（tccli 控制面）和 Helm（kubectl 数据面）。推荐 `InstallAddon`，因为它由 TKE 平台管理，版本升级、状态监控可直接通过 tccli 完成，无需额外配置 Helm 仓库。
- **命名空间**：KEDA 默认安装至 `keda` 命名空间，无需额外指定。
- **版本选择**：不指定版本时安装最新稳定版。如需固定版本（生产环境推荐），指定 `AddonVersion`。

#### 最小安装（不指定版本，安装最新版）

```bash
tccli tke InstallAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName keda
# expected: exit 0，返回 RequestId
```

**预期输出**：

```json
{
    "RequestId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
}
```

#### 增强配置（指定版本号）

先查询可用版本：

```bash
tccli tke DescribeAddonValues --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName keda
# expected: 返回可用版本列表
```

```json
{
  "Values": "<Values>",
  "DefaultValues": "<DefaultValues>",
  "RequestId": "<RequestId>"
}
```

```bash
tccli tke InstallAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName keda \
    --AddonVersion ADDON_VERSION
# expected: exit 0，返回 RequestId
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `REGION` | 地域 | 如 `ap-guangzhou` | `tccli configure list` |
| `CLUSTER_ID` | 目标集群 ID | 格式 `CLUSTER_ID` | `tccli tke DescribeClusters --region <Region>` |
| `ADDON_VERSION` | 组件版本 | 如 `2.14.0` | `tccli tke DescribeAddonValues --region <Region> --ClusterId CLUSTER_ID --AddonName keda` |

### 步骤 2：轮询 KEDA 安装状态

```bash
tccli tke DescribeAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName keda
# expected: Status: "Succeeded"
```

**预期输出**（安装完成后）：

```json
{
    "AddonName": "keda",
    "AddonVersion": "2.14.0",
    "Status": "Succeeded",
    "RequestId": "..."
}
```

### 步骤 3：验证 KEDA Pod 运行（数据面）

```bash
kubectl get pods -n keda
# expected: keda-operator 和 keda-metrics-apiserver Pod 均为 Running
```

**预期输出**：

```text
NAME                                      READY   STATUS    RESTARTS   AGE
keda-metrics-apiserver-xxxxxxxxxx-xxxxx   1/1     Running   0          1m
keda-operator-xxxxxxxxxx-xxxxx            1/1     Running   0          1m
```

```bash
# 确认 KEDA CRD 已注册
kubectl get crd | grep keda.sh
# expected: scaledobjects.keda.sh, scaledjobs.keda.sh, triggerauthentications.keda.sh
```

**预期输出**：

```text
scaledjobs.keda.sh                          2025-01-01T00:00:00Z
scaledobjects.keda.sh                       2025-01-01T00:00:00Z
triggerauthentications.keda.sh              2025-01-01T00:00:00Z
```

### 步骤 4：创建第一个 ScaledObject（验证伸缩链路）

#### 选择依据

- **触发器类型**：首个 ScaledObject 使用 `cpu` 触发器验证端到端链路——这是最简单的方式，不需要外部凭据或事件源。
- **目标工作负载**：需要先有一个 Deployment。这里创建 nginx Deployment 作为伸缩目标。
- **验证方式**：KEDA 创建后会自动生成对应的 HPA（命名 `keda-hpa-SCALED_OBJECT_NAME`），确认 HPA 存在即代表链路正常。

#### 最小配置（CPU 触发器，验证链路）

**Step 1 — 创建目标 Deployment：**

```bash
cat > nginx-deploy.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: NAMESPACE
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
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

kubectl apply -f nginx-deploy.yaml
# expected: deployment.apps/nginx created
```

**预期输出**：

```text
deployment.apps/nginx created
```

**Step 2 — 创建 CPU 触发的 ScaledObject：**

```bash
cat > cpu-scaledobject.yaml <<'EOF'
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: nginx-cpu-scaler
  namespace: NAMESPACE
spec:
  scaleTargetRef:
    name: nginx
  minReplicaCount: 1
  maxReplicaCount: 10
  triggers:
  - type: cpu
    metricType: Utilization
    metadata:
      value: "50"
EOF

kubectl apply -f cpu-scaledobject.yaml
# expected: scaledobject.keda.sh/nginx-cpu-scaler created
```

**预期输出**：

```text
scaledobject.keda.sh/nginx-cpu-scaler created
```

**Step 3 — 确认 HPA 自动创建：**

```bash
kubectl get hpa -n NAMESPACE
# expected: 出现 keda-hpa-nginx-cpu-scaler 条目
```

**预期输出**：

```text
NAME                          REFERENCE          TARGETS          MINPODS   MAXPODS   REPLICAS   AGE
keda-hpa-nginx-cpu-scaler     Deployment/nginx   1%/50%           1         10        1          30s
```

```bash
kubectl get scaledobject -n NAMESPACE
# expected: READY True，ACTIVE True
```

**预期输出**：

```text
NAME                 SCALETARGETKIND    SCALETARGETNAME   MIN   MAX   TRIGGERS   AUTHENTICATION   READY   ACTIVE   AGE
nginx-cpu-scaler     apps/v1.Deployment nginx              1    10    cpu                         True    True     30s
```

#### 增强配置（Cron + CPU 组合触发器）

```bash
cat > cron-cpu-scaledobject.yaml <<'EOF'
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: nginx-smart-scaler
  namespace: NAMESPACE
spec:
  scaleTargetRef:
    name: nginx
  minReplicaCount: 1
  maxReplicaCount: 20
  triggers:
  - type: cron
    metadata:
      timezone: Asia/Shanghai
      start: "0 8 * * 1-5"
      end: "0 20 * * 1-5"
      desiredReplicas: "5"
  - type: cpu
    metricType: Utilization
    metadata:
      value: "50"
EOF

kubectl apply -f cron-cpu-scaledobject.yaml
# expected: scaledobject.keda.sh/nginx-smart-scaler created
```

**预期输出**：

```text
scaledobject.keda.sh/nginx-smart-scaler created
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `NAMESPACE` | K8s 命名空间 | 已存在的命名空间 | `kubectl get ns` |

## 验证

### 控制面（tccli）

```bash
# 维度 1：确认 KEDA 组件状态
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
# 维度 2：确认集群状态
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

### 数据面

```bash
# 维度 3：确认 KEDA Pod 运行
kubectl get pods -n keda
# expected: keda-operator 和 keda-metrics-apiserver 均为 Running，READY 1/1
```

**预期输出**：

```text
NAME                                      READY   STATUS    RESTARTS   AGE
keda-metrics-apiserver-xxxxxxxxxx-xxxxx   1/1     Running   0          5m
keda-operator-xxxxxxxxxx-xxxxx            1/1     Running   0          5m
```

```bash
# 维度 4：确认 CRD 已注册
kubectl get crd | grep keda.sh
# expected: 3 个 CRD 均已注册
```

**预期输出**：

```text
scaledjobs.keda.sh                          2025-01-01T00:00:00Z
scaledobjects.keda.sh                       2025-01-01T00:00:00Z
triggerauthentications.keda.sh              2025-01-01T00:00:00Z
```

```bash
# 维度 5：确认 ScaledObject 就绪且 HPA 已创建
kubectl get scaledobject -n NAMESPACE
# expected: READY: True, ACTIVE: True
```

```text
NAME  STATUS  AGE
...
```

```bash
# 维度 6：确认 KEDA 日志无异常
kubectl logs -n keda -l app=keda-operator --tail=20
# expected: 无 FATAL/ERROR 日志，含 "Reconciling ScaledObject" 等信息
```

**预期输出**：

```text
INFO    Reconciling ScaledObject    {"controller": "scaledobject", "namespace": "NAMESPACE", "name": "nginx-cpu-scaler"}
INFO    Creating HPA                 {"controller": "scaledobject", "hpa": "NAMESPACE/keda-hpa-nginx-cpu-scaler"}
```

| 维度 | 命令 | 预期 |
|------|------|------|
| KEDA 组件状态 | `DescribeAddon --AddonName keda` | `Status: "Succeeded"` |
| KEDA Pod | `kubectl get pods -n keda` | keda-operator + keda-metrics-apiserver 均为 Running |
| CRD | `kubectl get crd \| grep keda.sh` | 3 个 CRD 已注册 |
| ScaledObject | `kubectl get scaledobject -n NAMESPACE` | READY: True |
| HPA | `kubectl get hpa -n NAMESPACE` | keda-hpa-* 已创建 |
| KEDA 日志 | `kubectl logs -n keda -l app=keda-operator --tail=20` | 无 ERROR |

## 清理

> **警告**：卸载 KEDA 后，所有 ScaledObject 和 ScaledJob 将失效，自动创建的 HPA 会被删除，工作负载副本数将保持不变（不会自动缩放至原始值）。卸载前确认无正在执行的伸缩操作，并检查是否有依赖 KEDA 自动伸缩的关键业务。

### 数据面

数据面资源清理在控制面之前。

```bash
# 清理前确认所有 ScaledObject
kubectl get scaledobject -A
# 确认所有待清理的 ScaledObject
```

```text
NAME  STATUS  AGE
...
```

```bash
# 删除 ScaledObject（KEDA 自动删除关联的 HPA）
kubectl delete scaledobject nginx-cpu-scaler -n NAMESPACE
kubectl delete scaledobject nginx-smart-scaler -n NAMESPACE
# expected: scaledobject.keda.sh "nginx-cpu-scaler" deleted
```

**预期输出**：

```text
scaledobject.keda.sh "nginx-cpu-scaler" deleted
scaledobject.keda.sh "nginx-smart-scaler" deleted
```

```bash
# 删除测试 Deployment
kubectl delete deployment nginx -n NAMESPACE
# expected: deployment.apps "nginx" deleted
```

```bash
# 确认 HPA 已自动删除
kubectl get hpa -n NAMESPACE
# expected: No resources found（或不再含 keda-hpa 前缀的条目）
```

**预期输出**：

```text
No resources found in NAMESPACE namespace.
```

### 控制面（tccli）

```bash
# 清理前状态检查
tccli tke DescribeAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName keda
# 确认待卸载的组件
```

```json
{
  "Addons": [],
  "AddonName": "<AddonName>",
  "AddonVersion": "<AddonVersion>",
  "RawValues": "<RawValues>",
  "Phase": "<Phase>",
  "Reason": "<Reason>",
  "CreateTime": "<CreateTime>",
  "RequestId": "<RequestId>"
}
```

```bash
# 卸载 KEDA 组件
tccli tke DeleteAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName keda
# expected: exit 0，返回 RequestId
```

**预期输出**：

```json
{
    "RequestId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
}
```

```bash
# 验证已卸载
tccli tke DescribeAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName keda
# expected: ResourceNotFound 或 AddonName 不在列表中
```

**预期输出**：

```json
{
    "Error": {
        "Code": "ResourceNotFound",
        "Message": "Addon not found"
    }
}
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `InstallAddon` 返回 `InvalidParameter.AddonName` | 检查 AddonName 是否为全小写 `keda` | AddonName 大小写错误，必须为小写 `keda` | 使用精确名称 `keda`（全小写） |
| `InstallAddon` 返回 `FailedOperation.AddonAlreadyExists` | `tccli tke DescribeAddon --region <Region> --ClusterId CLUSTER_ID --AddonName keda` | 组件已安装 | 如状态异常，先 `DeleteAddon` 卸载后重新安装 |
| `InstallAddon` 返回 `InvalidParameter.ClusterId` | `tccli tke DescribeClusters --region <Region>` 列出集群确认 ID | 集群 ID 不存在或不属于当前 region | 确认 `--ClusterId` 值与 `DescribeClusters` 返回的 `ClusterId` 一致；检查 region |
| `DescribeAddon` 始终返回 `Installing` | `tccli tke DescribeAddon --region <Region> --ClusterId CLUSTER_ID --AddonName keda` | KEDA Operator 镜像拉取慢或被拦截 | 等待 5 分钟；如仍 Installing → 检查集群节点能否访问 `ccr.ccs.tencentyun.com`；保留 RequestId 备查 |

### 安装成功但伸缩不生效

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| ScaledObject 为 Ready 但 HPA 未创建 | `kubectl describe scaledobject SCALED_OBJECT_NAME -n NAMESPACE` 查看 Events | 触发器 metadata 字段错误或缺失必填参数 | 检查 `triggers[].metadata` 与 Scaler 类型规范一致（如 `cpu` 需 `value` 和 `metricType`） |
| KEDA Pod 处于 CrashLoopBackOff | `kubectl logs -n keda -l app=keda-operator --tail=50` | KEDA Operator 启动失败，通常是 RBAC 权限不足或配置项错误 | 查看日志首行错误信息 → 若为 RBAC 错误，重新安装 KEDA → 若为配置错误，检查自定义 values |
| KEDA Operator 运行但 metrics-server 不可用 | `kubectl get apiservice v1beta1.external.metrics.k8s.io` 和 `kubectl get apiservice v1beta1.custom.metrics.k8s.io` | KEDA Metrics Adapter 未正确注册 API Service | `kubectl get pods -n keda` 确认 keda-metrics-apiserver 为 Running；否则 `kubectl describe pod` 查错 |
| CPU 触发器 HPA 显示 `unknown / 50%` | `kubectl describe hpa keda-hpa-SCALED_OBJECT_NAME -n NAMESPACE` | metrics-server 未安装或未就绪（此为环境限制，非命令错误） | `kubectl get pods -n kube-system \| grep metrics-server` 确认运行中；如未安装，在 TKE 组件市场安装 metrics-server |
| `kubectl apply -f` 返回 CRD 类型未找到 | `kubectl get crd \| grep keda.sh` | KEDA 安装为异步操作，CRD 注册有延迟 | 等待 1-2 分钟后重试；确认 `DescribeAddon` 状态为 `Succeeded` |

## 下一步

- [认识 KEDA](../认识%20KEDA/tccli%20操作.md) — KEDA 概念、HPA 对比与 Scaler 选择指南
- [定时水平伸缩（Cron 触发器）](../定时水平伸缩%20(Cron%20触发器)/tccli%20操作.md) — 基于时间表的伸缩配置
- [多级服务同步水平伸缩（Workload 触发器）](../多级服务同步水平伸缩（Workload%20触发器）/tccli%20操作.md) — 工作负载联动伸缩
- [基于 Prometheus 自定义指标的弹性伸缩](../基于%20Prometheus%20自定义指标的弹性伸缩/tccli%20操作.md) — 使用 Prometheus 指标驱动伸缩

## 控制台替代

登录 [容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) → 选择目标集群 → **组件管理** → **新建** → 搜索 `KEDA` → 选择版本 → **安装**。安装后在 **组件管理** 页可查看状态和版本。
