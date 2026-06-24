# 弹性伸缩（tccli）

> 对照官方：[弹性伸缩](https://cloud.tencent.com/document/product/457/45636) · page_id `45636`

## 概述

弹性伸缩是 TKE 集群应对负载波动的核心能力，涵盖 Pod 级（HPA/VPA/KEDA）和节点级（Cluster Autoscaler）两个维度。本页介绍四种弹性伸缩机制的 CLI 操作方式、策略选择依据及排障方法。

**本页为 HYBRID 页面**：控制面操作使用 `tccli tke`，数据面操作使用 `kubectl`。

> **注意**：以下 `kubectl` 命令假定你已配置 kubeconfig 并可访问集群 API Server。若 kubectl 不可达（如 API Server 仅内网暴露），请先通过 [集群访问配置](https://cloud.tencent.com/document/product/457/32191) 建立连接，或使用 TKE 控制台内置的 Cloud Shell。

### 四种弹性伸缩机制

| 机制 | 伸缩对象 | 触发条件 | 控制面/数据面 | 适用场景 |
|------|----------|----------|--------------|----------|
| HPA | Pod 副本数 | CPU/内存/自定义指标 | 数据面（kubectl） | 无状态应用水平扩缩容 |
| VPA | Pod 资源配额 | 历史用量 | 数据面（kubectl） | 有状态应用垂直调整 |
| KEDA | Pod 副本数 | 事件驱动（Prometheus/Kafka 等） | 数据面（kubectl） | 事件驱动、突发流量 |
| Cluster Autoscaler (CA) | 节点数 | Pod 调度失败/闲置节点 | 控制面（tccli） | 节点池弹性伸缩 |

## 控制台与 CLI 参数映射

### HPA（kubectl）

| 控制台参数 | CLI 参数 | 类型 | 必填 | 默认值 | 说明 | 幂等性 |
|-----------|----------|------|------|--------|------|--------|
| 命名空间 | `--namespace` / `-n` | string | 否 | `default` | 目标命名空间 | 指定同一值即为幂等 |
| 部署名称 | `DEPLOYMENT_NAME`（位置参数） | string | 是 | - | 目标 Deployment | 幂等（同名覆盖） |
| CPU 阈值 | `--cpu-percent` | int | 否 | `80` | CPU 使用率触发阈值（%） | 幂等 |
| 最小副本数 | `--min` | int | 否 | `1` | 最小副本数 | 幂等 |
| 最大副本数 | `--max` | int | 是 | - | 最大副本数 | 幂等 |
| 内存阈值 | `--memory-percent`（YAML） | int | 否 | - | 内存使用率触发阈值（%），仅 YAML 声明 | 幂等 |

### KEDA ScaledObject（kubectl）

| 控制台参数 | CLI 参数（YAML） | 类型 | 必填 | 默认值 | 说明 | 幂等性 |
|-----------|-----------------|------|------|--------|------|--------|
| 伸缩目标 | `spec.scaleTargetRef.name` | string | 是 | - | 目标 Deployment/StatefulSet 名称 | 幂等 |
| 触发器类型 | `spec.triggers[].type` | string | 是 | - | `prometheus`/`kafka`/`cron` 等 | 幂等 |
| Prometheus 地址 | `spec.triggers[].metadata.serverAddress` | string | 是 | - | Prometheus Server 地址 | 幂等 |
| 指标查询 | `spec.triggers[].metadata.query` | string | 是 | - | PromQL 查询语句 | 幂等 |
| 目标值 | `spec.triggers[].metadata.threshold` | string | 是 | - | 触发阈值 | 幂等 |
| 最小副本 | `spec.minReplicaCount` | int | 否 | `0` | 最小副本数（可缩至 0） | 幂等 |
| 最大副本 | `spec.maxReplicaCount` | int | 是 | - | 最大副本数 | 幂等 |

### Cluster Autoscaler（tccli tke CreateClusterAsGroup）

| 控制台参数 | CLI 参数 | 类型 | 必填 | 默认值 | 说明 | 幂等性 |
|-----------|----------|------|------|--------|------|--------|
| 集群 ID | `--ClusterId` | string | 是 | - | 目标集群 ID | 非幂等（多次创建会产生多组 AS 组） |
| 伸缩组名称 | `AutoScalingGroupPara.AutoScalingGroupName` | string | 是 | - | AS 伸缩组名称 | 非幂等 |
| 最大节点数 | `AutoScalingGroupPara.MaxSize` | int | 是 | - | 伸缩组最大节点数 | 幂等（更新场景） |
| 最小节点数 | `AutoScalingGroupPara.MinSize` | int | 是 | - | 伸缩组最小节点数 | 幂等（更新场景） |
| VPC ID | `AutoScalingGroupPara.VpcId` | string | 是 | - | VPC ID | 不可变 |
| 子网 ID 列表 | `AutoScalingGroupPara.SubnetIds` | array | 是 | - | 节点所在子网 | 幂等（更新场景） |
| 机型 | `AutoScalingGroupPara.InstanceTypes` | array | 否 | `["S5.MEDIUM4"]` | 实例机型列表 | 幂等（更新场景） |
| 伸缩策略 | `AutoScalingGroupPara.ServiceSettings.ScalingMode` | string | 否 | `CLASSIC_SCALING` | 伸缩模式 | 幂等（更新场景） |
| Label 标签 | `Labels` | array | 否 | - | 节点 Label | 幂等（更新场景） |
| 初始节点数 | `--InstanceAdvancedSettings.DesiredPodNumber` | int | 否 | `0` | 创建时初始节点数 | 仅首次生效 |

> **幂等性说明**：`CreateClusterAsGroup` 接口本身非幂等（重复调用会创建多个 AS 组）。如需更新已有伸缩组，请使用 `ModifyClusterAsGroupAttribute`。

## 前置条件

- 已完成[环境准备](../../环境准备.md)
- tccli 已配置凭据和地域

## 操作步骤

### Step 1: 安装 metrics-server（控制面）

HPA 依赖 metrics-server 提供 CPU/内存指标。若集群尚未安装，先执行安装。

#### 选择依据

- 新建 TKE 集群默认已安装 metrics-server
- 若不确定，先检查；缺少 metrics-server 时 HPA 无法获取 Pod 指标

```bash
tccli tke DescribeAddon --region <Region> --ClusterId CLUSTER_ID --AddonName metrics-server
```

输出示例：

```json
{
    "Addons": [
        {
            "AddonName": "metrics-server",
            "AddonVersion": "0.6.2",
            "Status": "Running",
            "Phase": "Running"
        }
    ]
}
```

若返回为空或 `Phase` 不为 `Running`，执行安装：

```bash
tccli tke InstallAddon --region <Region> --ClusterId CLUSTER_ID --AddonName metrics-server
```

输出示例：

```json
{
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

安装后轮询状态直到 `Phase` 为 `Running`：

```bash
tccli tke DescribeAddon --region <Region> --ClusterId CLUSTER_ID --AddonName metrics-server
```

输出示例：

```json
{
    "Addons": [
        {
            "AddonName": "metrics-server",
            "AddonVersion": "0.6.2",
            "Status": "Running",
            "Phase": "Running"
        }
    ]
}
```

### Step 2: 最小配置 — 部署示例应用 + 基于 CPU 的 HPA

#### 选择依据

- HPA 是最通用的水平伸缩方案，基于 CPU 的伸缩覆盖 80% 的无状态应用场景
- 最小配置：单指标（CPU）、`kubectl autoscale` 一键创建，适合快速验证

#### 部署示例应用

创建 `deployment-hpa-demo.yaml`：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: DEPLOYMENT_NAME
  namespace: NAMESPACE
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hpa-demo
  template:
    metadata:
      labels:
        app: hpa-demo
    spec:
      containers:
      - name: php-apache
        image: mirrors.tencent.com/tke/hpa-example:latest
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: "200m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "256Mi"
```

```bash
kubectl apply -f deployment-hpa-demo.yaml -n NAMESPACE
```

输出示例：

```
deployment.apps/DEPLOYMENT_NAME created
```

> **关键点**：HPA 依赖 `resources.requests` 计算使用率百分比。未设置 `requests` 的 Pod 无法被 HPA 管理。

#### 创建 HPA（最小配置）

```bash
kubectl autoscale deployment DEPLOYMENT_NAME --cpu-percent=50 --min=1 --max=10 -n NAMESPACE
```

输出示例：

```
horizontalpodautoscaler.autoscaling/DEPLOYMENT_NAME autoscaled
```

查看创建的 HPA YAML：

```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: DEPLOYMENT_NAME
  namespace: NAMESPACE
spec:
  maxReplicas: 10
  minReplicas: 1
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: DEPLOYMENT_NAME
  targetCPUUtilizationPercentage: 50
```

### Step 3: 增强配置 — 基于内存 + 自定义指标的 HPA

#### 选择依据

- 当应用对内存敏感（如 JVM 应用）时，需同时设置内存触发阈值
- 多指标 HPA 取所有指标中副本数最大的推荐值（即 `max` 策略），避免资源不足
- 适合已在最小配置基础上验证完毕、需要更精细伸缩控制的生产场景

#### 多指标 HPA YAML

创建 `hpa-multi-metrics.yaml`：

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: DEPLOYMENT_NAME
  namespace: NAMESPACE
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: DEPLOYMENT_NAME
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 70
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
```

```bash
kubectl apply -f hpa-multi-metrics.yaml -n NAMESPACE
```

输出示例：

```
horizontalpodautoscaler.autoscaling/DEPLOYMENT_NAME configured
```

#### 指标类型说明

| 指标类型 | 来源 | 典型场景 | 示例 |
|---------|------|---------|------|
| `Resource` | metrics-server | Pod CPU/内存使用率 | `averageUtilization: 50` |
| `Object` | Kubernetes 对象指标 | Ingress 请求数 | `ingress_requests` |
| `External` | 外部监控系统 | 消息队列积压 | RabbitMQ queue depth |
| `Pods` | 每个 Pod 的自定义指标 | 每个 Pod 的 RPS | `http_requests_per_second` |
| `ContainerResource` | 容器级资源 | 侧车容器内存 | sidecar memory usage |

> **行为配置 (`behavior`)**：`scaleDown.stabilizationWindowSeconds` 防止频繁缩容抖动；`scaleUp` 设置激进扩容策略应对突发流量。

### Step 4: KEDA 基于 Prometheus 指标弹性伸缩（增强配置）

#### 选择依据

- 当伸缩触发条件不限于 CPU/内存时（如根据队列长度、Prometheus 自定义指标），KEDA 提供更灵活的触发器体系
- KEDA 支持缩容至零（`minReplicaCount: 0`），适合低频调用或批处理场景
- 需提前部署 Prometheus 并确保其可采集应用指标

#### 安装 KEDA

```bash
helm repo add kedacore https://kedacore.github.io/charts
helm repo update
helm install keda kedacore/keda --namespace keda --create-namespace
```

输出示例：

```
NAME: keda
LAST DEPLOYED: ...
NAMESPACE: keda
STATUS: deployed
```

#### 创建 ScaledObject

创建 `scaledobject-prometheus.yaml`：

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: DEPLOYMENT_NAME
  namespace: NAMESPACE
spec:
  scaleTargetRef:
    name: DEPLOYMENT_NAME
  minReplicaCount: 0
  maxReplicaCount: 10
  cooldownPeriod: 300
  pollingInterval: 15
  triggers:
  - type: prometheus
    metadata:
      serverAddress: http://prometheus-server.monitoring.svc.cluster.local:9090
      metricName: http_requests_per_second
      threshold: "100"
      query: |
        sum(rate(http_requests_total{app="hpa-demo"}[2m]))
```

```bash
kubectl apply -f scaledobject-prometheus.yaml -n NAMESPACE
```

输出示例：

```
scaledobject.keda.sh/DEPLOYMENT_NAME created
```

> **注意**：KEDA 与原生 HPA 可共存。KEDA 创建 ScaledObject 时会自动管理底层 HPA 对象，不要同时手动创建同名 HPA。

### Step 5: 集群自动伸缩 CA — 控制面

#### 选择依据

- 当 Pod 因节点资源不足而 Pending 时，CA 自动扩容节点池
- 当节点利用率持续偏低时，CA 自动缩容节点池（前提是节点上的 Pod 可被重新调度）
- CA 是集群级能力，需提前创建节点池（伸缩组）

#### 配置伸缩组（--cli-input-json）

由于参数超过 4 个，使用 `--cli-input-json` 方式传参。创建 `cluster-asg.json`：

```json
{
    "ClusterId": "CLUSTER_ID",
    "AutoScalingGroupPara": {
        "AutoScalingGroupName": "tke-asg-demo",
        "MaxSize": 10,
        "MinSize": 0,
        "VpcId": "VPC_ID",
        "SubnetIds": [
            "subnet-xxxxxxxx"
        ],
        "InstanceTypes": [
            "S5.MEDIUM4",
            "S5.MEDIUM8"
        ],
        "ServiceSettings": {
            "ScalingMode": "CLASSIC_SCALING"
        }
    },
    "LaunchConfigurePara": {
        "InstanceName": "tke-node-",
        "SystemDisk": {
            "DiskType": "CLOUD_PREMIUM",
            "DiskSize": 50
        }
    },
    "InstanceAdvancedSettings": {
        "DesiredPodNumber": 0,
        "Labels": [
            {
                "Name": "node-type",
                "Value": "autoscaling"
            }
        ]
    }
}
```

```bash
tccli tke CreateClusterAsGroup --region <Region> --cli-input-json file://cluster-asg.json
```

输出示例：

```json
{
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
    "AutoScalingGroupId": "asg-xxxxxxxx"
}
```

#### 查看伸缩组状态

```bash
tccli tke DescribeClusterAsGroups --region <Region> --ClusterId CLUSTER_ID
```

输出示例：

```json
{
    "ClusterAsGroupSet": [
        {
            "AutoScalingGroupId": "asg-xxxxxxxx",
            "AutoScalingGroupName": "tke-asg-demo",
            "Status": "enabled",
            "MaxSize": 10,
            "MinSize": 0,
            "DesiredCapacity": 0
        }
    ],
    "TotalCount": 1
}
```

> **多个机型**：`InstanceTypes` 支持指定多种机型作为备选，当首选机型资源不足时自动切换到备选机型，提升扩容成功率。

### Step 6: 弹性伸缩策略选择决策

#### 选择依据

实际场景中通常组合使用多种伸缩机制。以下是决策矩阵：

| 场景 | 推荐方案 | 原因 |
|------|---------|------|
| 无状态 Web 服务，CPU/内存波动 | HPA（v2 多指标） | 标准方案，Kubernetes 内置，运维最简单 |
| 消息队列消费者，队列深度变化 | KEDA + Prometheus/Kafka trigger | 事件驱动，可缩至零，响应快 |
| 突发流量（秒杀/抢购） | KEDA cron scaling + HPA | Cron 预热 + 实时指标兜底 |
| 有状态服务，需增加 Pod 资源 | VPA（Recommender 模式） | 不改变副本数，仅调整 requests/limits |
| 定时扩容（每晚报表任务） | KEDA cron trigger | 按时间表精确控制 |
| Pod Pending 因节点资源不足 | Cluster Autoscaler (CA) | 自动扩缩节点，与 HPA/KEDA 协同 |
| 需要按 GPU 使用率伸缩 | HPA v2 + External/Pods 指标 | GPU 指标通过 DCGM Exporter 暴露 |
| 成本敏感，非工作时间缩减 | KEDA cron + CA 联动 | KEDA 缩 Pod，CA 缩节点 |

#### 组合策略示例

典型三层弹性伸缩体系：

```
KEDA（定时/事件） → 感知流量变化 → 调整 Pod 副本数
     ↓
HPA（指标兜底）  → 实时监控 Pod 使用率 → 精细调整副本数
     ↓
CA（资源感知）   → Pod Pending → 扩容节点池 → 新节点就绪
```

## 验证

### 控制面验证

**验证 metrics-server 运行状态**：

```bash
tccli tke DescribeAddon --region <Region> --ClusterId CLUSTER_ID --AddonName metrics-server
```

输出示例：

```json
{
    "Addons": [
        {
            "AddonName": "metrics-server",
            "Status": "Running",
            "Phase": "Running"
        }
    ]
}
```

**验证集群伸缩组**：

```bash
tccli tke DescribeClusterAsGroups --region <Region> --ClusterId CLUSTER_ID
```

输出示例：

```json
{
    "ClusterAsGroupSet": [
        {
            "AutoScalingGroupId": "asg-xxxxxxxx",
            "Status": "enabled"
        }
    ],
    "TotalCount": 1
}
```

### 数据面验证

**验证 HPA 状态**：

```bash
kubectl get hpa -n NAMESPACE
```

输出示例：

```
NAME              REFERENCE                    TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
DEPLOYMENT_NAME   Deployment/DEPLOYMENT_NAME   45%/50%         1         10        3          5m
```

> `TARGETS` 列格式为 `当前值/阈值`。若显示 `<unknown>/50%`，说明 metrics-server 尚未就绪。

**查看 Pod 资源使用**：

```bash
kubectl top pods -n NAMESPACE
```

输出示例：

```
NAME                               CPU(cores)   MEMORY(bytes)
DEPLOYMENT_NAME-xxxxxxxxxx-xxxxx   45m          96Mi
DEPLOYMENT_NAME-xxxxxxxxxx-yyyyy   68m          112Mi
DEPLOYMENT_NAME-xxxxxxxxxx-zzzzz   32m          84Mi
```

**验证 KEDA ScaledObject**：

```bash
kubectl get scaledobject -n NAMESPACE
```

输出示例：

```
NAME              SCALETARGETKIND      SCALETARGETNAME    TRIGGERS        AUTHENTICATION   READY   ACTIVE   FALLBACK   AGE
DEPLOYMENT_NAME   apps/v1.Deployment   DEPLOYMENT_NAME    prometheus                       True    True     False      5m
```

**验证节点伸缩**：

```bash
kubectl get nodes -l node-type=autoscaling
```

输出示例：

```
NAME                STATUS   ROLES    AGE   VERSION
tke-node-xxxxxxxx   Ready    <none>   2m    v1.28.3-tke.1
```

## 排障

| 症状 | 可能原因 | 排查方法 | 解决方案 |
|------|---------|---------|---------|
| HPA `TARGETS` 显示 `<unknown>` | metrics-server 未安装或未就绪 | `tccli tke DescribeAddon --ClusterId CLUSTER_ID --AddonName metrics-server` | `tccli tke InstallAddon` 安装 metrics-server |
| HPA 不触发扩容但 Pod CPU 已超阈值 | HPA `behavior.scaleUp` 策略配置过于保守 | `kubectl describe hpa DEPLOYMENT_NAME -n NAMESPACE` 查看事件 | 调整 `behavior.scaleUp` 策略，减小 `periodSeconds` |
| HPA 频繁扩缩容（抖动） | `scaleDown.stabilizationWindowSeconds` 过短 | `kubectl get hpa DEPLOYMENT_NAME -n NAMESPACE -w` 观察波动 | 增大 `stabilizationWindowSeconds` 到 300s+ |
| KEDA ScaledObject `READY` 为 `False` | Prometheus 地址不可达或认证失败 | `kubectl describe scaledobject DEPLOYMENT_NAME -n NAMESPACE` | 检查 Prometheus 地址、Namespace 网络策略 |
| KEDA 无法缩容到零 | `minReplicaCount` 未设为 0 | `kubectl get scaledobject DEPLOYMENT_NAME -n NAMESPACE -o yaml` | 设置 `minReplicaCount: 0` |
| CA 不扩容节点 | `ClusterAsGroup` 状态非 `enabled` 或资源不足 | `tccli tke DescribeClusterAsGroups` 检查 status 和 `InstanceTypes` | 开启伸缩组；添加备选机型 |
| Pod 长期 Pending 但 CA 已扩容 | `nodeSelector`/`affinity` 与新节点 Label 不匹配 | `kubectl describe pod <pending-pod> -n NAMESPACE` | 确保 `Labels` 配置在 `InstanceAdvancedSettings.Labels` 中 |
| `kubectl top` 报 `metrics not available yet` | metrics-server 启动后采集数据有延迟 | 等待 2-3 分钟 | 通常 metrics-server 就绪后 1-2 分钟内指标可用 |
| CA 不停缩容和扩容（循环） | Pod 资源请求与节点规格不匹配，导致频繁装箱调整 | `kubectl describe hpa` + CA 事件日志 | 调整 Pod `requests` 使装箱率合理；或增大缩容冷却时间 |
| HPA v2 多指标行为异常 | 多指标默认取 `max`，可能不符合预期 | 检查 HPA `status.conditions` | 若需 `min` 策略，使用 `metricSelector` 结合标签区分不同 HPA 对象 |

## 清理

> **警告**：删除 HPA 或 ScaledObject 后，应用的副本数将保持在最后伸缩的状态，不再自动扩缩容。若需恢复到初始副本数，请手动调整 Deployment 的 `replicas`。

**删除 HPA**：

```bash
kubectl delete hpa DEPLOYMENT_NAME -n NAMESPACE
```

输出示例：

```
horizontalpodautoscaler.autoscaling "DEPLOYMENT_NAME" deleted
```

**删除 KEDA ScaledObject**：

```bash
kubectl delete scaledobject DEPLOYMENT_NAME -n NAMESPACE
```

输出示例：

```
scaledobject.keda.sh "DEPLOYMENT_NAME" deleted
```

**删除示例 Deployment**：

```bash
kubectl delete deployment DEPLOYMENT_NAME -n NAMESPACE
```

输出示例：

```
deployment.apps "DEPLOYMENT_NAME" deleted
```

> **CA 节点清理与计费警告**：Cluster Autoscaler 创建的节点为计费资源（按量计费 CVM 实例）。仅删除伸缩组（通过 `DeleteClusterAsGroup`）不会自动退还已创建的节点。正确清理步骤：
> 1. 先删除伸缩组，阻止新增节点。
> 2. `kubectl drain` 并 `kubectl delete node` 逐个移除节点。
> 3. 在 CVM 控制台确认所有关联实例已退还。

## 控制台替代

[TKE 控制台](https://console.cloud.tencent.com/tke2)

## 下一步

- [HPA 使用手册](https://cloud.tencent.com/document/product/457/45636) — 原生 HPA 详细配置说明
- [KEDA 官方文档](https://keda.sh/docs/) — KEDA 全部触发器类型和配置参考
- [集群自动伸缩（CA）](https://cloud.tencent.com/document/product/457/32198) — TKE CA 完整操作指南
- [VPA 使用指南](https://cloud.tencent.com/document/product/457/45637) — 垂直伸缩适用场景与配置
- [metrics-server 安装说明](https://cloud.tencent.com/document/product/457/38371) — metrics-server 故障排查
