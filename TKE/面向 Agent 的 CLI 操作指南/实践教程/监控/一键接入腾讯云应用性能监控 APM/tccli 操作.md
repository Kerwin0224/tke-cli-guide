# 一键接入腾讯云应用性能监控 APM（tccli）

> 对照官方：[一键接入腾讯云应用性能监控 APM](https://cloud.tencent.com/document/product/457/107997) · page_id `107997`

## 概述

通过 TKE 组件管理功能一键安装腾讯云 APM（应用性能监控）Agent，实现对 TKE 集群中业务应用的分布式链路追踪、性能分析和故障定位。APM 基于 OpenTelemetry 标准协议，支持 Java、Go、Python、Node.js 等多语言自动/手动探针注入，无需修改业务代码即可采集 Trace、Metrics 和 Logs 数据。APM Agent 以 TKE Addon 形式安装在集群中，以 DaemonSet 或 Sidecar 模式运行，将链路数据上报至 APM 后端进行分析和展示。

> **kubectl 数据面不可达**：CAM拒绝公网端点(strategyId:240463971)，内网需VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

## 前置条件

### 环境检查

```bash
tccli --version
# expected: tccli version >= 1.0.0

tccli configure list
# expected: secretId 与 secretKey 已配置，默认 region 为 ap-guangzhou
```

验证 CAM 权限——以下为安装和配置 APM 所需全部 CAM Action：

```bash
tccli apm DescribeApmInstances \
    --region <Region>
# expected: 返回 APM 实例列表（可能为空），所需 CAM Action: apm:DescribeApmInstances

tccli apm DescribeApmAgent \
    --region <Region> \
    --InstanceId APM_INSTANCE_ID
# expected: 返回 Agent 配置信息，所需 CAM Action: apm:DescribeApmAgent

tccli tke DescribeClusters \
    --region <Region> \
    --ClusterIds '["CLUSTER_ID"]'
# expected: 返回集群详情，所需 CAM Action: tke:DescribeClusters

tccli tke DescribeAddon \
    --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName apm-agent
# expected: 返回 Addon 状态（未安装时 AddonStatus 为空），所需 CAM Action: tke:DescribeAddon
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

| 参数 | 说明 | 示例值 |
|------|------|--------|
| CLUSTER_ID | TKE 托管集群 ID | cls-xxxxxxxx |
| REGION | 腾讯云地域 | ap-guangzhou |
| APM_INSTANCE_ID | APM 实例 ID | apm-xxxxxxxx |
| NAMESPACE | Kubernetes 命名空间 | default |
| PROM_INSTANCE_ID | Prometheus 实例 ID（如 APM 关联 Prometheus） | prom-xxxxxxxx |

```bash
tccli tke DescribeClusters \
    --region <Region> \
    --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterStatus 为 "Running"，ClusterType 为 "MANAGED_CLUSTER"，ClusterVersion >= 1.30.0

tccli apm DescribeApmInstances \
    --region <Region> \
    --InstanceIds '["APM_INSTANCE_ID"]'
# expected: Instances 列表中 Status 为 "Running"，Region 与 TKE 集群一致
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

| 控制台操作 | tccli/kubectl 命令 | 幂等 |
|-----------|-------------------|:--:|
| 创建 APM 实例 | `tccli apm CreateApmInstance --region <Region> --cli-input-json file://create-apm.json` | 否 |
| 查询 APM 实例列表 | `tccli apm DescribeApmInstances --region <Region>` | 是 |
| 安装 APM Agent（集群组件） | `tccli tke InstallAddon --region <Region> --cli-input-json file://install-apm-agent.json` | 否 |
| 查看 APM Agent 安装状态 | `tccli tke DescribeAddon --region <Region> --ClusterId CLUSTER_ID --AddonName apm-agent` | 是 |
| 查看 APM Agent 配置 | `tccli apm DescribeApmAgent --region <Region> --InstanceId APM_INSTANCE_ID` | 是 |
| 应用注入 OpenTelemetry 探针 | `kubectl apply -f deployment.yaml`（需 VPN/IOA） | 是 |
| 查看链路追踪数据 | APM 控制台（内网/VPN） | 是 |

## 操作步骤

### 步骤 1：创建 APM 实例（如尚未创建）

#### 选择依据

- **APM vs 自建 Tracing（Jaeger/Zipkin）**：APM 免运维，自动聚合 Trace 数据并提供调用链拓扑、耗时分析、异常检测能力，与 TKE 共用腾讯云体系
- **地域选择**：APM 实例需与 TKE 集群同一地域，确保探针上报链路延迟最小
- **OpenTelemetry 兼容性**：APM 后端兼容 OpenTelemetry 协议（OTLP），支持 Java、Go、Python、Node.js、.NET 等语言 SDK
- **数据采样**：APM 支持固定比例采样、自适应采样等策略，平衡性能开销与数据完整性

#### 最小创建

```bash
cat > create-apm.json <<'EOF'
{
    "InstanceName": "tke-apm",
    "Description": "TKE 集群应用性能监控",
    "Tags": [
        {
            "Key": "env",
            "Value": "production"
        }
    ]
}
EOF
tccli apm CreateApmInstance \
    --region <Region> \
    --cli-input-json file://create-apm.json
# expected: 返回 InstanceId（格式 apm-xxxxxxxx），RequestId 非空
```

记录返回的 `InstanceId`，后续步骤使用 `APM_INSTANCE_ID` 代替。

#### 增强配置

如需配置数据保存时长和 Trace 采样率：

```bash
cat > create-apm-enhanced.json <<'EOF'
{
    "InstanceName": "tke-apm",
    "Description": "TKE 生产环境全链路追踪",
    "TraceDuration": 15,
    "SpanCountPerTrace": 500,
    "Tags": [
        {
            "Key": "env",
            "Value": "production"
        },
        {
            "Key": "team",
            "Value": "platform"
        }
    ]
}
EOF
tccli apm CreateApmInstance \
    --region <Region> \
    --cli-input-json file://create-apm-enhanced.json
# expected: 返回 InstanceId
```

### 步骤 2：在 TKE 集群中安装 APM Agent 组件

APM Agent 作为 TKE Addon 安装，负责接收应用上报的 OpenTelemetry 数据并转发至 APM 后端：

```bash
cat > install-apm-agent.json <<'EOF'
{
    "ClusterId": "CLUSTER_ID",
    "AddonName": "apm-agent",
    "AddonVersion": "ADDON_VERSION",
    "RawValues": "{\"apmInstanceId\":\"APM_INSTANCE_ID\",\"region\":\"REGION\",\"samplingRate\":\"0.1\"}"
}
EOF
tccli tke InstallAddon \
    --region <Region> \
    --cli-input-json file://install-apm-agent.json
# expected: exit 0，RequestId 非空
```

| 参数 | 说明 | 示例值 |
|------|------|--------|
| ADDON_VERSION | APM Agent Addon 版本号 | 1.2.0 |
| APM_INSTANCE_ID | APM 实例 ID | apm-xxxxxxxx |

### 步骤 3：查询 APM Agent 安装状态

等待 Addon 安装完成：

```bash
tccli tke DescribeAddon \
    --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName apm-agent
# expected: AddonStatus 为 "Running"，AddonVersion 与安装时指定版本一致
```

预期输出：

```json
{
    "Addons": [
        {
            "AddonName": "apm-agent",
            "AddonVersion": "ADDON_VERSION",
            "AddonStatus": "Running",
            "RawValues": "{\"apmInstanceId\":\"APM_INSTANCE_ID\",\"region\":\"REGION\"}"
        }
    ]
}
```

### 步骤 4：为业务应用注入 OpenTelemetry 探针

#### 最小创建

以 Java 应用为例，通过 Pod 注解实现自动探针注入（Java Agent 方式）：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: traced-app
  namespace: NAMESPACE
  labels:
    app: traced-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: traced-app
  template:
    metadata:
      labels:
        app: traced-app
      annotations:
        instrumentation.opentelemetry.io/inject-java: "true"
    spec:
      containers:
        - name: app
          image: APP_IMAGE
          ports:
            - containerPort: 8080
              name: http
              protocol: TCP
          env:
            - name: OTEL_EXPORTER_OTLP_ENDPOINT
              value: "http://apm-agent.NAMESPACE:4317"
            - name: OTEL_SERVICE_NAME
              value: "traced-app"
            - name: OTEL_RESOURCE_ATTRIBUTES
              value: "service.namespace=NAMESPACE,service.instance.id=$(POD_NAME)"
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
          resources:
            requests:
              memory: "512Mi"
              cpu: "250m"
            limits:
              memory: "1Gi"
              cpu: "500m"
```

| 参数 | 说明 | 示例值 |
|------|------|--------|
| APP_IMAGE | 业务应用容器镜像 | ccr.ccs.tencentyun.com/NAMESPACE/java-app:v1.0.0 |
| NAMESPACE | Kubernetes 命名空间 | default |

```bash
kubectl apply -f deployment-traced.yaml
# expected: deployment.apps/traced-app created
```

#### 增强配置

对于 Go、Python 等不支持 Java Agent 的语言，使用 OpenTelemetry SDK 手动埋点，并通过环境变量配置 OTLP Exporter：

```yaml
spec:
  containers:
    - name: app
      image: APP_IMAGE
      env:
        - name: OTEL_EXPORTER_OTLP_ENDPOINT
          value: "http://apm-agent.NAMESPACE:4317"
        - name: OTEL_EXPORTER_OTLP_PROTOCOL
          value: "grpc"
        - name: OTEL_TRACES_SAMPLER
          value: "parentbased_traceidratio"
        - name: OTEL_TRACES_SAMPLER_ARG
          value: "0.1"
        - name: OTEL_PROPAGATORS
          value: "tracecontext,baggage,b3"
```

### 步骤 5：查询 APM Agent 配置和数据上报状态

```bash
tccli apm DescribeApmAgent \
    --region <Region> \
    --InstanceId APM_INSTANCE_ID
# expected: 返回 Agent 配置信息，Status 为 "Running"，接入 Token 非空
```

## 验证

### 控制面（tccli）

```bash
tccli tke DescribeAddon \
    --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName apm-agent
# expected: AddonStatus 为 "Running"

tccli apm DescribeApmInstances \
    --region <Region> \
    --InstanceIds '["APM_INSTANCE_ID"]'
# expected: Instances[0].Status 为 "Running"

tccli apm DescribeApmAgent \
    --region <Region> \
    --InstanceId APM_INSTANCE_ID
# expected: Agent 配置正常，数据上报端点可访问
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

### 数据面（需 VPN/IOA）

```bash
kubectl get pods -n NAMESPACE \
    -l app=traced-app
# expected: 2 个 Pod，STATUS 为 Running，READY 1/1

kubectl describe pod -n NAMESPACE \
    -l app=traced-app | grep "opentelemetry"
# expected: 输出包含 OpenTelemetry 探针注入相关信息

kubectl logs -n NAMESPACE \
    -l app=traced-app \
    --tail=10 | grep -i "otel"
# expected: 日志中包含 OpenTelemetry 初始化信息，无 ERROR

kubectl get pods -n tke-apm \
    -l app=apm-agent
# expected: APM Agent DaemonSet Pod 状态均为 Running
```

```text
NAME  STATUS  AGE
...
```

通过 APM 控制台验证链路数据（需内网/VPN）：

```bash
echo "请在 APM 控制台确认链路数据已上报: https://console.cloud.tencent.com/apm"
# 浏览器访问 APM 控制台 -> 应用列表 -> traced-app -> 调用链查询
```

## 清理

在执行清理前注意以下事项：
- **计费警告**：`APM_INSTANCE_ID` 为按量计费资源（按 Span 数量计费），如不再使用请务必销毁实例以停止计费
- **数据丢失警告**：销毁 APM 实例将永久删除所有历史 Trace 数据和调用链分析结果，不可恢复
- **Addon 卸载影响**：卸载 APM Agent Addon 后集群中所有应用将停止上报链路数据，但不影响业务 Pod 运行
- **应用侧适配**：移除应用的 OpenTelemetry 环境变量和 Pod 注解，避免启动时报 OTEL Exporter 连接错误

### 数据面（需 VPN/IOA）

步骤 1：清理业务应用的 OpenTelemetry 配置（移除 Pod 注解和环境变量）：

```bash
kubectl delete deployment traced-app \
    -n NAMESPACE
# expected: deployment.apps "traced-app" deleted
```

步骤 2：验证 APM Agent 相关 Pod 状态（卸载 Addon 前快照）：

```bash
kubectl get pods -n tke-apm
# expected: APM Agent DaemonSet Pod 列表
```

```text
NAME  STATUS  AGE
...
```

### 控制面（tccli）

步骤 1：卸载 APM Agent Addon：

```bash
tccli tke DeleteAddon \
    --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName apm-agent
# expected: exit 0，RequestId 非空
```

步骤 2：确认 Addon 已卸载：

```bash
tccli tke DescribeAddon \
    --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName apm-agent
# expected: Addons 为空数组，或 AddonStatus 为 "Uninstalling"/不存在
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

步骤 3：销毁 APM 实例（需二次确认，不可逆）：

```bash
tccli apm DeleteApmInstance \
    --region <Region> \
    --InstanceId APM_INSTANCE_ID
# expected: exit 0，RequestId 非空
```

步骤 4：验证 APM 实例已销毁：

```bash
tccli apm DescribeApmInstances \
    --region <Region> \
    --InstanceIds '["APM_INSTANCE_ID"]'
# expected: TotalCount 为 0，或 Status 为 "Terminating"
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| APM Agent Addon 安装失败 | `tccli tke DescribeAddon --region <Region> --ClusterId CLUSTER_ID --AddonName apm-agent` | 集群版本不兼容或资源不足 | 确认 K8s 版本 >= 1.20；检查集群节点是否有足够资源调度 DaemonSet Pod；查看 `kubectl describe addon apm-agent` 详情 |
| 应用无链路数据上报 | `kubectl logs -n NAMESPACE -l app=APP_NAME \| grep -i otel` | OTLP Exporter 端点到 APM Agent 不通或环境变量配置错误 | 确认 `OTEL_EXPORTER_OTLP_ENDPOINT` 为 `http://apm-agent.NAMESPACE:4317`（grpc 端口为 4317）；`kubectl exec POD_NAME -n NAMESPACE -- nc -zv apm-agent.NAMESPACE 4317` 测试连通性 |
| OpenTelemetry 探针未自动注入 | `kubectl get pods -n NAMESPACE POD_NAME -o yaml \| grep -i "instrumentation\|otel"` | Pod 注解缺失或 Mutating Webhook 未生效 | 确认 Pod template 包含 `instrumentation.opentelemetry.io/inject-<language>: "true"` 注解；检查 APM Agent MutatingWebhookConfiguration 是否就绪：`kubectl get mutatingwebhookconfigurations \| grep apm` |
| 链路数据采样率为 0（无数据） | `kubectl exec deploy/traced-app -n NAMESPACE -- env \| grep OTEL_TRACES` | 采样策略配置导致所有 Trace 被丢弃 | 设置 `OTEL_TRACES_SAMPLER=parentbased_always_on`（调试）或 `parentbased_traceidratio` + `OTEL_TRACES_SAMPLER_ARG=0.1`（生产 10% 采样）；注意采样率变量名在 OTel SDK 1.x 中为 `OTEL_TRACES_SAMPLER_ARG` 非 `OTEL_TRACES_SAMPLING_RATE` |

## 下一步

- [腾讯云 Prometheus 一键关联监控容器服务](../腾讯云%20Prometheus%20一键关联监控容器服务/tccli%20操作.md) -- page_id `90923`
- [JVM 接入](../JVM%20接入/tccli%20操作.md) -- page_id `112791`
- [MySQL Exporter 接入](../MySQL%20Exporter%20接入/tccli%20操作.md) -- page_id `112792`

## 控制台替代

[APM 控制台](https://console.cloud.tencent.com/apm) -> 应用列表 -> 接入应用 -> 选择 TKE 集群 -> 一键接入。
