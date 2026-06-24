# 在 TKE 上使用自定义指标进行弹性伸缩（tccli）

> 对照官方：[在 TKE 上使用自定义指标进行弹性伸缩](https://cloud.tencent.com/document/product/457/50125) · page_id `50125`

## 概述

通过 Prometheus Adapter 将自定义指标（如请求 QPS、队列长度）暴露给 Kubernetes API，HPA 可基于这些指标进行弹性伸缩。

## 前置条件

- metrics-server 已安装
- Prometheus 已部署

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 安装 prometheus-adapter | `tccli tke InstallAddon --AddonName prometheus-adapter` | 是 |
| 创建 HPA 自定义指标 | `kubectl apply -f hpa-custom.yaml` | 是 |
| 查看自定义指标 | `kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1` | 是 |

## 操作步骤

### 1. 安装 Prometheus Adapter

```bash
tccli tke InstallAddon --region ap-guangzhou --cli-input-json "{\"ClusterId\":\"<ClusterId>\",\"AddonName\":\"prometheus-adapter\"}"
```

### 2. 验证自定义指标可用

```bash
kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1 | jq '.resources[].name'
```

### 3. 创建基于自定义指标的 HPA

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: app-custom-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "100"
```

## 验证

```bash
kubectl describe hpa app-custom-hpa
kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1/namespaces/default/pods/*/http_requests_per_second
```

```text
NAME  STATUS  AGE
...
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl` 命令报错/无响应（本环境实测） | `tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]' --filter "Clusters[0].ClusterStatus"` | CAM 拒绝公网端点（strategyId:240463971），数据面不可达 | 接入 VPN/IOA 内网后执行 kubectl；控制面 tccli 不受影响 |
| 自定义指标接口 404 | `tccli tke DescribeAddon --region ap-guangzhou --ClusterId <ClusterId> --AddonName prometheus-adapter` | prometheus-adapter 未安装或未就绪 | 安装/等待组件 Running，确认 Prometheus 数据源可达 |
| HPA `TARGETS` 显示 `<unknown>` | `kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1` | 指标名拼写错误或 Prometheus 未采集 | 核对 HPA `metric.name` 与 Prometheus 指标名一致 |

## 清理

```bash
kubectl delete hpa app-custom-hpa
```

## 下一步

- [HPA 弹性伸缩](../在%20TKE%20上利用%20HPA%20实现业务的弹性伸缩/tccli%20操作.md)
- [HPA 扩缩容灵敏度调节](../根据不同业务场景调节%20HPA%20扩缩容灵敏度/tccli%20操作.md)

## 控制台替代

控制台：工作负载 → HPA → 自定义指标模式。
