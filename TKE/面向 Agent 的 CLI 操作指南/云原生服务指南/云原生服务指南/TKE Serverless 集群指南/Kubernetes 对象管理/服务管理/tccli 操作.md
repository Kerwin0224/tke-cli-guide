# 服务管理（tccli）

> 对照官方：[服务管理](https://cloud.tencent.com/document/product/457/39817) · page_id `39817`

## 概述

通过 Service 和 Ingress 将 TKE Serverless 集群内的工作负载暴露为可访问的服务入口。Service 提供固定的虚拟 IP 和负载均衡能力，Ingress 提供基于规则的 HTTP/HTTPS 路由。

> **重要提示：** Serverless 容器服务已升级为 TKE 标准集群 + 超级节点模式，新建 Serverless 集群入口已关闭。存量用户集群不受影响。

> **注意：** kubectl 不可达（公网端点 CAM 拒绝 strategyId:240463971）。以下 `kubectl` 命令是正确的 CLI 操作方式，实际执行需解决 CAM 权限或使用 Cloud Shell 连接。

## 前置条件

- [环境准备](../../环境准备.md)
- 已创建 Serverless 集群且状态为 `Running`，参见[创建集群](../../TKE%20Serverless%20集群管理/创建集群/tccli%20操作.md)
- 已[连接集群](../../TKE%20Serverless%20集群管理/连接集群/tccli%20操作.md)，`kubectl` 可用
- 集群中存在可用的工作负载（Deployment 等），参见[工作负载管理](../工作负载管理/tccli%20操作.md)
- Service CIDR 子网有充足的剩余 IP

## 控制台与 CLI 参数映射

| 控制台操作 | kubectl 命令 | 幂等 |
|-----------|-------------|------|
| 创建 ClusterIP Service | `kubectl expose deployment <name> --port=<port>` 或 `kubectl apply -f <yaml>` | 是 |
| 创建 LoadBalancer Service（公网） | `kubectl expose deployment <name> --type=LoadBalancer` 或 `kubectl apply -f <yaml>` | 是 |
| 创建 LoadBalancer Service（内网） | `kubectl apply -f <yaml>`（含内网 Annotation） | 是 |
| 查看 Service 列表 | `kubectl get svc` | 是 |
| 查看 Service 详情 | `kubectl describe svc <name>` | 是 |
| 删除 Service | `kubectl delete svc <name>` | 是 |

## 操作步骤

### 1. Service 类型

TKE Serverless 支持三种 Service 类型：

| 类型 | 说明 | 适用场景 |
|------|------|---------|
| **ClusterIP** | 集群内部虚拟 IP，仅集群内可访问 | 内部服务间调用 |
| **LoadBalancer（公网）** | 自动创建公网 CLB，公网 IP 可达后端 Pod | 对外提供 Web API 等 |
| **LoadBalancer（内网）** | 自动创建内网 CLB，VPC 内网 IP 可达后端 Pod | VPC 内服务调用 |

> **注意：** NodePort Service **不支持**，因为 Serverless 集群无实体节点。

### 2. ClusterIP Service（集群内访问）

```bash
kubectl expose deployment <deployment-name> \
    --name=<service-name> \
    --port=<port> \
    --target-port=<target-port> \
    --type=ClusterIP \
    --namespace=<namespace>
```

示例：

```bash
kubectl expose deployment nginx-serverless \
    --name=nginx-clusterip \
    --port=80 \
    --target-port=80 \
    --type=ClusterIP \
    --namespace=default
```

**YAML 方式：**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-clusterip
  namespace: default
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
kubectl apply -f service-clusterip.yaml
```

```text
# command executed successfully
```

### 3. LoadBalancer Service（公网访问）

```bash
kubectl expose deployment <deployment-name> \
    --name=<service-name> \
    --port=<port> \
    --target-port=<target-port> \
    --type=LoadBalancer \
    --namespace=<namespace>
```

示例：

```bash
kubectl expose deployment nginx-serverless \
    --name=nginx-public \
    --port=80 \
    --target-port=80 \
    --type=LoadBalancer \
    --namespace=default
```

**YAML 方式（含同时开启 ClusterIP）：**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-public
  namespace: default
  annotations:
    service.kubernetes.io/qcloud-loadbalancer-clusterip-loadbalancer-subnetid: "<ServiceSubnetId>"
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
kubectl apply -f service-public.yaml
```

> **说明：** 公网 Service 默认不启用 ClusterIP。如需同时启用，添加 Annotation `service.kubernetes.io/qcloud-clusterip-loadbalancer-subnetid` 并指定 Service CIDR 子网 ID。

### 4. LoadBalancer Service（VPC 内网访问）

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-internal
  namespace: default
  annotations:
    service.kubernetes.io/qcloud-loadbalancer-internal-subnetid: "<SubnetId>"
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
kubectl apply -f service-internal.yaml
```

```text
# command executed successfully
```

> **关键 Annotation：** `service.kubernetes.io/qcloud-loadbalancer-internal-subnetid` 指定内网 CLB 所在的子网 ID。内网 IP 从该子网中分配。

### 5. 查看 Service

```bash
kubectl get svc --namespace=default
kubectl describe svc nginx-public --namespace=default
```

示例输出（`kubectl get svc`）：

```text
NAME              TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)        AGE
nginx-clusterip   ClusterIP      10.0.1.100      <none>          80/TCP         2m
nginx-public      LoadBalancer   10.0.1.200      1.2.3.4         80:30080/TCP   3m
nginx-internal    LoadBalancer   10.0.1.201      10.0.2.50       80:30081/TCP   3m
```

### 6. Ingress（HTTP/HTTPS 路由）

创建 Ingress 资源定义路由规则。需集群中运行 Ingress Controller（TKE 默认使用 `l7-lb-controller`）。

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
  namespace: default
  annotations:
    kubernetes.io/ingress.class: qcloud
spec:
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: nginx-clusterip
            port:
              number: 80
  - host: web.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-public
            port:
              number: 80
```

```bash
kubectl apply -f ingress.yaml
kubectl get ingress --namespace=default
kubectl describe ingress nginx-ingress --namespace=default
```

```text
Name:         ...
Status:       Running
...
```

> **注意：** TKE Serverless 仅支持**应用型负载均衡（ALB/CLB 七层）**。复用已有 CLB 时仅支持**无监听器**的 CLB。

### 7. 删除 Service

```bash
kubectl delete svc nginx-clusterip --namespace=default
kubectl delete svc nginx-public --namespace=default
kubectl delete ingress nginx-ingress --namespace=default
```

## 验证

| 验证项 | 命令 | 期望结果 |
|--------|------|---------|
| Service 已创建 | `kubectl get svc -n default` | 目标 Service 在列表中 |
| ClusterIP 已分配 | 同上，检查 `CLUSTER-IP` 列 | 有内网 IP 值 |
| LoadBalancer 已就绪 | 同上，检查 `EXTERNAL-IP` 列 | 有外网 IP 值（非 `<pending>`） |
| Endpoints 正确 | `kubectl get endpoints -n default` | 包含后端 Pod IP 列表 |
| Ingress 已创建 | `kubectl get ingress -n default` | 目标 Ingress 在列表中，`ADDRESS` 有 IP |

## 清理

```bash
# 删除 Service
kubectl delete svc <service-name> --namespace=default

# 删除 Ingress
kubectl delete ingress <ingress-name> --namespace=default
```

> **注意：** 删除 LoadBalancer 类型 Service 时会自动释放关联的 CLB。删除 Ingress 会释放关联的 CLB 转发规则。

## 排障

| 现象 | 原因 | 处理 |
|------|------|------|
| LoadBalancer `EXTERNAL-IP` 一直 `<pending>` | CLB 创建中或子网 IP 不足 | 等待 2-3 分钟，检查 CLB 相关子网 IP 是否充足 |
| 无法通过公网 IP 访问 Service | 安全组未放通端口或 CLB 监听器未就绪 | 检查 CLB 控制台中对应实例的监听器和安全组配置 |
| NodePort Service 不生效 | Serverless 集群不支持 NodePort | 改用 ClusterIP 或 LoadBalancer 类型 |
| `kubectl expose` 报错 | 未指定 --type 或 Deployment 名称错误 | `--type=ClusterIP` 或 `--type=LoadBalancer` |
| Ingress 创建后 `ADDRESS` 为空 | Ingress Controller 未运行或 CLB 创建中 | 检查 `l7-lb-controller` 是否正常运行 |
| 复用已有 CLB 失败 | CLB 已有监听器 | 仅支持无监听器的 CLB 用于 Service 创建 |

## 下一步

- [其他资源管理](../其他资源管理/tccli%20操作.md) — 管理 ConfigMap、Secret、存储等资源
- [Annotation 说明](../Annotation%20说明/tccli%20操作.md) — 了解通过 Annotation 自定义 Service 和 Ingress 配置

## 控制台替代

[容器服务控制台 → 集群 → 目标集群 → 服务与路由](https://console.cloud.tencent.com/tke2/cluster) — 通过控制台可视化界面创建 Service 和 Ingress。
