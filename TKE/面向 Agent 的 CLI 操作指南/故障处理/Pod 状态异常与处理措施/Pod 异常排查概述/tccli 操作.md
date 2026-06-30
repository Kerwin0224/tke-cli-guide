# Pod 异常排查概述（tccli）

> 对照官方：[Pod 异常排查概述](https://cloud.tencent.com/document/product/457/42945) · page_id `42945`

## 概述

系统化梳理 TKE 集群 Pod 常见异常状态（Pending、CrashLoopBackOff、ImagePullBackOff、Terminating、ContainerCreating），提供标准排查路径。

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
| 查看集群状态 | `tccli tke DescribeClusterStatus --ClusterIds '["CLUSTER_ID"]'` | 是 |
| 查看 Pod 列表 | `kubectl get pods -A`（需 VPN/IOA） | 是 |
| 查看 Pod 详情 | `kubectl describe pod POD_NAME -n NAMESPACE`（需 VPN/IOA） | 是 |
| 查看 Pod 事件 | `kubectl get events -n NAMESPACE --field-selector involvedObject.name=POD_NAME`（需 VPN/IOA） | 是 |
| 查看节点状态 | `kubectl get nodes`（需 VPN/IOA） | 是 |

## 操作步骤

### Pod 状态分类与排查决策树

#### 第一步：识别 Pod 状态

```bash
kubectl get pods -A
# expected: 列出所有 Pod 和状态
```

预期输出：

```
NAMESPACE   NAME        READY   STATUS             RESTARTS   AGE
default     app-pod     0/1     CrashLoopBackOff   5          10m
default     app-pod-2   0/1     ImagePullBackOff   0          2m
default     app-pod-3   0/1     Pending            0          1m
```

#### 第二步：按状态选择排查路径

| Pod 状态 | 说明 | 排查入口 | 详细文档 |
|---------|------|---------|---------|
| **Pending** | Pod 已创建但未调度到节点 | `kubectl describe pod` → Events 含调度失败原因 | [Pod 一直处于 Pending 状态](../Pod%20一直处于%20Pending%20状态/tccli%20操作.md) |
| **ContainerCreating** | Pod 已调度但容器未启动 | `kubectl describe pod` → Events 查看镜像拉取/卷挂载 | [Pod 一直处于 ContainerCreating 或 Waiting 状态](../Pod%20一直处于%20ContainerCreating%20或%20Waiting%20状态/tccli%20操作.md) |
| **ImagePullBackOff** | 镜像拉取失败 | `kubectl describe pod` → Events 含镜像错误 | [Pod 一直处于 ImagePullBackOff 状态](../Pod%20一直处于%20ImagePullBackOff%20状态/tccli%20操作.md) |
| **CrashLoopBackOff** | 容器反复崩溃重启 | `kubectl logs POD_NAME --previous` 查看上次崩溃日志 | [Pod 处于 CrashLoopBackOff 状态](../Pod%20处于%20CrashLoopBackOff%20状态/tccli%20操作.md) |
| **Terminating** | Pod 卡在删除中 | `kubectl describe pod` → 检查 finalizers | [Pod 一直处于 Terminating 状态](../Pod%20一直处于%20Terminating%20状态/tccli%20操作.md) |
| **健康检查失败** | Running 但 Not Ready | `kubectl describe pod` → Liveness/Readiness probe | [Pod 健康检查失败](../Pod%20健康检查失败/tccli%20操作.md) |

#### 第三步：通用排查命令

```bash
# 1. 查看 Pod 详情和事件
kubectl describe pod POD_NAME -n NAMESPACE
# expected: 含 Events 列表，最后几行通常是最新失败原因

# 2. 查看 Pod 日志
kubectl logs POD_NAME -n NAMESPACE --tail=50
# expected: 应用日志输出

# 3. 查看上一次崩溃日志
kubectl logs POD_NAME -n NAMESPACE --previous
# expected: 上一次运行的日志（CrashLoopBackOff 专用）

# 4. 查看节点状态
kubectl describe node NODE_NAME | grep -A10 "Conditions:"
# expected: MemoryPressure/DiskPressure/PIDPressure 为 False

# 5. 查看集群事件
kubectl get events -n NAMESPACE --sort-by='.lastTimestamp' | tail -20
# expected: 按时间排序的最近事件

# 6. 控制面：查看集群整体健康
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

## 验证

```bash
kubectl get pods -n NAMESPACE
# expected: 目标 Pod 状态恢复为 Running

kubectl describe pod POD_NAME -n NAMESPACE | grep "Conditions:"
# expected: Ready True
```

```text
NAME  STATUS  AGE
...
```

## 清理

排障概览页，无需清理。

## 排障

### Pod 状态速查表（4 列表格）

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Pod Pending | `kubectl describe pod POD_NAME` 查看 Events | 资源不足、节点选择器无匹配、PVC 未绑定 | 扩容节点/调整请求/创建 PVC |
| ContainerCreating | `kubectl describe pod` 查看 Events 和 Mounts | 镜像拉取慢/卷挂载失败/CNI 未就绪 | 检查镜像地址和拉取凭据/卷配置 |
| ImagePullBackOff | `kubectl describe pod` → Failed to pull image | 镜像不存在/仓库认证失败/网络不通 | `docker pull` 验证；配置 imagePullSecrets |
| CrashLoopBackOff | `kubectl logs --previous` | 应用启动失败/OOM/配置错误 | 分析崩溃日志修正应用配置 |
| Terminating | `kubectl describe pod` → finalizers | Finalizer 未完成/资源无法释放 | `kubectl patch pod POD_NAME -p '{"metadata":{"finalizers":[]}}'` |
| 健康检查失败 | `kubectl describe pod` → Liveness/Readiness | 探针路径错误/端口错误/超时太短 | 修正探针配置/增加 initialDelaySeconds |
| Pod 被驱逐 | `kubectl describe pod` → Evicted | 节点 DiskPressure/MemoryPressure | 释放节点资源；增加节点 |

## 下一步

- [Pod 一直处于 Pending 状态](../Pod%20一直处于%20Pending%20状态/tccli%20操作.md) -- page_id `42948`
- [Pod 处于 CrashLoopBackOff 状态](../Pod%20处于%20CrashLoopBackOff%20状态/tccli%20操作.md) -- page_id `43130`
- [通过 Exit Code 定位 Pod 异常退出和重启原因](../Pod%20异常排查工具/通过%20Exit%20Code%20定位%20Pod%20异常退出和重启原因/tccli%20操作.md) -- page_id `43125`
- [节点常见报错与处理](../../节点常见报错与处理/tccli%20操作.md) -- page_id `89869`

## 控制台替代

[容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) -> 集群 -> 工作负载 -> Pod -> 查看事件。
