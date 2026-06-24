# Pod 网络无法访问排查处理（tccli）

> 对照官方：[Pod 网络无法访问排查处理](https://cloud.tencent.com/document/product/457/40332) · page_id `40332`

## 概述

排查 Pod 网络无法访问的问题，涵盖 Pod 间通信、Pod 访问外部、外部访问 Pod 等场景。通过 tccli 检查集群和节点状态，配合 kubectl 进行网络诊断。

> **kubectl 数据面不可达**：CAM拒绝公网端点(strategyId:240463971)，内网需VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

## 前置条件

- [环境准备](../../环境准备.md)
- 已获取集群 kubeconfig（如需 kubectl 操作）

### 环境检查

```bash
tccli --version
# 预期输出: tccli version X.X.X
```

### 资源检查

```bash
tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]'
```

**预期输出**：

```json
{
  "TotalCount": 1,
  "Clusters": [
    {
      "ClusterId": "cls-xxxxxxxx",
      "ClusterName": "my-test-cluster",
      "ClusterStatus": "Running",
      "ClusterVersion": "1.30.0",
      "ClusterType": "MANAGED_CLUSTER"
    }
  ]
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli / CLI | 幂等 |
|---|---|---|
| 查看集群状态 | `tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]'` | 是 |
| 查看节点实例 | `tccli tke DescribeClusterInstances --region ap-guangzhou --ClusterId cls-xxxxxxxx` | 是 |
| 查看 Pod IP | `kubectl get pods -o wide`（需 VPN/IOA） | 是 |
| 查看 Service 和 Endpoints | `kubectl get svc,endpoints`（需 VPN/IOA） | 是 |
| 查看 NetworkPolicy | `kubectl get networkpolicy -A`（需 VPN/IOA） | 是 |

## 操作步骤

### 步骤1：控制面检查集群和节点

```bash
tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]'
```

**预期输出**：

```json
{
  "Clusters": [
    {
      "ClusterId": "cls-xxxxxxxx",
      "ClusterStatus": "Running",
      "ClusterVersion": "1.30.0"
    }
  ]
}
```

```bash
tccli tke DescribeClusterInstances --region ap-guangzhou --ClusterId cls-xxxxxxxx
```

**预期输出**：

```json
{
  "TotalCount": 3,
  "InstanceSet": [
    {
      "InstanceId": "ins-xxxxxxxx",
      "InstanceRole": "WORKER",
      "InstanceState": "running"
    }
  ]
}
```

### 步骤2：数据面网络诊断（需 VPN/IOA）

```bash
# 检查 Pod IP 分配
kubectl get pods -o wide -A
# 预期: 每个 Pod 有独立 IP

# 检查 Service 和 Endpoints
kubectl get svc,endpoints -A
# 预期: Service 的 Endpoints 包含后端 Pod IP

# 检查 NetworkPolicy 规则
kubectl get networkpolicy -A
# 预期: 查看是否有拒绝规则影响通信

# 从测试 Pod 验证连通性
kubectl run test-conn --image=busybox --rm -it --restart=Never -- wget -O- -T3 http://<svc-name>.<ns>.svc.cluster.local:<port>
# 预期: 返回预期内容
```

```text
NAME  STATUS  AGE
...
```

### 步骤3：诊断分析

Pod 网络问题按通信方向分为三类：

**Pod 间无法通信**

1. `kubectl get pods -o wide` 确认双方 Pod IP
2. `kubectl exec <src-pod> -- ping <dst-pod-ip>` 测试三层连通性
3. 检查 NetworkPolicy 是否拒绝了 Pod 间流量
4. 检查 CNI 插件状态：`kubectl get pods -n kube-system -l k8s-app=tke-cni`

**Pod 无法访问外网**

1. `kubectl exec <pod> -- ping 8.8.8.8` 测试外网连通性
2. 检查 Pod 所在节点的 NAT 网关配置
3. 检查 VPC 路由表是否有默认路由指向 NAT 网关
4. 检查安全组出站规则是否放通

**外部无法访问 Pod**

1. `kubectl get endpoints <svc-name>` 确认有后端 IP
2. 检查 Service type：ClusterIP（仅集群内）/ NodePort / LoadBalancer
3. `tccli clb DescribeTargetHealth --region ap-guangzhou --LoadBalancerId <lb-id>` 检查 CLB 后端健康
4. 检查安全组入站规则是否放通对应端口

## 验证

### 控制面（tccli）

```bash
tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]'
```

**预期输出**：

```json
{
  "Clusters": [
    {
      "ClusterId": "cls-xxxxxxxx",
      "ClusterStatus": "Running"
    }
  ]
}
```

### 数据面（需 VPN/IOA）

```bash
kubectl get nodes
# 预期: 所有节点 Ready

kubectl get endpoints -A --no-headers | grep -v "<none>"
# 预期: 所有 Service 有后端 Endpoints
```

```text
NAME  STATUS  AGE
...
```

## 清理

无需特殊清理（排障页）。测试用 Pod 使用 `--rm` 参数会自动清理。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|---|---|---|---|
| Pod IP 为空 | `kubectl describe pod` 显示 `FailedCreatePodSandBox` | CNI 插件异常或节点 IP 池耗尽 | 检查 CNI 组件状态；VPC-CNI 模式下确认子网有可用 IP |
| `ping` 不通其他 Pod | 同一节点 Pod 能通、跨节点不通 | VPC 路由或节点间网络配置异常 | 检查节点安全组和 VPC 路由表 |
| Service ClusterIP 无法访问 | `kubectl get endpoints` 为空 | selector 不匹配或后端 Pod 未就绪 | 修正 selector；确认 Pod readinessProbe 通过 |

### 操作成功但异常

| 现象 | 诊断 | 根因 | 修复 |
|---|---|---|---|
| 偶发网络不通 | `kubectl describe node` 查看节点状态 | kube-proxy 规则未同步 | 重启对应节点的 kube-proxy Pod |
| DNS 解析失败 | Pod 内 `nslookup` 超时 | CoreDNS 异常 | 转至 [集群 DNS 解析异常排障处理](../集群%20DNS%20解析异常排障处理/tccli%20操作.md) |

## 下一步

- [集群 DNS 解析异常排障处理](../集群%20DNS%20解析异常排障处理/tccli%20操作.md) -- page_id `80531`
- [Service&Ingress 常见报错和处理](../Service&Ingress%20常见报错和处理/tccli%20操作.md) -- page_id `80913`

## 控制台替代

[容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) -> 集群 -> 监控和事件 -> 网络监控。
