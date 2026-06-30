# Service&Ingress 常见报错和处理（tccli）

> 对照官方：[Service&Ingress 常见报错和处理](https://cloud.tencent.com/document/product/457/80913) · page_id `80913`

## 概述

汇总 TKE Service 和 Ingress 常见错误信息、原因分析和处理方法。通过 tccli 检查集群和 CLB 状态，配合 kubectl 查看 Service、Endpoints、Ingress 状态定位问题。

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
| 查看 Service 和 Endpoints | `kubectl get svc,endpoints -A`（需 VPN/IOA） | 是 |
| 查看 Ingress | `kubectl get ingress -A`（需 VPN/IOA） | 是 |
| 查看 CLB 配额 | `tccli clb DescribeQuota --region ap-guangzhou` | 是 |

## 操作步骤

### 步骤1：控制面检查集群

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

### 步骤2：数据面检查 Service 和 Ingress 状态（需 VPN/IOA）

```bash
# 检查所有 Service
kubectl get svc -A
# 预期: EXTERNAL-IP 非 <pending>

# 检查 Endpoints
kubectl get endpoints -A
# 预期: 每个 Service 有对应的 Endpoints，非 <none>

# 检查所有 Ingress
kubectl get ingress -A
# 预期: ADDRESS 列非空，显示 CLB VIP
```

```text
NAME  STATUS  AGE
...
```

### 步骤3：常见报错速查

| 错误现象 | 根因 | 修复 |
|---|---|---|
| Service EXTERNAL-IP 显示 `<pending>` | CLB 创建中或无可用 CLB 配额 | 等待 1-2 分钟；`tccli clb DescribeQuota` 检查配额 |
| Ingress ADDRESS 为空 | Ingress Controller 未能创建 CLB | `kubectl describe ingress <name>` 查看 Events |
| Ingress 返回 502/503 | 后端 Service 无就绪 Pod | `kubectl get endpoints <svc>` 确认有后端 IP |
| Ingress 返回 404 | Ingress 规则路径与请求不匹配 | 检查 `host` 和 `path` 配置 |
| TLS 证书错误 | 证书过期或域名与证书 CN 不匹配 | 更新 TLS Secret 中的证书 |

### 步骤4：Service 排障诊断（需 VPN/IOA）

```bash
# 确认 selector 匹配
kubectl get pod -l <selector> -n <ns>
# 预期: 返回匹配的 Pod 列表

# 确认 Endpoints 有后端
kubectl get endpoints <svc-name> -n <ns>
# 预期: ENDPOINTS 列有 Pod IP:Port

# 集群内测试
kubectl run test-svc --image=busybox --rm -it --restart=Never -- wget -O- -T3 http://<svc-name>.<ns>.svc.cluster.local:<port>
# 预期: 返回预期内容
```

```text
NAME  STATUS  AGE
...
```

### 步骤5：Ingress 排障诊断（需 VPN/IOA）

```bash
kubectl describe ingress <name> -n <ns> | grep -A20 Events
# 预期: Events 中查看 CLB 创建和后端绑定的错误
```

```text
Name:         ...
Status:       Running
...
```

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
kubectl get svc <svc-name> -n <ns>
# 预期: EXTERNAL-IP 非 <pending>

kubectl get ingress <name> -n <ns>
# 预期: ADDRESS 非空

curl -s -o /dev/null -w "%{http_code}" http://<ADDRESS>/<path>
# 预期: 返回 200
```

```text
NAME  STATUS  AGE
...
```

## 清理

```bash
# 删除测试资源（需 VPN/IOA）
kubectl delete ingress <name> -n <ns>
kubectl delete svc <svc-name> -n <ns>
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|---|---|---|---|
| `kubectl get svc` EXTERNAL-IP 持续 pending | `kubectl describe svc` Events 有 `Failed to create load balancer` | 集群 CLB 配额耗尽或子网 IP 不足 | `tccli clb DescribeQuota` 确认配额；更换或扩容子网 |
| Ingress Events 显示 `no endpoints` | `kubectl get endpoints` 对应 Service 为空 | Service selector 不匹配任何 Pod | 修正 Service selector 使其匹配 Running 的 Pod |
| curl 返回 `Connection Refused` | `kubectl logs` 查看 Ingress Controller 日志 | 后端 Pod 端口未监听或容器退出 | 检查 Pod 容器状态和端口监听情况 |

### 操作成功但异常

| 现象 | 诊断 | 根因 | 修复 |
|---|---|---|---|
| Ingress 间歇性 502 | 查看 CLB 健康检查状态 | 部分后端 Pod 健康检查失败导致被踢出 | 调整健康检查路径和阈值；确保所有 Pod 就绪 |
| 删除 Ingress 后 CLB 未释放 | `tccli clb DescribeLoadBalancers` 检查 CLB 列表 | CLB 删除 API 调用失败或超时 | 在 CLB 控制台手动清理孤儿 CLB |

## 下一步

- [CLB Ingress 创建报错排障处理](../CLB%20Ingress%20创建报错排障处理/tccli%20操作.md) -- page_id `75757`
- [Service&Ingress 网络无法访问排障处理](../Service&Ingress%20网络无法访问排障处理/tccli%20操作.md) -- page_id `80913`

## 控制台替代

[容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) -> 服务与路由 -> Service / Ingress -> 查看事件。
