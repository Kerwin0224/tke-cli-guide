# 在 TKE 上使用负载均衡直连 Pod（tccli）

> 对照官方：[在 TKE 上使用负载均衡直连 Pod](https://cloud.tencent.com/document/product/457/48793) · page_id `48793`

## 概述

CLB 直连 Pod 模式下，负载均衡直接绑定 Pod IP + Port，跳过节点 NodePort 转发。相比传统 NodePort 方式，消除 SNAT 性能损耗，天然保留客户端源 IP，简化会话保持配置。

## 前置条件

- [环境准备](../../../环境准备.md)
- Kubernetes 集群版本 ≥ 1.12
- Pod 需配置 ReadinessGate（`service.cloud.tencent.com/enable-readiness-gate: "true"`）确保 CLB 健康检测

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 创建直连 Service | `kubectl apply -f` (annotation `direct-access: "true"`) | 是 |
| 查看 Service 状态 | `kubectl get svc <name>` | 是 |
| 配置健康检查 | TkeServiceConfig CRD `spec.loadBalancer.l4Listeners[].healthCheck` | 是 |
| 配置会话保持 | TkeServiceConfig CRD `spec.loadBalancer.l4Listeners[].session` | 是 |
| 删除 Service | `kubectl delete svc <name>` | 否 |

## 操作步骤

### 1. 部署测试 Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-direct-pod
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-direct
  template:
    metadata:
      labels:
        app: nginx-direct
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
```

```bash
kubectl apply -f nginx-direct-deploy.yaml
```

```text
# command executed successfully
```

```output
deployment.apps/nginx-direct-pod created
```

### 2. 创建 CLB 直连 Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-direct-svc
  annotations:
    service.cloud.tencent.com/direct-access: "true"
spec:
  type: LoadBalancer
  selector:
    app: nginx-direct
  ports:
  - port: 80
    targetPort: 80
    protocol: TCP
```

```bash
kubectl apply -f nginx-direct-svc.yaml
```

```text
# command executed successfully
```

```output
service/nginx-direct-svc created
```

### 3. 验证 CLB 分配

```bash
kubectl get svc nginx-direct-svc
```

```text
NAME  STATUS  AGE
...
```

```output
NAME               TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
nginx-direct-svc   LoadBalancer   10.0.0.100      lb-example    80:30123/TCP   30s
```

### 4. 访问验证

```bash
curl http://<EXTERNAL-IP>
```

```output
<!DOCTYPE html><html><head><title>Welcome to nginx!</title>...
```

## 验证

```bash
kubectl get svc nginx-direct-svc -o jsonpath='{.metadata.annotations}'
kubectl get endpoints nginx-direct-svc
```

```text
NAME  STATUS  AGE
...
```

## 清理

```bash
kubectl delete svc nginx-direct-svc
kubectl delete deployment nginx-direct-pod
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl` 命令超时或连接被拒绝 | `tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]'`（控制面返回 Running） | CAM 策略 strategyId:240463971 拒绝公网端点，数据面需内网访问 | 通过 VPN/IOA 接入 VPC 内网后执行 kubectl；或使用控制台替代 |
| EXTERNAL-IP 为空 | `kubectl get svc <svc> -w` | CLB 正在创建中 | 等待 CLB 创建完成（约30秒） |
| 访问超时 | `tccli vpc DescribeSecurityGroups` 检查安全组规则 | 安全组未放通 80 端口 | 检查安全组是否放通 80 端口 |
| `direct-access` 不生效 | `tccli tke DescribeCluster --ClusterId <ClusterId> --filter "Cluster.NetworkSettings.Cni"` | 集群非 VPC-CNI 模式 | 确认集群 VPC-CNI 模式或使用已有 CLB |

## 下一步

- [在 TKE 中获取客户端真实源 IP](../在%20TKE%20中获取客户端真实源%20IP/tccli%20操作.md)
- [TKE 基于弹性网卡直连 Pod 的网络负载均衡](../TKE%20基于弹性网卡直连%20Pod%20的网络负载均衡/tccli%20操作.md)

## 控制台替代

控制台：Service → 新建 → 访问方式选择"负载均衡" → 勾选"直连 Pod"。
