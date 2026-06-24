# 使用 LoadBalancer 直连 Pod 模式 Service（tccli）

> 对照官方：[使用 LoadBalancer 直连 Pod 模式 Service](https://cloud.tencent.com/document/product/457/41897) · page_id `41897`

## 概述

原生 LoadBalancer 模式 Service 通过集群 NodePort 转发至集群内，再通过 iptables/ipvs 二次转发。直连 Pod 模式 Service 让 CLB 直接绑定 Pod IP，适用于以下场景：
- 需要获取来源 IP（非直连模式必须额外开启 Local 转发）
- 要求更高转发性能（减少一层 CLB 转发）
- 需要完整的 Pod 层级健康检查和会话保持

TKE 支持 VPC-CNI 和 GlobalRouter 两种容器网络模式开启直连功能。Serverless 集群默认为直连 Pod 模式，无需额外操作。**Cilium-Overlay 容器网络模式不支持直连 Pod 模式。**

## 前置条件

- [环境准备](../../../../环境准备.md)
- 已 [连接集群](../../../../集群配置/集群管理/连接集群/tccli%20操作.md)，kubectl 可正常访问 APIServer
- VPC-CNI 模式：集群 K8s 版本 > 1.12，已开启 VPC-CNI 弹性网卡模式
- GlobalRouter 模式：已开启集群维度直连能力（见步骤1）

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 查看集群 | `tccli tke DescribeClusters` | 是 |
| VPC-CNI 直连 | annotation `service.cloud.tencent.com/direct-access: "true"` | 是(apply) |
| GR 直连（集群开关） | ConfigMap `GlobalRouteDirectAccess: "true"` | 是(apply) |
| GR 直连（Service 开关） | annotation `service.cloud.tencent.com/direct-access: "true"` | 是(apply) |
| Ingress 直连 | annotation `ingress.cloud.tencent.com/direct-access: "true"` | 是(apply) |
| 关联 TkeServiceConfig | annotation `service.cloud.tencent.com/tke-service-config:<name>` | 是(apply) |

## 操作步骤

### VPC-CNI 模式

#### 使用限制

- 集群 K8s 版本需高于 1.12
- 集群网络模式必须开启 VPC-CNI 弹性网卡模式
- 直连 Service 使用的工作负载需使用 VPC-CNI 弹性网卡模式
- 非带宽上移账号不支持直连超级节点 Pod
- CLB 后端数量限制通常为 100 或 200，超过需提工单提升配额
- 满足 CLB 绑定弹性网卡的功能限制
- 开启直连模式的工作负载更新时，根据 CLB 健康检查状态滚动更新，可能影响更新速度

> **从 NodePort 迁移至直连的注意事项**：
> - 确认弹性网卡工作负载的安全组出入流量放通
> - 确认来源 IP 的变化对业务没有影响（直连避免了 FullNAT 转发，来源 IP 不再是节点 IP）
> - 确认工作负载当前没有处于滚动更新状态
> - 建议为 TCP 监听器开启双向 RESET 功能

#### YAML 示例

```yaml
kind: Service
apiVersion: v1
metadata:
  annotations:
    service.cloud.tencent.com/direct-access: "true"  ## 开启直连 Pod 模式
  name: my-service
spec:
  selector:
    app: MyApp
  ports:
  - protocol: TCP
    port: 80
    targetPort: 9376
  type: LoadBalancer
```

```bash
kubectl apply -f my-service-direct.yaml
```

```text
# command executed successfully
```

关闭直连模式只需删除注解或将注解值设为 `"false"`：

```bash
kubectl annotate service my-service service.cloud.tencent.com/direct-access-
```

#### Annotation 扩展

```yaml
metadata:
  annotations:
    service.cloud.tencent.com/direct-access: "true"
    service.cloud.tencent.com/tke-service-config: <tke-service-config-name>
```

### GlobalRouter 模式

#### 使用限制

- 单个工作负载仅能运行在一种网络模式下
- 仅支持带宽上移账号
- CLB 后端数量限制通常为 100 或 200
- 需确保 CVM 安全组放开工作负载对应端口
- 开启直连后默认启用 ReadinessGate 就绪检查
- 不支持黑石机型 BMG5n.24XLARGE384

#### 步骤1：开启集群维度直连配置

在 `kube-system/tke-service-controller-config` ConfigMap 中新增 `GlobalRouteDirectAccess: "true"`：

```bash
kubectl edit configmap tke-service-controller-config -n kube-system
```

在 `data` 下新增：

```yaml
data:
  GlobalRouteDirectAccess: "true"
  VERSION: v2.x.x
```

#### 步骤2：Service 开启直连模式

```yaml
kind: Service
apiVersion: v1
metadata:
  annotations:
    service.cloud.tencent.com/direct-access: "true"  ## 开启直连 Pod 模式
  name: my-service
spec:
  selector:
    app: MyApp
  ports:
  - protocol: TCP
    port: 80
    targetPort: 9376
  type: LoadBalancer
```

```bash
kubectl apply -f my-service-gr-direct.yaml
```

#### Ingress 开启直连模式

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    ingress.cloud.tencent.com/direct-access: "true"
  name: my-ingress
spec:
  rules:
  - host: example.io
    http:
      paths:
      - backend:
          service:
            name: my-service
            port:
              number: 80
        path: /
        pathType: ImplementationSpecific
```

### ReadinessGate 就绪检查

直连模式下，滚动更新时 Kubernetes 判断 Pod 启动完成的标识不包括该 Pod 在 CLB 上健康检查的状态。通过引入 ReadinessGate 特性，TKE 接入层组件在确认后端绑定成功且健康检查通过后，才使 Pod 达到 Ready 状态，推动工作负载滚动更新。

> **注意**：ReadinessGate 仅负责流量就绪检查，与 Endpoints 职责不同。Pod 启动就绪之后的健康状态不会在 ReadinessGate Status 中体现。

## 验证

### 数据面（kubectl）

```bash
# 查看 Service，确认 EXTERNAL-IP
kubectl get service my-service

# 查看 Service 注解，确认直连模式
kubectl get service my-service -o yaml | grep direct-access

# GlobalRouter 模式：确认 ConfigMap 配置
kubectl get configmap tke-service-controller-config -n kube-system -o yaml | grep GlobalRouteDirectAccess

# 查看 Pod 就绪状态（带 ReadinessGate）
kubectl get pods -o wide -l app=MyApp
```

```text
NAME  STATUS  AGE
...
```

## 清理

### 数据面（kubectl）

```bash
kubectl delete service my-service --ignore-not-found
kubectl delete service my-ingress --ignore-not-found
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `Unable to connect to the server` | `kubectl cluster-info` | 公网端点被 CAM 策略 strategyId:240463971 以 `tke:clusterExtranetEndpoint=true` 条件拒绝 | 通过 IOA/VPN 连接内网，或使用同 VPC CVM 执行 kubectl |
| Cilium-Overlay 模式直连不生效 | `kubectl describe <resource>` | Cilium-Overlay 不支持直连 Pod 模式 | Cilium-Overlay 不支持直连 Pod 模式 |
| GR 模式直连不生效 | `GlobalRouteDirectAccess: "true"` | ConfigMap 中 `GlobalRouteDirectAccess: "true"` 已配置 | 确认 ConfigMap 中 `GlobalRouteDirectAccess: "true"` 已配置 |
| 直连模式下 Pod 滚动更新慢 | `tccli clb DescribeLoadBalancers` | 直连模式会根据 CLB 健康状态进行滚动更新 | 直连模式会根据 CLB 健康检查状态进行滚动更新，检查健康检查配置 |
| 非带宽上移账号直连失败 | `kubectl describe <resource>` | 非带宽上移账号不支持直连超级节点 Pod | 非带宽上移账号不支持直连超级节点 Pod，需升级账号类型 |
| CLB 后端数量超限 | `tccli clb DescribeLoadBalancers` | 提工单提升负载均衡 CLB 配额 | 提工单提升负载均衡 CLB 配额 |
| 安全组导致流量不通 | `kubectl describe <resource>` | CVM 安全组已放开工作负载对应端口 | 确认 CVM 安全组已放开工作负载对应端口 |
| ReadinessGate 不生效 | `kubectl describe <resource>` | 集群版本 >= 1.12 | 确认集群版本 >= 1.12，组件版本满足要求 |

## 下一步

- [Service 基本功能](../Service%20基本功能/tccli%20操作.md)
- [多 Service 复用 CLB](../多%20Service%20复用%20CLB/tccli%20操作.md)
- [Service 优雅停机](../Service%20优雅停机/tccli%20操作.md)
- [Pod 优雅删除](../Pod%20优雅删除/tccli%20操作.md)
- [全局开关配置说明](../全局开关配置说明/tccli%20操作.md)

## 控制台替代

[控制台 → 集群 → 服务与路由 → Service](https://console.cloud.tencent.com/tke2/cluster) 新建时勾选"采用负载均衡直连 Pod 模式"。
