# Pod 处于 CrashLoopBackOff 状态（tccli）

> 对照官方：[Pod 处于 CrashLoopBackOff 状态](https://cloud.tencent.com/document/product/457/43130) · page_id `43130`

## 概述

Pod 反复启动后立即退出，kubelet 指数退避重启（CrashLoopBackOff）。常见原因：应用启动错误、OOMKilled、探针失败、命令错误。通过 tccli 检查集群状态，配合 kubectl 查看退出码和上一轮日志进行诊断。

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
| 查看退出原因 | `kubectl describe pod <pod>`（需 VPN/IOA） | 是 |
| 查看上一轮日志 | `kubectl logs <pod> --previous`（需 VPN/IOA） | 是 |
| 查看重启次数 | `kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[0].restartCount}'`（需 VPN/IOA） | 是 |

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

### 步骤2：数据面查看退出信息（需 VPN/IOA）

```bash
# 查看退出码和退出原因
kubectl describe pod <pod-name> | grep -A10 "Last State"
# 预期: 显示 Exit Code 和 Reason（如 OOMKilled、Error、Completed）

# 查看上一轮容器日志
kubectl logs <pod-name> --previous
# 预期: 退出前的最后日志，定位错误信息
```

```text
Name:         ...
Status:       Running
...
```

### 步骤3：按 Exit Code 分类诊断

| Exit Code | 含义 | 常见原因 | 修复 |
|---|---|---|---|
| 0 | 正常退出 | command 执行完毕即退出（非长期运行进程） | 使用 Job/CronJob 或改为前台进程 |
| 1 | 应用错误 | 程序启动失败、配置缺失、依赖不满足 | 查看 `--previous` 日志定位错误 |
| 137 | SIGKILL | OOMKilled 被内核杀或外部 kill | 增加 memory limit；检查是否有外部进程 kill |
| 139 | SIGSEGV | 段错误（Segfault） | 应用程序 bug，检查代码和依赖库 |
| 143 | SIGTERM | 正常终止信号 | 可能 preStop hook 超时或被驱逐 |
| 255 | 容器启动失败 | 容器入口点错误或镜像问题 | 检查 command/args 是否正确 |

### 步骤4：常见场景处理

**OOMKilled（Exit Code 137）**

```bash
kubectl describe pod <pod-name> | grep -i "OOMKilled\|Reason"
# 若显示 OOMKilled：容器内存使用超过 limits
```

```text
Name:         ...
Status:       Running
...
```

修复：增加 `spec.containers[].resources.limits.memory` 或排查内存泄漏。

**应用启动错误（Exit Code 1）**

```bash
kubectl logs <pod-name> --previous | tail -50
# 预期: 根据错误日志修复应用配置或代码
```

```text
...log output...
```

**启动即退出且无日志**

判断 `/proc/1` 进程是否正常：
- 检查 command 是否拼写错误
- 检查 args 中引用的文件或环境变量是否存在
- 使用 `kubectl run test --image=busybox --rm -it -- <cmd>` 交互式排查

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
kubectl get pod <pod-name> -w
# 预期: RESTARTS 不再增长，STATUS 稳定为 Running
```

```text
NAME  STATUS  AGE
...
```

## 清理

无需特殊清理（排障页）。修复后 Pod 自动恢复正常。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|---|---|---|---|
| `--previous` 日志为空 | `kubectl describe pod` 确认容器是否曾运行 | 容器未成功启动就退出，未产生日志 | 检查镜像 ENTRYPOINT 和 CMD 是否正确 |
| `kubectl describe` 显示 `Back-off restarting failed container` | Pod 在指数退避等待中 | 等待间隔为 10s -> 20s -> 40s -> ... 最大 5 分钟 | 修复根因后 Pod 自动恢复；如需立即重试，删除 Pod |

### 操作成功但异常

| 现象 | 诊断 | 根因 | 修复 |
|---|---|---|---|
| 修复后仍偶发重启 | `kubectl top pod` 查看资源使用 | 资源 limits 仍然偏紧，OOM 阈值波动 | 进一步提高 memory limits；配置 HPA |
| 多副本同时 CrashLoopBackOff | 检查是否有滚动更新或 ConfigMap 变更 | 新版镜像或配置存在全局 bug | 回滚到上一版本：`kubectl rollout undo deploy <name>` |

## 下一步

- [通过 Exit Code 定位 Pod 异常退出和重启原因](../通过%20Exit%20Code%20定位%20Pod%20异常退出和重启原因/tccli%20操作.md)
- [Pod 健康检查失败](../Pod%20健康检查失败/tccli%20操作.md) -- page_id `43129`

## 控制台替代

[容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) -> 工作负载 -> Pod -> 日志 Tab -> 选择"上一个容器"查看退出前日志。
