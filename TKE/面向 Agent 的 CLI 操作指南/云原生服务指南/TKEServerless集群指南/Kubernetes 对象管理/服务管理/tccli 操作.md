# 服务管理

> 对照官方：[服务管理](https://cloud.tencent.com/document/product/457/39817) · page_id `39817`

## 概述

TKE Serverless 集群中通过 Kubernetes Service 暴露工作负载。支持的 Service 类型包括 ClusterIP（集群内访问）和 LoadBalancer（公网/内网负载均衡）。LoadBalancer 类型会自动创建腾讯云 CLB 实例，公网 CLB 提供外部访问能力，内网 CLB 用于 VPC 内访问。Serverless 集群不支持 NodePort 类型，因为不存在物理节点。

**计费提醒**：LoadBalancer 类型 Service 创建的 CLB 实例单独计费，按 CLB 实例规格和使用量收费。

## 前置条件

- [环境准备](../../../../环境准备.md)

### 集群可达性检查（Before you begin）

服务操作为 kubectl-only 操作。确认集群连接正常后方可继续。

```bash
# 1. 确认集群处于 Running 状态
tccli tke DescribeEKSClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterStatus: Running

# 2. 验证 kubectl 连接
kubectl --kubeconfig ~/.kube/config-eks-CLUSTER_ID cluster-info
# expected: Kubernetes control plane is running at https://...

# 3. 确认已有工作负载在运行
kubectl --kubeconfig ~/.kube/config-eks-CLUSTER_ID get deployments -n NAMESPACE
# expected: 至少 1 个 Deployment 处于 READY 状态
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

| 控制台操作 | kubectl 命令 | 幂等 |
|-----------|-----------|:--:|
| 创建 ClusterIP Service | `kubectl expose deployment/NAME --port=80 --target-port=80 -n NAMESPACE` | 是 |
| 创建 LoadBalancer Service | `kubectl expose deployment/NAME --type=LoadBalancer --port=80 -n NAMESPACE` | 否 |
| 查看 Service | `kubectl get svc -n NAMESPACE` | 是 |
| 查看 Endpoints | `kubectl get endpoints -n NAMESPACE` | 是 |
| 删除 Service | `kubectl delete svc/NAME -n NAMESPACE` | 是 |

## 操作步骤

### 步骤 1: 创建 ClusterIP Service

ClusterIP 用于集群内部通信，不暴露到公网。

```bash
# 方式 A: kubectl expose 命令快速创建
kubectl --kubeconfig ~/.kube/config-eks-CLUSTER_ID expose deployment/nginx-deployment \
    --name=nginx-clusterip \
    --port=80 \
    --target-port=80 \
    -n NAMESPACE
# expected: service/nginx-clusterip exposed

# 方式 B: YAML 声明式创建
```

`nginx-clusterip-svc.yaml`：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-clusterip
  namespace: NAMESPACE
  labels:
    app: nginx
spec:
  type: ClusterIP
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
    protocol: TCP
```

```bash
kubectl --kubeconfig ~/.kube/config-eks-CLUSTER_ID apply -f nginx-clusterip-svc.yaml
# expected: service/nginx-clusterip created 或 unchanged
```

### 步骤 2: 创建 LoadBalancer Service（公网）

LoadBalancer 类型会自动创建腾讯云公网 CLB，分配公网 IP：

```bash
kubectl --kubeconfig ~/.kube/config-eks-CLUSTER_ID expose deployment/nginx-deployment \
    --name=nginx-lb \
    --type=LoadBalancer \
    --port=80 \
    --target-port=80 \
    -n NAMESPACE
# expected: service/nginx-lb exposed
```

或使用 YAML：

`nginx-lb-svc.yaml`：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-lb
  namespace: NAMESPACE
  annotations:
    service.cloud.tencent.com/direct-access: "true"
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
    protocol: TCP
```

```bash
kubectl --kubeconfig ~/.kube/config-eks-CLUSTER_ID apply -f nginx-lb-svc.yaml
# expected: service/nginx-lb created
```

### 步骤 3: 创建内网 LoadBalancer Service

通过 annotation `service.kubernetes.io/qcloud-loadbalancer-internal-subnetid` 创建内网 CLB：

`nginx-internal-lb-svc.yaml`：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-internal-lb
  namespace: NAMESPACE
  annotations:
    service.kubernetes.io/qcloud-loadbalancer-internal-subnetid: "SUBNET_ID"
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
    protocol: TCP
```

```bash
kubectl --kubeconfig ~/.kube/config-eks-CLUSTER_ID apply -f nginx-internal-lb-svc.yaml
# expected: service/nginx-internal-lb created
```

### 步骤 4: 查看 Service 状态

LoadBalancer 类型 Service 创建后需等待 CLB 分配 EXTERNAL-IP（通常 30 秒 - 2 分钟）：

```bash
kubectl --kubeconfig ~/.kube/config-eks-CLUSTER_ID get svc -n NAMESPACE
# expected: nginx-lb 的 EXTERNAL-IP 不为 <pending>
```

预期输出：

```
NAME                TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)        AGE
nginx-clusterip     ClusterIP      10.0.0.100      <none>           80/TCP         2m
nginx-lb            LoadBalancer   10.0.0.200      1.2.3.4          80:30000/TCP   1m
nginx-internal-lb   LoadBalancer   10.0.0.201      10.0.1.100       80:30001/TCP   30s
```

### 步骤 5: 验证 Endpoints

```bash
kubectl --kubeconfig ~/.kube/config-eks-CLUSTER_ID get endpoints -n NAMESPACE
# expected: 各 Service 对应的 Endpoints 有后端 Pod IP
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `NAMESPACE` | Kubernetes 命名空间 | 需预先存在 | `kubectl get ns` |
| `CLUSTER_ID` | 集群 ID | `cls-xxxxxxxx` 格式 | `tccli tke DescribeEKSClusters --region <Region>` |
| `REGION` | 地域 | 如 `ap-guangzhou` | `tccli configure list` |
| `SUBNET_ID` | 内网 CLB 子网 ID | 须与集群同 VPC | `tccli vpc DescribeSubnets --region <Region>` |

## 验证

| 维度 | 命令 | 预期 |
|------|------|------|
| Service 列表 | `kubectl get svc -n NAMESPACE` | 显示所有 Service，LB 类型有 EXTERNAL-IP |
| Endpoints | `kubectl get endpoints -n NAMESPACE` | 有后端 Pod IP |
| 集群内访问 | `kubectl run -it --rm debug --image=busybox:1.36 -n NAMESPACE -- wget -qO- nginx-clusterip` | 返回 nginx 欢迎页 |
| 公网访问 | `curl -s http://EXTERNAL_IP` | 返回 nginx 欢迎页 |
| CLB 查询 | `tccli clb DescribeLoadBalancers --region <Region>` | 包含自动创建的 CLB 实例 |

```bash
# 综合验证
kubectl --kubeconfig ~/.kube/config-eks-CLUSTER_ID get svc,endpoints -n NAMESPACE
# expected: Service 和 Endpoints 状态正常
```

## 清理

> **警告**：删除 LoadBalancer Service 会**同步删除**关联的 CLB 实例及其公网 IP。如 CLB 被多个 Service 共享，请确认后再操作。

```bash
# 删除 ClusterIP Service
kubectl --kubeconfig ~/.kube/config-eks-CLUSTER_ID delete svc/nginx-clusterip -n NAMESPACE
# expected: service "nginx-clusterip" deleted

# 删除 LoadBalancer Service（公网 CLB 将同步销毁）
kubectl --kubeconfig ~/.kube/config-eks-CLUSTER_ID delete svc/nginx-lb -n NAMESPACE
# expected: service "nginx-lb" deleted

# 删除内网 LoadBalancer Service
kubectl --kubeconfig ~/.kube/config-eks-CLUSTER_ID delete svc/nginx-internal-lb -n NAMESPACE
# expected: service "nginx-internal-lb" deleted

# 验证 CLB 已释放
tccli clb DescribeLoadBalancers --region <Region>
# expected: 目标 CLB 不再出现（可能延迟 1-2 分钟）
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl expose` 返回 `service already exists` | `kubectl get svc -n NAMESPACE` 检查 | Service 名称冲突（此为操作错误） | 删除已有 Service 或使用不同名称 |
| LoadBalancer EXTERNAL-IP 一直 `<pending>` | `kubectl describe svc/NAME -n NAMESPACE \| grep -A10 Events` | CLB 创建中或失败 | 检查 Events 错误信息；内网 LB 确认子网 ID 正确 |
| `kubectl expose` 返回 `cannot expose a deployment with no selector` | `kubectl describe deployment/NAME -n NAMESPACE` 检查标签 | Deployment 没有 selector | 确认 Deployment 的 `spec.selector.matchLabels` 已定义 |
| Endpoints 为空 | `kubectl describe svc/NAME -n NAMESPACE \| grep Endpoints` | Service selector 未匹配到 Pod | 确认 Service 的 `spec.selector` 与 Pod 的 `metadata.labels` 一致 |
| CLB 创建失败 `Failed to create load balancer` | `kubectl describe svc/NAME -n NAMESPACE` 查看 Events | 子网无可用 IP 或 CLB 配额不足（此为环境限制） | 检查子网 IP 余量；确认 CLB 配额 |

### 操作成功但异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 公网 LB 可访问但响应慢 | `curl -s -o /dev/null -w '%{time_total}' http://EXTERNAL_IP` 测试延迟 | 首次访问需建立连接 | 正常范围内（1-3 秒）；持续高延迟检查 Pod 资源和健康状态 |
| 内网 LB EXTERNAL-IP 显示公网 IP 而非内网 IP | 检查 Service annotation `service.kubernetes.io/qcloud-loadbalancer-internal-subnetid` | 缺少内网子网 annotation | 添加内网子网 annotation 重新创建 Service |
| 删除 Service 后 CLB 未自动删除 | `tccli clb DescribeLoadBalancers --region <Region>` 检查 | CLB 自动删除有 1-2 分钟延迟 | 等待后确认；若持续残留，手动通过 CLB 控制台删除 |

## 下一步

- [其他资源管理](../其他资源管理/tccli%20操作.md) — 管理 ConfigMap、Secret 等配置资源
- [监控和告警](../../运维中心/监控和告警/tccli%20操作.md) — 为 Service 配置监控
- [工作负载管理](../工作负载管理/tccli%20操作.md) — 返回工作负载管理

## 控制台替代

控制台：容器服务控制台 → 集群 → 选择目标 Serverless 集群 → 服务与路由 → Service → 新建。
