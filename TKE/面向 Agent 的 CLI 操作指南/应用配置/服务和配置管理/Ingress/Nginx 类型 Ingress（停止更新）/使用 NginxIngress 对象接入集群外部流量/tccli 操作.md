# 使用 NginxIngress 对象接入集群外部流量（tccli）

> 对照官方：[使用 NginxIngress 对象接入集群外部流量](https://cloud.tencent.com/document/product/457/50504) · page_id `50504`

## 概述

> **注意：** NginxIngress 扩展组件已停止更新，详情见 [NginxIngress 扩展组件停止更新公告](https://cloud.tencent.com/document/product/457/108517)。

通过创建 Nginx 类型的 Ingress 对象，将集群外部流量路由到集群内部的 Service。Nginx Ingress 通过 Annotations 扩展了原生 Kubernetes Ingress 的功能。

## 前置条件

- [环境准备](../../../../../环境准备.md)
- 已 [连接集群](../../../../../集群配置/集群管理/连接集群/tccli 操作.md)，kubectl 可正常访问 APIServer
- 集群内已 [安装 NginxIngress 实例](../安装%20NginxIngress%20实例/tccli 操作.md)

## 控制台与 CLI 参数映射

| Console 操作 | CLI 等价操作 | 幂等 | 说明 |
|---|---|---|---|------|
| 创建 Nginx 类型 Ingress | `kubectl create -f <ingress>.yaml` | 否(同名报错) | Class 选 NginxIngress 实例 |
| 指定 IngressClass | `kubernetes.io/ingress.class: "nginx-public"` | 否(首次) | 对应 NginxIngress 实例名称 |
| 查看 Ingress | `kubectl get ingress` | 是 | 查看状态 |
| 更新 Ingress | `kubectl edit ingress/<name>` | 否 | 编辑规则 |

## 操作步骤

### 创建 Nginx 类型 Ingress

```yaml
# nginx-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: "nginx-public"
  name: my-nginx-ingress
  namespace: default
spec:
  rules:
    - host: <domain>
      http:
        paths:
          - backend:
              service:
                name: <service-name>
                port:
                  number: 80
            path: /
            pathType: ImplementationSpecific
```

> `kubernetes.io/ingress.class` 的值对应 TKE 集群 NginxIngress 组件中的 NginxIngress 实例名称。

```bash
kubectl create -f nginx-ingress.yaml
```

### NginxIngress 对象使用模型

| 规则 | 说明 |
|---|---|
| 排序规则 | 多个 Ingress 作用于同一 Nginx 实体时，按 CreationTimestamp 排序（旧规则优先） |
| 路径冲突 | 同一主机相同路径，最早的规则获胜 |
| TLS 冲突 | 同一主机 TLS 部分，最早的规则获胜 |
| Server 块注解冲突 | 最早定义的注解获胜 |
| Server 合并 | 按 hostname 创建 NGINX Server，同一 host 不同路径合并 |
| 注解作用域 | Ingress 的注解应用于该 Ingress 中的所有路径 |

### 触发更新 nginx.conf 的场景

- 创建新的 Ingress 对象
- 为 Ingress 添加新的 TLS
- Ingress 注解的更改（影响 server 配置的）
- 为 Ingress 添加/删除路径
- 删除 Ingress、Service、Secret
- Ingress 关联对象状态变化（Service 或 Secret）
- 更新 Secret

## 验证

### 数据面（kubectl）

```bash
kubectl get ingress my-nginx-ingress
kubectl describe ingress my-nginx-ingress
```

```text
NAME  STATUS  AGE
...
```

```bash
# 查看 Nginx Ingress Controller Pod
kubectl get pods -n kube-system | grep nginx-ingress
```

```text
NAME  STATUS  AGE
...
```

## 清理

### 数据面（kubectl）

```bash
kubectl delete ingress my-nginx-ingress
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `Unable to connect to the server` | `kubectl cluster-info` | 公网端点被 CAM 策略 strategyId:240463971 以 `tke:clusterExtranetEndpoint=true` 条件拒绝 | 通过 IOA/VPN 连接内网，或使用同 VPC CVM 执行 kubectl |
| Ingress 未生效 | `ingress.class` | NginxIngress 实例已安装 | 确认 NginxIngress 实例已安装，`ingress.class` 与实例名称匹配 |
| 路径冲突 | `kubectl describe <resource>` | 是否有更早创建的 Ingress 定义了相同的主机和路径 | 检查是否有更早创建的 Ingress 定义了相同的主机和路径 |
| nginx.conf 未更新 | `kubectl describe <resource>` | 是否触发了更新条件 | 检查是否触发了更新条件 |

## 下一步

- [安装 NginxIngress 实例](../安装%20NginxIngress%20实例/tccli 操作.md)
- [NginxIngress 日志配置](../NginxIngress%20日志配置/tccli 操作.md)
- [Ingress Controllers 说明](../../Ingress%20Controllers%20说明/tccli 操作.md)

## 控制台替代

在控制台 **新建 Ingress** 中选择 Ingress 类型为 **Nginx Ingress Controller**，选择一个 NginxIngress 实例。
