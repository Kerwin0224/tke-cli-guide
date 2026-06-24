# Pod 一直处于 Terminating 状态（tccli）

> 对照官方：[Pod 一直处于 Terminating 状态](https://cloud.tencent.com/document/product/457/43238) · page_id `43238`

## 概述

Pod 卡在 Terminating 状态的原因：finalizer 未完成、kubelet 不可达、preStop hook 卡住、存储卷卸载失败。通过 tccli 检查集群状态，配合 kubectl 查看 Pod 详情并执行强制删除。

> **kubectl 数据面不可达**：CAM拒绝公网端点(strategyId:240463971)，内网需VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

## 前置条件

- [环境准备](../../../环境准备.md)
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
| 查看 Pod 详情 | `kubectl describe pod <pod>`（需 VPN/IOA） | 是 |
| 强制删除 | `kubectl delete pod <pod> --force --grace-period=0`（需 VPN/IOA） | 否 |
| 移除 finalizer | `kubectl patch pod <pod> -p '{"metadata":{"finalizers":[]}}' --type=merge`（需 VPN/IOA） | 否 |

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

### 步骤2：数据面诊断卡住原因（需 VPN/IOA）

```bash
kubectl describe pod <pod-name> -n <ns> | grep -A5 "Status:\|Finalizers\|Conditions"
# 预期: 查看 finalizer 列表和 Pod 条件

kubectl get pod <pod-name> -n <ns> -o json | jq '.metadata.finalizers'
# 预期: 列出未完成的 finalizer
```

```text
NAME  STATUS  AGE
...
```

### 步骤3：常见原因与修复

**Finalizer 阻塞**

自定义 Controller（如日志采集、网络策略）注册了 finalizer，控制器异常导致 finalizer 未移除。

```bash
# 查看 finalizer
kubectl get pod <pod-name> -n <ns> -o jsonpath='{.metadata.finalizers}'

# 手动移除（确认控制器已不管理该资源后）
kubectl patch pod <pod-name> -n <ns> -p '{"metadata":{"finalizers":[]}}' --type=merge
```

```text
NAME  STATUS  AGE
...
```

**preStop hook 卡住**

Pod 的 `terminationGracePeriodSeconds` 内 preStop 未执行完毕。

```bash
kubectl logs <pod-name> -n <ns> --previous
# 预期: 查看 preStop 是否阻塞
```

```text
...log output...
```

**节点不可达**

Pod 所在节点 NotReady，kubelet 无法执行容器停止操作。

```bash
kubectl get nodes
# 预期: 检查 Pod 所在节点状态
```

```text
NAME  STATUS  AGE
...
```

**存储卷卸载失败**

```bash
kubectl describe pod <pod-name> -n <ns> | grep -i "unmount\|detach"
# 预期: 是否有存储卸载错误
```

```text
Name:         ...
Status:       Running
...
```

### 步骤4：强制删除

当无法正常终止时，使用强制删除：

```bash
kubectl delete pod <pod-name> -n <ns> --force --grace-period=0
# 预期: warning 提示但 Pod 被立即删除
```

### 步骤5：清理 finalizer 后删除

```bash
# 先移除 finalizer
kubectl patch pod <pod-name> -n <ns> -p '{"metadata":{"finalizers":[]}}' --type=merge

# 再删除
kubectl delete pod <pod-name> -n <ns>
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
kubectl get pod <pod-name> -n <ns>
# 预期: Error from server (NotFound): pods "<pod-name>" not found
```

```text
NAME  STATUS  AGE
...
```

## 清理

Pod 删除成功后无需额外清理。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|---|---|---|---|
| `--force --grace-period=0` 后 Pod 仍存在 | `kubectl get pod <pod> -o json` 查看 metadata | finalizer 阻止了 API Server 删除 | 先 `kubectl patch` 移除 finalizer，再执行删除 |
| `kubectl patch` 移除 finalizer 返回 `the object has been modified` | 并发修改冲突 | 控制器同时尝试更新 finalizer | 重试 patch 操作 |
| 强制删除后 Pod 残留数据 | 检查 PVC 状态 | `--force` 跳过了优雅终止流程 | 手动清理残留 PVC 和数据 |

### 操作成功但异常

| 现象 | 诊断 | 根因 | 修复 |
|---|---|---|---|
| 删除 Pod 后新 Pod 仍卡 Terminating | 检查 Yaml 中是否配置了相同的 finalizer | Deployment 模板中包含 finalizer | 移除 Deployment 模板中的 finalizer 配置 |
| 存储资源泄漏 | 删除 Pod 后 PV/PVC 未释放 | Pod 强制删除导致 CSI unmount 未执行 | 手动在节点上 umount；在 CBS 控制台解挂磁盘 |

## 下一步

- [容器进程主动退出](../容器进程主动退出/tccli%20操作.md) -- page_id `43148`
- [Pod 处于 CrashLoopBackOff 状态](../Pod%20处于%20CrashLoopBackOff%20状态/tccli%20操作.md) -- page_id `43130`

## 控制台替代

[容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) -> 工作负载 -> Pod -> 删除 -> 勾选"强制删除"。
