# 资源利用率提升工具大全（tccli）

> 对照官方：[资源利用率提升工具大全](https://cloud.tencent.com/document/product/457/80787) · page_id `80787`

## 概述

TKE 生态提供了多种资源利用率提升工具，覆盖成本分配、资源推荐、自动扩缩容、自定义监控等场景。本页对比 Crane、KubeCost、VPA recommender、TKE 原生成本洞察和 Prometheus + Grafana 五类工具，帮助按照实际需求选择合适的方案。

> **kubectl 数据面不可达**：CAM 拒绝公网端点（strategyId:240463971），内网需 VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。控制面操作（集群管理、组件安装）通过 tccli 完成。

## 前置条件

- [环境准备](../../../环境准备.md)
- 目标集群已就绪（`ClusterStatus: "Running"`）
- 如使用 kubectl，已通过 VPN/IOA 连接至集群内网

```bash
# 检查集群状态
tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
```

```json
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "CLUSTER_ID",
            "ClusterName": "CLUSTER_NAME",
            "ClusterStatus": "Running"
        }
    ]
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 查看集群列表 | `tccli tke DescribeClusters` | 是 |
| 安装 Crane 组件 | `tccli tke InstallAddon --AddonName craned` | 是 |
| 查看组件状态 | `tccli tke DescribeAddon --AddonName craned` | 是 |
| 查看集群已安装组件 | `tccli tke DescribeAddonList --ClusterId CLUSTER_ID` | 是 |
| 卸载组件 | `tccli tke DeleteAddon --AddonName craned` | 否 |
| 查看节点用量 | `kubectl top nodes` | 是 |
| 查看 Pod 用量 | `kubectl top pods -A` | 是 |
| 安装 KubeCost | `helm install kubecost` | 否（重复执行会升级） |
| 查看推荐结果 | `kubectl get recommendation` | 是 |
| 查看 VPA 推荐 | `kubectl describe vpa <vpa-name>` | 是 |

## 操作步骤

> 本章节聚焦于工具对比与选择决策，非顺序执行的操作流程。根据需求选择对应的工具章节。

### 工具概览

| 工具名称 | 安装方式 | 核心能力 | 适用场景 | 推荐度 |
|---------|---------|---------|---------|:----:|
| **Crane（TKE 原生）** | `tccli tke InstallAddon --AddonName craned` | 资源预测、智能推荐、自动扩缩容、混部调度 | 需要自动化资源优化和成本节省的 TKE 集群 | ★★★★★ |
| **KubeCost** | `helm install kubecost`（数据面） | 成本分配、费用归因、效率指标、多集群统一看板 | 多团队成本核算、FinOps 治理、费用 showback | ★★★★ |
| **metrics-server + VPA recommender** | kubectl apply + `updateMode: "Off"`（数据面） | 基于历史用量的资源推荐建议（不自动调整） | 仅需推荐值、不想自动变更资源配置的场景 | ★★★ |
| **TKE 原生成本洞察** | 控制台开启 / TCOP 上报 | 集群级成本可视化、命名空间/工作负载级费用分析 | 快速查看 TKE 集群成本构成，无需额外安装 | ★★★★ |
| **Prometheus + Grafana** | `helm install kube-prometheus-stack`（数据面） | 自定义资源监控面板、灵活告警规则、长期存储 | 已有 Prometheus 体系、需要高度定制化的监控 | ★★★ |

### 选择依据

根据目标需求选择工具：

```
需要成本分配和费用归因？
├── 是 → 多团队场景？ → KubeCost
│         └── 快速查看 → TKE 原生成本洞察
│
需要资源自动优化和推荐？
├── 是 → 自动化优化 → Crane（推荐 + 自动伸缩 + 混部）
│         └── 仅推荐不自动 → VPA recommender（updateMode: Off）
│
需要自定义监控面板？
├── 是 → 已有 Prometheus？ → kube-prometheus-stack + Grafana
│         └── 无 Prometheus → 先装 kube-prometheus-stack
│
仅查看节点/Pod 用量？
└── 是 → kubectl top（metrics-server 已预装）
```

```text
# command executed successfully
```

#### 工具组合建议

| 场景 | 推荐组合 | 说明 |
|------|---------|------|
| 成本管控入门 | TKE 原生成本洞察 + Crane | 先可视化成本，再自动优化 |
| FinOps 治理 | KubeCost + Crane | 成本归因 + 资源优化 |
| 精细化监控 | Prometheus + Grafana + VPA recommender | 自定义面板 + 推荐值参考 |
| 全面自动化 | Crane + KubeCost + Prometheus | 预测 → 推荐 → 自动调整 → 成本核算 |

### Crane 安装与使用（控制面 + 数据面）

Crane 是 TKE 原生集成的资源优化组件，通过 `craned` 插件一键安装，提供资源预测、智能推荐和自动扩缩容能力。

```bash
# 1. 安装 Craned 组件
tccli tke InstallAddon --region <Region> --ClusterId CLUSTER_ID --AddonName craned
```

```json
{
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

```bash
# 2. 轮询组件安装状态
tccli tke DescribeAddon --region <Region> --ClusterId CLUSTER_ID --AddonName craned
```

```json
{
    "Name": "craned",
    "Status": "Running"
}
```

```bash
# 3. 查看资源推荐结果（须 VPN/IOA）
kubectl get recommendations -A
```

```text
NAMESPACE    NAME                    AGE   TARGETKIND        TARGETNAME       READY
default      recommendation-xxx     10m   Deployment        DEPLOYMENT_NAME  True
```

```bash
# 4. 查看推荐详情（须 VPN/IOA）
kubectl get recommendation recommendation-xxx -o json | jq '.status.recommendedResources'
```

```json
{
  "containers": [
    {
      "containerName": "CONTAINER_NAME",
      "target": {
        "cpu": "200m",
        "memory": "256Mi"
      }
    }
  ]
}
```

### KubeCost 安装与使用（数据面，须 VPN/IOA）

KubeCost 是开源 Kubernetes 成本分配工具，通过 Helm 安装，提供命名空间、工作负载、标签等维度的成本归因。

```bash
# 1. 添加 KubeCost Helm 仓库
helm repo add kubecost https://kubecost.github.io/cost-analyzer/
helm repo update
```

```text
"kubecost" has been added to your repositories
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "kubecost" chart repository
Update Complete. Happy Helming!
```

```bash
# 2. 安装 KubeCost
helm install kubecost kubecost/cost-analyzer \
  --namespace kubecost --create-namespace \
  --set kubecostToken="<KUBECOST_TOKEN>" \
  --set prometheus.server.persistentVolume.enabled=false
```

```text
NAME: kubecost
LAST DEPLOYED: ...
NAMESPACE: kubecost
STATUS: deployed
```

```bash
# 3. 查看 KubeCost Pod 状态
kubectl get pods -n kubecost
```

```text
NAME                                          READY   STATUS    RESTARTS   AGE
kubecost-cost-analyzer-xxxxxxxxxx-xxxxx       1/1     Running   0          2m
kubecost-prometheus-server-xxxxxxxxxx-xxxxx   1/1     Running   0          2m
```

```bash
# 4. 访问 KubeCost 面板（端口转发）
kubectl port-forward --namespace kubecost deployment/kubecost-cost-analyzer 9090:9090
```

```text
Forwarding from 127.0.0.1:9090 -> 9090
# 浏览器访问 http://localhost:9090
```

### VPA recommender 使用（数据面，须 VPN/IOA）

Vertical Pod Autoscaler (VPA) 的 recommender 模式仅提供资源推荐建议，不自动调整 Pod 的 Request/Limit。

```bash
# 1. 安装 VPA（从官方仓库）
git clone https://github.com/kubernetes/autoscaler.git
cd autoscaler/vertical-pod-autoscaler
./hack/vpa-up.sh
```

```text
# 确认 VPA 组件已部署
customresourcedefinition.apiextensions.k8s.io/verticalpodautoscalers.autoscaling.k8s.io created
deployment.apps/vpa-admission-controller created
deployment.apps/vpa-recommender created
deployment.apps/vpa-updater created
```

```bash
# 2. 创建 VPA 对象（仅推荐，不自动调整）
cat <<EOF | kubectl apply -f -
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: DEPLOYMENT_NAME-vpa
  namespace: NAMESPACE
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind: Deployment
    name: DEPLOYMENT_NAME
  updatePolicy:
    updateMode: "Off"
EOF
```

```text
verticalpodautoscaler.autoscaling.k8s.io/DEPLOYMENT_NAME-vpa created
```

```bash
# 3. 等待推荐生成后查看
kubectl describe vpa DEPLOYMENT_NAME-vpa -n NAMESPACE
```

```text
...
Recommendation:
  Container Recommendations:
    Container Name:  CONTAINER_NAME
    Lower Bound:
      Cpu:     200m
      Memory:  256Mi
    Target:
      Cpu:     500m
      Memory:  512Mi
    Upper Bound:
      Cpu:     1
      Memory:  1Gi
...
```

### TKE 原生成本洞察（控制面 + 数据面）

TKE 内置成本洞察功能，通过控制台或 TCOP 监控 API 获取集群成本数据。

```bash
# 查看 TCOP 成本数据（通过监视器 API）
tccli monitor GetMonitorData --region <Region> --cli-input-json '{
    "Namespace": "QCE/TKE",
    "MetricName": "CostTotal",
    "Instances": [{
        "Dimensions": [
            {"Name": "tke_cluster_instance_id", "Value": "CLUSTER_ID"}
        ]
    }],
    "Period": 3600,
    "StartTime": "START_TIME",
    "EndTime": "END_TIME"
}'
```

```json
{
    "DataPoints": [
        {
            "Timestamps": [],
            "Values": []
        }
    ],
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

> **注意**：成本洞察新版通过 CLS 日志服务关联真实账单数据，配置步骤参见[成本洞察（新版）](../../../可观测配置/成本洞察和优化/成本洞察（新版）/tccli%20操作.md/tccli%20操作.md)。

### Prometheus + Grafana 自定义监控（数据面，须 VPN/IOA）

通过 kube-prometheus-stack 搭建自定义资源监控面板，灵活配置告警规则和可视化。

```bash
# 1. 安装 kube-prometheus-stack
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace
```

```text
NAME: kube-prometheus-stack
LAST DEPLOYED: ...
NAMESPACE: monitoring
STATUS: deployed
```

```bash
# 2. 验证组件状态
kubectl get pods -n monitoring
```

```text
NAME                                                       READY   STATUS    RESTARTS   AGE
kube-prometheus-stack-grafana-xxxxxxxxxx-xxxxx             2/2     Running   0          2m
kube-prometheus-stack-operator-xxxxxxxxxx-xxxxx            1/1     Running   0          2m
kube-prometheus-stack-prometheus-0                          2/2     Running   0          2m
```

```bash
# 3. 访问 Grafana 面板
kubectl port-forward --namespace monitoring svc/kube-prometheus-stack-grafana 3000:80
```

```text
Forwarding from 127.0.0.1:3000 -> 3000
# 浏览器访问 http://localhost:3000（默认 admin/prom-operator）
```

## 验证

### 控制面（tccli）

| 验证项 | 命令 | 期望结果 |
|--------|------|---------|
| 集群就绪 | `tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'` | `ClusterStatus: "Running"` |
| Crane 已安装 | `tccli tke DescribeAddon --region <Region> --ClusterId CLUSTER_ID --AddonName craned` | `Status: "Running"` |
| 组件列表含 craned | `tccli tke DescribeAddonList --region <Region> --ClusterId CLUSTER_ID` | 列表中包含 `craned` |

### 数据面（须 VPN/IOA）

| 验证项 | 命令 | 期望结果 |
|--------|------|---------|
| Crane 推荐已生成 | `kubectl get recommendations -A` | 至少一条推荐，`READY: True` |
| KubeCost 运行中 | `kubectl get pods -n kubecost` | 所有 Pod `STATUS: Running` |
| VPA 推荐已生成 | `kubectl describe vpa DEPLOYMENT_NAME-vpa -n NAMESPACE` | 包含 `Recommendation` 和 `Target` 值 |
| Prometheus 运行中 | `kubectl get pods -n monitoring \| grep prometheus` | Pod `STATUS: Running` |
| Grafana 可访问 | `kubectl port-forward --namespace monitoring svc/kube-prometheus-stack-grafana 3000:80` | 端口转发成功，面板可访问 |

```bash
# 综合验证：检查各工具组件状态
tccli tke DescribeAddon --region <Region> --ClusterId CLUSTER_ID --AddonName craned
kubectl get pods -n kubecost --no-headers 2>/dev/null || echo "KubeCost 未安装"
kubectl get pods -n monitoring --no-headers 2>/dev/null || echo "Prometheus 未安装"
kubectl get recommendations -A --no-headers 2>/dev/null || echo "Crane 推荐尚未生成"
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

## 清理

> **警告**：清理操作不可逆。卸载组件不会自动删除已生成的分析数据和推荐结果。删除集群会级联删除所有关联资源，请提前备份业务数据。

### 卸载 Crane

```bash
# 卸载 Craned 组件
tccli tke DeleteAddon --region <Region> --ClusterId CLUSTER_ID --AddonName craned
```

```json
{
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

```bash
# 验证已卸载
tccli tke DescribeAddon --region <Region> --ClusterId CLUSTER_ID --AddonName craned
# expected: 返回组件不存在或空
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

### 卸载 KubeCost（须 VPN/IOA）

```bash
# 卸载 KubeCost
helm uninstall kubecost -n kubecost
kubectl delete namespace kubecost --ignore-not-found
```

```text
release "kubecost" uninstalled
```

### 卸载 VPA（须 VPN/IOA）

```bash
# 删除 VPA 对象
kubectl delete vpa DEPLOYMENT_NAME-vpa -n NAMESPACE --ignore-not-found
```

```text
verticalpodautoscaler.autoscaling.k8s.io "DEPLOYMENT_NAME-vpa" deleted
```

### 卸载 Prometheus + Grafana（须 VPN/IOA）

```bash
# 卸载 kube-prometheus-stack
helm uninstall kube-prometheus-stack -n monitoring
kubectl delete namespace monitoring --ignore-not-found
```

```text
release "kube-prometheus-stack" uninstalled
```

### 删除集群（如为本页新建的测试集群）

```bash
tccli tke DeleteCluster --region <Region> --ClusterId CLUSTER_ID --InstanceDeleteMode terminate
```

```json
{
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `InstallAddon craned` 返回 `FailedOperation.AddonAlreadyInstalled` | `tccli tke DescribeAddon --region <Region> --ClusterId CLUSTER_ID --AddonName craned` | 组件已安装 | 直接使用已安装的组件，如需升级使用 `UpdateAddon` |
| `helm install kubecost` 报网络错误 | `helm repo list` 确认仓库连通性 | 无法访问 KubeCost Helm 仓库 | 配置代理或使用内网镜像仓库拉取 Chart |
| `kubectl get recommendations` 返回 `No resources found` | `kubectl get analytics -A` 查看分析任务状态 | Craned 尚未完成分析或集群无工作负载 | 等待 30 分钟后重新查询；确保集群有可分析的工作负载 |
| `kubectl describe vpa` 推荐为空 | `kubectl top pods -n NAMESPACE` 查看监控数据 | metrics-server 未运行或 VPA recommender 尚未收集足够历史数据 | 确认 metrics-server 正常运行；等待至少 24 小时让 VPA 收集数据 |
| `kubectl top` 返回 `Metrics API not available` | `kubectl get pods -n kube-system \| grep metrics-server` | metrics-server 未安装或异常 | 检查 metrics-server 组件状态，必要时重建 |
| Prometheus Pod 一直 Pending | `kubectl describe pod -n monitoring PROMETHEUS_POD_NAME` | 存储卷未绑定（PVC 挂起）或节点资源不足 | 配置持久化存储或设置 `persistentVolume.enabled=false` |
| Grafana 面板无数据 | `kubectl logs -n monitoring PROMETHEUS_POD_NAME` | Prometheus 未成功采集指标 | 检查 Prometheus ServiceMonitor 配置，确认 metrics 端点可达 |
| `DeleteAddon` 返回 `FailedOperation.AddonNotFound` | 检查 AddonName 拼写 | 组件未安装或名称错误 | 确认组件名称（`craned` 全小写） |

### 功能异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Crane 推荐值始终为 0 | `kubectl logs -n crane-system deployment/craned` 查看日志 | metrics-server 数据未上报或集群无足够监控数据 | 确认 `kubectl top pods -A` 有数据；等待至少 1 天收集历史数据 |
| KubeCost 显示成本为 0 | `kubectl logs -n kubecost deployment/kubecost-cost-analyzer` | Prometheus 数据未采集或节点无定价模型 | 检查 Prometheus Target 状态；确认节点机型在云服务定价数据库中 |
| VPA 推荐值与实际用量偏差大 | 对比 `kubectl top pods` 实时值与 VPA Target 推荐 | VPA 仅基于历史数据，突发性负载导致偏差 | 使用更多历史数据的窗口期（默认 8 天），或结合 Crane 实时预测 |
| 成本洞察控制台无数据 | `tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'` | 集群未开启 TCOP 数据上报 | 在控制台开启 TCOP 数据上报；仅标准集群支持 |
| 多工具同时运行时推荐值冲突 | 分别查看 Crane `kubectl get recommendations` 和 VPA `kubectl describe vpa` | 不同工具基于不同算法和历史窗口 | 选择一个主要工具（推荐 Crane），其他工具仅作为参考 |
| 组件卸载后推荐数据仍然显示 | `kubectl get recommendations -A` | CRD 和推荐数据不随组件自动清理 | 手动删除 Recommendation CRD 资源：`kubectl delete recommendations --all -A` |

## 下一步

- [Crane 调度器介绍](../../../调度配置/Crane%20调度器介绍/tccli%20操作.md) -- 深入了解 Crane 调度器原理和配置
- [TKE 集群混部原理与部署介绍](../../../Data%20应用实践/TKE%20集群混部原理与部署介绍/tccli%20操作.md) -- 在线离线混部提升利用率
- [Request 智能推荐](../../../可观测配置/成本洞察和优化/成本优化/Request%20智能推荐/tccli%20操作.md) -- 基于 Crane 的资源推荐实操
- [Node Map](../../../可观测配置/成本洞察和优化/Node%20Map/tccli%20操作.md) -- 节点可视化热力图
- [成本洞察](../../../可观测配置/成本洞察和优化/成本洞察/tccli%20操作.md) -- TKE 成本可视化面板
- [KubeCost 官方文档](https://www.kubecost.com/docs/) -- KubeCost 官方使用指南
- [Crane 项目地址](https://github.com/gocrane/crane) -- Crane (FinOps) 开源项目

## 控制台替代

- **Crane 组件管理**：[容器服务控制台 → 组件管理](https://console.cloud.tencent.com/tke2/cluster)
- **成本洞察**：[容器服务控制台 → 成本洞察](https://console.cloud.tencent.com/tke2/cost/insight)
- **Node Map**：[容器服务控制台 → 节点 Map](https://console.cloud.tencent.com/tke2/node/map)
- **KubeCost 面板**：`kubectl port-forward` 后访问 `http://localhost:9090`
