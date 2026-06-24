# Pod 无限重启且流量异常（tccli）

> 对照官方：[Pod 无限重启且流量异常](https://cloud.tencent.com/document/product/457/78805) · page_id `78805`

## 概述

Pod 频繁重启导致 Service 流量中断或 5xx 错误。排查重启原因、检查滚动更新策略、确认探针配置。通过 tccli 检查集群状态，配合 kubectl 查看重启历史和日志进行诊断。

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
| 查看重启次数 | `kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[0].restartCount}'`（需 VPN/IOA） | 是 |
| 查看历史日志 | `kubectl logs <pod> --previous`（需 VPN/IOA） | 是 |
| 暂停服务 | `kubectl scale deploy <name> --replicas=0`（需 VPN/IOA） | 否 |

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

### 步骤2：数据面查看重启历史（需 VPN/IOA）

```bash
# 查看 Pod 重启次数和推出信息
kubectl describe pod <pod-name> | grep -A10 "Last State"
# 预期: 查看 Exit Code 和退出原因

kubectl get pod <pod-name> -o jsonpath='{.status.containerStatuses[*].restartCount}'
# 预期: 重启计数

# 查看 Deployment 滚动更新状态
kubectl rollout status deploy <deploy-name> -n <ns>
# 预期: 查看是否在滚动更新中
```

```text
NAME  STATUS  AGE
...
```

### 步骤3：按症状诊断

**重启伴随 502/503 错误**

- 原因：容器启动慢但 readinessProbe 过早触发，导致 Pod 被加入 Service 后端但尚未就绪
- 诊断：`kubectl describe pod` 查看 readinessProbe 失败时间与重启时间关联
- 修复：增大 `readinessProbe.initialDelaySeconds`，确保应用完全启动后再探测

**OOMKilled 导致重启**

- 原因：容器内存使用超过 `resources.limits.memory`
- 诊断：`kubectl describe pod` 显示 `Reason: OOMKilled`，Exit Code 137
- 修复：增加 memory limits；排查内存泄漏

**多 Pod 同时重启**

- 原因：滚动更新触发了全部 Pod 重建，但新版本有 bug
- 诊断：`kubectl rollout history deploy <name>` 查看变更历史
- 修复：`kubectl rollout undo deploy <name>` 回滚到上一版本

**启动即退出**

- 原因：应用配置错误或依赖服务不可达
- 诊断：`kubectl logs <pod> --previous` 查看退出前日志
- 修复：修正配置或添加 initContainer 等待依赖就绪

### 步骤4：临时缓解措施

```bash
# 紧急暂停流量入口
kubectl scale deploy <name> --replicas=0 -n <ns>

# 回滚到稳定版本
kubectl rollout undo deploy <name> -n <ns>

# 修改探针配置延长等待时间后恢复服务
# 编辑 Deployment，调整 readinessProbe.initialDelaySeconds
kubectl scale deploy <name> --replicas=3 -n <ns>
```

### 步骤5：防止滚动更新雪崩

```yaml
spec:
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0  # 确保更新过程中服务不中断
  minReadySeconds: 30    # Pod Ready 后等待 30 秒再继续更新
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
kubectl get pod <pod> -w
# 预期: RESTARTS 计数稳定，不再频繁增长

curl -s -o /dev/null -w "%{http_code}" http://<svc-ip>:<port>/<healthz>
# 预期: 持续返回 200
```

```text
NAME  STATUS  AGE
...
```

## 清理

无需特殊清理（排障页）。修复后 Pod 自动恢复稳定。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|---|---|---|---|
| `rollout undo` 后仍重启 | 检查回滚到的版本配置 | 回滚到的旧版本也存在同样问题 | 使用 `--to-revision=N` 指定更早的稳定版本 |
| `scale --replicas=0` 后恢复的 Pod 仍重启 | 问题不在 Deployment 而在集群环境 | 存储卷数据损坏或 ConfigMap 有错误值 | 检查持久化数据和 ConfigMap/Secret 内容 |

### 操作成功但异常

| 现象 | 诊断 | 根因 | 修复 |
|---|---|---|---|
| 每次部署新版本都经历重启周期 | 观察重启是否集中在新版 Pod 上线后 | 应用启动时间超过 `initialDelaySeconds` | 在 Deployment 中设置保守的探针初始延迟 |
| 滚动更新中旧 Pod 被误杀 | 查看 Events 和 kubelet 日志 | `terminationGracePeriodSeconds` 太短 | 增加优雅终止时间（建议 30-60 秒） |

## 下一步

- [Pod 处于 CrashLoopBackOff 状态](../Pod%20处于%20CrashLoopBackOff%20状态/tccli%20操作.md) -- page_id `43130`
- [通过 Exit Code 定位 Pod 异常退出和重启原因](../通过%20Exit%20Code%20定位%20Pod%20异常退出和重启原因/tccli%20操作.md)

## 控制台替代

[容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) -> 工作负载 -> Pod -> 监控 -> 重启次数。
