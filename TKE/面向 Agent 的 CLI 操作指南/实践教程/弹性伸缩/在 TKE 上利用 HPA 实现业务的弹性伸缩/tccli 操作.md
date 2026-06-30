# 在 TKE 上利用 HPA 实现业务的弹性伸缩（tccli）

> 对照官方：[在 TKE 上利用 HPA 实现业务的弹性伸缩](https://cloud.tencent.com/document/product/457/50084) · page_id `50084`
## 概述

通过 Kubernetes Horizontal Pod Autoscaler（HPA）基于 CPU、内存或自定义指标自动调整 Pod 副本数，实现弹性伸缩。

> **kubectl 数据面不可达**：CAM拒绝公网端点(strategyId:240463971)，内网需VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

## 前置条件

- [环境准备](../../../环境准备.md)：`tccli` 已配置，`kubectl` >= v1.28
- 已获取集群 kubeconfig：

```bash
tccli tke DescribeClusterKubeconfig --region <Region> --ClusterId <ClusterId> \
    --filter "Kubeconfig" > kubeconfig.yaml
export KUBECONFIG=$(pwd)/kubeconfig.yaml
# expected: 生成 kubeconfig.yaml
```

```json
{
  "Kubeconfig": "<Kubeconfig>",
  "RequestId": "<RequestId>"
}
```

- 需要 CAM 权限：`tke:DescribeClusters`、`tke:DescribeClusterKubeconfig`

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 获取集群凭证 | `tccli tke DescribeClusterKubeconfig` | 是 |
| 查看集群状态 | `tccli tke DescribeClusterStatus --region <Region> --ClusterIds '["<ClusterId>"]'` | 是 |
| 相关 kubectl 操作 | 见 Procedure | -- |

## 操作步骤

### 1. 前置要求

- 已安装 metrics-server 组件
- 工作负载已配置 `resources.requests`

### 2. 控制面：获取集群凭证

```bash
tccli tke DescribeClusterKubeconfig --region <Region> --ClusterId <ClusterId> \
    --filter "Kubeconfig" > kubeconfig.yaml
export KUBECONFIG=$(pwd)/kubeconfig.yaml
# expected: kubeconfig 写入本地文件
```

### 3. 数据面：创建 Deployment（需 VPN/IOA）

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hpa-demo
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
          image: mirrors.tencent.com/google_containers/hpa-example:latest
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 200m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 256Mi
```

```bash
kubectl apply -f hpa-deployment.yaml
# expected: deployment.apps/hpa-demo created
```

### 4. 数据面：创建 HPA

```bash
kubectl autoscale deployment hpa-demo \
  --cpu-percent=50 \
  --min=1 \
  --max=10
# expected: horizontalpodautoscaler.autoscaling/hpa-demo autoscaled
```

或使用 YAML：

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: hpa-demo
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: hpa-demo
  minReplicas: 1
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50
```

| 参数 | 说明 | 推荐值 |
|------|------|--------|
| `minReplicas` | 最小副本数 | >= 2 |
| `maxReplicas` | 最大副本数 | 根据业务评估 |
| `averageUtilization` | 目标 CPU 利用率(%) | 50-70 |
| `stabilizationWindowSeconds` | 缩容稳定窗口 | 300 (5min) |

### 5. 数据面：查看 HPA 状态

```bash
kubectl get hpa
# expected: TARGETS 和 REPLICAS 列
```

预期输出：

```text
NAME       REFERENCE             TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
hpa-demo   Deployment/hpa-demo   5%/50%    1         10        1          5m
```

### 6. 数据面：生成负载触发扩容

```bash
# 创建负载 Pod 生成 CPU 压力
kubectl run -it --rm load-generator --image=busybox --restart=Never \
  -- /bin/sh -c "while true; do wget -q -O- http://hpa-demo; done"
# expected: 持续发送请求

# 观察 HPA 扩容
kubectl get hpa -w
# expected: REPLICAS 随时间增加
```

```text
NAME  STATUS  AGE
...
```

## 验证

### 控制面

```bash
tccli tke DescribeClusterStatus --region <Region> --ClusterIds '["<ClusterId>"]'
# expected: ClusterState "Running"
```

```json
{
  "ClusterStatusSet": "<ClusterStatusSet>",
  "ClusterId": "<ClusterId>",
  "ClusterState": "<ClusterState>",
  "ClusterInstanceState": "<ClusterInstanceState>",
  "ClusterBMonitor": "<ClusterBMonitor>",
  "ClusterInitNodeNum": "<ClusterInitNodeNum>"
}
```

### 数据面（需 VPN/IOA）

```bash
kubectl get hpa
# expected: HPA 状态正常

kubectl get pods -l app=hpa-demo
# expected: Pod 副本数按 HPA 调整
```

```text
NAME  STATUS  AGE
...
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl` 命令报错/无响应（本环境实测） | `tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]' --filter "Clusters[0].ClusterStatus"` | CAM 拒绝公网端点（strategyId:240463971），数据面不可达 | 接入 VPN/IOA 内网后执行 kubectl；控制面 tccli 不受影响 |
| HPA `TARGETS` 显示 `<unknown>` | `tccli tke DescribeAddon --region ap-guangzhou --ClusterId <ClusterId> --AddonName metrics-server` | metrics-server 未就绪或未安装 | 等待组件 Running；参考 [在 TKE 上安装 metrics-server](../../监控/在%20TKE%20上安装%20metrics-server/tccli%20操作.md) |
| HPA 不扩容 | `kubectl top pods` 查看实际使用率 | metrics 未达到阈值 | 持续施压使 CPU 超过 `averageUtilization` |
| HPA 频繁抖动 | `kubectl get hpa -w` 观察 REPLICAS 波动 | 阈值设置过低、缩容窗口过短 | 提高 targetUtilization；增大 `stabilizationWindowSeconds` |

## 清理

```bash
kubectl delete hpa hpa-demo
kubectl delete deployment hpa-demo
# expected: 资源删除成功
```

## 下一步

- [在 TKE 上安装 metrics-server](../../监控/在%20TKE%20上安装%20metrics-server/tccli%20操作.md) -- page_id `50074`
- [HPC 说明](../../../应用配置/组件和应用管理/组件管理/HPC%20说明/tccli%20操作.md) -- page_id `56753`
- [不同业务场景调节 HPA 扩缩容灵敏度](../../弹性伸缩/根据不同业务场景调节%20HPA%20扩缩容灵敏度/tccli%20操作.md)

## 控制台替代

[容器服务控制台](https://console.cloud.tencent.com/tke2/cluster)
