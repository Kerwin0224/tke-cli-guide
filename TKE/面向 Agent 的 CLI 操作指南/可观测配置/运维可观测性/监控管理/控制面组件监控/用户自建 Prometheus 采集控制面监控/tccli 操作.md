# 用户自建 Prometheus 采集控制面监控（tccli）

> 对照官方：[用户自建 Prometheus 采集控制面监控](https://cloud.tencent.com/document/product/457/116448) · page_id `116448`

## 概述

托管集群（`MANAGED_CLUSTER`）的控制面组件（kube-apiserver、kube-scheduler、kube-controller-manager）对外暴露 `/metrics` 端点。本页介绍如何通过自建 Prometheus 采集这些控制面组件的指标数据。托管集群的控制面由腾讯云运维，控制面组件运行在腾讯云基础设施上，用户无法直接通过 kubectl 访问控制面节点，但可以通过集群内或集群外 Prometheus 采集指标。

| 方案 | 适用场景 | 复杂度 |
|------|---------|:--:|
| 集群内 Prometheus | Prometheus 部署在同集群内，通过 Kubernetes Service 采集 | 低 |
| 集群外 Prometheus | Prometheus 部署在其他集群或外部，通过 CLB/公网端点采集 | 中 |

推荐将 Prometheus 部署在目标集群内，通过 ServiceAccount + RBAC 授权直接访问控制面 `/metrics` 端点，无需额外网络配置。

## 前置条件

- `tccli` 已配置凭证
- kubectl 不可达：演示集群 cls-xxxxxxxx 为托管集群，公网端点可能受 CAM 策略限制（strategyId:240463971）；内网无 VPN 时不可达。以下 kubectl 命令作为文档参考。
- `helm` 已安装（v3.x+），用于安装 kube-prometheus-stack
- 集群类型为托管集群（`MANAGED_CLUSTER`）

```bash
tccli tke DescribeClusters --ClusterIds '["cls-xxxxxxxx"]' --region ap-guangzhou
```

```json
{
  "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "TotalCount": 1,
  "Clusters": [
    {
      "ClusterId": "cls-xxxxxxxx",
      "ClusterName": "my-test-cluster",
      "ClusterType": "MANAGED_CLUSTER",
      "ClusterStatus": "Running",
      "ClusterVersion": "1.30.0",
      "ClusterNodeNum": 2
    }
  ]
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 查看集群状态 | `tccli tke DescribeClusters` | 是 |
| 获取集群 kubeconfig | `tccli tke DescribeClusterKubeconfig` | 是 |
| 查看集群端点 | `tccli tke DescribeClusterEndpoint` | 是 |
| 开启公网访问端点 | `tccli tke CreateClusterEndpoint` | 否 |
| 关闭公网访问端点 | `tccli tke DeleteClusterEndpoint` | 是 |
| 部署 Prometheus | `helm install kube-prometheus-stack` | 是 |
| 创建 ServiceMonitor | `kubectl apply -f servicemonitor.yaml` | 是 |
| 验证 Targets | Prometheus UI / `kubectl port-forward` + curl | 是 |
| 清理卸载 | `helm uninstall` + `kubectl delete` | — |

## 操作步骤

### 步骤 1：获取 kubeconfig 并验证集群连通性

对于集群内 Prometheus 方案，获取 kubeconfig 主要是为了在开发机上通过 kubectl 部署 Prometheus 和 ServiceMonitor 资源。

```bash
tccli tke DescribeClusterKubeconfig \
  --ClusterId cls-xxxxxxxx \
  --region ap-guangzhou \
  | jq -r '.Kubeconfig' > ~/.kube/config-tke
```

```json
{
  "Kubeconfig": "YXBpVmVyc2lvbjogdjEKY2x1c3RlcnM6Ci0g...",
  "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

```bash
export KUBECONFIG=~/.kube/config-tke
kubectl cluster-info
# expected: Kubernetes control plane is running at https://...
# 需 kubectl 可达环境
```

### 步骤 2：创建 Prometheus 命名空间和 RBAC

Prometheus 需要 `get`、`list`、`watch` 集群范围内资源的权限来发现监控目标。使用 ClusterRole + ClusterRoleBinding 而非 Role，避免命名空间限制。

`prometheus-rbac.yaml`：

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: monitoring
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus
  namespace: monitoring
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus
rules:
  - apiGroups: [""]
    resources:
      - nodes
      - nodes/metrics
      - services
      - endpoints
      - pods
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources:
      - configmaps
    verbs: ["get"]
  - nonResourceURLs:
      - "/metrics"
    verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: prometheus
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus
subjects:
  - kind: ServiceAccount
    name: prometheus
    namespace: monitoring
```

```bash
kubectl apply -f prometheus-rbac.yaml
# expected: namespace/monitoring created, serviceaccount/prometheus created,
# clusterrole.rbac.authorization.k8s.io/prometheus created,
# clusterrolebinding.rbac.authorization.k8s.io/prometheus created
# 需 kubectl 可达环境
```

### 步骤 3：通过 Helm 部署 Prometheus

使用 `kube-prometheus-stack`（包含 Prometheus Operator + Grafana + Alertmanager + 预配置采集规则），一步到位。

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
# expected: Hang tight while we grab the latest...

helm upgrade --install prometheus prometheus-community/kube-prometheus-stack \
    --namespace monitoring \
    --set prometheus.prometheusSpec.serviceAccountName=prometheus \
    --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false \
    --set prometheus.prometheusSpec.podMonitorSelectorNilUsesHelmValues=false
# expected: Release "prometheus" has been upgraded. Happy Helming!
# 需 kubectl 可达环境
```

### 步骤 4：配置 ServiceMonitor 采集控制面组件

kube-apiserver 指标通过 `kubernetes.default.svc` Service 暴露，kube-scheduler 和 kube-controller-manager 的指标端点通过 API Server 代理暴露。创建以下 ServiceMonitor：

`servicemonitor-control-plane.yaml`：

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: kube-apiserver
  namespace: monitoring
spec:
  endpoints:
    - interval: 30s
      port: https
      scheme: https
      bearerTokenFile: /var/run/secrets/kubernetes.io/serviceaccount/token
      tlsConfig:
        insecureSkipVerify: true
  selector:
    matchLabels:
      component: apiserver
      provider: kubernetes
  namespaceSelector:
    matchNames:
      - default
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: kube-controller-manager
  namespace: monitoring
spec:
  endpoints:
    - interval: 30s
      port: https
      scheme: https
      bearerTokenFile: /var/run/secrets/kubernetes.io/serviceaccount/token
      tlsConfig:
        insecureSkipVerify: true
      metricRelabelings:
        - action: keep
          regex: >-
            up|leader_election_master_status|workqueue_.+|rest_client_.+|node_collector_.+
          sourceLabels: [__name__]
  selector:
    matchLabels:
      component: kube-controller-manager
      provider: kubernetes
  namespaceSelector:
    matchNames:
      - kube-system
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: kube-scheduler
  namespace: monitoring
spec:
  endpoints:
    - interval: 30s
      port: https
      scheme: https
      bearerTokenFile: /var/run/secrets/kubernetes.io/serviceaccount/token
      tlsConfig:
        insecureSkipVerify: true
      metricRelabelings:
        - action: keep
          regex: >-
            up|leader_election_master_status|scheduler_.+|rest_client_.+
          sourceLabels: [__name__]
  selector:
    matchLabels:
      component: kube-scheduler
      provider: kubernetes
  namespaceSelector:
    matchNames:
      - kube-system
```

```bash
kubectl apply -f servicemonitor-control-plane.yaml
# expected: servicemonitor.monitoring.coreos.com/kube-apiserver created
# （controller-manager、scheduler 同样）
# 需 kubectl 可达环境
```

## 验证

### Control plane (tccli)

```bash
tccli tke DescribeClusters --ClusterIds '["cls-xxxxxxxx"]' --region ap-guangzhou
```

```json
{
  "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "TotalCount": 1,
  "Clusters": [
    {
      "ClusterId": "cls-xxxxxxxx",
      "ClusterName": "my-test-cluster",
      "ClusterType": "MANAGED_CLUSTER",
      "ClusterStatus": "Running",
      "ClusterVersion": "1.30.0",
      "ClusterNodeNum": 2
    }
  ]
}
```

### Data plane (kubectl) — 需 kubectl 可达环境

```bash
# 1. 确认 Prometheus Pod 正常运行
kubectl get pods -n monitoring -l app.kubernetes.io/name=prometheus
# expected: 至少 1 个 prometheus Pod，状态 Running，READY 1/1

# 2. 确认 ServiceMonitor 已注册
kubectl get servicemonitors -n monitoring
# expected: kube-apiserver, kube-controller-manager, kube-scheduler

# 3. 端口转发访问 Prometheus UI
kubectl port-forward -n monitoring svc/prometheus-operated 9090:9090 &
# expected: Forwarding from 127.0.0.1:9090 -> 9090

# 4. 查询 targets 状态
curl -s http://localhost:9090/api/v1/targets | jq '.data.activeTargets[] | {labels: .labels.job, health: .health}'
```

```text
{"labels":"kube-apiserver","health":"up"}
{"labels":"kube-controller-manager","health":"up"}
{"labels":"kube-scheduler","health":"up"}
```

验证维度汇总：

| 维度 | 命令 | 预期 |
|------|------|------|
| 集群状态 | `DescribeClusters --ClusterIds '["cls-xxxxxxxx"]'` | `ClusterStatus: "Running"` |
| Prometheus Pod | `kubectl get pods -n monitoring` | 状态 `Running`，1/1 Ready |
| ServiceMonitor | `kubectl get servicemonitors -n monitoring` | 3 个控制面 ServiceMonitor |
| Targets 健康 | Prometheus API `/api/v1/targets` | 所有 target 健康状态为 `up` |

## 清理

> **警告**：卸载 Prometheus 会删除所有历史监控数据（指标、告警规则、Grafana 仪表盘）。生产环境操作前确认已备份关键数据。

### Data plane (kubectl) — 需 kubectl 可达环境

清理顺序：ServiceMonitor → Helm Release → Namespace → RBAC。

```bash
# 1. 删除 ServiceMonitor
kubectl delete servicemonitor -n monitoring kube-apiserver kube-controller-manager kube-scheduler
# expected: servicemonitor "kube-apiserver" deleted (x3)

# 2. 卸载 Prometheus Stack
helm uninstall prometheus -n monitoring
# expected: release "prometheus" uninstalled

# 3. 删除监控命名空间
kubectl delete namespace monitoring
# expected: namespace "monitoring" deleted

# 4. 删除 RBAC 资源
kubectl delete clusterrole prometheus
kubectl delete clusterrolebinding prometheus
# expected: clusterrole "prometheus" deleted, clusterrolebinding "prometheus" deleted
```

### Control plane (tccli)

```bash
# 清理 kubeconfig 文件（可选）
rm -f ~/.kube/config-tke
unset KUBECONFIG
```

### 验证 cleanup

```bash
# 确认命名空间已删除
kubectl get ns monitoring 2>&1
# expected: Error from server (NotFound): namespaces "monitoring" not found

# 确认集群未受影响
tccli tke DescribeClusters --ClusterIds '["cls-xxxxxxxx"]' --region ap-guangzhou
```

```json
{
  "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "TotalCount": 1,
  "Clusters": [
    {
      "ClusterId": "cls-xxxxxxxx",
      "ClusterName": "my-test-cluster",
      "ClusterStatus": "Running"
    }
  ]
}
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DescribeClusters` 返回 `ResourceNotFound` | `tccli tke DescribeClusters --region ap-guangzhou` 列出所有集群确认 ID | 集群不存在或 region 不匹配 | 用 `tccli configure list` 确认 region，用 DescribeClusters 列出所有集群获取正确 ID |
| `DescribeClusterKubeconfig` 返回 `UnauthorizedOperation` | `tccli configure list` 检查凭据 | 子账号缺少 `tke:DescribeClusterKubeconfig` 权限（环境限制） | 联系主账号授权 `tke:DescribeClusterKubeconfig` |
| `kubectl cluster-info` 连接超时 | 检查 `~/.kube/config` 中 server 地址 | 集群 API Server 公网/内网端点未开启；CAM 策略限制公网端点访问（strategyId:240463971） | 确保 Prometheus 部署在集群内；或开启外网访问：`tccli tke CreateClusterEndpoint --ClusterId cls-xxxxxxxx --IsExtranet true --region ap-guangzhou` |
| `helm install` 返回 `context deadline exceeded` | `kubectl get pods -n kube-system` 检查系统组件 | Kubernetes API 响应超时 | 重试 helm install；检查集群网络是否正常 |

### 部署成功但无法采集指标

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Prometheus targets 显示 `down` | Prometheus UI → Status → Targets 查看具体 error | ServiceAccount 权限不足或 tlsConfig 配置错误 | 检查 ClusterRole 是否包含 `nonResourceURLs: ["/metrics"]`；确认 `tlsConfig.insecureSkipVerify: true` 已设置 |
| `kubectl get servicemonitors` 为空 | `kubectl api-resources \| grep servicemonitor` | 未安装 ServiceMonitor CRD | 重新安装 kube-prometheus-stack，确认 CRD 已创建 |
| ServiceMonitor 未发现 target | `kubectl describe servicemonitor -n monitoring kube-apiserver` | `namespaceSelector.matchNames` 不匹配 Service 所在命名空间 | kube-apiserver Service 在 `default` 命名空间 → `matchNames: ["default"]` |
| 部分指标未采集 | PromQL `up{job="kube-controller-manager"}` | 托管集群控制面部分内部指标不对外暴露 | 此为预期行为。kube-apiserver 指标完整可用 |
| kubectl 不可达 | `kubectl cluster-info` 返回超时或拒绝连接 | 托管集群公网端点可能受 CAM 策略限制（strategyId:240463971）；内网无 VPN | 确认 kubeconfig server 地址可 ping；内网地址需 VPN；所有 kubectl 命令均需可达环境 |

### 已知限制

- 托管集群（`MANAGED_CLUSTER`）的 kube-scheduler 和 kube-controller-manager `/metrics` 端点默认仅暴露少量健康指标，完整内部指标由腾讯云后端运维。
- 部署 Prometheus 产生的 PVC 存储会持续产生 CBS 云硬盘费用，清理时需删除 PVC 以停止计费。

## 下一步

- [查看 TKE 集群控制面组件监控](../查看 TKE 集群控制面组件监控/tccli 操作.md) -- TKE 控制台集成的控制面组件监控
- [kube-apiserver 组件指标说明](../kube-apiserver 组件指标说明/tccli 操作.md) -- apiserver 各指标含义与告警建议
- [kube-scheduler 组件指标说明](../kube-scheduler 组件指标说明/tccli 操作.md) -- scheduler 指标参考
- [kube-controller-manager 组件指标说明](../kube-controller-manager 组件指标说明/tccli 操作.md) -- controller-manager 指标参考
- [日志采集概述](../../../日志管理/日志采集概述/tccli 操作.md) -- 采集容器日志到 CLS
- [Prometheus Operator 官方文档](https://prometheus-operator.dev/) -- ServiceMonitor/PodMonitor 完整配置参考

## 控制台替代

登录 [容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) > 选择集群 > 监控，控制台内置的控制面组件监控已默认开启（无需自建），可查看 kube-apiserver、kube-scheduler、kube-controller-manager 的预置仪表盘。
