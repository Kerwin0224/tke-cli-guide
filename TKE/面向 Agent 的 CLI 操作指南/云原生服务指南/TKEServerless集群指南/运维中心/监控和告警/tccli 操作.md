# 监控和告警

> 对照官方：[监控和告警](https://cloud.tencent.com/document/product/457/39820) · page_id `39820`

## 概述

TKE Serverless 集群的监控和告警涵盖集群状态监控、Pod 资源使用监控、Horizontal Pod Autoscaler（HPA）自动扩缩容，以及可选的 Prometheus 托管服务（TMP）集成。通过 `tccli tke` 可查询集群级别状态和事件，通过 kubectl 可查看 Pod/节点资源使用并配置 HPA。对于高级监控需求，可集成 TMP 实现自定义指标的采集、存储和告警。

**Serverless 特性**：
- 节点为虚拟节点，`kubectl top nodes` 显示虚拟节点
- HPA 基于 Pod 的 CPU/内存使用自动扩缩容
- Pod 资源使用通过 `eks.tke.cloud.tencent.com/cpu|mem` annotation 定义

## 前置条件

- [环境准备](../../../../环境准备.md)

### 集群可达性检查（Before you begin）

监控操作为 hybrid（tccli + kubectl）操作。确认集群连接正常后方可继续。

```bash
# 1. 确认集群处于 Running 状态
tccli tke DescribeEKSClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterStatus: Running

# 2. 验证 kubectl 连接
kubectl --kubeconfig ~/.kube/config-eks-CLUSTER_ID cluster-info
# expected: Kubernetes control plane is running at https://...

# 3. 检查 metrics-server 是否运行（kubectl top 依赖）
kubectl --kubeconfig ~/.kube/config-eks-CLUSTER_ID get deployment metrics-server -n kube-system
# expected: READY 1/1（Serverless 集群默认已部署）
```

```json
{
  "TotalCount": "<TotalCount>",
  "Clusters": "<Clusters>",
  "ClusterId": "<ClusterId>",
  "ClusterName": "<ClusterName>",
  "VpcId": "<VpcId>",
  "SubnetIds": "<SubnetIds>"
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli / kubectl 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看集群状态 | `tccli tke DescribeEKSClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'` | 是 |
| 查看 Pod 资源 | `kubectl top pods -n NAMESPACE` | 是 |
| 查看集群事件 | `tccli tke DescribeEKSClusterEvent --region <Region> --ClusterId CLUSTER_ID` | 是 |
| 创建 HPA | `kubectl autoscale deployment/NAME --min=1 --max=10 --cpu-percent=80 -n NAMESPACE` | 否 |
| TMP 集成 | `tccli monitor CreatePrometheusClusterAgent --region <Region> --cli-input-json file://input.json` | 否 |

## 操作步骤

### 步骤 1: 查看集群整体状态

```bash
# 集群级别状态
tccli tke DescribeEKSClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]' | jq '.Clusters[0] | {ClusterStatus, ClusterVersion, ClusterName}'
# expected: ClusterStatus: "Running"

# 查看集群近期事件
tccli tke DescribeEKSClusterEvent --region <Region> \
    --ClusterId CLUSTER_ID | jq '.Events[:5]'
# expected: 列出最近 5 条事件
```

```json
{
  "TotalCount": "<TotalCount>",
  "Clusters": "<Clusters>",
  "ClusterId": "<ClusterId>",
  "ClusterName": "<ClusterName>",
  "VpcId": "<VpcId>",
  "SubnetIds": "<SubnetIds>"
}
```

### 步骤 2: 查看 Pod 资源使用

```bash
# 查看 Pod 实时资源使用
kubectl --kubeconfig ~/.kube/config-eks-CLUSTER_ID top pods -n NAMESPACE
# expected: 显示 CPU(cores) 和 MEMORY(bytes) 列

# 查看虚拟节点资源
kubectl --kubeconfig ~/.kube/config-eks-CLUSTER_ID top nodes
# expected: 显示虚拟节点资源使用
```

预期输出：

```
NAME                                CPU(cores)   MEMORY(bytes)
nginx-deployment-xxxx-xxxxx         10m          128Mi
nginx-deployment-xxxx-yyyyy         8m           120Mi
```

### 步骤 3: 配置 HPA 自动扩缩容

Serverless 集群中的 HPA 基于 metrics-server 提供的 CPU/内存指标自动扩缩容：

```bash
# 为 Deployment 创建 HPA
kubectl --kubeconfig ~/.kube/config-eks-CLUSTER_ID autoscale deployment/nginx-deployment \
    --min=1 \
    --max=10 \
    --cpu-percent=80 \
    -n NAMESPACE
# expected: horizontalpodautoscaler.autoscaling/nginx-deployment autoscaled

# 查看 HPA 状态
kubectl --kubeconfig ~/.kube/config-eks-CLUSTER_ID get hpa -n NAMESPACE
# expected: 显示 NAME, REFERENCE, TARGETS, MINPODS, MAXPODS, REPLICAS

# 查看 HPA 详情
kubectl --kubeconfig ~/.kube/config-eks-CLUSTER_ID describe hpa nginx-deployment -n NAMESPACE
# expected: 显示 Metrics、Conditions、Events
```

预期输出：

```
NAME               REFERENCE                     TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
nginx-deployment   Deployment/nginx-deployment   <unknown>/80%   1         10        2          1m
```

或使用 YAML 声明式创建：

`nginx-hpa.yaml`：

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: nginx-deployment
  namespace: NAMESPACE
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx-deployment
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 80
```

```bash
kubectl --kubeconfig ~/.kube/config-eks-CLUSTER_ID apply -f nginx-hpa.yaml
# expected: horizontalpodautoscaler.autoscaling/nginx-deployment configured
```

### 步骤 4: 集成 TMP Prometheus 监控（可选）

对于需要自定义指标、告警规则和长期监控数据的场景，可集成腾讯云托管 Prometheus（TMP）：

```bash
# 前提：已有 TMP 实例
tccli monitor DescribePrometheusInstances --region <Region>
# expected: 返回 InstanceId

# 关联 TKE Serverless 集群到 TMP
tccli monitor CreatePrometheusClusterAgent --region <Region> \
    --InstanceId TMP_INSTANCE_ID \
    --ClusterId CLUSTER_ID \
    --ClusterType tke-serverless
# expected: exit 0，关联成功后可在 TMP 实例中查看集群指标
```

### 步骤 5: 配置基于自定义指标的 HPA（TMP 集成后）

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: nginx-custom-hpa
  namespace: NAMESPACE
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx-deployment
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Pods
    pods:
      metric:
        name: nginx_ingress_controller_requests
      target:
        type: AverageValue
        averageValue: "100"
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `NAMESPACE` | Kubernetes 命名空间 | 需预先存在 | `kubectl get ns` |
| `CLUSTER_ID` | 集群 ID | `cls-xxxxxxxx` 格式 | `tccli tke DescribeEKSClusters --region <Region>` |
| `REGION` | 地域 | 如 `ap-guangzhou` | `tccli configure list` |
| `TMP_INSTANCE_ID` | TMP 实例 ID | 需预先创建 | `tccli monitor DescribePrometheusInstances --region <Region>` |

## 验证

| 维度 | 命令 | 预期 |
|------|------|------|
| 集群健康 | `tccli tke DescribeEKSClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'` | `ClusterStatus: Running` |
| metrics-server 运行 | `kubectl get deploy metrics-server -n kube-system` | READY 1/1 |
| Pod 资源可见 | `kubectl top pods -n NAMESPACE` | 显示 CPU 和 MEMORY 数值 |
| HPA 监控指标 | `kubectl get hpa -n NAMESPACE` | TARGETS 列不为 `<unknown>` |
| HPA 扩缩容事件 | `kubectl describe hpa nginx-deployment -n NAMESPACE \| grep -A5 Events` | 显示扩缩容事件 |

```bash
# 综合监控验证
tccli tke DescribeEKSClusters --region <Region> --ClusterIds '["CLUSTER_ID"]' | jq '.Clusters[0].ClusterStatus' && \
kubectl --kubeconfig ~/.kube/config-eks-CLUSTER_ID top pods -n NAMESPACE && \
kubectl --kubeconfig ~/.kube/config-eks-CLUSTER_ID get hpa -n NAMESPACE
# expected: 三个命令均 exit 0
```

```json
{
  "TotalCount": "<TotalCount>",
  "Clusters": "<Clusters>",
  "ClusterId": "<ClusterId>",
  "ClusterName": "<ClusterName>",
  "VpcId": "<VpcId>",
  "SubnetIds": "<SubnetIds>"
}
```

## 清理

> **警告**：HPA 删除后 Pod 副本数不会自动缩减，需手动 scale down。TMP 关联解除后历史数据仍保留在 TMP 实例中（至数据保留期结束）。

```bash
# 删除 HPA
kubectl --kubeconfig ~/.kube/config-eks-CLUSTER_ID delete hpa nginx-deployment -n NAMESPACE
# expected: horizontalpodautoscaler.autoscaling "nginx-deployment" deleted

# 解除 TMP 关联（如有集成）
tccli monitor DescribePrometheusClusterAgents --region <Region> \
    --InstanceId TMP_INSTANCE_ID
# 记录 AgentId，然后在 TMP 控制台或通过 API 解绑
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl top pods` 返回 `error: Metrics API not available` | `kubectl get deployment metrics-server -n kube-system` 检查部署 | metrics-server 未部署或异常 | 确认 metrics-server 状态：`kubectl get pods -n kube-system \| grep metrics-server` |
| `kubectl top pods` 返回 `metrics not available yet` | 等待 1-2 分钟重试 | metrics-server 刚启动，尚未采集数据 | 等待后重试，默认采集间隔为 60 秒 |
| HPA TARGETS 显示 `<unknown>/80%` | `kubectl describe hpa NAME -n NAMESPACE` 查看 Conditions | metrics-server 未就绪或 Pod 无资源定义 | 确认 metrics-server 运行正常，Pod 有 `eks.tke.cloud.tencent.com/cpu` annotation |
| HPA 不自动扩容 | `kubectl top pods -n NAMESPACE` 查看实际 CPU 使用 | 实际使用率未达到阈值 | 确认 HPA 阈值设置；检查 `kubectl describe hpa` 的 Events |
| `CreatePrometheusClusterAgent` 返回 `ResourceNotFound.ClusterId` | `tccli tke DescribeEKSClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'` | 集群不存在或 region 不对（此为操作错误） | 确认 CLUSTER_ID 和 REGION 匹配 |

### 操作成功但异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| metrics-server 运行但 `kubectl top pods` 无数据 | `kubectl logs deployment/metrics-server -n kube-system` 查看日志 | metrics-server 无法与 kubelet 通信 | 检查 metrics-server 配置，Serverless 集群通常无需手动处理 |
| HPA 频繁扩容又缩容（震荡） | `kubectl describe hpa NAME -n NAMESPACE` 查看 Events 和 Conditions | HPA 阈值设置过激或负载波动大 | 增大 `--horizontal-pod-autoscaler-tolerance` 或调整指标阈值 |
| 自定义指标 HPA 不工作 | `kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1` 检查 API | TMP adapter 未正确安装 | 确认 TMP 关联成功，prometheus-adapter 在集群中运行 |
| Prometheus 集成后无数据上报 | `kubectl get pods -n tke-monitor` 检查 agent 组件 | TMP agent 组件未正确安装 | 确认 `CreatePrometheusClusterAgent` 返回成功，检查 tke-monitor 命名空间下的 Pod 状态；验证集群 VPC 到 TMP 实例的网络连通性 |

### 补充排障（监控和告警专项）

#### Metrics 不显示

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl top pods` 返回空列表 | `kubectl get apiservice v1beta1.metrics.k8s.io` 检查 API 服务 | metrics-server 对应的 APIService 不可用 | 检查 metrics-server deployment 和 apiservice 状态：`kubectl get apiservice v1beta1.metrics.k8s.io -o yaml` |
| TMP 控制台无指标数据 | `kubectl get pods -n tke-monitor` 检查采集器 Pod | Prometheus agent 采集器异常 | `kubectl logs -n tke-monitor POD_NAME` 查看采集器日志 |
| 指标数据延迟大 | `kubectl top pods -n NAMESPACE` 多次采样对比 | metrics-server 采集间隔导致 | 正常延迟 60s；超过 5 分钟则检查 metrics-server 日志 |

#### HPA 不缩放

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| HPA 始终不扩容 | `kubectl describe hpa NAME -n NAMESPACE \| grep -A10 Conditions` | Pod 缺少 `resources.requests` 定义 | Serverless 集群中 Pod 使用 annotation 定义资源，确保 `eks.tke.cloud.tencent.com/cpu` 已设置 |
| HPA 缩容延迟 | 查看 HPA `--horizontal-pod-autoscaler-downscale-stabilization` 参数 | 默认缩容稳定窗口 5 分钟 | 正常行为；如需加速可在 HPA behavior 中配置 |
| 资源指标正常但 HPA 无动作 | `kubectl get hpa -n NAMESPACE -o yaml` 检查 `minReplicas` / `maxReplicas` | 当前副本数已达 min 或 max 边界 | 调整 minReplicas/maxReplicas 范围 |

#### Prometheus 集成失败

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `CreatePrometheusClusterAgent` 返回 `FailedOperation` | 检查 TMP 实例与集群是否同 VPC | TMP 实例与 Serverless 集群不在同一 VPC | 确认 TMP 实例创建时绑定的 VPC 与集群 VPC 一致；不一致则需重建 TMP 实例 |
| 关联成功但采集器 Pod CrashLoopBackOff | `kubectl describe pod -n tke-monitor POD_NAME` | 网络不通或权限不足 | 检查集群 VPC 安全组是否允许到 TMP 实例 IP 的出站流量 |
| TMP 告警规则不生效 | 在 TMP 控制台查看告警规则状态 | 告警规则语法错误或条件未满足 | 使用 PromQL 验证规则表达式，确认指标在 TMP 实例中可查询 |

## 下一步

- [事件管理](../事件管理/事件日志/tccli 操作.md) — 查看和管理集群事件
- [日志采集](../日志采集/tccli%20操作.md) — 配置 CLS 日志采集
- [审计管理](../审计管理/审计日志/tccli 操作.md) — 查看 Kubernetes 审计日志
- [常见问题](../../常见问题/tccli%20操作.md) — Serverless 集群常见问题

## 控制台替代

控制台：容器服务控制台 → 集群 → 选择目标 Serverless 集群 → 监控 → 查看集群监控面板、事件、HPA 配置。TMP 集成通过「运维功能 → Prometheus 监控」操作。
