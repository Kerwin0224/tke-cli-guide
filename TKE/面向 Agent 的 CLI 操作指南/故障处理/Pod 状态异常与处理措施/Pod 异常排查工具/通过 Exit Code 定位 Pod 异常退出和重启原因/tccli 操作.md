# 通过 Exit Code 定位 Pod 异常退出和重启原因（tccli）

> 对照官方：[通过 Exit Code 定位 Pod 异常退出和重启原因](https://cloud.tencent.com/document/product/457/43125) · page_id `43125`

## 概述

Pod 容器退出时的 Exit Code 包含崩溃原因的关键线索。本文提供 Exit Code 速查表、诊断命令和修复方案。

> **kubectl 数据面不可达**：CAM拒绝公网端点(strategyId:240463971)，内网需VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

## 前置条件

### 环境检查

```bash
tccli --version
# expected: tccli version >= 1.0.0

tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterStatus "Running"
```

```json
{
  "TotalCount": "<TotalCount>",
  "Clusters": "<Clusters>",
  "ClusterId": "<ClusterId>",
  "ClusterName": "<ClusterName>",
  "ClusterDescription": "<ClusterDescription>",
  "ClusterVersion": "<ClusterVersion>"
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli/kubectl 命令 | 幂等 |
|-----------|-------------------|:--:|
| 查看 Pod 状态 | `kubectl get pods -A`（需 VPN/IOA） | 是 |
| 查看退出码 | `kubectl describe pod POD_NAME -n NAMESPACE`（需 VPN/IOA） | 是 |
| 查看上次崩溃日志 | `kubectl logs POD_NAME -n NAMESPACE --previous`（需 VPN/IOA） | 是 |
| 查看节点事件 | `kubectl get events -n NAMESPACE`（需 VPN/IOA） | 是 |

## 操作步骤

### 步骤 1：获取退出码

```bash
# 查看 Pod 详情中的退出码和重启次数
kubectl describe pod POD_NAME -n NAMESPACE | grep -E "State:|Exit Code:|Restart Count:|Reason:"
# expected: 含 Last State → Exit Code 和 Reason
```

预期输出：

```
State:          Waiting
  Reason:       CrashLoopBackOff
Last State:     Terminated
  Reason:       Error
  Exit Code:    137
  Started:      ...
  Finished:     ...
```

### 步骤 2：查看崩溃前日志

```bash
kubectl logs POD_NAME -n NAMESPACE --previous --tail=100
# expected: 上一次容器运行的日志
```

```text
NAME  STATUS  AGE
...
```

### 步骤 3：按退出码定位根因

#### Exit Code 速查与修复表

| Exit Code | 含义 | 常见根因 | 诊断命令 | 修复 |
|-----------|------|---------|---------|------|
| **0** | 正常退出 | 容器任务完成（Job/CronJob） | `kubectl logs POD_NAME` | 正常行为，无需修复 |
| **1** | 应用错误 | 代码逻辑错误/配置错误 | `kubectl logs --previous` | 检查应用日志，修复代码或配置 |
| **2** | 参数错误 | 命令行参数/环境变量错误 | `kubectl describe pod POD_NAME` 查看 Command/Args | 修正启动命令和参数 |
| **126** | 权限不足 | 脚本不可执行 | `kubectl exec POD_NAME -- ls -la /entrypoint.sh` | `chmod +x` 或修改 CMD |
| **127** | 命令未找到 | 容器中缺少可执行文件 | `kubectl exec POD_NAME -- which COMMAND` | 检查镜像或修改 CMD 路径 |
| **137** | SIGKILL（128+9） | OOMKilled / 强制终止 | `kubectl describe pod POD_NAME` 查看 OOMKilled | 增加 memory limit；优化应用内存使用 |
| **139** | SIGSEGV（128+11） | 段错误/内存访问越界 | `kubectl logs --previous` 查看堆栈 | 修复代码 bug；`dmesg` 查看内核日志 |
| **143** | SIGTERM（128+15） | 优雅终止/滚动更新 | `kubectl describe pod POD_NAME` 查看事件 | 正常行为；如卡死则在 preStop 处理 |

#### 重点诊断：Exit Code 137（OOMKilled）

```bash
# 1. 确认是否 OOMKilled
kubectl describe pod POD_NAME -n NAMESPACE | grep -A5 "Last State"
# expected: Reason: OOMKilled

# 2. 查看当前内存限制
kubectl get pod POD_NAME -n NAMESPACE -o jsonpath='{.spec.containers[0].resources.limits.memory}'
# expected: 当前 memory limit 值

# 3. 查看节点 OOM 事件
kubectl describe node NODE_NAME | grep -A10 "Conditions:"
# expected: MemoryPressure 为 False（节点内存充足，Pod 内存超限）

# 4. 控制面：查看集群实例资源
tccli tke DescribeClusterInstances --region <Region> --ClusterId CLUSTER_ID
# expected: InstanceSet 含节点规格
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

## 验证

```bash
kubectl get pods -n NAMESPACE
# expected: Pod 状态 Running，RESTARTS 不再增长

kubectl describe pod POD_NAME -n NAMESPACE | grep "Exit Code"
# expected: 新容器 Exit Code 为空（Running 状态无 Exit Code）
```

```text
NAME  STATUS  AGE
...
```

## 清理

排障页无需清理。如需重建问题 Pod：

```bash
kubectl delete pod POD_NAME -n NAMESPACE
# expected: Pod 删除并重建（如由 Deployment 管理）
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Exit Code 137 但未显示 OOMKilled | `kubectl describe pod` → Reason 非 OOMKilled | Pod 被 kubelet 主动 Kill（非 OOM） | 检查是否被驱逐（DiskPressure）或手动删除 |
| Exit Code 0 但 Pod 重启 | `kubectl describe pod` → Restart Count > 0 | 容器正常退出但 restartPolicy: Always | 修改 restartPolicy 或修改应用使其常驻 |
| 多次不同 Exit Code | `kubectl describe pod` → Last State 变化 | 根因在逐次重试中变化（如先 1 后 137） | 按最近一次退出码优先处理 |
| 退出码为空 | `kubectl describe pod` → State: Waiting（非 Terminated） | 容器尚未崩溃，处于等待状态 | 查看 Events 确定等待原因 |

## 下一步

- [Pod 异常排查概述](../../Pod%20异常排查概述/tccli%20操作.md) -- page_id `42945`
- [Pod 处于 CrashLoopBackOff 状态](../../Pod%20处于%20CrashLoopBackOff%20状态/tccli%20操作.md) -- page_id `43130`
- [使用 Systemtap 定位 Pod 异常退出原因](../使用%20Systemtap%20定位%20Pod%20异常退出原因/tccli%20操作.md) -- page_id `43111`
- [节点高负载排障处理](../../../节点高负载排障处理/tccli%20操作.md) -- page_id `43127`

## 控制台替代

[容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) -> 工作负载 -> Pod -> 事件/日志。
