# CLB 回环现象指南（tccli）

> 对照官方：[CLB 回环现象指南](https://cloud.tencent.com/document/product/457/128648) · page_id `128648`

## 概述

CLB 回环现象：当 Pod 通过 CLB 公网/内网 IP 访问自身所在集群的服务时，流量从 Pod 发出 → CLB → 又被调度回同一 Pod，导致连接超时或异常。典型场景包括 Pod 调用同集群的 LoadBalancer Service、hostNetwork 模式 Pod 访问 CLB、或 Pod 通过 CLB 调用同节点其他 Pod。

> **kubectl 数据面不可达**：CAM拒绝公网端点(strategyId:240463971)，内网需VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

## 前置条件

- [环境准备](../../环境准备.md)：`tccli` 已配置
- 已获取集群 kubeconfig（如需 kubectl 操作）
- 了解 CLB LoadBalancer Service 工作原理和 iptables/IPVS 基本概念
- 集群为 MANAGED_CLUSTER 类型，K8s 1.30.0，ap-guangzhou

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 查看集群状态 | `tccli tke DescribeClusterStatus --region <Region> --ClusterIds '["CLUSTER_ID"]'` | 是 |
| 查看 CLB Service | `kubectl get svc -A --field-selector spec.type=LoadBalancer`（需 VPN/IOA） | 是 |
| 查看 CLB 详情 | `tccli clb DescribeLoadBalancers --region <Region> --LoadBalancerIds '["CLB_ID"]'` | 是 |
| 查看节点 iptables 规则 | 登录节点 `iptables -t nat -L -n`（需 SSH） | 是 |
| 查看 Service Endpoints | `kubectl get endpoints <svc> -n NAMESPACE`（需 VPN/IOA） | 是 |

## 操作步骤

### 1. 控制面：确认集群和 CLB 状态

```bash
tccli tke DescribeClusterStatus --region <Region> --ClusterIds '["CLUSTER_ID"]'
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

```bash
tccli clb DescribeLoadBalancers --region <Region>
# expected: 返回 CLB 列表，确认状态为 1（正常）
```

### 2. 数据面：识别回环场景（需 VPN/IOA）

**场景识别：判断是否属于回环问题**

```bash
# 列出所有 LoadBalancer 类型 Service 及其 CLB IP
kubectl get svc -A --field-selector spec.type=LoadBalancer -o custom-columns=\
NAMESPACE:.metadata.namespace,\
NAME:.metadata.name,\
EXTERNAL-IP:.status.loadBalancer.ingress[0].ip,\
CLUSTER-IP:.spec.clusterIP,\
PORTS:.spec.ports[*].port
# expected: 记录 EXTERNAL-IP 列（CLB IP）

# 进入 Pod 测试访问 CLB IP
kubectl exec -it <pod> -n NAMESPACE -- curl -v -m 5 http://<CLB-IP>:<PORT>
# expected（回环场景）: connection timeout 或无响应
# expected（正常）: 返回 HTTP 响应

# 对比：访问 ClusterIP 是否正常
kubectl exec -it <pod> -n NAMESPACE -- curl -v -m 5 http://<ClusterIP>:<PORT>
# expected: 返回 HTTP 响应（ClusterIP 通常正常）
```

```text
NAME  STATUS  AGE
...
```

**判断是否回环：** 若 Pod 访问 CLB-IP 超时但访问 ClusterIP 正常，则为回环问题。

### 3. 数据面：确认回环节点（需 VPN/IOA）

```bash
# 查看 CLB 后端 RS（Real Server）绑定
tccli clb DescribeTargets --region <Region> --LoadBalancerId <CLB_ID>
# expected: 返回绑定的后端节点 IP:Port 列表

# 查看 Service Endpoints
kubectl get endpoints <svc-name> -n NAMESPACE -o yaml
# expected: subsets 列出所有后端 Pod IP:Port

# 从 Pod 中 curl 测试自身节点 IP:NodePort
kubectl exec -it <pod> -n NAMESPACE -- curl -v -m 5 http://<NodeIP>:<NodePort>
# expected（回环场景）: connection timeout
# expected（非回环）: 返回 HTTP 响应
```

```text
NAME  STATUS  AGE
...
```

### 4. 数据面：针对不同回环场景的修复方案（需 VPN/IOA）

**方案 A：使用 `externalTrafficPolicy: Local`（推荐）**

```bash
kubectl patch svc <svc-name> -n NAMESPACE -p '{"spec":{"externalTrafficPolicy":"Local"}}'
# expected: service/<svc-name> patched
```

效果：流量仅转发到接收节点上的 Pod，源 IP 保留，避免跨节点 NAT 回环。

**方案 B：使用内部 Service（访问集群内服务）**

```bash
# 集群内调用应使用 ClusterIP，而非 CLB IP
kubectl get svc <svc-name> -n NAMESPACE -o jsonpath='{.spec.clusterIP}'
# expected: 返回 ClusterIP，内网可达
```

```text
NAME  STATUS  AGE
...
```

**方案 C：CLB 后端不绑定 Pod 所在节点（不推荐生产）**

此方案需通过 CLB 控制台手动管理后端 RS 绑定，排除回环节点。一般不推荐，因为会破坏负载均衡。

**方案 D：hostNetwork Pod 场景特殊处理**

```bash
# 检查是否为 hostNetwork 模式 Pod
kubectl get pod <pod> -n NAMESPACE -o jsonpath='{.spec.hostNetwork}'
# expected: true（hostNetwork Pod）

# hostNetwork Pod 直接使用节点网络，访问 CLB 时源 IP 为节点 IP，
# 若 CLB 将流量转发回同一节点且该节点上 Pod 为终点则触发回环
# 修复：改为 ClusterIP 访问；或使用 externalTrafficPolicy: Local
```

```text
NAME  STATUS  AGE
...
```

## 验证

### 控制面（tccli）

```bash
tccli tke DescribeClusterStatus --region <Region> --ClusterIds '["CLUSTER_ID"]'
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

```bash
tccli clb DescribeLoadBalancers --region <Region> --LoadBalancerIds '["CLB_ID"]'
# expected: Status 为 1（正常）
```

### 数据面（需 VPN/IOA）

```bash
# 验证 Pod 可正常访问 CLB
kubectl exec -it <pod> -n NAMESPACE -- curl -s -o /dev/null -w "%{http_code}" http://<CLB-IP>:<PORT>
# expected: 200（修复前超时/无响应）

# 确认 externalTrafficPolicy 已生效
kubectl get svc <svc-name> -n NAMESPACE -o jsonpath='{.spec.externalTrafficPolicy}'
# expected: Local

# 确认 Endpoints 健康
kubectl get endpoints <svc-name> -n NAMESPACE
# expected: ENDPOINTS 列有 IP:Port
```

```text
NAME  STATUS  AGE
...
```

## 清理

排障页通常无需清理。如有临时修改的 Service 配置需恢复：

```bash
# 恢复 externalTrafficPolicy
kubectl patch svc <svc-name> -n NAMESPACE -p '{"spec":{"externalTrafficPolicy":"Cluster"}}'
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl patch svc` 报错 "field is immutable" | `kubectl get svc <name> -n NAMESPACE -o yaml` | 部分 Service 字段（如 `spec.clusterIP`）不可变 | 仅修改可变字段；若需修改不可变字段，需 `kubectl delete svc` 后重建 |
| `curl` CLB IP 返回 `Connection refused` 而非 `timeout` | `kubectl describe svc <name> -n NAMESPACE \| grep -A5 "Port:"` | 非回环问题：CLB 后端端口未监听或健康检查失败 | 检查 CLB 监听器端口配置和健康检查状态：`tccli clb DescribeListeners --region <Region> --LoadBalancerId CLB_ID` |
| `tccli clb DescribeTargets` 返回空绑定 | 控制台查看 CLB 后端绑定 | Service 类型非 LoadBalancer；TKE 组件未自动创建 CLB | 确认 Service type 为 LoadBalancer：`kubectl get svc <name> -n NAMESPACE -o jsonpath='{.spec.type}'` |

### 创建成功但状态异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 设置 `externalTrafficPolicy: Local` 后部分节点上 Pod 无法接收流量 | `kubectl get pods -n NAMESPACE -o wide \| grep <svc>` 对比节点分布 | `Local` 模式下流量仅发送到有 Pod 在运行的节点，节点无 Pod 则返回 `Connection refused` | 确保每个节点至少有一个 Pod 副本（增加副本数），或使用 DaemonSet 部署 |
| `externalTrafficPolicy: Local` 后源 IP 仍不是客户端 IP | 节点上 `iptables -t nat -L KUBE-NODEPORTS -n` | 节点 iptables 规则中仍有 SNAT/MASQUERADE | 检查 kube-proxy 模式：IPVS 模式需额外配置；确认 iptables 规则中 KUBE-NODEPORTS 链无 MASQUERADE 本地流量 |
| hostNetwork Pod 访问 CLB 时偶发超时 | `kubectl exec <hostnet-pod> -- curl -m 5 http://<CLB-IP>:<PORT>` 多次测试 | iptables hairpin 模式：Linux bridge 默认不转发从同一 bridge 端口发出的包回来 | 节点上执行：`sysctl net.bridge.bridge-nf-call-iptables=1`；或 Pod 使用 ClusterIP 而非 CLB IP |
| CLB 健康检查后端节点显示异常 | `tccli clb DescribeTargetHealth --region <Region> --LoadBalancerId CLB_ID` | CLB 健康检查目标端口未监听；`externalTrafficPolicy: Local` 下节点无 Pod | 确认健康检查端口与 NodePort 一致；对于 `Local` 模式使用含 Pod 的节点做健康检查 |
| CoreDNS 检测到回环（loop detected） | `kubectl logs -n kube-system -l k8s-app=kube-dns \| grep -i "loop detected"` | CoreDNS 的 forward 上游指向了自身 IP，形成 DNS 解析回环 | 修改 CoreDNS ConfigMap 中 forward 地址为非自身 IP：`kubectl edit configmap coredns -n kube-system`，将 `forward .` 指向外部可靠 DNS |

## 下一步

- [Service&Ingress 常见报错和处理](../Service&Ingress%20常见报错和处理/tccli%20操作.md) -- page_id `75758`
- [CLB Ingress 创建报错排障处理](../CLB%20Ingress%20创建报错排障处理/tccli%20操作.md) -- page_id `75757`
- [集群 Kube-Proxy 异常排障处理](../集群%20Kube-Proxy%20异常排障处理/tccli%20操作.md) -- page_id `130407`

## 控制台替代

[容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) → 集群 → 服务与路由 → Service → 查看 YAML/事件。
