# CoreDNS 升级常见错误和处理（tccli）

> 对照官方：[CoreDNS 升级常见错误和处理](https://cloud.tencent.com/document/product/457/130406) · page_id `130406`

## 概述

TKE 集群升级 CoreDNS 版本时可能遇到的常见问题：版本不兼容、ConfigMap 配置迁移失败、Corefile 语法错误、插件废弃等。本文提供系统化排障步骤。

> **kubectl 数据面不可达**：CAM拒绝公网端点(strategyId:240463971)，内网需VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

## 前置条件

- [环境准备](../../环境准备.md)：`tccli` 已配置
- 已获取集群 kubeconfig（如需 kubectl 操作）
- 已确认目标 CoreDNS 版本号
- 集群为 MANAGED_CLUSTER 类型，K8s 1.30.0，ap-guangzhou

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 查看集群状态 | `tccli tke DescribeClusterStatus --region <Region> --ClusterIds '["CLUSTER_ID"]'` | 是 |
| 查看 CoreDNS 版本 | `tccli tke DescribeAddon --region <Region> --ClusterId CLUSTER_ID --AddonName coredns` | 是 |
| 查看 CoreDNS Pod | `kubectl get pods -n kube-system -l k8s-app=kube-dns`（需 VPN/IOA） | 是 |
| 查看 CoreDNS ConfigMap | `kubectl get configmap coredns -n kube-system -o yaml`（需 VPN/IOA） | 是 |
| 查看 CoreDNS 日志 | `kubectl logs -n kube-system -l k8s-app=kube-dns --tail=50`（需 VPN/IOA） | 是 |
| 触发组件升级 | `tccli tke UpdateAddon --region <Region> --ClusterId CLUSTER_ID --AddonName coredns --AddonVersion <target>` | 否 |

## 操作步骤

### 1. 控制面：确认当前 CoreDNS 版本

```bash
tccli tke DescribeAddon --region <Region> --ClusterId CLUSTER_ID --AddonName coredns
# expected: 返回当前 AddonVersion 和状态 "Running"
```

```json
{
  "Addons": "<Addons>",
  "AddonName": "<AddonName>",
  "AddonVersion": "<AddonVersion>",
  "RawValues": "<RawValues>",
  "Phase": "<Phase>",
  "Reason": "<Reason>"
}
```

```bash
tccli tke DescribeClusterInstances --region <Region> --ClusterId CLUSTER_ID \
    --filter "InstanceSet[].InstanceState"
# expected: 节点状态 running
```

```json
{
  "TotalCount": "<TotalCount>",
  "InstanceSet": "<InstanceSet>",
  "InstanceId": "<InstanceId>",
  "InstanceRole": "<InstanceRole>",
  "FailedReason": "<FailedReason>",
  "InstanceState": "<InstanceState>"
}
```

### 2. 数据面：升级前检查（需 VPN/IOA）

```bash
# 检查 CoreDNS Pod 当前状态
kubectl get pods -n kube-system -l k8s-app=kube-dns -o wide
# expected: 所有 Pod Running，RESTARTS 数稳定

# 备份当前 Corefile
kubectl get configmap coredns -n kube-system -o yaml > coredns-configmap-backup.yaml
# expected: 备份文件生成成功

# 检查 CoreDNS 日志无报错
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=50
# expected: 无 ERROR 级别日志
```

```text
NAME  STATUS  AGE
...
```

### 3. 控制面：执行升级

```bash
tccli tke UpdateAddon --region <Region> --ClusterId CLUSTER_ID --AddonName coredns --AddonVersion "<target-version>"
# expected: RequestId 返回，升级任务已提交
```

## 验证

### 控制面（tccli）

```bash
tccli tke DescribeAddon --region <Region> --ClusterId CLUSTER_ID --AddonName coredns
# expected: AddonVersion 为目标版本，Status "Running"
```

```json
{
  "Addons": "<Addons>",
  "AddonName": "<AddonName>",
  "AddonVersion": "<AddonVersion>",
  "RawValues": "<RawValues>",
  "Phase": "<Phase>",
  "Reason": "<Reason>"
}
```

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

### 数据面（需 VPN/IOA）

```bash
# 检查新版本 Pod 已运行
kubectl get pods -n kube-system -l k8s-app=kube-dns
# expected: 所有 Pod Running，AGE 显示为刚重建

# 检查 Corefile 配置已迁移
kubectl get configmap coredns -n kube-system -o jsonpath='{.data.Corefile}' | head -30
# expected: Corefile 语法正确，包含新版插件块

# 测试 DNS 解析
kubectl run -it --rm dns-test --image=busybox:1.28 --restart=Never -- nslookup kubernetes.default.svc.CLUSTER_ID
# expected: 返回 Service ClusterIP，无超时
```

```text
NAME  STATUS  AGE
...
```

## 清理

排障页通常无需清理。如需回滚，执行 `tccli tke UpdateAddon` 指定旧版本号即可。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `tccli tke UpdateAddon` 返回 "Addon version not available" | `tccli tke DescribeAddon` 查看可用版本列表 | 目标版本不存在或与集群 K8s 版本不兼容 | 选择 `DescribeAddon` 返回的可用版本列表中兼容的版本号 |
| 升级后 `kubectl get pods` 显示 ImagePullBackOff | `kubectl describe pod -n kube-system -l k8s-app=kube-dns \| grep -A5 Events` | CoreDNS 镜像 tag 在镜像仓库中不存在 | 检查 `DescribeAddon` 返回的镜像地址；若私有仓库则确认镜像已同步 |
| `kubectl apply` 更新 ConfigMap 时报错 "Corefile syntax error" | `kubectl get configmap coredns -n kube-system -o jsonpath='{.data.Corefile}'` 逐段检查 | Corefile 插件配置语法错误：缺少花括号、插件名称拼写错误、参数格式不对 | 对照 [CoreDNS 官方 Corefile 文档](https://coredns.io/manual/toc/) 修正语法，或从备份恢复：`kubectl apply -f coredns-configmap-backup.yaml` |
| 升级后 CoreDNS Pod CrashLoopBackOff | `kubectl logs -n kube-system <coredns-pod> \| tail -20` | Corefile 中引用了已废弃且移除的插件（如 `upstream` 在 1.8+ 中行为变更、`federation` 在 1.7+ 中废弃） | `kubectl edit configmap coredns -n kube-system`，删除废弃插件，替换为等效新配置（如 `proxy` → `forward . <upstream>`）|
| 升级后 CoreDNS Pod 一直 Pending | `kubectl describe pod -n kube-system <coredns-pod> \| grep -A10 Events` | 新版 CoreDNS 对资源 requests/limits 有更高要求，当前节点资源不足 | 扩容节点或降低 CoreDNS resource requests/limits：`kubectl edit deployment coredns -n kube-system` |

### 创建成功但状态异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| CoreDNS Pod Running 但 DNS 解析偶尔超时 | `kubectl logs -n kube-system -l k8s-app=kube-dns \| grep -i "timeout\|error\|i/o"` | Corefile 中 `forward .` 指向的上游 DNS 不可达或响应慢 | 修改 Corefile 中 forward 上游为可靠地址（如 `forward . 183.60.83.19 183.60.82.98`） |
| DNS 请求返回 NXDOMAIN（域名不存在） | `kubectl run -it --rm debug --image=busybox:1.28 --restart=Never -- nslookup test.CLUSTER_ID` | `prometheus` 插件未启用、`errors` 插件未在正确位置 | 确保 Corefile 中启用 `errors` 和 `log` 插件：`errors` 应放在第一个插件位置 |
| 升级后 Service 间 DNS 解析中断 | `kubectl exec -it <any-pod> -- nslookup <svc>.<ns>` | Corefile 中 `kubernetes` 插件配置丢失或 cluster domain 不匹配 | `kubectl get configmap coredns -n kube-system -o yaml`，确认 `kubernetes CLUSTER_ID in-addr.arpa ip6.arpa` 块存在且域名后缀正确 |
| Pod 中 `/etc/resolv.conf` 仍指向旧 DNS | `kubectl exec <pod> -- cat /etc/resolv.conf` | kubelet 未更新 Pod 的 DNS 配置，需重建 Pod | 滚动重启受影响的 Deployment/StatefulSet：`kubectl rollout restart deployment <name> -n NAMESPACE` |
| CoreDNS 升级后内存占用持续增长 | `kubectl top pods -n kube-system -l k8s-app=kube-dns` | Corefile 中 `cache` 插件配置过大或缺少 TTL 限制 | `kubectl edit configmap coredns -n kube-system`，在 `cache` 块中添加 `success 9984 30` 和 `denial 9984 5` 限制缓存 TTL |

## 下一步

- [集群 DNS 解析异常排障处理](../集群%20DNS%20解析异常排障处理/tccli%20操作.md) -- page_id `130406`
- [集群 Kube-Proxy 异常排障处理](../集群%20Kube-Proxy%20异常排障处理/tccli%20操作.md) -- page_id `130407`
- [节点常见报错与处理](../节点常见报错与处理/tccli%20操作.md) -- page_id `89869`

## 控制台替代

[容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) → 集群 → 组件管理 → CoreDNS → 升级/配置。
