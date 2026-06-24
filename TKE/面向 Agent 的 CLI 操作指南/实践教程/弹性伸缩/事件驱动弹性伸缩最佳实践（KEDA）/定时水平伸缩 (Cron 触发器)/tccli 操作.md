# 定时水平伸缩（Cron 触发器）（tccli）

> 对照官方：[定时水平伸缩（Cron 触发器）](https://cloud.tencent.com/document/product/457/106147) · page_id `106147`

## 概述

通过 KEDA Cron Scaler 实现按时间计划的自动扩缩容。适用于周期性负载——例如办公时段扩容（8:00-20:00 保持 5 副本）、夜间缩容至 1 副本、周末进一步缩容至最小甚至 0。与 HPA 不同，Cron Scaler 不依赖外部指标，仅按时间表精确控制副本数。

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
# 5. 确认目标集群存在且 KEDA 已安装
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
            "ClusterVersion": "1.30.0"
        }
    ],
    "RequestId": "..."
}
```

```bash
# 6. 确认 KEDA 组件已安装
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
# 确认连通
kubectl get ns
# expected: 返回 default 等系统命名空间
```

```json
{
  "Kubeconfig": "<Kubeconfig>",
  "RequestId": "<RequestId>"
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli / kubectl 命令 | 幂等 |
|-----------|---------------------|:--:|
| 查看集群状态 | `tccli tke DescribeClusters --ClusterIds '["CLUSTER_ID"]'` | 是 |
| 查看 KEDA 组件 | `tccli tke DescribeAddon --AddonName keda` | 是 |
| 创建 Cron ScaledObject | `kubectl apply -f cron-scaledobject.yaml`（需 VPN/IOA） | 是 |
| 查看定时伸缩规则 | `kubectl describe scaledobject SCALED_OBJECT_NAME -n NAMESPACE`（需 VPN/IOA） | 是 |
| 查看伸缩状态 | `kubectl get hpa -n NAMESPACE`（需 VPN/IOA） | 是 |

## 关键字段说明

KEDA Cron Scaler 的 `triggers[].metadata` 字段：

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `timezone` | String | 是 | IANA 时区格式，如 `Asia/Shanghai`、`UTC`、`America/New_York`。**默认 UTC，必须显式设置** | 未设置 → 使用 UTC 时间，比北京时间晚 8 小时，触发时机错误 |
| `start` | String | 是 | 5 字段 cron 表达式（分 时 日 月 周），如 `0 8 * * 1-5`。所有时间基于 `timezone` 时区 | 格式错误 → 触发器无法解析，ScaledObject 状态异常 |
| `end` | String | 是 | 5 字段 cron 表达式，必须晚于 `start` 时间 | end 早于 start → 时间窗口无效，触发器永不激活 |
| `desiredReplicas` | String | 是 | 目标副本数，**值类型为字符串**（如 `"5"`）。必须满足 `minReplicaCount <= desiredReplicas <= maxReplicaCount` | 超出范围 → 实际副本数受 min/max 约束，不会达到期望值 |
| `minReplicaCount` | Integer | 是 | 最小副本数，在 `end` 时间之外生效。可为 `0` | 设为 0 → 非活跃时段无 Pod 运行，但事件到达后自动激活 |
| `maxReplicaCount` | Integer | 是 | 最大副本数，防止 Cron 导致过度扩容 | 未设置 → 无法限制上限 |

### Cron 表达式速查

| 字段位置 | 含义 | 取值范围 | 示例 |
|:--:|------|------|------|
| 1 | 分钟 | 0-59 | `0`（整点） |
| 2 | 小时 | 0-23 | `8`（8 点） |
| 3 | 日期 | 1-31 | `*`（每天） |
| 4 | 月份 | 1-12 | `*`（每月） |
| 5 | 星期 | 0-6（0=周日）或 MON-SUN | `1-5`（周一至周五） |

## 操作步骤

### 步骤 1：部署目标工作负载

#### 选择依据

- **工作负载类型**：选择 Deployment（长期运行服务），Cron Scaler 不适用于 Job 型任务（应使用 ScaledJob）。
- **初始副本数**：设为 Cron 触发器非活跃时段的最低值，如此处设 `replicas: 1`。
- **资源限制**：设置 requests/limits 确保 KEDA 在扩容时调度器能正常分配资源。

#### 最小配置（nginx Deployment）

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

```bash
kubectl get pods -n NAMESPACE -l app=nginx
# expected: 1 个 Pod 为 Running
```

```text
NAME  STATUS  AGE
...
```

### 步骤 2：创建 Cron 触发 ScaledObject

#### 选择依据

- **时区**：必须显式设置 `timezone: Asia/Shanghai`，KEDA 默认 UTC，直接使用会导致比北京时间晚 8 小时触发。
- **时间窗口设计**：工作日上午 8:00 扩容至 5 副本，下午 18:00 缩回 1 副本。weekday 用 `1-5`（周一至周五），不包含周末。
- **desiredReplicas**：`"5"` 必须在 `minReplicaCount: 1` 和 `maxReplicaCount: 10` 之间。
- **多 Cron 触发器**：如需要多个时间窗口（如中午高峰额外扩容），可在 `triggers` 数组添加多个 `cron` 条目，KEDA 取所需副本数的最大值。

#### 最小配置（单一 Cron 触发器）

```bash
cat > cron-scaledobject.yaml <<'EOF'
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: nginx-cron-scaler
  namespace: NAMESPACE
spec:
  scaleTargetRef:
    name: nginx
  minReplicaCount: 1
  maxReplicaCount: 10
  triggers:
  - type: cron
    metadata:
      timezone: Asia/Shanghai
      start: "0 8 * * 1-5"
      end: "0 18 * * 1-5"
      desiredReplicas: "5"
EOF

kubectl apply -f cron-scaledobject.yaml
# expected: scaledobject.keda.sh/nginx-cron-scaler created
```

**预期输出**：

```text
scaledobject.keda.sh/nginx-cron-scaler created
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `NAMESPACE` | K8s 命名空间 | 已存在的命名空间 | `kubectl get ns` |

### 步骤 3：查询 ScaledObject 状态与下次触发时间

```bash
kubectl get scaledobject nginx-cron-scaler -n NAMESPACE
# expected: READY: True, ACTIVE: True（如果在活跃时间窗口内），或 False（非活跃窗口）
```

**预期输出**：

```text
NAME                SCALETARGETKIND    SCALETARGETNAME   MIN   MAX   TRIGGERS   AUTHENTICATION   READY   ACTIVE   AGE
nginx-cron-scaler   apps/v1.Deployment nginx              1    10    cron                         True    False    1m
```

```bash
kubectl describe scaledobject nginx-cron-scaler -n NAMESPACE
# expected: Conditions 含 ScaledObjectReady=True，Status 含 cron 触发器详情
```

**预期输出**（关键片段）：

```text
Status:
  Conditions:
    Type:    Ready
    Status:  True
    Type:    Active
    Status:  False
    Type:    Fallback
    Status:  False
  External Metric Names:
    s0-cron-Asia-Shanghai-0-8-*-*-1-5
```

```bash
# 确认 KEDA 创建的 HPA
kubectl get hpa -n NAMESPACE
# expected: keda-hpa-nginx-cron-scaler 存在
```

**预期输出**：

```text
NAME                          REFERENCE          TARGETS          MINPODS   MAXPODS   REPLICAS   AGE
keda-hpa-nginx-cron-scaler    Deployment/nginx   unknown/5 (avg)  1         10        1          1m
```

### 步骤 4：增强配置——多时间窗口 + Cron + CPU 组合

#### 选择依据

- **多时间窗口**：增加一个中午高峰窗口（12:00-14:00 扩容至 10），以及夜间更低的副本数（使用 `minReplicaCount: 0` 在非活跃窗口完全缩容节约成本）。
- **Cron + CPU 组合**：同时加入 CPU 触发器作为兜底——在活跃时间窗口内，如果 CPU 超过 80%，进一步扩容至 `maxReplicaCount`。
- **时区**：所有 Cron 触发器使用相同 `timezone: Asia/Shanghai`。

```bash
cat > cron-enhanced-scaledobject.yaml <<'EOF'
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: nginx-cron-enhanced
  namespace: NAMESPACE
spec:
  scaleTargetRef:
    name: nginx
  minReplicaCount: 1
  maxReplicaCount: 15
  triggers:
  - type: cron
    metadata:
      timezone: Asia/Shanghai
      start: "0 8 * * 1-5"
      end: "0 12 * * 1-5"
      desiredReplicas: "5"
  - type: cron
    metadata:
      timezone: Asia/Shanghai
      start: "0 12 * * 1-5"
      end: "0 14 * * 1-5"
      desiredReplicas: "10"
  - type: cron
    metadata:
      timezone: Asia/Shanghai
      start: "0 14 * * 1-5"
      end: "0 18 * * 1-5"
      desiredReplicas: "5"
  - type: cpu
    metricType: Utilization
    metadata:
      value: "80"
EOF

kubectl apply -f cron-enhanced-scaledobject.yaml
# expected: scaledobject.keda.sh/nginx-cron-enhanced created
```

**预期输出**：

```text
scaledobject.keda.sh/nginx-cron-enhanced created
```

> **注意**：多个 Cron 触发器叠加时，KEDA 取 `desiredReplicas` 的最大值。例如 12:00-14:00 期间 `desiredReplicas: "10"` 为最大值，实际副本数为 10。

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
# expected: nginx-cron-scaler 和 nginx-cron-enhanced READY: True
```

**预期输出**：

```text
NAME                    SCALETARGETKIND    SCALETARGETNAME   MIN   MAX   TRIGGERS   READY   ACTIVE   AGE
nginx-cron-scaler       apps/v1.Deployment nginx              1    10    cron        True    False    5m
nginx-cron-enhanced     apps/v1.Deployment nginx              1    15    cron,cpu   True    False    1m
```

```bash
# 维度 4：查看 Cron 触发器详情（确认时间窗口和下次触发）
kubectl describe scaledobject nginx-cron-enhanced -n NAMESPACE | grep -A 20 "Triggers:"
# expected: 显示所有 cron 触发器及其 start/end/desiredReplicas
```

```text
NAME  STATUS  AGE
...
```

```bash
# 维度 5：确认 HPA 已创建
kubectl get hpa -n NAMESPACE
# expected: 每个 ScaledObject 对应一个 keda-hpa-* HPA
```

**预期输出**：

```text
NAME                              REFERENCE          TARGETS          MINPODS   MAXPODS   REPLICAS   AGE
keda-hpa-nginx-cron-scaler        Deployment/nginx   unknown/5 (avg)  1         10        1          5m
keda-hpa-nginx-cron-enhanced      Deployment/nginx   unknown/5 (avg)  1         15        1          1m
```

```bash
# 维度 6：查看 KEDA Operator 日志（确认 Cron 触发器被识别）
kubectl logs -n keda -l app=keda-operator --tail=20 | grep -i cron
# expected: 含 "cron" 相关日志，无 ERROR
```

```text
NAME  STATUS  AGE
...
```

| 维度 | 命令 | 预期 |
|------|------|------|
| 集群状态 | `DescribeClusters --ClusterIds '["CLUSTER_ID"]'` | `ClusterStatus: "Running"` |
| KEDA 状态 | `DescribeAddon --AddonName keda` | `Status: "Succeeded"` |
| ScaledObject | `kubectl get scaledobject -n NAMESPACE` | READY: True |
| 触发器详情 | `kubectl describe scaledobject -n NAMESPACE` | 各 Cron 触发器 start/end 正确 |
| HPA | `kubectl get hpa -n NAMESPACE` | keda-hpa-* 存在 |
| KEDA 日志 | `kubectl logs -n keda -l app=keda-operator` | 无 ERROR，cron 触发器已注册 |

## 清理

> **警告**：删除 ScaledObject 后，KEDA 自动删除关联的 HPA，工作负载副本数将**立即恢复为 Deployment 原始 `replicas` 值**（而非 Cron 最后设置的副本数）。如果此时处于业务高峰，可能因副本骤降导致服务不可用。建议在非活跃时段执行清理。

### 数据面

数据面资源清理在控制面之前。

```bash
# 清理前确认所有 ScaledObject
kubectl get scaledobject -n NAMESPACE
# 确认目标 ScaledObject 名称
```

```text
NAME  STATUS  AGE
...
```

```bash
# 删除 ScaledObject（清理关联 HPA）
kubectl delete scaledobject nginx-cron-scaler -n NAMESPACE
kubectl delete scaledobject nginx-cron-enhanced -n NAMESPACE
# expected: scaledobject.keda.sh "nginx-cron-scaler" deleted
```

**预期输出**：

```text
scaledobject.keda.sh "nginx-cron-scaler" deleted
scaledobject.keda.sh "nginx-cron-enhanced" deleted
```

```bash
# 确认 HPA 已自动清理
kubectl get hpa -n NAMESPACE
# expected: No resources found（或不再含 keda-hpa 前缀的条目）
```

**预期输出**：

```text
No resources found in NAMESPACE namespace.
```

```bash
# 删除测试 Deployment
kubectl delete deployment nginx -n NAMESPACE
# expected: deployment.apps "nginx" deleted
```

### 控制面（tccli）

Cron 伸缩的资源全部在集群内部（CRD + Deployment），无 tccli 控制面资源需要清理。确认 KEDA 组件状态正常即可：

```bash
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
| `kubectl apply -f` 返回 `error: unable to recognize ... no matches for kind "ScaledObject"` | `kubectl get crd \| grep keda.sh` 检查 CRD 是否已注册 | KEDA 未安装或 CRD 未注册 | 安装 KEDA（参见 [在 TKE 上部署 KEDA](../在%20TKE%20上部署%20KEDA/tccli%20操作.md)）；已安装则等待 `DescribeAddon` 状态为 `Succeeded` |
| `kubectl apply -f` 返回 `field is immutable` | `kubectl get scaledobject SCALED_OBJECT_NAME -n NAMESPACE -o yaml` 对比变更 | 尝试修改不可变字段（如 `spec.scaleTargetRef.name`） | `kubectl delete scaledobject` 后重新 apply |
| ScaledObject 创建成功但 READY 为 False | `kubectl describe scaledobject SCALED_OBJECT_NAME -n NAMESPACE` 查看 Conditions | Cron 表达式格式错误或 `desiredReplicas` 超出 `minReplicaCount`/`maxReplicaCount` 范围 | 检查 `start`/`end` 为 5 字段格式；确保 `minReplicaCount <= desiredReplicas <= maxReplicaCount` |

### 创建成功但伸缩异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 到达 `start` 时间但副本数未变化 | `kubectl describe scaledobject SCALED_OBJECT_NAME -n NAMESPACE` 查看 `ACTIVE` 状态 | 时区设置错误，KEDA 默认 UTC，未设置 `timezone: Asia/Shanghai` 导致比北京时间晚 8 小时 | 添加 `timezone: Asia/Shanghai`（或目标时区 IANA 格式）。修改后 `kubectl delete scaledobject` 重建 |
| 到达 `end` 时间但副本数未缩回 | `kubectl get hpa -n NAMESPACE` 和 `kubectl get pods -n NAMESPACE` | KEDA 缩放有轮询周期（约 30s-60s），非瞬间切换。如果有 CPU 触发器，CPU 使用率可能阻止缩容 | 等待 1-2 分钟；如仍不缩容，检查是否有多触发器导致取最大值 |
| 多 Cron 触发器中副本数不符合预期 | `kubectl describe scaledobject SCALED_OBJECT_NAME -n NAMESPACE` 查看各触发器 desiredReplicas | KEDA 多 Cron 触发器取 `desiredReplicas` 最大值，而非按时间精确切换 | 检查各 Cron 时间窗口是否重叠，确保重叠时段 `desiredReplicas` 最大值符合预期 |
| 周末工作负载仍在运行（minReplicaCount=1） | `kubectl get pods -n NAMESPACE -l app=nginx` | Cron `start` 范围仅覆盖 `1-5`（周一至周五），但 `minReplicaCount: 1` 保证至少有 1 个副本 | 如需周末完全停机，设置 `minReplicaCount: 0` |
| `desiredReplicas` 设置为 `"3"` 但实际副本数为 1 | `kubectl get scaledobject SCALED_OBJECT_NAME -n NAMESPACE -o yaml \| grep -E "minReplicaCount\|maxReplicaCount\|desiredReplicas"` | `desiredReplicas` 字段必须是**字符串类型** `"3"`，而非数字 `3` | 修改为字符串 `"3"`，重新 apply |

## 下一步

- [认识 KEDA](../认识%20KEDA/tccli%20操作.md) — KEDA 概念与 Scaler 类型概览
- [多级服务同步水平伸缩（Workload 触发器）](../多级服务同步水平伸缩（Workload%20触发器）/tccli%20操作.md) — 工作负载联动伸缩
- [基于 Prometheus 自定义指标的弹性伸缩](../基于%20Prometheus%20自定义指标的弹性伸缩/tccli%20操作.md) — 使用 Prometheus 指标驱动伸缩
- [在 TKE 上部署 KEDA](../在%20TKE%20上部署%20KEDA/tccli%20操作.md) — KEDA 安装与卸载

## 控制台替代

登录 [容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) → 选择目标集群 → **工作负载** → **Deployment** → 选择目标 Deployment → **弹性伸缩** → **定时策略** → 添加规则（设置时间、时区、目标副本数）。
