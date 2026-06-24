# Ingress 配置双算法证书（tccli）

> 对照官方：[Ingress 配置双算法证书](https://cloud.tencent.com/document/product/457/115386) · page_id `115386`

## 概述

TKE Ingress 支持为一个域名配置两个不同算法类型的证书（如 ECC 和 RSA），通过 Opaque 类型的 Secret 以逗号分隔多个证书 ID。

## 前置条件

- [环境准备](../../../../../环境准备.md)
- 已 [连接集群](../../../../../集群配置/集群管理/连接集群/tccli 操作.md)，kubectl 可正常访问 APIServer
- 已准备好两个不同算法类型的 SSL 证书

## 控制台与 CLI 参数映射

| Console 操作 | CLI 等价操作 | 说明 |
|---|---|---|
| 创建 Opaque Secret | `kubectl create -f <secret>.yaml` | 证书 ID 逗号分隔 |
| 创建 Ingress | `kubectl create -f <ingress>.yaml` | spec.tls 引用 Secret |
| 更换证书 | `kubectl edit secret <name>` | 手动更新证书 ID |

## 操作步骤

> **注意：**
> - Secret 最多支持两个不同算法类型的证书
> - Secret 类型必须为 `Opaque`
> - 使用双算法证书的 Secret 暂不支持 SSL 控制台自动更新，需手动替换证书 ID
> - 使用双算法证书的 Secret 暂时只支持配置给 Ingress，不支持特殊协议监听器
> - Secret 需与 Ingress 在同一命名空间
> - ingress-controller 组件版本最低要求 v2.5.0

### 1. 创建 Opaque 类型 Secret

#### 方式 1：使用 stringData（自动 BASE64 编码）

```yaml
apiVersion: v1
stringData:
  qcloud_cert_id: <CertID1>,<CertID2>
kind: Secret
metadata:
  name: kateway-cert
type: Opaque
```

#### 方式 2：手动 BASE64 编码

```yaml
apiVersion: v1
data:
  qcloud_cert_id: <base64-encoded CertID1,CertID2>
kind: Secret
metadata:
  name: kateway-cert
type: Opaque
```

```bash
kubectl create -f kateway-cert.yaml
```

### 2. 部署工作负载

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx:latest
        imagePullPolicy: Always
        name: nginx
```

### 3. 创建 NodePort Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
    service: nginx
spec:
  ports:
  - name: http
    port: 80
    targetPort: 80
  selector:
    app: nginx
  type: NodePort
```

### 4. 创建 Ingress 关联双算法证书 Secret

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx
spec:
  rules:
  - host: <domain>
    http:
      paths:
      - backend:
          service:
            name: nginx
            port:
              number: 80
        path: /
        pathType: ImplementationSpecific
  tls:
  - hosts:
    - <domain>
    secretName: kateway-cert
```

```bash
kubectl apply -f nginx-deployment.yaml
kubectl apply -f nginx-service.yaml
kubectl apply -f nginx-ingress.yaml
```

```text
# command executed successfully
```

## 验证

### 数据面（kubectl）

```bash
kubectl get secret kateway-cert
kubectl describe ingress nginx
```

```text
NAME  STATUS  AGE
...
```

确认 Ingress 状态正常且 CLB HTTPS 监听器已绑定双算法证书。

## 清理

### 数据面（kubectl）

```bash
kubectl delete ingress nginx
kubectl delete service nginx
kubectl delete deployment nginx
kubectl delete secret kateway-cert
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `Unable to connect to the server` | `kubectl cluster-info` | 公网端点被 CAM 策略 strategyId:240463971 以 `tke:clusterExtranetEndpoint=true` 条件拒绝 | 通过 IOA/VPN 连接内网，或使用同 VPC CVM 执行 kubectl |
| 双证书不生效 | `kubectl get secret` | ingress-controller 版本 ≥ v2.5.0 | 确认 ingress-controller 版本 ≥ v2.5.0 |
| Secret 类型错误 | `Opaque` | 必须为 `Opaque` 类型 | 必须为 `Opaque` 类型 |
| 证书更换后未同步 | `kubectl get secret` | 双算法证书 Secret 暂不支持自动更新 | 双算法证书 Secret 暂不支持自动更新，需手动修改 Secret 的证书 ID |

## 下一步

- [Ingress 证书配置](../Ingress%20证书配置/tccli 操作.md)
- [多 Ingress 复用 CLB](../多%20Ingress%20复用%20CLB/tccli 操作.md)

## 控制台替代

控制台支持在新建 Secret 时配置双算法证书（证书 ID 用英文逗号分隔），再在 Ingress 中引用。
