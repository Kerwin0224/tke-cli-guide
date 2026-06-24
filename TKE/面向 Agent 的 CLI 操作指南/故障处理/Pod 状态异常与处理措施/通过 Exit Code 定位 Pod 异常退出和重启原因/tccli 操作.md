# 通过 Exit Code 定位 Pod 异常退出和重启原因（tccli）

> 对照官方：[通过 Exit Code 定位 Pod 异常退出和重启原因](https://cloud.tencent.com/document/product/457/43125) · page_id `43125`

## 概述

容器退出码（Exit Code）是定位 Pod 异常退出和重启原因的首要线索。结合 `kubectl describe pod` 查看退出码和退出原因，配合 `kubectl logs --previous` 查看崩溃前的日志输出，可快速定位根因。

> **kubectl 数据面不可达**：CAM拒绝公网端点(strategyId:240463971)，内网需VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

## 前置条件

- [环境准备](../../../环境准备.md)：`tccli` 已配置
- 已获取集群 kubeconfig（如需 kubectl 操作）
- 了解 Kubernetes Pod 生命周期和容器退出码基本概念
- 集群为 MANAGED_CLUSTER 类型，K8s 1.30.0，ap-guangzhou

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 查看集群状态 | `tccli tke DescribeClusterStatus --region <Region> --ClusterIds '["CLUSTER_ID"]'` | 是 |
| 查看 Pod 退出码 | `kubectl describe pod POD_NAME -n NAMESPACE \| grep -A5 "Last State"`（需 VPN/IOA） | 是 |
| 查看前一容器日志 | `kubectl logs POD_NAME -n NAMESPACE --previous --tail=100`（需 VPN/IOA） | 是 |
| 查看 Pod 事件 | `kubectl get events -n NAMESPACE --field-selector involvedObject.name=POD_NAME`（需 VPN/IOA） | 是 |
| 查看节点资源 | `kubectl top node NODE_ID`（需 VPN/IOA） | 是 |
| 查看重启次数 | `kubectl get pod POD_NAME -n NAMESPACE`（需 VPN/IOA） | 是 |

## 操作步骤

### 1. 控制面：确认集群正常

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

### 2. 数据面：获取 Pod 退出码（需 VPN/IOA）

```bash
# 查看 Pod 当前状态和重启次数
kubectl get pod POD_NAME -n NAMESPACE
# expected: RESTARTS 列显示重启次数

# 查看退出码和退出原因
kubectl describe pod POD_NAME -n NAMESPACE | grep -A10 "Last State"
# expected 输出示例:
#   Last State:     Terminated
#     Reason:       OOMKilled
#     Exit Code:    137
#     Started:      ...
#     Finished:     ...

# 使用 jsonpath 精确提取退出码
kubectl get pod POD_NAME -n NAMESPACE -o jsonpath='{range .status.containerStatuses[*]}{.name}{"\t"}{.lastState.terminated.exitCode}{"\t"}{.lastState.terminated.reason}{"\n"}{end}'
# expected: 容器名    退出码    退出原因

# 查看多容器 Pod 的所有容器退出码
kubectl get pod POD_NAME -n NAMESPACE -o jsonpath='{range .status.containerStatuses[*]}{"container: "}{.name}{"\n"}{"  exitCode: "}{.lastState.terminated.exitCode}{"\n"}{"  reason: "}{.lastState.terminated.reason}{"\n"}{"---\n"}{end}'
# expected: 逐个列出每个容器的退出码和原因
```

```text
NAME  STATUS  AGE
...
```

### 3. 退出码速查与诊断

#### 退出码对照表

| Exit Code | 含义 | 信号/场景 | 典型根因 | 诊断命令 |
|-----------|------|----------|----------|----------|
| **0** | 正常退出 | -- | 容器命令/入口执行完毕后正常结束。常见于 Job/CronJob 完成任务后退出，或容器入口点设计为执行完即退出 | `kubectl logs POD_NAME -n NAMESPACE` 查看最后输出 |
| **1** | 通用错误 | -- | 应用程序自身逻辑错误：配置文件解析失败、数据库连接失败、参数校验失败等 | `kubectl logs POD_NAME -n NAMESPACE --tail=50` |
| **2** | 误用 Shell 内建命令 | -- | Shell 语法错误，命令不存在 | 查看容器日志中的 shell 错误输出 |
| **126** | 命令无法执行 | -- | 容器入口点（command）指定的二进制文件权限不足（不可执行）或不是有效的可执行文件 | `kubectl logs POD_NAME -n NAMESPACE --previous` 查看 "Permission denied" |
| **127** | 命令未找到 | -- | 容器入口点（command）指定的二进制文件在容器镜像中不存在 | 检查 Dockerfile CMD/ENTRYPOINT 指令的路径和镜像内容 |
| **128+N** | 致命信号 | N 为信号编号 | 容器收到信号 N 后终止（信号编号见下表） | 查看 exit code 推断信号编号 N |
| **137** | SIGKILL (128+9) | 信号 9 | **最常见退出码**。容器被强制杀死：OOMKilled（cgroup 内存超限）、kubelet 驱逐、`kubectl delete pod --force`。`kubectl describe` 中显示 `Reason: OOMKilled` 即为 OOM | `kubectl describe pod` 查看 Reason；`kubectl top pod` 查看内存；`kubectl describe node NODE_ID` 查看是否有资源压力 |
| **139** | SIGSEGV (128+11) | 信号 11 | 段错误（Segmentation Fault）：空指针解引用、访问非法内存地址、栈溢出 | `kubectl logs POD_NAME -n NAMESPACE --previous` 查看崩溃前日志；开启 core dump |
| **143** | SIGTERM (128+15) | 信号 15 | Kubernetes 正常终止信号：Deployment 缩容、滚动更新替换 Pod、`kubectl delete pod`（优雅终止） | 检查 `terminationGracePeriodSeconds` 是否充足、PreStop 钩子逻辑是否正确 |
| **130** | SIGINT (128+2) | 信号 2 | Ctrl+C 中断信号 | 用户或脚本发送的中断 |
| **141** | SIGPIPE (128+13) | 信号 13 | Broken pipe：管道写入端已关闭但仍尝试写入 | 检查容器内管道通信逻辑 |

#### 信号编号速查表

| 信号编号 | 信号名 | 退出码 | 含义 |
|---------|--------|--------|------|
| 1 | SIGHUP | 129 | 终端挂起 |
| 2 | SIGINT | 130 | 中断（Ctrl+C） |
| 3 | SIGQUIT | 131 | 退出（Ctrl+\），生成 core dump |
| 6 | SIGABRT | 134 | abort() 调用，生成 core dump |
| 9 | SIGKILL | 137 | 强制杀死（不可捕获） |
| 11 | SIGSEGV | 139 | 段错误 |
| 13 | SIGPIPE | 141 | 管道破裂 |
| 15 | SIGTERM | 143 | 终止（可捕获，默认 kill） |

### 4. 按退出码的场景化诊断

#### 场景一：Exit Code 137 — OOMKilled

```bash
# 诊断步骤
# 1. 确认 OOMKilled
kubectl describe pod POD_NAME -n NAMESPACE | grep -E "OOMKilled|Exit Code"
# expected: Reason: OOMKilled, Exit Code: 137

# 2. 查看内存限制
kubectl get pod POD_NAME -n NAMESPACE -o jsonpath='{.spec.containers[*].resources.limits.memory}'
# expected: 显示 memory limits 值

# 3. 查看内存使用趋势（需 metrics-server）
kubectl top pod POD_NAME -n NAMESPACE
# expected: 当前内存使用量，对比 limits

# 4. 查看节点资源状态
kubectl describe node NODE_ID | grep -A5 "Allocated resources"
# expected: 节点资源分配情况

# 修复：
# - 增加 memory limits：kubectl set resources deployment/DEP_NAME -n NAMESPACE --limits=memory=512Mi
# - 排查应用内存泄漏：kubectl logs --previous 查看内存溢出相关日志
```

```text
NAME  STATUS  AGE
...
```

#### 场景二：Exit Code 1 — 应用程序错误

```bash
# 诊断步骤
# 1. 查看退出前日志
kubectl logs POD_NAME -n NAMESPACE --previous --tail=100
# expected: 显示应用错误信息（配置错误、连接失败等）

# 2. 检查事件
kubectl get events -n NAMESPACE --field-selector involvedObject.name=POD_NAME --sort-by='.lastTimestamp'
# expected: 相关事件序列

# 修复：根据日志中的错误信息修正配置或代码
```

```text
NAME  STATUS  AGE
...
```

#### 场景三：Exit Code 143 — 优雅终止相关问题

```bash
# 诊断步骤
# 1. 检查终止宽限期
kubectl get pod POD_NAME -n NAMESPACE -o jsonpath='{.spec.terminationGracePeriodSeconds}'
# expected: 30（默认值）或其他设置值

# 2. 检查 PreStop 钩子
kubectl get pod POD_NAME -n NAMESPACE -o yaml | grep -A10 "preStop"
# expected: 若配置了 PreStop，显示钩子配置

# 3. 查看终止事件
kubectl describe pod POD_NAME -n NAMESPACE | grep -A5 "Events"
# expected: Killing 事件的时间

# 修复：
# - 若 PreStop 处理时间超过 terminationGracePeriodSeconds，增加宽限期
# - 确保应用能正确处理 SIGTERM 信号并优雅关闭
```

```text
NAME  STATUS  AGE
...
```

#### 场景四：Exit Code 139 — 段错误

```bash
# 诊断步骤
# 1. 确认 SIGSEGV
kubectl describe pod POD_NAME -n NAMESPACE | grep -E "Exit Code|Reason"
# expected: Exit Code: 139

# 2. 查看崩溃前日志
kubectl logs POD_NAME -n NAMESPACE --previous --tail=100
# expected: 崩溃前最后输出，可能无明确错误信息

# 修复：
# - 开启 core dump 分析：[容器 coredump 持久化](../../../实践教程/服务部署/容器%20coredump%20持久化/tccli%20操作.md)
# - 使用 Systemtap 追踪：[使用 Systemtap 定位 Pod 异常退出原因](../使用%20Systemtap%20定位%20Pod%20异常退出原因/tccli%20操作.md)
# - 使用 AddressSanitizer (ASan) 编译应用排查内存错误
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

### 数据面（需 VPN/IOA）

```bash
# 确认修复后 Pod 退出码为 0（或不再异常退出）
kubectl get pod POD_NAME -n NAMESPACE
# expected: STATUS Running, RESTARTS 不增长

# 对于已完成 Job，确认退出码
kubectl get pod -l job-name=JOB_NAME -n NAMESPACE -o jsonpath='{.items[0].status.containerStatuses[0].state.terminated.exitCode}'
# expected: 0
```

```text
NAME  STATUS  AGE
...
```

## 清理

排障页通常无需清理。如有测试 Pod 需删除：

```bash
kubectl delete pod <test-pod> -n NAMESPACE
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl describe pod` 无 "Last State" 输出 | `kubectl get pod POD_NAME -n NAMESPACE -o jsonpath='{.status.containerStatuses[0].restartCount}'` | 容器从未重启，无 lastState 记录 | 查看 `state.running` 下的 `startedAt`；若 Pod 正在运行且无异常，无需排查退出码 |
| `kubectl logs --previous` 返回空 | `kubectl get pod POD_NAME -n NAMESPACE -o jsonpath='{.status.containerStatuses[0].restartCount}'` | 容器仅运行一次即崩溃，日志未持久化；或容器 stdout/stderr 未输出任何内容 | 查看 `kubectl describe pod` 的 Events 部分；确认应用日志输出到 stdout/stderr 而非文件 |
| `kubectl describe` 中 `Reason: Error` 但 Exit Code 不明确 | `kubectl get pod POD_NAME -n NAMESPACE -o jsonpath='{.status.containerStatuses[0].lastState.terminated}'` | 退出原因未分类为 OOMKilled，Exit Code 需手动解析 | 根据 Exit Code 数值对照本文退出码表确定信号场景；查 `kubectl logs --previous` |
| 多容器 Pod 中无法确定哪个容器退出引起重启 | `kubectl describe pod POD_NAME -n NAMESPACE \| grep -A3 "Container ID"` 逐容器检查 | 多容器 Pod 中任一容器退出（exit code 非 0 且 restartPolicy 非 Never）都会触发重启 | 逐个检查每个容器的 lastState.terminated；重点看 exitCode 非 0 且 finishedAt 最近的容器 |

### 创建成功但状态异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Exit Code 137 但 `Reason` 不是 OOMKilled | `kubectl describe pod POD_NAME -n NAMESPACE \| grep -B2 -A10 "Last State"`；`dmesg` 查看 OOM 记录（需节点 SSH） | 非 OOM 导致的 SIGKILL：kubelet 驱逐（Eviction）、`kubectl delete pod --force --grace-period=0`、容器运行时异常 | 检查 Events 中是否有 Evicted 或 Killing 事件；`kubectl describe node NODE_ID` 查看 Taints 和 Conditions |
| Exit Code 143 但 Pod 长时间 Terminating | `kubectl get pod POD_NAME -n NAMESPACE -o yaml \| grep -A5 "deletionTimestamp\|finalizers"` | PreStop 钩子挂起；Finalizer 阻塞删除；应用未正确处理 SIGTERM | 检查 PreStop 脚本是否 hang；确认应用捕获 SIGTERM 信号进行优雅关闭；若有 finalizer 阻塞：`kubectl patch pod POD_NAME -n NAMESPACE -p '{"metadata":{"finalizers":[]}}' --type=merge` |
| RESTARTS 持续增长但 STATUS 仍 Running | `kubectl logs POD_NAME -n NAMESPACE --previous \| tail -30` 和 `kubectl describe pod` 查看退出码和原因 | Liveness Probe 配置过于激进（initialDelaySeconds 太小、timeoutSeconds 太短）；应用启动时间超过探针设置 | 增加的 liveness probe 的 `initialDelaySeconds` 和 `failureThreshold`：`kubectl edit deployment DEP_NAME -n NAMESPACE` |
| Job Pod 完成（Completed, exit 0）但预期任务未执行 | `kubectl logs POD_NAME -n NAMESPACE` | 容器入口命令不是预期的任务执行命令，导致空操作即退出 | 检查 Job/CronJob YAML 中 `spec.template.spec.containers[].command` 和 `args` 是否正确 |
| Exit Code 126/127 反复出现 | `kubectl get pod POD_NAME -n NAMESPACE -o jsonpath='{.spec.containers[0].command}'` | 镜像中缺少入口点指定的二进制文件或路径错误 | 修正 Deployment/Pod 中 `command` 为正确路径；`kubectl run test --image=<image> --rm -it -- /bin/sh` 进入镜像验证二进制文件存在 |

## 下一步

- [使用 Systemtap 定位 Pod 异常退出原因](../使用%20Systemtap%20定位%20Pod%20异常退出原因/tccli%20操作.md) -- page_id `43111`
- [Pod 处于 CrashLoopBackOff 状态](../Pod%20处于%20CrashLoopBackOff%20状态/tccli%20操作.md) -- page_id `42948`
- [容器 coredump 持久化](../../../实践教程/服务部署/容器%20coredump%20持久化/tccli%20操作.md) -- page_id `43115`
- [Pod 异常排查概述](../tccli%20操作.md) -- page_id `42945`

## 控制台替代

[容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) → 集群 → 工作负载 → Pod → 详情 → 查看容器状态/退出码。
