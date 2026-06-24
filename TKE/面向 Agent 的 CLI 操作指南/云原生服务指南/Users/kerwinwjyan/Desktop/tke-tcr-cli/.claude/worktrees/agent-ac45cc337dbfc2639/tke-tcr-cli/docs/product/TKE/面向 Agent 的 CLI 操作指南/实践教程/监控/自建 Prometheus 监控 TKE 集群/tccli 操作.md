# 自建 Prometheus 监控 TKE 集群（tccli）

> 对照官方：[自建 Prometheus 监控 TKE 集群](https://cloud.tencent.com/document/product/457/82640) · page_id `82640`

## 概述

在 TKE 中部署自建 Prometheus + Grafana 监控集群和应用。通过 ServiceMonitor CRD 自动发现监控目标，配合 NodePort 或 CLB 暴露 Grafana。

## 前置条件

- [环境准备](../../../../环境准备.md)

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 安装 Prometheus Operator | `kubectl apply -f https://github.com/prometheus-operator/.../bundle.yaml` | 是 |
| 创建 ServiceMonitor | `kubectl apply -f servicemonitor.yaml` | 是 |
| 暴露 Grafana | `kubectl expose deploy grafana --type=LoadBalancer --port=3000` | 是 |

## 操作步骤

### 1. 安装 Prometheus Stack

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install prometheus prometheus-community/kube-prometheus-stack -n monitoring --create-namespace
```

### 2. 创建应用 ServiceMonitor

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: app-monitor
spec:
  selector:
    matchLabels:
      app: myapp
  endpoints:
  - port: metrics
    interval: 30s
```

### 3. 访问 Grafana

```bash
kubectl port-forward svc/prometheus-grafana 3000:80 -n monitoring
# 或
kubectl expose deploy prometheus-grafana -n monitoring --type=LoadBalancer --port=3000
```

## 验证

```bash
kubectl get pods -n monitoring
kubectl get servicemonitors -A
```

```text
NAME  STATUS  AGE
...
```

## 清理

```bash
helm uninstall prometheus -n monitoring
```

## 下一步

- [腾讯云 Prometheus 一键关联监控容器服务](../腾讯云%20Prometheus%20一键关联监控容器服务/tccli%20操作.md)
- [监控及告警指标列表](../../../可观测配置/运维可观测性/监控及告警指标列表/tccli%20操作.md)

## 控制台替代

控制台：组件管理 → 安装 prometheus。
