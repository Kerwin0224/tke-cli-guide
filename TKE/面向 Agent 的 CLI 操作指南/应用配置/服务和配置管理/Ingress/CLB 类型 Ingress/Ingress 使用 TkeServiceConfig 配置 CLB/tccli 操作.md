# Ingress 使用 TkeServiceConfig 配置 CLB（tccli）

> 对照官方：[Ingress 使用 TkeServiceConfig 配置 CLB](https://cloud.tencent.com/document/product/457/45700) · page_id `45700`

## 概述

TkeServiceConfig 是 TKE 提供的自定义 CRD 资源，通过它可以对 Ingress 关联的 CLB 进行更精细的配置（健康检查、会话保持、转发策略等），弥补 Ingress YAML 语义无法定义的负载均衡参数。

## 前置条件

- [环境准备](../../../../../环境准备.md)
- 已 [连接集群](../../../../../集群配置/集群管理/连接集群/tccli 操作.md)，kubectl 可正常访问 APIServer

## 控制台与 CLI 参数映射

| Console 操作 | CLI 等价操作 | 幂等 | 说明 |
|---|---|---|---|------|
| 设置健康检查 | TkeServiceConfig `healthCheck` 字段 | 是(apply) | 配置健康检查参数 |
| 设置会话保持 | TkeServiceConfig `session` 字段 | 是(apply) | 配置会话保持超时 |
| 设置转发策略 | TkeServiceConfig `scheduler` 字段 | 是(apply) | WRR / LEAST_CONN / IP_HASH |
| 自定义监听器名称 | TkeServiceConfig `listenerName` 字段 | 是(apply) | 需组件版本 ≥ 2.11.0 |
| 引用配置 | annotation `ingress.cloud.tencent.com/tke-service-config: <name>` | 是(apply) | 指定已有 TkeServiceConfig |
| 自动创建配置 | annotation `ingress.cloud.tencent.com/tke-service-config-auto: "true"` | 是(apply) | 自动创建 `<IngressName>-auto-ingress-config` |

## 操作步骤

> **注意：**
> - TkeServiceConfig 资源需要和 Ingress 处于同一命名空间
> - `tke-service-config` 和 `tke-service-config-auto` 两个注解不可同时使用
> - TkeServiceConfig 不会配置协议、端口、域名以及转发路径，仅对已存在的监听器做配置修改
> - `forwardType` 支持 HTTP/HTTPS/GRPC

### 1. 创建 Deployment

```yaml
# jetty-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: jetty
  name: jetty-deployment
  namespace: default
spec:
  replicas: 3
  selector:
    matchLabels:
      app: jetty
  template:
    metadata:
      labels:
        app: jetty
    spec:
      containers:
      - image: jetty:9.4.27-jre11
        imagePullPolicy: IfNotPresent
        name: jetty
        ports:
        - containerPort: 80
          protocol: TCP
        - containerPort: 443
          protocol: TCP
```

### 2. 创建 Service

```yaml
# jetty-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: jetty-service
  namespace: default
spec:
  ports:
  - name: tcp-80-80
    port: 80
    protocol: TCP
    targetPort: 80
  - name: tcp-443-443
    port: 443
    protocol: TCP
    targetPort: 443
  selector:
    app: jetty
  type: NodePort
```

### 3. 创建 TkeServiceConfig

```yaml
# jetty-ingress-config.yaml
apiVersion: cloud.tencent.com/v1alpha1
kind: TkeServiceConfig
metadata:
  name: jetty-ingress-config
  namespace: default
spec:
  loadBalancer:
    l7Listeners:
    - protocol: HTTP
      port: 80
      domains:
      - domain: ""
        rules:
        - url: "/health"
          forwardType: HTTP
          healthCheck:
            enable: false
    - protocol: HTTPS
      port: 443
      defaultServer: "sample.tencent.com"
      domains:
      - domain: "sample.tencent.com"
        http2: true
        rules:
        - url: "/"
          forwardType: HTTPS
          session:
            enable: true
            sessionExpireTime: 3600
          healthCheck:
            enable: true
            intervalTime: 10
            timeout: 5
            healthNum: 2
            unHealthNum: 2
            httpCheckPath: "/checkHealth"
            httpCheckDomain: "sample.tencent.com"
            httpCheckMethod: HEAD
            httpCode: 31
            sourceIpType: 0
          scheduler: WRR
```

| 字段 | 说明 |
|---|---|
| `snatEnable` | 透传客户端源 IP（非 snat 白名单用户勿声明） |
| `keepaliveEnable` | 监听器长连接（非 keepalive 白名单用户勿声明） |
| `forwardType` | 后端协议：HTTP / HTTPS / GRPC |
| `scheduler` | 负载均衡策略：WRR / LEAST_CONN / IP_HASH |
| `checkType` | 健康检查协议：HTTP / HTTPS / TCP / GRPC（2024.06 后新建集群支持） |
| `checkPort` | 自定义健康检查端口（组件版本 ≥ 2.10.0） |
| `sourceIpType` | 0=CLB VIP 探测，1=100.64.0.0/10 网段探测。域名化 CLB 仅支持 1 |

### 4. 创建 Ingress 并关联 TkeServiceConfig

```yaml
# jetty-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.rule-mix: "true"
    kubernetes.io/ingress.http-rules: '[{"path":"/health","backend":{"serviceName":"jetty-service","servicePort":"80"}}]'
    kubernetes.io/ingress.https-rules: '[{"path":"/","backend":{"serviceName":"jetty-service","servicePort":"443","host":"sample.tencent.com"}}]'
    ingress.cloud.tencent.com/tke-service-config: jetty-ingress-config
  name: jetty-ingress
  namespace: default
spec:
  rules:
    - http:
        paths:
          - backend:
              service:
                name: jetty-service
                port:
                  number: 80
            path: /health
            pathType: ImplementationSpecific
    - host: "sample.tencent.com"
      http:
        paths:
          - backend:
              service:
                name: jetty-service
                port:
                  number: 80
            path: /
            pathType: ImplementationSpecific
  tls:
    - secretName: jetty-cert-secret
```

```bash
kubectl apply -f jetty-deployment.yaml
kubectl apply -f jetty-service.yaml
kubectl apply -f jetty-ingress-config.yaml
kubectl apply -f jetty-ingress.yaml
```

```text
# command executed successfully
```

## Ingress 与 TkeServiceConfig 关联行为

| 场景 | 行为 |
|---|---|
| 创建 Ingress 时指定 `tke-service-config-auto: "true"` | 自动创建 `<IngressName>-auto-ingress-config` |
| 更新 Ingress 新增转发规则 | 自动在 TkeServiceConfig 中添加对应配置片段 |
| 删除 Ingress 资源 | 级联删除自动创建的 TkeServiceConfig |
| 手动创建的 TkeServiceConfig，添加注解 | 负载均衡即刻同步 |
| 手动创建的 TkeServiceConfig，删除注解 | 负载均衡保持不变 |
| 修改 TkeServiceConfig 配置 | 引用该配置的 Ingress 的 CLB 同步更新 |
| 自定义配置名称 | 不能以 `-auto-service-config` 或 `-auto-ingress-config` 为后缀 |

## 验证

### 数据面（kubectl）

```bash
kubectl get pods
```

```text
NAME  STATUS  AGE
...
```

```bash
kubectl get tkeserviceconfigs.cloud.tencent.com
```

```text
NAME  STATUS  AGE
...
```

```bash
kubectl describe tkeserviceconfig.cloud.tencent.com jetty-ingress-config
```

```text
Name:         ...
Status:       Running
...
```

## 清理

### 数据面（kubectl）

```bash
kubectl delete ingress jetty-ingress
kubectl delete tkeserviceconfig.cloud.tencent.com jetty-ingress-config
kubectl delete service jetty-service
kubectl delete deployment jetty-deployment
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `Unable to connect to the server` | `kubectl cluster-info` | 公网端点被 CAM 策略 strategyId:240463971 以 `tke:clusterExtranetEndpoint=true` 条件拒绝 | 通过 IOA/VPN 连接内网，或使用同 VPC CVM 执行 kubectl |
| 健康检查不通过 | `intervalTime` | `intervalTime` > `timeout` | 确认 `intervalTime` > `timeout`，健康检查域名非泛域名 |
| 泛域名健康检查失败 | `httpCheckDomain` | 使用 `httpCheckDomain` 字段明确指定具体域名 | 使用 `httpCheckDomain` 字段明确指定具体域名 |
| TkeServiceConfig 配置不生效 | `kubectl describe svc` | TkeServiceConfig 与 Ingress 在同一命名空间 | 确认 TkeServiceConfig 与 Ingress 在同一命名空间 |
| 修改配置后未同步 | `kubectl describe <resource>` | 注解已正确引用 TkeServiceConfig 名称 | 确认注解已正确引用 TkeServiceConfig 名称 |

## 下一步

- [Ingress 基本功能](../Ingress%20基本功能/tccli 操作.md)
- [Ingress 跨 VPC 绑定](../Ingress%20跨%20VPC%20绑定/tccli 操作.md)
- [Ingress Annotation 说明](../Ingress%20Annotation%20说明/tccli 操作.md)

## 控制台替代

控制台不支持直接配置 TkeServiceConfig，需通过 YAML 方式创建。
