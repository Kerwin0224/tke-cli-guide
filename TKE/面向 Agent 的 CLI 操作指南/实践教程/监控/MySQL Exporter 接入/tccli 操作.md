# MySQL Exporter 接入（tccli）

> 对照官方：[MySQL Exporter 接入](https://cloud.tencent.com/document/product/457/112792) · page_id `112792`

## 概述

通过 Prometheus MySQL Exporter 采集 MySQL 数据库（CDB/自建）的运行指标，包括连接数、QPS、慢查询、InnoDB 状态、复制延迟等，接入腾讯云 Prometheus 监控服务（TMP）实现数据库可观测性。MySQL Exporter 以 Sidecar 或独立 Deployment 方式部署在 TKE 集群中，通过 ServiceMonitor 自动发现并推送指标，配合 Grafana 预置大盘（ID: 7362）实现一站式数据库监控。

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
# expected: 返回集群详情，所需 CAM Action: tke:DescribeClusters

tccli cdb DescribeDBInstances \
    --region <Region>
# expected: 返回 CDB 实例列表，所需 CAM Action: cdb:DescribeDBInstances
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
| INSTANCE_ID | CDB 实例 ID | cdb-xxxxxxxx |
| VPC_ID | CDB 实例所属 VPC | vpc-xxxxxxxx |
| SUBNET_ID | TKE/Exporter 所在子网 | subnet-xxxxxxxx |
| SECURITY_GROUP_ID | CDB 安全组 ID（需允许 Exporter 访问） | sg-xxxxxxxx |

```bash
tccli monitor DescribePrometheusInstances \
    --region <Region> \
    --InstanceIds '["PROM_INSTANCE_ID"]'
# expected: InstanceStatus 为 "Running"

tccli tke DescribeClusters \
    --region <Region> \
    --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterStatus 为 "Running"，ClusterType 为 "MANAGED_CLUSTER"

tccli cdb DescribeDBInstances \
    --region <Region> \
    --InstanceIds '["INSTANCE_ID"]'
# expected: Status 为 1（运行中），VpcId 与 TKE 集群同 VPC
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
| 确认 CDB 实例可用 | `tccli cdb DescribeDBInstances --region <Region> --InstanceIds '["INSTANCE_ID"]'` | 是 |
| 确认 Prometheus 实例可用 | `tccli monitor DescribePrometheusInstances --region <Region>` | 是 |
| 创建数据库凭据 Secret | `kubectl create secret generic mysql-credentials -n NAMESPACE`（需 VPN/IOA） | 否（覆盖更新须加 --dry-run） |
| 部署 MySQL Exporter | `kubectl apply -f mysql-exporter.yaml`（需 VPN/IOA） | 是 |
| 创建 ServiceMonitor | `kubectl apply -f servicemonitor.yaml`（需 VPN/IOA） | 是 |
| 查看数据库指标 | `kubectl port-forward svc/mysql-exporter 9104:9104`（需 VPN/IOA） | 是 |

## 操作步骤

### 步骤 1：配置 MySQL Exporter 连接信息

#### 选择依据

- **部署模式**：独立 Deployment（推荐），与目标 MySQL 实例分离，便于独立扩缩容和维护
- **Sidecar 模式**：如 MySQL 也运行在 TKE 中，可将 Exporter 作为 Sidecar 注入同一 Pod
- **Exporter 镜像**：`prom/mysqld-exporter` 官方镜像，通过 `DATA_SOURCE_NAME` 环境变量指定连接串
- **采集指标范围**：通过 `--collect.xxx` 参数按需开启采集器，避免过多指标导致 Prometheus 存储压力
- **关键指标**：`mysql_global_status_threads_connected`（当前连接数）、`mysql_global_status_queries`（查询总数）、`mysql_global_status_slow_queries`（慢查询数）、`mysql_global_status_bytes_sent`/`bytes_received`（流量）、`mysql_slave_status_seconds_behind_master`（主从延迟）

#### 最小创建

创建数据库连接凭证 Secret：

```bash
kubectl create secret generic mysql-credentials \
    -n NAMESPACE \
    --from-literal=username=MYSQL_USER \
    --from-literal=password=MYSQL_PASSWORD
# expected: secret/mysql-credentials created
```

如需使用文件方式（生产推荐，参数超过 4 个使用 --cli-input-json 风格 YAML）：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysql-credentials
  namespace: NAMESPACE
type: Opaque
stringData:
  username: "MYSQL_USER"
  password: "MYSQL_PASSWORD"
  host: "INSTANCE_ID.mysql.internal.tencentcdb.com"
  port: "3306"
```

```bash
kubectl apply -f mysql-secret.yaml
# expected: secret/mysql-credentials created
```

#### 增强配置

部署 MySQL Exporter，配置完整采集器参数并绑定 Service：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql-exporter
  namespace: NAMESPACE
  labels:
    app: mysql-exporter
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysql-exporter
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: mysql-exporter
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9104"
        prometheus.io/path: "/metrics"
    spec:
      containers:
        - name: exporter
          image: prom/mysqld-exporter:v0.15.1
          imagePullPolicy: IfNotPresent
          env:
            - name: DATA_SOURCE_NAME
              valueFrom:
                secretKeyRef:
                  name: mysql-credentials
                  key: username
          args:
            - "--collect.info_schema.tables"
            - "--collect.info_schema.innodb_metrics"
            - "--collect.info_schema.processlist"
            - "--collect.slave_status"
            - "--collect.engine_innodb_status"
            - "--collect.global_status"
            - "--collect.global_variables"
            - "--web.listen-address=:9104"
          ports:
            - containerPort: 9104
              name: metrics
              protocol: TCP
          livenessProbe:
            httpGet:
              path: /
              port: 9104
            initialDelaySeconds: 15
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /metrics
              port: 9104
            initialDelaySeconds: 10
            periodSeconds: 5
          resources:
            requests:
              memory: "128Mi"
              cpu: "100m"
            limits:
              memory: "256Mi"
              cpu: "200m"
---
apiVersion: v1
kind: Service
metadata:
  name: mysql-exporter
  namespace: NAMESPACE
  labels:
    app: mysql-exporter
spec:
  selector:
    app: mysql-exporter
  ports:
    - name: metrics
      port: 9104
      targetPort: 9104
      protocol: TCP
  type: ClusterIP
```

| 参数 | 说明 | 示例值 |
|------|------|--------|
| MYSQL_USER | CDB 只读账号（建议使用只读权限账号，最小权限原则） | monitor |
| MYSQL_PASSWORD | CDB 账号密码 | Pa\$\$w0rd |
| NAMESPACE | Kubernetes 命名空间 | default |

```bash
kubectl apply -f mysql-exporter.yaml
# expected: deployment.apps/mysql-exporter created
# expected: service/mysql-exporter created
```

### 步骤 2：创建 ServiceMonitor 实现自动发现

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: mysql-exporter-monitor
  namespace: NAMESPACE
  labels:
    release: prometheus
spec:
  selector:
    matchLabels:
      app: mysql-exporter
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
# expected: servicemonitor.monitoring.coreos.com/mysql-exporter-monitor created
```

### 步骤 3：验证指标暴露

```bash
kubectl port-forward svc/mysql-exporter 9104:9104 \
    -n NAMESPACE &
# expected: Forwarding from 127.0.0.1:9104 -> 9104

curl -s http://localhost:9104/metrics | grep "mysql_global_status_threads_connected"
# expected:
# mysql_global_status_threads_connected 5

curl -s http://localhost:9104/metrics | grep "mysql_global_status_queries"
# expected:
# mysql_global_status_queries 1.28342e+06

curl -s http://localhost:9104/metrics | grep "mysql_global_status_slow_queries"
# expected:
# mysql_global_status_slow_queries 12

kill %1
```

## 验证

### 控制面（tccli）

```bash
tccli cdb DescribeDBInstances \
    --region <Region> \
    --InstanceIds '["INSTANCE_ID"]'
# expected: Status 为 1，VpcId 与 TKE 集群相同

tccli monitor DescribePrometheusTargets \
    --region <Region> \
    --InstanceId PROM_INSTANCE_ID
# expected: Targets 列表中包含 mysql-exporter-monitor，Health 为 "up"

tccli monitor DescribePrometheusInstances \
    --region <Region> \
    --InstanceIds '["PROM_INSTANCE_ID"]'
# expected: InstanceStatus 为 "Running"
```

### 数据面（需 VPN/IOA）

```bash
kubectl get pods -n NAMESPACE \
    -l app=mysql-exporter
# expected: 1 个 Pod，STATUS 为 Running，READY 1/1

kubectl get servicemonitor mysql-exporter-monitor \
    -n NAMESPACE
# expected: NAME mysql-exporter-monitor，状态正常

kubectl logs -n NAMESPACE \
    deploy/mysql-exporter \
    --tail=10
# expected: 日志中无 ERROR 级别记录，显示 "Listening on :9104" 及 "Starting HTTP server"

kubectl exec -n NAMESPACE \
    deploy/mysql-exporter \
    -- curl -s localhost:9104/metrics | head -5
# expected: 返回 Prometheus 格式指标数据，首行为 mysql_exporter 相关指标
```

```text
NAME  STATUS  AGE
...
```

## 清理

在执行清理前注意以下事项：
- **数据库连接影响**：删除 MySQL Exporter 不会影响 CDB 数据库实例，仅停止指标采集
- **Secret 清理**：`mysql-credentials` Secret 含数据库明文密码，务必清理避免凭据泄露
- **CDB 实例计费**：`INSTANCE_ID` 对应的 CDB 实例为包年包月/按量计费资源，清理 Exporter 不会销毁 CDB 实例——如需释放 CDB 请在控制台手动操作
- **Prometheus 实例计费**：`PROM_INSTANCE_ID` 为按量计费，如不再使用请手动销毁

### 数据面（需 VPN/IOA）

先清理 ServiceMonitor 再删除关联资源（避免 Prometheus 产生孤儿 target 并持续报错）：

```bash
kubectl delete servicemonitor mysql-exporter-monitor \
    -n NAMESPACE
# expected: servicemonitor.monitoring.coreos.com "mysql-exporter-monitor" deleted

kubectl delete deployment mysql-exporter \
    -n NAMESPACE
# expected: deployment.apps "mysql-exporter" deleted

kubectl delete service mysql-exporter \
    -n NAMESPACE
# expected: service "mysql-exporter" deleted

kubectl delete secret mysql-credentials \
    -n NAMESPACE
# expected: secret "mysql-credentials" deleted
```

### 控制面（tccli）

验证 Kubernetes 资源已清理，并确认 Prometheus target 已移除：

```bash
kubectl get pods -n NAMESPACE \
    -l app=mysql-exporter
# expected: No resources found in NAMESPACE namespace.

tccli monitor DescribePrometheusTargets \
    --region <Region> \
    --InstanceId PROM_INSTANCE_ID
# expected: Targets 列表中不再包含 mysql-exporter-monitor
```

```text
NAME  STATUS  AGE
...
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Exporter 无法连接 MySQL | `kubectl logs deploy/mysql-exporter -n NAMESPACE \| grep -i error` | 数据库连接串格式错误或安全组未放通 | 确认 `DATA_SOURCE_NAME` 格式为 `username:password@tcp(host:port)/`；在 CDB 安全组 `SECURITY_GROUP_ID` 中放通 TKE 节点/容器网段访问 3306 端口 |
| 指标数据为空 | `kubectl exec deploy/mysql-exporter -n NAMESPACE -- curl -s localhost:9104/metrics \| wc -l` | 数据库用户权限不足或采集器未启用 | 授予监控账号 `PROCESS`、`REPLICATION CLIENT`、`SELECT` 权限；确认 args 中包含 `--collect.global_status` 等采集器参数 |
| Exporter Pod CrashLoopBackOff | `kubectl describe pod -n NAMESPACE -l app=mysql-exporter` | Secret 未创建或 key 名不匹配 | `kubectl get secret mysql-credentials -n NAMESPACE -o yaml` 确认 username 和 password key 存在；重新创建 Secret |
| Prometheus 无 MySQL Target | `tccli monitor DescribePrometheusTargets --region <Region> --InstanceId PROM_INSTANCE_ID` | ServiceMonitor 标签选择器不匹配或 Prometheus 未配置 namespaceSelector | `kubectl get servicemonitor mysql-exporter-monitor -n NAMESPACE -o yaml` 确认 `spec.selector.matchLabels` 与 Service 标签一致，且 ServiceMonitor 含 `release: prometheus` 标签 |

## 下一步

- [JVM 接入](../JVM%20接入/tccli%20操作.md) -- page_id `112791`
- [腾讯云 Prometheus 一键关联监控容器服务](../腾讯云%20Prometheus%20一键关联监控容器服务/tccli%20操作.md) -- page_id `90923`
- [一键接入腾讯云应用性能监控 APM](../一键接入腾讯云应用性能监控%20APM/tccli%20操作.md) -- page_id `107997`

## 控制台替代

[Prometheus 控制台](https://console.cloud.tencent.com/monitor/prometheus) -> 选择实例 -> 集成中心 -> MySQL Exporter -> 查看 Grafana 大盘（Dashboard ID: 7362）。
