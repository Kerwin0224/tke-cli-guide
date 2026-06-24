# 应用高可用部署（tccli）

> 对照官方：[应用高可用部署](https://cloud.tencent.com/document/product/457/40212) · page_id `40212`
## 概述

通过 Pod 反亲和（podAntiAffinity）、多副本、跨可用区部署等策略，在 TKE 中实现应用高可用部署，避免单点故障。

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

### 1. 控制面：获取集群凭证

```bash
tccli tke DescribeClusterKubeconfig --region <Region> --ClusterId <ClusterId> \
    --filter "Kubeconfig" > kubeconfig.yaml
export KUBECONFIG=$(pwd)/kubeconfig.yaml
# expected: kubeconfig 写入本地文件
```

### 2. 数据面：创建高可用 Deployment（需 VPN/IOA）

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ha-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: ha-app
  template:
    metadata:
      labels:
        app: ha-app
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: app
                    operator: In
                    values:
                      - ha-app
              topologyKey: kubernetes.io/hostname
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              preference:
                matchExpressions:
                  - key: topology.kubernetes.io/zone
                    operator: In
                    values:
                      - ap-guangzhou-3
                      - ap-guangzhou-4
      containers:
        - name: nginx
          image: nginx:alpine
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
```

```bash
kubectl apply -f ha-deployment.yaml
# expected: deployment.apps/ha-app created
```

### 3. 数据面：验证 Pod 分布

```bash
kubectl get pods -l app=ha-app -o wide
# expected: 3 Pods 分布在不同 Node/Zone
```

### 4. 数据面：创建多副本 Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: ha-app-svc
spec:
  type: LoadBalancer
  selector:
    app: ha-app
  ports:
    - port: 80
      targetPort: 80
```

```bash
kubectl apply -f ha-service.yaml
# expected: service/ha-app-svc created
```

### 高可用策略总结

| 策略 | 实现方式 | 防护 |
|------|----------|------|
| 多副本 | `replicas >= 2` | Pod 级故障 |
| 反亲和 | `podAntiAffinity` | 单节点故障 |
| 跨可用区 | `nodeAffinity` 多 Zone | 可用区故障 |
| 就绪探针 | `readinessProbe` | 流量路由到健康 Pod |
| PDB | `PodDisruptionBudget` | 主动驱逐保护 |

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
kubectl get pods -l app=ha-app -o wide
# expected: 3 Pods Running，分布在不同 Node
```

```text
NAME  STATUS  AGE
...
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl` 连接超时/拒绝 | `tccli tke DescribeClusters` 返回集群 Running 但 kubectl 不可达 | CAM 策略 240463971 拒绝公网访问 APIServer 端点 | 通过 VPN/IOA 接入内网后执行 kubectl 命令 |
| Pod 集中在同一节点 | `kubectl get pods -l app=ha-app -o wide` 查看 NODE 列 | podAntiAffinity 的 label selector 或 topologyKey 配置错误 | 检查 `podAntiAffinity` label selector 和 `topologyKey: kubernetes.io/hostname` |
| 跨可用区调度失败 | `kubectl describe pod <pod>` 查看 Events | nodeAffinity 指定的 zone 无可用节点或资源不足 | 扩容节点或降低 `resources.requests`；调整 zone 列表 |

## 清理

```bash
kubectl delete deployment ha-app
kubectl delete service ha-app-svc
# expected: 资源删除成功
```

## 下一步

- [工作负载平滑升级](../../发布/工作负载平滑升级/tccli%20操作.md) -- page_id `77969`
- [设置工作负载的调度规则](../../../应用配置/工作负载管理/设置工作负载的调度规则/tccli%20操作.md) -- page_id `32814`

## 控制台替代

[容器服务控制台](https://console.cloud.tencent.com/tke2/cluster)
