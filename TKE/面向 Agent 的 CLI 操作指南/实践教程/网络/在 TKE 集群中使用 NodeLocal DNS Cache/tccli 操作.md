# 在 TKE 集群中使用 NodeLocal DNS Cache（tccli）

> 对照官方：[在 TKE 集群中使用 NodeLocal DNS Cache](https://cloud.tencent.com/document/product/457/40613) · page_id `40613`

## 概述

NodeLocal DNS Cache 以 DaemonSet 形式在每个节点部署 DNS 缓存代理（169.254.20.10），拦截 Pod 的 DNS 请求，减少 CoreDNS 压力。TKE 支持三级注入优先级（Pod 级 > Namespace 级 > 集群级），IPVS 模式下需显式配置 DNSConfig。

## 前置条件

- [环境准备](../../../环境准备.md)
- 集群网络模式非 Cilium-Overlay、非超级节点
- `tke-eni-ip-webhook` 组件已安装（DNSConfig 自动注入依赖）

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 安装 NodeLocal DNS Cache | `tccli tke InstallAddon --AddonName localdns` | 是 |
| 启用 Namespace 级注入 | `kubectl label namespace <ns> localdns-injector=enabled` | 是 |
| 启用 Pod 级注入 | `kubectl run --labels localdns-injector=enabled` | 否 |
| 验证 DNS 解析路径 | `kubectl exec <pod> -- nslookup kubernetes.default` | 是 |
| 卸载组件 | `tccli tke DeleteAddon --AddonName localdns` | 否 |

## 操作步骤

### 1. 安装 NodeLocal DNS Cache

```bash
tccli tke InstallAddon --region ap-guangzhou --cli-input-json file://install-localdns.json
```

```json
{"ClusterId": "<ClusterId>", "AddonName": "localdns", "AddonVersion": "<AddonVersion>"}
```

```output
{"RequestId": "..."}
```

### 2. 启用 DNSConfig 自动注入

**Namespace 级启用（批量控制）：**

```bash
kubectl label namespace default localdns-injector=enabled
```

```output
namespace/default labeled
```

**Pod 级启用（精细控制）：**

```bash
kubectl run test-pod --image=nginx:alpine --labels="localdns-injector=enabled" --restart=Never
```

### 3. 验证注入效果

```bash
kubectl get pod test-pod -o jsonpath='{.spec.dnsConfig}'
```

```text
NAME  STATUS  AGE
...
```

```output
{"nameservers":["169.254.20.10","10.0.0.10"],"options":[{"name":"ndots","value":"3"},{"name":"attempts","value":"2"},{"name":"timeout","value":"1"}]}
```

### 4. 验证 DNS 解析路径

```bash
kubectl exec test-pod -- nslookup kubernetes.default 2>&1
```

```output
Server:    169.254.20.10
Address:   169.254.20.10#53
Name:      kubernetes.default.svc.cluster.local
Address:   10.0.0.1
```

### IPVS 集群特殊配置

IPVS 模式下需修改 CoreDNS Corefile 添加 `prefer_udp`：

```bash
kubectl edit configmap coredns -n kube-system
```

在对应 server block 中添加：

```
forward . 183.60.83.19 183.60.82.98 {
    prefer_udp
}
```

## 验证

### Data plane (kubectl)

```bash
kubectl get ds node-local-dns -n kube-system
kubectl exec test-pod -- cat /etc/resolv.conf
```

```text
NAME  STATUS  AGE
...
```

## 清理

### Data plane (kubectl)

```bash
kubectl label namespace default localdns-injector-
kubectl delete pod test-pod
```

### Control plane (tccli)

```bash
kubectl -n kube-system delete ds node-local-dns --cascade=orphan
tccli tke DeleteAddon --region ap-guangzhou --cli-input-json file://delete-localdns.json
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl` 命令超时或连接被拒绝 | `tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]'`（控制面返回 Running） | CAM 策略 strategyId:240463971 拒绝公网端点，数据面需内网访问 | 通过 VPN/IOA 接入 VPC 内网后执行 kubectl；或使用控制台替代 |
| DNSConfig 未注入 | `kubectl get namespace <ns> --show-labels` | `localdns-injector` label 未设置，或 Pod 为 hostNetwork | 确认 `localdns-injector` label 已设置，Pod 非 hostNetwork |
| DNS 解析失败 | `kubectl get cm coredns -n kube-system -o yaml \| grep prefer_udp` | CoreDNS Corefile 未配置 `prefer_udp` | 检查 CoreDNS Corefile 是否配置 `prefer_udp` |
| NodeLocalDNS Pod Crash | `kubectl logs -n kube-system <node-local-dns-pod>` | 节点 53 端口被 named 服务占用 | Ubuntu 20.04 上需关闭 named 服务：`systemctl stop named` |

## 下一步

- [TKE DNS 最佳实践](../DNS%20相关/TKE%20DNS%20最佳实践/tccli%20操作.md)
- [在 TKE 中实现自定义域名解析](../DNS%20相关/在%20TKE%20中实现自定义域名解析/tccli%20操作.md)

## 控制台替代

控制台：集群 → 组件管理 → 安装 → nodelocal-dns → 完成安装。
