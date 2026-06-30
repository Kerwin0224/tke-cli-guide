# 容器进程主动退出（tccli）

> 对照官方：[容器进程主动退出](https://cloud.tencent.com/document/product/457/43148) · page_id `43148`

## 概述

容器进程主动退出（Exit Code 0 或 1）的场景排查。通常因 command 执行完毕、应用内部错误或 OOM。通过 tccli 检查集群状态，配合 kubectl 查看退出码和上一轮日志进行诊断。

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
| 查看退出码 | `kubectl describe pod <pod> \| grep "Exit Code"`（需 VPN/IOA） | 是 |
| 查看上一轮日志 | `kubectl logs <pod> --previous`（需 VPN/IOA） | 是 |
| 保留现场 | 设置 `restartPolicy: Never`（需 VPN/IOA） | 否 |

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
kubectl describe pod <pod-name> -n <ns> | grep -A10 "Last State"
# 预期: 显示 Exit Code、Reason、StartedAt、FinishedAt
```

```text
Name:         ...
Status:       Running
...
```

### 步骤3：按退出场景诊断

**command 为一次性任务（Exit Code 0）**

容器执行完 command 后正常退出。对于需要持续运行的服务，应确保容器内进程以前台模式运行。

修复：
- 使用 Job/CronJob 而非 Deployment 运行一次性任务
- 或确保 entrypoint 是前台进程（如 nginx 使用 `daemon off;`）

**应用启动失败（Exit Code 1）**

```bash
kubectl logs <pod-name> -n <ns> --previous
# 预期: 查看退出前的错误日志
```

```text
...log output...
```

常见原因：配置文件缺失、环境变量错误、依赖服务不可达。

修复：
- 检查 ConfigMap/Secret 挂载是否正确
- 检查环境变量值
- 使用 initContainer 等待依赖就绪

**依赖服务不可达**

```bash
# 从集群内测试依赖服务
kubectl run test-dep --image=busybox --rm -it --restart=Never -- nc -zv <svc-name>.<ns>.svc.cluster.local <port>
# 预期: Connection succeeded
```

修复示例（initContainer 等待数据库就绪）：

```yaml
initContainers:
- name: wait-for-db
  image: busybox
  command: ['sh', '-c', 'until nc -z db-svc 3306; do echo waiting for db; sleep 2; done']
```

**PID 1 进程退出**

容器中主进程作为 PID 1，退出后容器即停止。若主子进程未正确处理信号，可能导致异常退出。

修复：
- 使用 `tini` 或 `dumb-init` 作为 PID 1
- 或镜像中配置 `STOPSIGNAL` 正确处理 SIGTERM

### 步骤4：临时保留现场

为排查 Pod 退出原因，可临时设置 `restartPolicy: Never` 保留退出后的现场：

```yaml
spec:
  restartPolicy: Never
```

设置后 Pod 退出后不会重启，可以查看完整的退出状态和日志。

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
kubectl logs <pod> -n <ns> --previous
# 预期: 查看退出前日志定位根因

kubectl get pod <pod> -n <ns>
# 预期: 修复后 STATUS Running，不再重启
```

```text
NAME  STATUS  AGE
...
```

## 清理

无需特殊清理（排障页）。修复后 Pod 自动恢复。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|---|---|---|---|
| `--previous` 日志为空 | `kubectl describe pod` 查看 Previous State | 上一轮容器运行时无日志输出 | 检查日志输出配置；在镜像中增加启动诊断日志 |
| Exit Code 137 但未显示 OOMKilled | `kubectl describe pod` 查看 Reason | 可能由外部 kill 或 preStop hook 超时触发 | 检查是否有外部进程管理；增加 `terminationGracePeriodSeconds` |

### 操作成功但异常

| 现象 | 诊断 | 根因 | 修复 |
|---|---|---|---|
| 进程偶尔退出但自动恢复 | 查看退出时间模式 | 定时任务（如 cron）在容器内触发 | 将定时任务迁移到 Kubernetes CronJob |
| 进程不退但应用不响应 | `kubectl exec <pod> -- kill -0 1` | PID 1 进程僵尸，未正确 reap 子进程 | 使用 tini 作为 init 进程管理子进程 |

## 下一步

- [通过 Exit Code 定位 Pod 异常退出和重启原因](../通过%20Exit%20Code%20定位%20Pod%20异常退出和重启原因/tccli%20操作.md)
- [Pod 处于 CrashLoopBackOff 状态](../Pod%20处于%20CrashLoopBackOff%20状态/tccli%20操作.md) -- page_id `43130`

## 控制台替代

[容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) -> 工作负载 -> Pod -> 日志 -> 选择"上一个容器"。
