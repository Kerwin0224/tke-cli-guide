# JVM 接入（tccli）

> 对照官方：[JVM 接入](https://cloud.tencent.com/document/product/457/112791) · page_id `112791`

## 概述

通过 JMX Exporter 将 Java 应用 JVM 指标（堆内存、GC、线程、类加载等）接入腾讯云 Prometheus 监控服务（TMP），在 TKE 集群中实现对 JVM 应用的可观测性。JMX Exporter 以 Java Agent 方式运行，无侵入性，通过 HTTP 端点（默认 9404）暴露 Prometheus 格式的指标数据，配合 ServiceMonitor CRD 实现自动发现和采集。适用于 Spring Boot、Tomcat 等主流 Java 框架。

> **kubectl 数据面不可达**：CAM拒绝公网端点(strategyId:240463971)，内网需VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

## 前置条件

### 环境检查

```bash
tccli --version
# expected: tccli version >= 1.0.0

tccli configure list
# expected: secretId 与 secretKey 已配置，默认 region 为 ap-guangzhou
```

验证 CAM 权限——执行以下 Describe* 只读接口，确认无权限错误：

```bash
tccli monitor DescribePrometheusInstances \
    --region <Region>
# expected: 返回 Prometheus 实例列表，所需 CAM Action: monitor:DescribePrometheusInstances

tccli tke DescribeClusters \
    --region <Region> \
    --ClusterIds '["CLUSTER_ID"]'
# expected: 返回集群详情且 Status 为 Running，所需 CAM Action: tke:DescribeClusters
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
| PROM_INSTANCE_ID | Prometheus 监控实例 ID | prom-xxxxxxxx |
| NAMESPACE | Kubernetes 命名空间 | default |

```bash
tccli monitor DescribePrometheusInstances \
    --region <Region> \
    --InstanceIds '["PROM_INSTANCE_ID"]'
# expected: InstanceStatus 为 "Running"，TotalCount >= 1

tccli tke DescribeClusters \
    --region <Region> \
    --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterStatus 为 "Running"，ClusterType 为 "MANAGED_CLUSTER"，ClusterVersion >= 1.30.0
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli/kubectl 命令 | 幂等 |
|-----------|-------------------|:--:|
| 确认 Prometheus 实例可用 | `tccli monitor DescribePrometheusInstances --region <Region>` | 是 |
| 部署 JMX Exporter 应用 | `kubectl apply -f java-app.yaml`（需 VPN/IOA） | 是 |
| 创建 ServiceMonitor | `kubectl apply -f servicemonitor.yaml`（需 VPN/IOA） | 是 |
| 验证 JVM 指标采集 | `kubectl port-forward svc/java-app 9404:9404`（需 VPN/IOA） | 是 |
| 查看 Grafana JVM 大盘 | Prometheus 控制台 → Grafana（需内网/VPN） | 是 |

## 操作步骤

### 步骤 1：准备 JMX Exporter 配置文件

#### 选择依据

- **JMX Exporter** 以 Java Agent 方式运行，对应用代码零侵入，仅需添加 `-javaagent` 启动参数
- **暴露端口**：默认 9404（可通过配置修改）
- **关键指标**：`jvm_memory_bytes_used`（堆内存）、`jvm_gc_collection_seconds_sum`（GC 耗时）、`jvm_threads_current`（活跃线程数）、`jvm_classes_loaded`（已加载类数）
- **适用场景**：所有运行在 JVM 上的应用（Spring Boot、Tomcat、Jetty 等），无需修改业务代码

#### 最小创建

创建 JMX Exporter 配置 ConfigMap，定义需要采集的 JVM 指标规则：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: jmx-exporter-config
  namespace: NAMESPACE
data:
  config.yaml: |
    startDelaySeconds: 0
    hostPort: 0.0.0.0:9404
    rules:
    - pattern: "java.lang<type=Memory><>(HeapMemoryUsage|NonHeapMemoryUsage)"
      name: jvm_memory_$2_bytes
      type: GAUGE
    - pattern: "java.lang<type=GarbageCollector, name=(.*)><>(CollectionCount|CollectionTime)"
      name: jvm_gc_$2_seconds
      type: COUNTER
    - pattern: "java.lang<type=Threading><>(ThreadCount|DaemonThreadCount)"
      name: jvm_threads_$2
      type: GAUGE
    - pattern: "java.lang<type=ClassLoading><>(LoadedClassCount|UnloadedClassCount)"
      name: jvm_classes_$2
      type: COUNTER
```

```bash
kubectl apply -f jmx-exporter-configmap.yaml
# expected: configmap/jmx-exporter-config created
```

#### 增强配置

在 Deployment 中集成 JMX Exporter Java Agent，并添加 Prometheus 注解实现自动发现：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: java-app
  namespace: NAMESPACE
  labels:
    app: java-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: java-app
  template:
    metadata:
      labels:
        app: java-app
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9404"
        prometheus.io/path: "/metrics"
    spec:
      containers:
        - name: app
          image: JAVA_APP_IMAGE
          ports:
            - containerPort: 9404
              name: metrics
              protocol: TCP
          env:
            - name: JAVA_TOOL_OPTIONS
              value: "-javaagent:/opt/jmx_exporter/jmx_prometheus_javaagent.jar=9404:/etc/jmx-exporter/config.yaml"
          volumeMounts:
            - name: jmx-config
              mountPath: /etc/jmx-exporter
              readOnly: true
          resources:
            requests:
              memory: "512Mi"
              cpu: "250m"
            limits:
              memory: "1Gi"
              cpu: "500m"
      volumes:
        - name: jmx-config
          configMap:
            name: jmx-exporter-config
---
apiVersion: v1
kind: Service
metadata:
  name: java-app
  namespace: NAMESPACE
  labels:
    app: java-app
spec:
  selector:
    app: java-app
  ports:
    - name: metrics
      port: 9404
      targetPort: 9404
      protocol: TCP
    - name: http
      port: 8080
      targetPort: 8080
      protocol: TCP
  type: ClusterIP
```

| 参数 | 说明 | 示例值 |
|------|------|--------|
| JAVA_APP_IMAGE | Java 应用容器镜像（需内置 JMX Exporter JAR） | ccr.ccs.tencentyun.com/NAMESPACE/java-app:v1.0.0 |
| NAMESPACE | Kubernetes 命名空间 | default |

```bash
kubectl apply -f java-app.yaml
# expected: deployment.apps/java-app created
# expected: service/java-app created
```

### 步骤 2：创建 ServiceMonitor 实现自动发现

ServiceMonitor 告知 Prometheus Operator 从哪些 Service 端点抓取指标：

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: java-app-monitor
  namespace: NAMESPACE
  labels:
    release: prometheus
spec:
  selector:
    matchLabels:
      app: java-app
  namespaceSelector:
    matchNames:
      - NAMESPACE
  endpoints:
    - port: metrics
      interval: 30s
      scrapeTimeout: 10s
      path: /metrics
```

```bash
kubectl apply -f servicemonitor.yaml
# expected: servicemonitor.monitoring.coreos.com/java-app-monitor created
```

### 步骤 3：验证 JVM 指标暴露

```bash
kubectl port-forward svc/java-app 9404:9404 \
    -n NAMESPACE &
# expected: Forwarding from 127.0.0.1:9404 -> 9404

curl -s http://localhost:9404/metrics | grep "jvm_memory_bytes_used"
# expected:
# jvm_memory_HeapMemoryUsage_bytes{area="heap"} 2.68435456E8
# jvm_memory_NonHeapMemoryUsage_bytes{area="nonheap"} 1.18226944E8

curl -s http://localhost:9404/metrics | grep "jvm_threads_current"
# expected:
# jvm_threads_ThreadCount 42.0

kill %1
```

## 验证

### 控制面（tccli）

```bash
tccli monitor DescribePrometheusInstances \
    --region <Region> \
    --InstanceIds '["PROM_INSTANCE_ID"]'
# expected: InstanceStatus 为 "Running"，GrafanaURL 非空

tccli monitor DescribePrometheusTargets \
    --region <Region> \
    --InstanceId PROM_INSTANCE_ID
# expected: Targets 列表中包含 java-app-monitor 相关 target，Health 为 "up"
```

### 数据面（需 VPN/IOA）

```bash
kubectl get pods -n NAMESPACE \
    -l app=java-app
# expected: 2 个 Pod，STATUS 均为 Running，READY 1/1

kubectl get servicemonitor java-app-monitor \
    -n NAMESPACE
# expected: NAME java-app-monitor，AGE 显示创建时长

kubectl logs -n NAMESPACE \
    -l app=java-app \
    --tail=5
# expected: 应用启动日志，无异常堆栈
```

```text
NAME  STATUS  AGE
...
```

## 清理

在执行清理前注意以下事项：
- **数据丢失警告**：删除 Deployment 和 ServiceMonitor 后，Prometheus 将停止采集 JVM 指标，历史数据由 Prometheus 实例的 DataRetentionTime 决定保留时长
- **依赖检查**：确认无其他应用通过 ServiceMonitor 引用 java-app-monitor，避免影响其他采集链路
- **Prometheus 实例**：PROM_INSTANCE_ID 对应的 Prometheus 实例会产生计费，如不再使用请手动销毁

### 数据面（需 VPN/IOA）

先清理 Kubernetes 资源（ServiceMonitor 必须在关联的 Deployment/Service 之前删除，避免 Prometheus Operator 产生孤儿 target 告警）：

```bash
kubectl delete servicemonitor java-app-monitor \
    -n NAMESPACE
# expected: servicemonitor.monitoring.coreos.com "java-app-monitor" deleted

kubectl delete deployment java-app \
    -n NAMESPACE
# expected: deployment.apps "java-app" deleted

kubectl delete service java-app \
    -n NAMESPACE
# expected: service "java-app" deleted

kubectl delete configmap jmx-exporter-config \
    -n NAMESPACE
# expected: configmap "jmx-exporter-config" deleted
```

### 控制面（tccli）

验证资源已清理：

```bash
kubectl get pods -n NAMESPACE \
    -l app=java-app
# expected: No resources found in NAMESPACE namespace.

tccli monitor DescribePrometheusTargets \
    --region <Region> \
    --InstanceId PROM_INSTANCE_ID
# expected: Targets 列表中不再包含 java-app-monitor（如 Prometheus 实例仍保留）
```

```text
NAME  STATUS  AGE
...
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Prometheus Target 中无 JVM 指标 | `tccli monitor DescribePrometheusTargets --region <Region> --InstanceId PROM_INSTANCE_ID` | ServiceMonitor 标签选择器未匹配到 Service，或 Prometheus Operator 不在同一命名空间 | `kubectl get servicemonitor -n NAMESPACE java-app-monitor -o yaml` 确认 selector.matchLabels 与 Service labels 一致，确认 ServiceMonitor 含有 `release: prometheus` 标签 |
| 9404 端口无法访问 | `kubectl exec -it POD_NAME -n NAMESPACE -- netstat -tlnp` | JMX Exporter Java Agent 未启动或 JAR 文件缺失 | 检查 Pod 环境变量 `JAVA_TOOL_OPTIONS` 是否含 `-javaagent` 参数；确认容器镜像中 `/opt/jmx_exporter/jmx_prometheus_javaagent.jar` 存在 |
| 指标数据全为 0 或空 | `kubectl exec -it POD_NAME -n NAMESPACE -- curl -s localhost:9404/metrics` | JVM 尚未完成初始化或 JMX Exporter 配置规则不匹配 | 等待应用完全启动（通常 30-60 秒）后重试；确认 ConfigMap 中 rules.pattern 与 JVM MXBean 名称一致 |
| Pod 启动失败 CrashLoopBackOff | `kubectl describe pod POD_NAME -n NAMESPACE` | JAVA_TOOL_OPTIONS 中的 javaagent 路径错误或 JAR 不可执行 | 将 `-javaagent:` 路径修正为镜像中 JMX Exporter JAR 的实际路径；考虑使用 initContainer 下载 JAR 到共享卷 |

## 下一步

- [MySQL Exporter 接入](../MySQL%20Exporter%20接入/tccli%20操作.md) -- page_id `112792`
- [腾讯云 Prometheus 一键关联监控容器服务](../腾讯云%20Prometheus%20一键关联监控容器服务/tccli%20操作.md) -- page_id `90923`
- [一键接入腾讯云应用性能监控 APM](../一键接入腾讯云应用性能监控%20APM/tccli%20操作.md) -- page_id `107997`

## 控制台替代

[Prometheus 控制台](https://console.cloud.tencent.com/monitor/prometheus) -> 选择实例 -> 集成中心 -> JVM 监控 -> 查看 Grafana 大盘。
