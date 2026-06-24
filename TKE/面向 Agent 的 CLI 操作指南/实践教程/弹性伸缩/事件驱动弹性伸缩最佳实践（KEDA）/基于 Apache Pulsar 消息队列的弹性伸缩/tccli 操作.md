# 基于 Apache Pulsar 消息队列的弹性伸缩（tccli）

> 对照官方：[基于 Apache Pulsar 消息队列的弹性伸缩](https://cloud.tencent.com/document/product/457/106149) · page_id `106149`

## 概述

通过 KEDA Pulsar Scaler 将 Apache Pulsar 消息队列的订阅积压量（message backlog）映射为弹性伸缩信号。当消费者跟不上生产者速率时，自动增加消费者 Pod 副本数以加速消息处理，积压消除后自动缩容。

> **kubectl 数据面不可达**：CAM拒绝公网端点(strategyId:240463971)，内网需VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

### 方案选择

Pulsar Scaler 与其他消息队列 Scaler 的对比：

| Scaler 类型 | 适用场景 | 核心指标 | 连接方式 |
|------------|---------|---------|---------|
| **Pulsar** | 已有 Pulsar 集群，基于订阅积压量伸缩 | `msgBacklog`（订阅积压消息数） | HTTP admin API + token 认证 |
| Kafka | Kafka 集群消费者组伸缩 | `lagThreshold`（消费延迟） | Bootstrap servers |
| RabbitMQ | RabbitMQ 队列长度触发伸缩 | `queueLength` | Management API |

**选择 Pulsar Scaler 的依据**：

- 业务使用 Apache Pulsar 作为消息中间件
- 消费者处理能力与消息生产速率不匹配，需要动态调整消费者副本数
- 已有 Pulsar 集群的管理 API 可访问（网络可达）

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

# 6. 确认 Pulsar 集群网络可达（如在同 VPC 内）
tccli vpc DescribeVpcs --region <Region> --Filters '[{"Name":"vpc-id","Values":["VPC_ID"]}]'
# expected: 返回 VPC 详情，确认与集群在同一 VPC 或已通过云联网/对等连接互通
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

Pulsar ScaledObject 中与 Pulsar 连接和订阅相关的核心参数：

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `adminURL` | String | 是 | Pulsar Admin API 地址，格式 `http://<host>:8080` 或 `https://<host>:8443` | 地址不可达 → ScaledObject `ACTIVE` 为 `False`，Events 显示连接超时 |
| `subscription` | String | 是 | Pulsar 消费者订阅名，如 `my-subscription`。需与消费者使用的订阅名一致 | 订阅名不匹配 → 查询不到积压数据，指标始终为 0 |
| `topic` | String | 是 | Pulsar Topic 完整路径，格式 `persistent://<tenant>/<namespace>/<topic>` | Topic 不存在或格式错误 → Scaler 返回错误 |
| `msgBacklog` | String | 是 | 触发扩容的积压消息数阈值，如 `"100"` | 阈值过低 → 频繁扩容；过高 → 消息积压严重 |
| `authModes` | String | 否 | `"tls"` 或 `"bearer"`。Pulsar 开启认证时必须配置 | 未配置认证 → 连接被 Pulsar 拒 |

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

### 步骤 2：部署 Pulsar 消费者应用（数据面）

部署一个 Pulsar 消费者应用，从指定的 Topic 和订阅中消费消息。

`pulsar-consumer.yaml`：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: DEPLOYMENT_NAME
  namespace: NAMESPACE
  labels:
    app: pulsar-consumer
spec:
  replicas: 1
  selector:
    matchLabels:
      app: pulsar-consumer
  template:
    metadata:
      labels:
        app: pulsar-consumer
    spec:
      containers:
        - name: consumer
          image: apachepulsar/pulsar:latest
          command:
            - /bin/bash
            - -c
            - |
              bin/pulsar-client consume \
                persistent://public/default/my-topic \
                -s my-subscription \
                -n 0
          env:
            - name: PULSAR_SERVICE_URL
              value: "pulsar://pulsar-proxy:6650"
          resources:
            requests:
              cpu: 200m
              memory: 256Mi
            limits:
              cpu: 500m
              memory: 512Mi
```

```bash
kubectl apply -f pulsar-consumer.yaml
# expected: deployment.apps/DEPLOYMENT_NAME created
```

**预期输出**：

```text
deployment.apps/pulsar-consumer created
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `DEPLOYMENT_NAME` | 消费者部署名称 | 长度 1-63 字符 | 自定义，如 `pulsar-consumer` |
| `NAMESPACE` | 命名空间 | 需已存在 | `kubectl get ns` 确认 |

### 步骤 3：创建 Pulsar 认证 Secret（数据面）

#### 选择依据

若 Pulsar 集群启用了认证（推荐），需将 admin token 存储为 Kubernetes Secret。Pulsar Admin API 使用 Bearer Token 认证方式。

```bash
kubectl create secret generic pulsar-auth \
    --namespace NAMESPACE \
    --from-literal=admin-token="PULSAR_ADMIN_TOKEN"
# expected: secret/pulsar-auth created
```

**预期输出**：

```text
secret/pulsar-auth created
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `PULSAR_ADMIN_TOKEN` | Pulsar Admin API token | 来自 Pulsar 集群配置 | Pulsar 管理员提供或从 `pulsar-admin` 获取 |

### 步骤 4：创建 TriggerAuthentication（数据面）

创建 TriggerAuthentication 引用上一步创建的 Secret。

`trigger-auth-pulsar.yaml`：

```yaml
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: keda-pulsar-auth
  namespace: NAMESPACE
spec:
  secretTargetRef:
    - parameter: bearerToken
      name: pulsar-auth
      key: admin-token
```

```bash
kubectl apply -f trigger-auth-pulsar.yaml
# expected: triggerauthentication.keda.sh/keda-pulsar-auth created
```

**预期输出**：

```text
triggerauthentication.keda.sh/keda-pulsar-auth created
```

### 步骤 5：创建 ScaledObject（数据面）

#### 选择依据

Pulsar Scaler 的关键决策：

- **`adminURL`**：Pulsar Admin API 地址。需确保从 KEDA operator 所在网络可达该地址。若 Pulsar 部署在同一 K8s 集群内，可使用 Service 名称：`http://pulsar-broker.pulsar:8080`。
- **`subscription`**：消费者订阅名，必须与实际消费者使用的订阅名一致。不同的订阅名对应不同的积压量。
- **`topic`**：Pulsar Topic 的完整路径，格式为 `persistent://<tenant>/<namespace>/<topic-name>`。
- **`msgBacklog`**：积压阈值。当订阅积压消息数超过此值时触发扩容。设为 `"100"` 意味着积压超过 100 条消息时开始增加消费者副本。

`scaledobject-pulsar.yaml`：

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: pulsar-scaledobject
  namespace: NAMESPACE
spec:
  scaleTargetRef:
    name: DEPLOYMENT_NAME
  minReplicaCount: 1
  maxReplicaCount: 10
  cooldownPeriod: 120
  pollingInterval: 15
  triggers:
    - type: pulsar
      metadata:
        adminURL: http://pulsar-broker.pulsar:8080
        subscription: my-subscription
        topic: persistent://public/default/my-topic
        msgBacklog: "100"
      authenticationRef:
        name: keda-pulsar-auth
```

```bash
kubectl apply -f scaledobject-pulsar.yaml
# expected: scaledobject.keda.sh/pulsar-scaledobject created
```

**预期输出**：

```text
scaledobject.keda.sh/pulsar-scaledobject created
```

| 参数字段 | 说明 | 推荐值 | 调整建议 |
|---------|------|--------|---------|
| `minReplicaCount` | 最小副本数 | `1` | 生产环境建议 ≥ 2 保证高可用 |
| `maxReplicaCount` | 最大副本数 | `10` | 根据 Pulsar 分区数和处理能力上限设定 |
| `cooldownPeriod` | 缩容冷却时间（秒） | `120` | 调整取决于消息处理时延 |
| `pollingInterval` | Pull 拉取间隔（秒） | `15` | 需权衡时效性与 Pulsar Admin API 负载 |
| `msgBacklog` | 积压阈值 | `"100"` | 根据单条消息处理时间和消费者吞吐量设定 |

### 步骤 6：生产测试消息（数据面）

通过 Pulsar 生产者发送消息，模拟消息积压场景以触发扩容。

```bash
# 在生产端执行——向 Topic 发送大量消息
kubectl run -it --rm pulsar-producer \
    --image=apachepulsar/pulsar:latest -n NAMESPACE \
    --restart=Never \
    -- /bin/bash -c "
  for i in \$(seq 1 500); do
    bin/pulsar-client produce persistent://public/default/my-topic -m \"message-\$i\";
  done
"
# expected: 500 条消息成功发送
```

**预期输出**：

```text
... (500 条 produce 成功日志)
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
# 1. 确认消费者 Pod 正常运行
kubectl get pods -n NAMESPACE -l app=pulsar-consumer
# expected: Pod Running，至少 1 个副本
```

**预期输出**：

```text
NAME                               READY   STATUS    RESTARTS   AGE
pulsar-consumer-6d4f7b8c9-x2k5m   1/1     Running   0          5m
```

```bash
# 2. 确认 ScaledObject 就绪
kubectl get scaledobject -n NAMESPACE
# expected: READY True
```

**预期输出**：

```text
NAME                   SCALETARGETKIND      SCALETARGETNAME    MIN   MAX   TRIGGERS   AUTHENTICATION      READY   ACTIVE   AGE
pulsar-scaledobject    apps/v1.Deployment   pulsar-consumer    1     10    pulsar     keda-pulsar-auth    True    True     5m
```

```bash
# 3. 确认 HPA 已自动生成
kubectl get hpa -n NAMESPACE
# expected: 存在 keda-hpa- 前缀的 HPA
```

```text
NAME  STATUS  AGE
...
```

```bash
# 4. 消息积压时观察 Pod 扩容
kubectl get pods -n NAMESPACE -l app=pulsar-consumer -w
# expected: Pod 数量随积压量增加
```

```text
NAME  STATUS  AGE
...
```

| 验证维度 | 命令 | 预期 |
|---------|------|------|
| 消费者 Pod Running | `kubectl get pods -n NAMESPACE -l app=pulsar-consumer` | 至少 1 个 Pod，状态 Running |
| ScaledObject 就绪 | `kubectl get scaledobject -n NAMESPACE` | `READY: True` |
| HPA 自动生成 | `kubectl get hpa -n NAMESPACE` | 存在 `keda-hpa-pulsar-scaledobject` |
| 消息积压触发扩容 | 生产消息后观察 `kubectl get pods -n NAMESPACE -w` | 副本数从 1 增加 |
| 积压消除后缩容 | 消费者处理完消息后等待 `cooldownPeriod` | 副本数恢复至 `minReplicaCount` |

## 清理

> **警告**：删除 ScaledObject 会**同时删除关联的 HPA**，Deployment 将保持当前副本数不再伸缩。Pulsar 认证 Secret 含敏感信息，确认不再需要后及时删除。

### 数据面清理（先执行）

```bash
# 1. 删除测试生产者 Pod（如仍在运行）
kubectl delete pod pulsar-producer -n NAMESPACE --ignore-not-found
# expected: pod "pulsar-producer" deleted

# 2. 删除 ScaledObject
kubectl delete scaledobject pulsar-scaledobject -n NAMESPACE
# expected: scaledobject.keda.sh "pulsar-scaledobject" deleted

# 3. 删除 TriggerAuthentication
kubectl delete triggerauthentication keda-pulsar-auth -n NAMESPACE
# expected: triggerauthentication.keda.sh "keda-pulsar-auth" deleted

# 4. 删除 Pulsar 认证 Secret
kubectl delete secret pulsar-auth -n NAMESPACE
# expected: secret "pulsar-auth" deleted

# 5. 删除消费者部署
kubectl delete -f pulsar-consumer.yaml
# expected: deployment.apps "pulsar-consumer" deleted
```

### 验证数据面清理

```bash
kubectl get deployment,scaledobject,secret -n NAMESPACE -l app=pulsar-consumer
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
| `kubectl create secret` 返回 `Error from server (AlreadyExists)` | `kubectl get secret pulsar-auth -n NAMESPACE` | 同名 Secret 已存在 | 删除旧 Secret 后重建：`kubectl delete secret pulsar-auth -n NAMESPACE`；或使用不同 Secret 名称 |
| `kubectl apply` ScaledObject 返回验证错误 | `kubectl apply --dry-run=client -f scaledobject-pulsar.yaml` 验证 YAML 语法 | `topic` 格式不符合 `persistent://<tenant>/<namespace>/<topic>` 规范 | 修正 topic 格式，确保格式为 `persistent://<tenant>/<namespace>/<topic>` |

### 操作提交成功但异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| ScaledObject `READY` 为 `False` | `kubectl describe scaledobject pulsar-scaledobject -n NAMESPACE` 查看 Events | Pulsar admin API 不可达、认证失败或 topic 不存在 | 检查 Events 中的错误；验证 `adminURL` 可达：`kubectl run -it --rm debug --image=busybox --restart=Never -n NAMESPACE -- wget -q -O- http://pulsar-broker.pulsar:8080/admin/v2/persistent/public/default/my-topic/stats`；检查 Secret 中的 token 是否正确 |
| ScaledObject `ACTIVE` 始终为 `False`（无消息时属正常） | `kubectl describe scaledobject pulsar-scaledobject -n NAMESPACE` | 订阅 `my-subscription` 下无积压消息，积压量 = 0 | 向 Topic 生产消息进行验证；确认消费者订阅名与 ScaledObject 中 `subscription` 一致 |
| 消息积压超过阈值但未扩容 | `kubectl get hpa -n NAMESPACE` 查看 HPA TARGETS 列；`kubectl describe hpa keda-hpa-pulsar-scaledobject -n NAMESPACE` | HPA 冷启动期未过或 KEDA operator 拉取间隔未到 | 等待至少 1 个 `pollingInterval`（15s）；确认消费者的 `subscription` 与 ScaledObject 中 `subscription` 字段完全一致（包括大小写） |
| Pod 扩容到 `maxReplicaCount` 仍无法消费完积压 | `kubectl top pods -n NAMESPACE -l app=pulsar-consumer` 检查资源使用 | 单 Pod 处理能力不足，或 Pulsar 分区数限制了并行消费 | 增大 `maxReplicaCount`（不超过 Pulsar Topic 分区数）；优化消费者逻辑或增大单 Pod 资源配额；考虑拆分 Topic |
| 缩容后消息重新积压 | 观察 `kubectl get hpa -n NAMESPACE -w` 的 TARGETS 变化 | `cooldownPeriod` 过短，缩容过早，剩余 Pod 无法处理持续流入的消息 | 增大 `cooldownPeriod`（如 300s），确保缩容后剩余副本能稳定消费 |

## 下一步

- [认识 KEDA](https://cloud.tencent.com/document/product/457/106143) -- page_id `106143`
- [在 TKE 上部署 KEDA](https://cloud.tencent.com/document/product/457/106144) -- page_id `106144`
- [基于 Prometheus 自定义指标的弹性伸缩](https://cloud.tencent.com/document/product/457/107998) -- page_id `107998`
- [多级服务同步水平伸缩（Workload 触发器）](https://cloud.tencent.com/document/product/457/106150) -- page_id `106150`
- [Pulsar 消息队列](https://cloud.tencent.com/document/product/1179) -- TDMQ Pulsar 版文档

## 控制台替代

[TKE 控制台 → 集群 → 工作负载](https://console.cloud.tencent.com/tke2/cluster)：在 KEDA 控制台中可视化创建和管理 Pulsar ScaledObject；[Pulsar 控制台](https://console.cloud.tencent.com/tdmq/pulsar-cluster)：管理 Pulsar 集群、Topic 和订阅。
