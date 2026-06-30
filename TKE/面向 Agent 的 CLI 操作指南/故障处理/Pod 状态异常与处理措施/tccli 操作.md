# Pod 异常排查概述（tccli）

> 对照官方：[Pod 异常排查概述](https://cloud.tencent.com/document/product/457/42945) · page_id `42945`

## 概述

Kubernetes Pod 生命周期中包含多种异常状态。本文作为 Pod 排障的入口索引，提供 Pod 状态分类体系、诊断决策树和通用排查命令，帮助快速定位问题并导航到对应排障子页面。

> **kubectl 数据面不可达**：CAM拒绝公网端点(strategyId:240463971)，内网需VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

## 前置条件

- [环境准备](../../环境准备.md)：`tccli` 已配置
- 已获取集群 kubeconfig（如需 kubectl 操作）
- 了解 Kubernetes Pod 生命周期基本概念
- 集群为 MANAGED_CLUSTER 类型，K8s 1.30.0，ap-guangzhou

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 查看集群状态 | `tccli tke DescribeClusterStatus --region <Region> --ClusterIds '["CLUSTER_ID"]'` | 是 |
| 查看集群实例 | `tccli tke DescribeClusterInstances --region <Region> --ClusterId CLUSTER_ID` | 是 |
| 查看 Pod 列表 | `kubectl get pods -A`（需 VPN/IOA） | 是 |
| 查看 Pod 详情/事件 | `kubectl describe pod POD_NAME -n NAMESPACE`（需 VPN/IOA） | 是 |
| 查看 Pod 日志 | `kubectl logs POD_NAME -n NAMESPACE --tail=100`（需 VPN/IOA） | 是 |
| 查看前一容器日志 | `kubectl logs POD_NAME -n NAMESPACE --previous`（需 VPN/IOA） | 是 |
| 查看事件 | `kubectl get events -n NAMESPACE --sort-by='.lastTimestamp'`（需 VPN/IOA） | 是 |

## 操作步骤

### 1. 控制面：确认集群和节点状态

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

### 2. 数据面：快速扫描异常 Pod（需 VPN/IOA）

```bash
# 列出所有非 Running Pod
kubectl get pods -A --field-selector=status.phase!=Running
# expected: 列出 Pending/Failed/Unknown 的 Pod

# 按状态统计
kubectl get pods -A --no-headers | awk '{print $4}' | sort | uniq -c | sort -rn
# expected: 各状态 Pod 数量分布

# 按重启次数排序（找出 CrashLoopBackOff Pod）
kubectl get pods -A --sort-by='.status.containerStatuses[0].restartCount' | tail -20
# expected: RESTARTS 列降序排列
```

### 3. Pod 状态分类与诊断决策树

#### Pod 状态分类速查表

| Pod STATUS | 阶段 | 含义 | 典型根因 | 排障入口 |
|-----------|------|------|----------|----------|
| `Pending` | 调度/初始化 | Pod 已创建但容器未启动 | 资源不足、节点不可调度、PVC 未 Bound、镜像拉取中 | [Pod 一直处于 Pending](./Pod%20一直处于%20Pending%20状态/tccli%20操作.md) |
| `ContainerCreating` | 容器创建 | 容器镜像拉取中或存储挂载中 | 镜像仓库不可达、镜像体积大、CBS 挂载超时 | [Pod 处于 ContainerCreating](./Pod%20一直处于%20ContainerCreating%20或%20Waiting%20状态/tccli%20操作.md) |
| `ImagePullBackOff` | 镜像拉取 | 镜像拉取失败，正在退避重试 | 镜像不存在、imagePullSecret 无效、镜像仓库网络不通 | [Pod 处于 ImagePullBackOff](./Pod%20一直处于%20ImagePullBackOff%20状态/tccli%20操作.md) |
| `ErrImagePull` | 镜像拉取 | 镜像拉取出错（权限/网络） | 私有仓库认证失败、拉取受限 | [Pod 处于 ImagePullBackOff](./Pod%20一直处于%20ImagePullBackOff%20状态/tccli%20操作.md) |
| `CrashLoopBackOff` | 崩溃循环 | 容器启动后崩溃，反复重启 | 应用程序启动失败、OOMKilled、健康检查配置不当 | [Pod 处于 CrashLoopBackOff](./Pod%20处于%20CrashLoopBackOff%20状态/tccli%20操作.md) |
| `Error` | 异常退出 | 容器以非零退出码结束 | 应用逻辑错误（exit code 1-255） | [通过 Exit Code 排障](./通过%20Exit%20Code%20定位%20Pod%20异常退出和重启原因/tccli%20操作.md) |
| `Completed` | 正常完成 | Job/CronJob 容器正常结束（exit 0） | 非异常；检查 container command 是否符合预期 | -- |
| `Terminating` | 终止中 | Pod 被删除但未完全终止 | GracePeriod 内、Finalizer 阻塞、PreStop 钩子未完成 | [Pod 处于 Terminating](./Pod%20一直处于%20Terminating%20状态/tccli%20操作.md) |
| `Evicted` | 被驱逐 | Pod 被节点驱逐 | 节点 DiskPressure、MemoryPressure、PIDPressure | [节点排障页](../节点常见报错与处理/tccli%20操作.md) |
| `Unknown` | 状态未知 | kubelet 失联，无法报告 Pod 状态 | 节点 NotReady、网络分区 | [节点排障页](../节点常见报错与处理/tccli%20操作.md) |

#### 诊断决策树

```
Pod 状态异常
|
+-- STATUS = Pending
|   +-- Events 显示 FailedScheduling -----> 资源不足，亲和性/污点不满足
|   +-- Events 显示 PersistentVolumeClaim 未绑定 -----> PVC/PV 排障
|   +-- 等待时间 < 镜像拉取时间 -----> 正常 ContainerCreating（等待）
|
+-- STATUS = ContainerCreating / Waiting
|   +-- Events 显示 pulling image -----> 等待镜像拉取完成
|   +-- Events 显示 FailedMount ----------> CBS/CFS 存储挂载排障
|   +-- 长时间无事件 ---------------------> 镜像仓网络问题
|
+-- STATUS = ImagePullBackOff / ErrImagePull
|   +-- Events 显示 authentication required -> 检查 imagePullSecret
|   +-- Events 显示 manifest unknown --------> 镜像 tag 不存在
|   +-- Events 显示 timeout ------------------> 镜像仓库不可达
|
+-- STATUS = CrashLoopBackOff
|   +-- describe 显示 OOMKilled -----------> 增加内存 limits 或排查内存泄漏
|   +-- describe 显示 Error (exit code) ---> 按 exit code 排查
|   +-- logs 显示应用启动错误 -------------> 修复应用配置/代码
|   +-- events 显示 Liveness probe failed --> 调整健康检查参数
|
+-- STATUS = Terminating (超过 terminationGracePeriodSeconds)
|   +-- kubectl get po -o yaml 有 finalizers -> 移除 blocking finalizer
|   +-- PreStop 钩子挂起 ---------------------> 修复 PreStop 逻辑
|   +-- 存储卸载阻塞 -------------------------> 强制删除（--force --grace-period=0）
|
+-- STATUS = Evicted
    +-- 节点状态 NodeCondition 显示 DiskPressure -> 清理节点磁盘
    +-- 节点状态 NodeCondition 显示 MemoryPressure -> 扩容节点或降低 requests
```

```text
NAME  STATUS  AGE
...
```

### 4. 通用排障命令

```bash
# 查看 Pod 完整信息（状态、事件、Conditions）
kubectl describe pod POD_NAME -n NAMESPACE
# expected: Events 区域显示最近事件序列，Conditions 区域显示就绪状态

# 查看 Pod 当前容器日志
kubectl logs POD_NAME -n NAMESPACE --tail=100
# expected: 容器标准输出

# 查看 Pod 上一个崩溃容器的日志
kubectl logs POD_NAME -n NAMESPACE --previous --tail=100
# expected: 上次运行的容器日志（CrashLoopBackOff 场景关键）

# 查看命名空间下最近事件
kubectl get events -n NAMESPACE --sort-by='.lastTimestamp' | tail -30
# expected: 按时间排序的事件列表

# 查看 Pod 资源使用（需 metrics-server）
kubectl top pod POD_NAME -n NAMESPACE
# expected: CPU/Memory 当前使用量

# 进入 Pod 调试（仅 Running 状态可用）
kubectl exec -it POD_NAME -n NAMESPACE -- /bin/sh
# expected: 进入容器 Shell
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
# 扫描异常 Pod
kubectl get pods -A --field-selector=status.phase!=Running
# expected: 列出需要关注的 Pod

# 查看特定 Pod 状态恢复
kubectl get pod POD_NAME -n NAMESPACE -w
# expected: 实时观察 Pod 状态变化
```

```text
NAME  STATUS  AGE
...
```

## 清理

排障页通常无需清理，但如有自建测试资源需删除：

```bash
kubectl delete pod <test-pod> -n NAMESPACE
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl get pods` 返回 "Unable to connect to the server" | `kubectl cluster-info` 测试连接 | kubeconfig 未配置或 API Server 不可达 | 确认 kubeconfig 有效：`kubectl config view`；确认 VPN/IOA 已连接 |
| `kubectl describe pod` 无 Events 输出 | `kubectl get pod POD_NAME -n NAMESPACE -o yaml \| grep -A20 status` | Events 被 TTL 清理或 Pod 已删除 | 查看 Pod status.conditions 和 status.containerStatuses；或查看集群审计日志 |
| `kubectl logs --previous` 返回空或无此选项 | `kubectl get pod POD_NAME -n NAMESPACE -o jsonpath='{.status.containerStatuses[0].restartCount}'` | 容器未曾重启，无 previous 日志 | 直接使用 `kubectl logs` 查看当前日志；首次崩溃的容器无 previous 日志 |
| `kubectl top pod` 返回 "metrics not available yet" | `kubectl get deployment metrics-server -n kube-system` | metrics-server 未安装或未就绪 | 安装或重启 metrics-server：`kubectl rollout restart deployment metrics-server -n kube-system` |

### 创建成功但状态异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Pod Running 但 READY 显示 0/1 | `kubectl describe pod POD_NAME -n NAMESPACE \| grep -A5 "Conditions\|Readiness"` | Readiness Probe 失败，Pod 未就绪 | 检查 readiness probe 配置：端口、路径、超时时间是否匹配；`kubectl logs` 确认应用是否监听正确端口 |
| Pod 频繁重启（RESTARTS > 10）但 STATUS 仍 Running | `kubectl get pod POD_NAME -n NAMESPACE -o yaml \| grep -E "exitCode\|reason\|startedAt"` | OOMKilled、Liveness Probe 误判、应用启动后快速退出 | 按 exit code 排查（见 Exit Code 排障页）；增加 memory limits；延长 liveness probe `initialDelaySeconds` |
| Pod Running 但无网络（ping 不通其他 Pod） | `kubectl exec POD_NAME -n NAMESPACE -- ping <other-pod-ip>` | CNI 插件异常、NetworkPolicy 拦截、Pod 网络配置错误 | 检查 CNI 插件状态和 NetworkPolicy：`kubectl get networkpolicies -n NAMESPACE` |
| `kubectl get pods -A` 显示多个 Evicted Pod | `kubectl describe node NODE_ID \| grep -A5 Conditions` | 节点资源压力导致 Pod 驱逐 | 参考 [节点排障页](../节点常见报错与处理/tccli%20操作.md) 处理节点资源压力；启用 PodDisruptionBudget 保护关键 Pod |

## 下一步

- [Pod 一直处于 Pending 状态](./Pod%20一直处于%20Pending%20状态/tccli%20操作.md) -- page_id `42945`
- [Pod 一直处于 ContainerCreating 或 Waiting 状态](./Pod%20一直处于%20ContainerCreating%20或%20Waiting%20状态/tccli%20操作.md) -- page_id `42946`
- [Pod 一直处于 ImagePullBackOff 状态](./Pod%20一直处于%20ImagePullBackOff%20状态/tccli%20操作.md) -- page_id `42947`
- [Pod 处于 CrashLoopBackOff 状态](./Pod%20处于%20CrashLoopBackOff%20状态/tccli%20操作.md) -- page_id `42948`
- [通过 Exit Code 定位 Pod 异常退出和重启原因](./通过%20Exit%20Code%20定位%20Pod%20异常退出和重启原因/tccli%20操作.md) -- page_id `43125`
- [使用 Systemtap 定位 Pod 异常退出原因](./使用%20Systemtap%20定位%20Pod%20异常退出原因/tccli%20操作.md) -- page_id `43111`

## 控制台替代

[容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) → 集群 → 工作负载 → Pod → 查看事件/日志/YAML。
