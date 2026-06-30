# Service 跨 VPC 绑定（tccli）

> 对照官方：[Service 跨 VPC 绑定](https://cloud.tencent.com/document/product/457/59094) · page_id `59094`

## 概述

公网 CLB Service 默认在当前集群所在 VPC 内随机可用区生成 CLB。TKE 公网 CLB Service 已支持跨 VPC 绑定和指定可用区，包括其他地域的可用区。

使用场景：
- CLB 跨地域接入或跨 VPC 接入（CLB 所在的 VPC 与集群所在的 VPC 不在同一 VPC）
- 指定 CLB 的可用区以实现资源统一管理

## 前置条件

- [环境准备](../../../../环境准备.md)
- 已 [连接集群](../../../../集群配置/集群管理/连接集群/tccli%20操作.md)，kubectl 可正常访问 APIServer
- 跨 VPC 绑定仅支持"带宽上移账户"。若无法确定账户类型，请参见 [判断账户类型](https://cloud.tencent.com/document/product/1199/49090#judge)
- 需先通过 [云联网](https://cloud.tencent.com/document/product/877/18752) 打通当前集群 VPC 和 CLB 所在的 VPC
- 云联网需提前规划各个地域 VPC 的网段，不能出现冲突
- 集群所在 VPC 不能同时加入多个云联网
- VPC 打通后，需通过 [在线咨询](https://cloud.tencent.com/act/event/connect-service) 申请使用该功能

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 查看集群 | `tccli tke DescribeClusters` | 是 |
| 查看地域 ID | [地域和可用区](https://cloud.tencent.com/document/product/457/44787) | 是 |
| 指定本集群 VPC 可用区 | annotation `service.kubernetes.io/service.extensiveParameters: '{"ZoneId":"ap-guangzhou-1"}'` | 否(首次) |
| 跨 VPC 自动创建 CLB | annotation `service.cloud.tencent.com/cross-vpc-id: "vpc-<id>"` | 否(首次) |
| 跨地域自动创建 CLB | annotation `service.cloud.tencent.com/cross-region-id: "ap-guangzhou"` |
| 跨地域使用已有 CLB | annotation `service.cloud.tencent.com/cross-region-id + service.kubernetes.io/tke-existed-lbid` |

## 操作步骤

### 示例1：指定本集群 VPC 可用区

仅需指定本集群所在 VPC 的可用区，例如集群 VPC 在广州地域，指定 CLB 在广州一区：

```yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    service.kubernetes.io/service.extensiveParameters: '{"ZoneId":"ap-guangzhou-1"}'
  name: echo-server-service
  namespace: default
spec:
  ports:
  - port: 80
    protocol: TCP
  selector:
    app: echo
  type: LoadBalancer
```

```bash
kubectl apply -f echo-server-zone.yaml
```

### 示例2：创建跨地域跨 VPC 的 CLB

```yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    service.cloud.tencent.com/cross-region-id: "ap-chongqing"
    service.cloud.tencent.com/cross-vpc-id: "vpc-mjekzyps"
  name: echo-server-service
  namespace: default
spec:
  ports:
  - port: 80
    protocol: TCP
  selector:
    app: echo
  type: LoadBalancer
```

```bash
kubectl apply -f echo-server-cross-vpc.yaml
```

> **注意**：如需同时指定可用区，还需添加示例1中的 `extensiveParameters` annotation。

### 示例3：跨地域使用已有负载均衡

```yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    service.cloud.tencent.com/cross-region-id: "ap-guangzhou"
    service.kubernetes.io/tke-existed-lbid: "lb-342wppll"
  name: echo-server-service
  namespace: default
spec:
  ports:
  - port: 80
    protocol: TCP
  selector:
    app: echo
  type: LoadBalancer
```

```bash
kubectl apply -f echo-server-cross-existed.yaml
```

### 完整 YAML 示例

```yaml
# 创建异地接入的负载均衡
apiVersion: v1
kind: Service
metadata:
  annotations:
    service.cloud.tencent.com/cross-region-id: "ap-chongqing"
    service.cloud.tencent.com/cross-vpc-id: "vpc-mjekzyps"
  name: echo-server-service
  namespace: default
spec:
  ports:
  - port: 80
    protocol: TCP
  selector:
    app: echo
  type: LoadBalancer
---
# 复用其他地域已有负载均衡
apiVersion: v1
kind: Service
metadata:
  annotations:
    service.cloud.tencent.com/cross-region-id: "ap-chongqing"
    service.kubernetes.io/tke-existed-lbid: "lb-o8ugf2wb"
  name: echo-server-service-existed
  namespace: default
spec:
  ports:
  - port: 80
    protocol: TCP
  selector:
    app: echo
  type: LoadBalancer
```

## 验证

### 数据面（kubectl）

```bash
# 查看 Service 状态
kubectl get service echo-server-service

# 查看 Service 注解
kubectl get service echo-server-service -o yaml
```

```text
NAME  STATUS  AGE
...
```

## 清理

### 数据面（kubectl）

```bash
kubectl delete service echo-server-service echo-server-service-existed --ignore-not-found
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `Unable to connect to the server` | `kubectl cluster-info` | 公网端点被 CAM 策略 strategyId:240463971 以 `tke:clusterExtranetEndpoint=true` 条件拒绝 | 通过 IOA/VPN 连接内网，或使用同 VPC CVM 执行 kubectl |
| 跨 VPC CLB 创建失败 | `tccli clb DescribeLoadBalancers` | 已通过云联网打通 VPC； VPC 网段无冲突 | 确认已通过云联网打通 VPC；确认 VPC 网段无冲突 |
| 云联网路由冲突 | 检查云联网路由配置 | 集群 VPC 不能同时加入多个云联网 | 集群 VPC 不能同时加入多个云联网，冲突路由规则不会生效 |
| 带宽上移账号检查 | `kubectl describe <resource>` | 非带宽上移账号不支持跨 VPC 绑定 | 非带宽上移账号不支持跨 VPC 绑定，需升级账号类型 |
| 需白名单开通 | `kubectl describe <resource>` | VPC 打通后需通过在线咨询申请使用 | VPC 打通后需通过在线咨询申请使用 |
| 地域 ID 无效 | `kubectl describe <resource>` | 参考 [地域和可用区](https://cloud.tencent.com/document/product/45... | 参考 [地域和可用区](https://cloud.tencent.com/document/product/457/44787) 确认正确的地域 ID |

## 下一步

- [Service 基本功能](../Service%20基本功能/tccli%20操作.md)
- [Service 后端选择](../Service%20后端选择/tccli%20操作.md)
- Service Annotation 说明请参见官方文档

## 控制台替代

[控制台 → 集群 → 服务与路由 → Service](https://console.cloud.tencent.com/tke2/cluster) 新建时在"访问设置"中选择"其他 VPC"。
