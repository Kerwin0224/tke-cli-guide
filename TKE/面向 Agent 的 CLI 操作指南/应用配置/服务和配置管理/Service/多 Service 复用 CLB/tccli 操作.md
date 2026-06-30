# 多 Service 复用 CLB（tccli）

> 对照官方：[多 Service 复用 CLB](https://cloud.tencent.com/document/product/457/46370) · page_id `46370`

## 概述

多个 Service 可复用同一个负载均衡器 CLB，主要用于支持在同一个 VIP 上同时暴露 TCP 及 UDP 的相同端口。其他场景下不建议使用多个 Service 复用相同的 CLB。

## 前置条件

- [环境准备](../../../../环境准备.md)
- 已 [连接集群](../../../../集群配置/集群管理/连接集群/tccli%20操作.md)，kubectl 可正常访问 APIServer
- 已在目标 VPC 中创建 CLB 实例

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 查看集群 | `tccli tke DescribeClusters` | 是 |
| 复用已有 CLB（标准集群） | annotation `service.kubernetes.io/tke-existed-lbid: lb-<id>` | 否(首次) |
| 复用已有 CLB（Serverless 集群） | 需额外 annotation `service.kubernetes.io/qcloud-share-existed-lb: "true"` | 否(首次) |
| 创建 Service 复用 CLB | 多个 Service 配置相同 `tke-existed-lbid` | 是(注解同) |
| 查看复用状态 | `kubectl describe service <name>` | 是 |

## 操作步骤

### 说明事项

- **2025年1月2日起创建的 TKE 集群（Master 托管），默认开启 Service 复用。**
- **2020年8月17日至2025年1月1日创建的 TKE 集群，默认关闭 Service 复用。** 需通过 [在线咨询](https://cloud.tencent.com/online-service?from=doc_457) 申请开启。
- TKE Serverless 集群默认已开启 CLB 复用能力，但用于复用的 CLB 必须为用户手动购买，不能是 Serverless 集群自动购买的 CLB。

| Serverless 集群复用 CLB 注解 | 说明 |
|------------------------------|------|
| `service.kubernetes.io/qcloud-share-existed-lb: "true"` | 标记复用行为 |
| `service.kubernetes.io/tke-existed-lbid: lb-xxx` | 指定已有 CLB ID |

> **注意**：Service 和 CLB 之间配置的管理和同步由以 CLB ID 为名字的 LoadBalancerResource 类型 CRD 对象负责，请勿对该 CRD 进行任何操作，否则容易导致 Service 失效。

> **业务发版注意事项**：在滚动更新时，请避免同时对这些 Service 关联的后端服务进行发版。同步复用相同 CLB 的 Service 操作是串行的，如果多个 Service 后端同时在更新，部分后端流量摘除可能不及时，[优雅停机](../Service%20优雅停机/tccli%20操作.md) 能力也可能不能真正生效。

### 使用限制

- 多个 Service 复用相同 CLB 时，Service 的端口不能重复。
- 单个 CLB 管理的监听器数量由 CLB 的 TOTAL_LISTENER_QUOTA 限制。
- 只能使用用户自行创建的 CLB，不能使用 TKE 自动创建的 CLB（否则可能导致 CLB 资源泄漏）。
- 负载均衡资源被复用后，该 CLB 的生命周期不由 TKE 管理，需自行维护。

### 步骤1：创建负载均衡实例

在 CLB 控制台或通过 tccli 创建集群所在 VPC 下的公网或内网类型负载均衡：

```bash
tccli clb CreateLoadBalancer \
  --LoadBalancerType "OPEN" \
  --VpcId "vpc-<VpcId>" \
  --LoadBalancerName "shared-clb"
```

### 步骤2：创建第一个 Service（使用已有 CLB）

```yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    service.kubernetes.io/tke-existed-lbid: lb-<LoadBalancerId>
  name: my-service-tcp
spec:
  ports:
    - name: 80-80-tcp
      port: 80
      protocol: TCP
      targetPort: 80
  selector:
    app: my-app
  type: LoadBalancer
```

```bash
kubectl apply -f my-service-tcp.yaml
```

### 步骤3：创建第二个 Service（复用相同 CLB）

```yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    service.kubernetes.io/tke-existed-lbid: lb-<LoadBalancerId>
  name: my-service-udp
spec:
  ports:
    - name: 8080-8080-udp
      port: 8080
      protocol: UDP
      targetPort: 8080
  selector:
    app: my-app
  type: LoadBalancer
```

```bash
kubectl apply -f my-service-udp.yaml
```

```text
# command executed successfully
```

## 验证

### 数据面（kubectl）

```bash
# 查看所有 Service，确认使用相同 CLB
kubectl get services

# 查看每个 Service 的注解确认 CLB ID
kubectl get service my-service-tcp -o jsonpath='{.metadata.annotations}'
kubectl get service my-service-udp -o jsonpath='{.metadata.annotations}'
```

```text
NAME  STATUS  AGE
...
```

## 清理

### 数据面（kubectl）

```bash
kubectl delete service my-service-tcp my-service-udp --ignore-not-found
```

> 删除 Service 后，复用 CLB 绑定的后端 CVM 需要自行解绑，同时需自行清理 `tke-clusterId` 标签。CLB 本身不会被释放。

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `Unable to connect to the server` | `kubectl cluster-info` | 公网端点被 CAM 策略 strategyId:240463971 以 `tke:clusterExtranetEndpoint=true` 条件拒绝 | 通过 IOA/VPN 连接内网，或使用同 VPC CVM 执行 kubectl |
| 复用失败，端口冲突 | `tccli clb DescribeLoadBalancers` | 多个 Service 复用相同 CLB 时端口不能重复 | 多个 Service 复用相同 CLB 时端口不能重复 |
| Serverless 集群复用 CLB 报错 | `qcloud-share-existed-lb: "true"` | CLB 为用户手动购买 | 确认 CLB 为用户手动购买，非 Serverless 集群自动购买；确认添加了 `qcloud-share-existed-lb: "true"` 注解 |
| 存量集群无法复用 | `kubectl describe <resource>` | 2020年8月17日-2025年1月1日创建的集群需提工单开启复用功能 | 2020年8月17日-2025年1月1日创建的集群需提工单开启复用功能 |
| 复用 CLB 的 Service 删除后 CLB 未被释放 | `kubectl describe svc` | 复用场景下 CLB 生命周期不由 TKE 管理 | 复用场景下 CLB 生命周期不由 TKE 管理，需自行管理 |
| 滚动更新时流量异常 | `tccli clb DescribeLoadBalancers` | 避免同时更新复用同一 CLB 的多个 Service 后端 | 避免同时更新复用同一 CLB 的多个 Service 后端 |
| 监听器数量超限 | `tccli clb DescribeLoadBalancers` | CLB TOTAL_LISTENER_QUOTA 限制 | 检查 CLB TOTAL_LISTENER_QUOTA 限制 |

## 下一步

- [Service 使用已有 CLB](../Service%20使用已有%20CLB/tccli%20操作.md)
- [Service 扩展协议](../Service%20扩展协议/tccli%20操作.md)
- [Service 优雅停机](../Service%20优雅停机/tccli%20操作.md)

## 控制台替代

[控制台 → 集群 → 服务与路由 → Service](https://console.cloud.tencent.com/tke2/cluster) 新建时选择"使用已有"并选择相同负载均衡实例。
