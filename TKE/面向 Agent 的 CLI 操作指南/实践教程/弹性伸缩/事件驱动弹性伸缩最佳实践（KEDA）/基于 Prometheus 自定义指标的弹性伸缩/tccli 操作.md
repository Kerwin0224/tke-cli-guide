# 基于 Prometheus 自定义指标的弹性伸缩（tccli）

> 对照官方：[基于 Prometheus 自定义指标的弹性伸缩](https://cloud.tencent.com/document/product/457/107998) · page_id `107998`

## 概述

通过 KEDA Prometheus Scaler 将 Prometheus 中采集的自定义指标（如 HTTP 请求速率、队列深度、错误率）映射为弹性伸缩的触发信号。Prometheus 作为事实标准的监控后端，可覆盖绝大多数业务指标场景。

> **kubectl 数据面不可达**：CAM拒绝公网端点(strategyId:240463971)，内网需VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

### 方案选择

Prometheus Scaler 适用于以下场景，与其他 KEDA Scaler 的区别：

| Scaler 类型 | 适用场景 | 典型指标 | 本页用途 |
|------------|---------|---------|---------|
| **Prometheus** | 已有 Prometheus 监控，指标灵活自定义 | HTTP 请求速率、错误率、延迟 p99 | 当前页面 |
| CPU/Memory | 标准化资源监控，无需额外组件 | CPU 利用率、内存使用 | 见 HPA 文档 |
| Cron | 定时扩缩容，与指标无关 | 时间窗口 | 见 Cron Scaler 文档 |

**选择 Prometheus Scaler 的依据**：

- 已有 Prometheus 部署且采集了业务指标
- 需要基于复杂 PromQL 查询结果（如 `rate()`、`histogram_quantile()`）进行弹性伸缩
- 需要多维度组合指标（如 QPS > 100 且 p99 < 500ms）

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
#    tke:DescribeClusters, tke:InstallAddon, tke:DescribeAddon
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

Prometheus ScaledObject 中与 Prometheus 连接和查询相关的核心参数：

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `serverAddress` | String | 是 | Prometheus server 地址，格式 `http://<service>.<namespace>:9090` | 地址不可达 → ScaledObject 无法获取指标，`Active` 状态为 `False` |
| `metricName` | String | 是 | KEDA 内部指标名称，用于 HPA 追踪，可自定义 | 命名不规范 → HPA 中指标名难识别 |
| `query` | String | 是 | PromQL 查询语句，返回标量值（单个数字），如 `rate(http_requests_total{app="myapp"}[2m])` | 查询返回空或多值 → Scaler 返回错误 |
| `threshold` | String | 是 | 触发扩缩容的阈值，与查询返回值单位一致 | 阈值不合理 → 频繁扩缩或永不触发 |
| `authModes` | String | 否 | 认证方式：`"bearer"`（默认）或 `"basic"`。无认证时留空 | 认证方式不匹配 → 连接 Prometheus 被拒 |

## 操作步骤

### 步骤 1：验证集群状态

#### 选择依据

确认目标集群处于 `Running` 状态，且网络配置正常。集群必须已部署 KEDA 组件。如未安装，先参考 KEDA 安装文档安装 `keda` Addon。

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
                "VpcId": "vpc-example",
                "ClusterCIDR": "10.100.0.0/16"
            }
        }
    ],
    "RequestId": "..."
}
```

### 步骤 2：安装 Prometheus Addon（控制面）

#### 选择依据

TKE 提供 `prometheus` 托管 Addon，可快速在集群内搭建 Prometheus 服务。选择此 Addon 而非自行部署的理由：
- 与控制台集成，自动配置 ServiceMonitor
- 托管运维，无需自行管理 Prometheus 的 PVC 和持久化

`prometheus-addon.json`：

```json
{
    "ClusterId": "CLUSTER_ID",
    "AddonName": "prometheus",
    "AddonVersion": "1.0.0",
    "RawValues": "{\"prometheus\":{\"retention\":\"7d\",\"storage\":\"50Gi\"}}"
}
```

```bash
tccli tke InstallAddon --region <Region> --cli-input-json file://prometheus-addon.json
# expected: exit 0
```

**预期输出**：

```json
{
    "RequestId": "..."
}
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `CLUSTER_ID` | 集群 ID | 格式 `cls-xxxxxxxx` | `tccli tke DescribeClusters --region <Region>` |
| `REGION` | 地域 | 如 `ap-guangzhou` | `tccli configure list` 查看 region |

### 步骤 3：验证 Addon 状态（控制面）

Addon 安装是异步操作，需轮询确认状态为 `Succeeded`。

```bash
tccli tke DescribeAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName prometheus
# expected: Status 为 "Succeeded"
```

**预期输出**：

```json
{
    "Addons": [
        {
            "AddonName": "prometheus",
            "AddonVersion": "1.0.0",
            "AddonStatus": "Succeeded",
            "RawValues": "..."
        }
    ],
    "RequestId": "..."
}
```

| 验证维度 | 命令 | 预期 |
|---------|------|------|
| 状态 | `DescribeAddon --AddonName prometheus` | `AddonStatus: "Succeeded"` |
| 版本 | 同上，检查 `AddonVersion` | 与安装时指定版本一致 |

### 步骤 4：部署示例应用（数据面）

部署一个暴露 Prometheus 指标端点的示例应用，模拟 HTTP 请求计数场景。

`sample-app.yaml`：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: DEPLOYMENT_NAME
  namespace: NAMESPACE
  labels:
    app: prometheus-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus-demo
  template:
    metadata:
      labels:
        app: prometheus-demo
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/metrics"
    spec:
      containers:
        - name: sample-app
          image: nginx:latest
          ports:
            - containerPort: 80
              name: http
---
apiVersion: v1
kind: Service
metadata:
  name: prometheus-demo-svc
  namespace: NAMESPACE
spec:
  selector:
    app: prometheus-demo
  ports:
    - port: 80
      targetPort: 80
      name: http
  type: ClusterIP
```

```bash
kubectl apply -f sample-app.yaml
# expected: deployment.apps/DEPLOYMENT_NAME created, service/prometheus-demo-svc created
```

**预期输出**：

```text
deployment.apps/prometheus-demo created
service/prometheus-demo-svc created
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `DEPLOYMENT_NAME` | 部署名称 | 长度 1-63 字符 | 自定义，如 `prometheus-demo` |
| `NAMESPACE` | 命名空间 | 需已存在 | `kubectl get ns` 确认 |

### 步骤 5：创建 TriggerAuthentication（数据面）

创建 TriggerAuthentication 用于 KEDA 连接到 Prometheus server。若 Prometheus 无需认证则跳过此步（在 ScaledObject 中不指定 `authenticationRef`）。

`trigger-auth-prometheus.yaml`：

```yaml
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: keda-prometheus-auth
  namespace: NAMESPACE
spec:
  secretTargetRef:
    - parameter: bearerToken
      name: prometheus-token
      key: token
    - parameter: ca
      name: prometheus-token
      key: ca
```

```bash
kubectl apply -f trigger-auth-prometheus.yaml
# expected: triggerauthentication.keda.sh/keda-prometheus-auth created
```

**预期输出**：

```text
triggerauthentication.keda.sh/keda-prometheus-auth created
```

### 步骤 6：创建 ScaledObject（数据面）

#### 选择依据

配置 Prometheus Scaler 的关键决策：

- **`query`**：PromQL 查询表达式。本例以 HTTP 请求速率为指标：`rate(http_requests_total{app="prometheus-demo"}[2m])`。实际使用时替换为你的指标名和标签选择器。
- **`threshold`**：当查询结果超过此值时触发扩容。设为 `"10"` 表示每秒 10 个请求以上开始扩容。
- **`serverAddress`**：Prometheus 服务地址。TKE 托管 Prometheus 地址格式为 `http://prometheus.monitoring:9090`。通过 `kubectl get svc -n monitoring` 确认。

`scaledobject-prometheus.yaml`：

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: prometheus-scaledobject
  namespace: NAMESPACE
spec:
  scaleTargetRef:
    name: DEPLOYMENT_NAME
  minReplicaCount: 1
  maxReplicaCount: 10
  cooldownPeriod: 300
  pollingInterval: 15
  triggers:
    - type: prometheus
      metadata:
        serverAddress: http://prometheus.monitoring:9090
        metricName: http_requests_per_second
        threshold: "10"
        query: rate(http_requests_total{app="prometheus-demo"}[2m])
      authenticationRef:
        name: keda-prometheus-auth
```

```bash
kubectl apply -f scaledobject-prometheus.yaml
# expected: scaledobject.keda.sh/prometheus-scaledobject created
```

**预期输出**：

```text
scaledobject.keda.sh/prometheus-scaledobject created
```

| 参数字段 | 说明 | 推荐值 | 调整建议 |
|---------|------|--------|---------|
| `minReplicaCount` | 最小副本数 | `1` | 生产环境建议 ≥ 2 |
| `maxReplicaCount` | 最大副本数 | `10` | 根据集群资源和业务上限设定 |
| `cooldownPeriod` | 缩容冷却时间（秒） | `300` | 防止频繁缩容抖动 |
| `pollingInterval` | Prometheus 指标拉取间隔（秒） | `15` | 越小反应越快，但对 Prometheus 压力越大 |
| `query` | PromQL 查询 | 业务自定义 | 必须是瞬时向量查询，返回单个标量值 |
| `threshold` | 触发阈值 | 业务自定义 | 与 `query` 返回值单位一致 |

## 验证

### 控制面（tccli）

```bash
# 1. 确认集群 Running
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterStatus: "Running"

# 2. 确认 Prometheus Addon Succeeded
tccli tke DescribeAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName prometheus
# expected: AddonStatus: "Succeeded"
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
# 1. 确认 ScaledObject 状态
kubectl get scaledobject -n NAMESPACE
# expected: READY True, ACTIVE True/False
```

**预期输出**：

```text
NAME                      SCALETARGETKIND      SCALETARGETNAME     MIN   MAX   TRIGGERS      AUTHENTICATION          READY   ACTIVE   AGE
prometheus-scaledobject   apps/v1.Deployment   prometheus-demo     1     10    prometheus    keda-prometheus-auth    True    False    2m
```

```bash
# 2. 确认 HPA 已自动创建
kubectl get hpa -n NAMESPACE
# expected: HPA 由 ScaledObject 自动生成
```

**预期输出**：

```text
NAME                            REFERENCE                   TARGETS             MINPODS   MAXPODS   REPLICAS   AGE
keda-hpa-prometheus-scaledobject Deployment/prometheus-demo   0/10 (avg)          1         10        1          2m
```

```bash
# 3. 检查 Prometheus 指标端点可达
kubectl exec -it -n NAMESPACE \
    $(kubectl get pod -n NAMESPACE -l app=prometheus-demo -o jsonpath='{.items[0].metadata.name}') \
    -- curl -s http://localhost:80
# expected: 返回 nginx 欢迎页或指标数据

# 4. 发送测试流量触发扩容
kubectl run -it --rm load-generator \
    --image=busybox --restart=Never -n NAMESPACE \
    -- /bin/sh -c "while true; do wget -q -O- http://prometheus-demo-svc; done"
# expected: 持续发送 HTTP 请求

# 5. 观察 Pod 扩容
kubectl get pods -n NAMESPACE -l app=prometheus-demo -w
# expected: Pod 数量从 1 开始增加
```

```text
NAME  STATUS  AGE
...
```

| 验证维度 | 命令 | 预期 |
|---------|------|------|
| ScaledObject 就绪 | `kubectl get scaledobject -n NAMESPACE` | `READY: True` |
| HPA 自动生成 | `kubectl get hpa -n NAMESPACE` | 存在 `keda-hpa-` 前缀的 HPA |
| 触发器激活 | 同上，`ScaledObject` `ACTIVE` | 流量持续时变为 `True` |
| Pod 扩容 | `kubectl get pods -n NAMESPACE` | 副本数随负载增加 |
| 指标采集 | Prometheus 查询 `/metrics` 端点 | `http_requests_total` 计数器递增 |

## 清理

> **警告**：删除 ScaledObject 会**同时删除关联的 HPA**，Deployment 将保持当前副本数不再自动伸缩。Prometheus Addon 卸载后会删除所有采集数据和配置，且不可恢复。Prometheus Addon 持续运行会产生**存储费用**（PVC），请确认后操作。

### 数据面清理（先执行）

```bash
# 1. 删除负载生成器（如仍在运行）
kubectl delete pod load-generator -n NAMESPACE --ignore-not-found
# expected: pod "load-generator" deleted

# 2. 删除 ScaledObject
kubectl delete scaledobject prometheus-scaledobject -n NAMESPACE
# expected: scaledobject.keda.sh "prometheus-scaledobject" deleted

# 3. 删除 TriggerAuthentication
kubectl delete triggerauthentication keda-prometheus-auth -n NAMESPACE
# expected: triggerauthentication.keda.sh "keda-prometheus-auth" deleted

# 4. 删除示例应用
kubectl delete -f sample-app.yaml
# expected: deployment.apps "prometheus-demo" deleted, service "prometheus-demo-svc" deleted
```

### 控制面清理（后执行）

```bash
# 5. 清理前确认 Addon 状态
tccli tke DescribeAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName prometheus
# expected: 确认是目标 Addon，记录 AddonVersion

# 6. 卸载 Prometheus Addon
tccli tke DeleteAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName prometheus
# expected: exit 0

# 7. 验证已卸载
tccli tke DescribeAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName prometheus
# expected: 返回资源不存在或空列表
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
| `InstallAddon` 返回 `FailedOperation.AddonAlreadyInstalled` | `tccli tke DescribeAddon --region <Region> --ClusterId CLUSTER_ID --AddonName prometheus` 查看 Addon 状态 | 集群已安装同名 Addon | 如 Addon 状态正常则无需重复安装；如需重装，先 `DeleteAddon` 后再 `InstallAddon` |
| `InstallAddon` 返回 `InvalidParameter.ClusterId` | 检查 `--ClusterId` 参数格式 | 集群 ID 格式错误或不存在 | 用 `tccli tke DescribeClusters --region <Region>` 列出可用集群，确认正确的 `ClusterId` |
| `InstallAddon` 返回 `InvalidParameterVersion` | `tccli tke DescribeAddon --region <Region> --ClusterId CLUSTER_ID --AddonName prometheus`（如已安装）或查看 Addon 版本列表 | `AddonVersion` 字段指定的版本不可用 | 校正 `AddonVersion` 为有效版本号 |
| `DescribeAddon` 返回空数组 `"Addons": []` | 检查 AddonName 拼写 | Addon 未安装或名称大小写错误 | `AddonName` 必须为 `"prometheus"`（全小写），确认大小写；如未安装则先执行 `InstallAddon` |
| `kubectl apply` 返回 `error: unable to recognize "scaledobject.yaml"` | `kubectl api-resources \| grep scaledobjects` 检查 CRD 是否存在 | KEDA 未安装，ScaledObject CRD 未注册 | 先在集群中安装 KEDA Addon：`tccli tke InstallAddon --region <Region> --cli-input-json file://keda-install.json`，AddonName 为 `keda` |

### 操作提交成功但异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| ScaledObject `READY` 为 `False` | `kubectl describe scaledobject prometheus-scaledobject -n NAMESPACE` 查看 Events | Prometheus 连接失败、PromQL 语法错误或认证失败 | 检查 Events 中的错误信息；验证 `serverAddress` 可达：`kubectl run -it --rm debug --image=busybox --restart=Never -n NAMESPACE -- wget -q -O- http://prometheus.monitoring:9090/api/v1/query?query=up` |
| ScaledObject `ACTIVE` 始终为 `False` | `kubectl describe scaledobject prometheus-scaledobject -n NAMESPACE` 查看触发器状态；`kubectl get hpa -n NAMESPACE` 查看 HPA TARGETS | PromQL 查询返回值为 0 或低于阈值 | 手动查询 Prometheus 确认数据：`kubectl port-forward -n monitoring svc/prometheus 9090:9090` 然后在浏览器访问 Prometheus UI 验证 PromQL；调整 `query` 或检查应用是否暴露指标 |
| Addon 状态长时间非 `Succeeded` | `tccli tke DescribeAddon --region <Region> --ClusterId CLUSTER_ID --AddonName prometheus` 查看 `AddonStatus` 和 `Reason` | 集群资源不足或网络限流导致安装卡住 | 检查集群节点资源：`kubectl top nodes`；超过 10 分钟则保留 region、ClusterId、RequestId → 登录 [TKE 控制台](https://console.cloud.tencent.com/tke2/cluster) 查看 Addon 安装细节 → 仍无法解决则 [提交工单](https://console.cloud.tencent.com/workorder) |
| Pod 实际未扩容但指标已超阈值 | `kubectl get hpa -n NAMESPACE` 查看 HPA 的 `TARGETS` 列；`kubectl describe hpa keda-hpa-prometheus-scaledobject -n NAMESPACE` | HPA 计算显示指标未达到目标值（单位不匹配或冷启动期） | 检查 `threshold` 与 `query` 返回值单位一致；等待至少 1 个 `pollingInterval` + 冷启动期（默认 300s） |

## 下一步

- [认识 KEDA](https://cloud.tencent.com/document/product/457/106143) -- page_id `106143`
- [在 TKE 上部署 KEDA](https://cloud.tencent.com/document/product/457/106144) -- page_id `106144`
- [基于 Apache Pulsar 消息队列的弹性伸缩](https://cloud.tencent.com/document/product/457/106149) -- page_id `106149`
- [多级服务同步水平伸缩（Workload 触发器）](https://cloud.tencent.com/document/product/457/106150) -- page_id `106150`
- [TKE Addon 管理](https://cloud.tencent.com/document/product/457/54682) -- 通过 CLI 管理集群组件

## 控制台替代

[TKE 控制台 → 集群 → 组件管理](https://console.cloud.tencent.com/tke2/cluster)：在「组件管理」中安装和管理 Prometheus Addon；[KEDA 控制台 → ScaledObject](https://console.cloud.tencent.com/tke2/keda)：可视化创建和管理 ScaledObject。
