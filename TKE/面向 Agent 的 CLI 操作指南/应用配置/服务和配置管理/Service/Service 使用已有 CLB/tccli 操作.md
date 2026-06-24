# Service 使用已有 CLB（tccli）

> 对照官方：[Service 使用已有 CLB](https://cloud.tencent.com/document/product/457/45491) · page_id `45491`
## 概述

将已有腾讯云 CLB 实例绑定到 Kubernetes Service，复用已配置的 CLB 实例（域名、证书、安全组等）。通过 Annotation `service.kubernetes.io/loadbalance-id` 指定已有 CLB ID。

> **kubectl 数据面不可达**：CAM拒绝公网端点(strategyId:240463971)，内网需VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。
## 前置条件

- [环境准备](../../../../环境准备.md)：`tccli` 已配置，`kubectl` >= v1.28
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

- 需要 CAM 权限：`tke:DescribeClusterKubeconfig`

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 获取集群凭证 | `tccli tke DescribeClusterKubeconfig` | 是 |
| 创建 Service | `kubectl apply -f service.yaml` | 是 |
| 查看 Service 列表 | `kubectl get svc` | 是 |
| 查看 Service 详情 | `kubectl describe svc/<name>` | 是 |
| 删除 Service | `kubectl delete svc/<name>` | 是 |

## 操作步骤

### 1. 控制面：获取集群凭证

```bash
tccli tke DescribeClusterKubeconfig --region <Region> --ClusterId <ClusterId> \
    --filter "Kubeconfig" > kubeconfig.yaml
export KUBECONFIG=$(pwd)/kubeconfig.yaml
# expected: kubeconfig 写入本地文件
```

### 2. 数据面：创建 Service（需 VPN/IOA）

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-existing-lb
  annotations:
    service.kubernetes.io/loadbalance-id: lb-example
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
kubectl apply -f service.yaml
# expected: service/<name> created
```

### 前提条件

- 已有 CLB 实例，记录其 `LoadBalancerId`（如 `lb-example`）
- 通过 tccli 查询已有 CLB：

```bash
tccli clb DescribeLoadBalancers --region <Region>
# expected: 返回 LoadBalancerSet，含 LoadBalancerId
```

### 注意事项

- Service 删除时不会删除已有 CLB（仅解绑）
- 已有 CLB 的监听器端口不能与 Service 端口冲突
- 跨 VPC 绑定时需确认网络可达

### 3. 数据面：验证 Service

```bash
kubectl get svc
# expected: 含目标 Service

kubectl describe svc/<name>
# expected: 显示 Service 详情、Endpoints
```

```text
NAME  STATUS  AGE
...
```

## 验证

### 控制面（tccli）

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

### 数据面（kubectl，需 VPN/IOA）

```bash
kubectl get svc
# expected: 目标 Service 状态正常

kubectl get endpoints <name>
# expected: 有后端 Pod IP
```

```text
NAME  STATUS  AGE
...
```

## 清理

```bash
kubectl delete svc/<name>
# expected: service "<name>" deleted
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Service EXTERNAL-IP `<pending>` | `kubectl get svc -w` | CLB 创建中 | 等待 1–2 分钟 |
| Endpoints 为空 | `kubectl get pods --show-labels` | selector 不匹配 Pod | `kubectl get pods --show-labels` 确认标签 |
| `connection refused` | `kubectl get pods` | 后端 Pod 未就绪 | `kubectl get pods` 确认 Running |

## 下一步

- [Service 概述](../../../../应用配置/服务和配置管理/Service/概述/tccli%20操作.md) -- page_id `45487`
- [Ingress 概述](../../../../应用配置/服务和配置管理/Ingress/概述/tccli%20操作.md) -- page_id `45685`

## 控制台替代

[容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) -> 集群 -> 服务与路由 -> Service -> 新建。
