# NodeProblemDetectorPlus 说明（tccli）

> 对照官方：https://cloud.tencent.com/document/product/457/49422

## 概述

Node-Problem-Detector-Plus（NPD Plus）是 Kubernetes 集群节点的健康监测组件。它作为 DaemonSet 在 TKE 环境中运行，实时检测各种节点异常，并将结果上报至上游 kube-apiserver。监测结果通过 **Node Condition** 和 **Event** 对象体现。

用户可以监控指标提前感知资源压力，在 Kubernetes 驱逐 Pod 前手动释放或扩容资源，防止因资源回收或节点不可用带来的损失。

### 部署的 Kubernetes 对象

| 对象名称 | 类型 | 资源 | 命名空间 |
|---------|------|------|---------|
| node-problem-detector | DaemonSet | 0.1C / 200M | kube-system |
| node-problem-detector | ServiceAccount | — | kube-system |
| node-problem-detector | ClusterRole | — | 集群级 |
| node-problem-detector | ClusterRoleBinding | — | 集群级 |

### 检测项清单（Node Conditions）

| Condition 类型 | 默认值 | 说明 |
|---------------|--------|------|
| ReadonlyFilesystem | False | 文件系统是否只读 |
| FDPressure | False | 文件描述符是否达到最大值的 80% |
| FrequentKubeletRestart | False | Kubelet 在 20 分钟内是否重启超过 5 次 |
| CorruptDockerOverlay2 | False | Docker 镜像是否存在问题（overlay2 损坏） |
| KubeletProblem | False | Kubelet 服务是否 Running |
| KernelDeadlock | False | 内核是否存在死锁 |
| FrequentDockerRestart | False | Docker 在 20 分钟内是否重启超过 5 次 |
| FrequentContainerdRestart | False | Containerd 在 20 分钟内是否重启超过 5 次 |
| DockerdProblem | False | Docker 服务是否 Running（Containerd 运行时下始终为 False） |
| ContainerdProblem | False | Containerd 服务是否 Running（Docker 运行时下始终为 False） |
| ThreadPressure | False | 系统线程数是否达到最大值的 90% |
| NetworkUnavailable | False | NTP 服务是否 Running（与网络可用性关联） |
| SerfFailed | False | 分布式检测节点网络健康状况 |
| CustomPIDPressure | False | 当前节点 PID 数是否达到最大值的 90% |
| NFConntrackPressure | False | Conntrack 表使用量是否达到总容量的 90% |

> **说明：** Condition 类型值 `True` 表示异常，`False` 表示正常。`Unknown` 表示无法判断。

### 组件 RBAC 权限（控制面已自动创建）

组件安装时自动创建 ClusterRole `node-problem-detector`：

| 功能 | API 资源 | 操作 |
|------|---------|------|
| 上报故障信息，修改节点 Condition | nodes/status | patch |
| 发送事件通知 | events | create, patch, update |
| 获取节点信息 | nodes | get |

## 前置条件

环境准备参见 [环境准备](../../../../环境准备.md)，连接集群参见 [连接集群](../../../../集群配置/集群管理/连接集群/tccli 操作.md)。

先确认集群状态：

```bash
tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'
# expected: 返回集群信息，ClusterId 与 <ClusterId> 一致，ClusterStatus 为 Running
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

其他前置条件：

- 集群处于 Running 状态，kubectl 可正常访问 APIServer。
- 节点资源充足，每个节点至少预留 0.1C / 200M 给 DaemonSet。

CAM 权限：

- `tke:InstallAddon`
- `tke:DescribeAddon`
- `tke:DescribeAddonValues`
- `tke:UpdateAddon`
- `tke:DeleteAddon`

## 控制台与 CLI 参数映射

| 操作 | 控制台路径 | tccli 命令 |
|---|---|---|
| 查看集群信息 | 集群管理 | `tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'` |
| 查看组件列表 | 组件管理 | `tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId>` |
| 查看组件详情 | 组件管理 → NodeProblemDetectorPlus | `tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName node-problem-detector-plus` |
| 安装组件 | 组件管理 → 新建 | `tccli tke InstallAddon --region <Region> --ClusterId <ClusterId> --AddonName node-problem-detector-plus` |
| 升级组件 | 组件管理 → 更新 | `tccli tke UpdateAddon --region <Region> --ClusterId <ClusterId> --AddonName node-problem-detector-plus` |
| 删除组件 | 组件管理 → 删除 | `tccli tke DeleteAddon --region <Region> --ClusterId <ClusterId> --AddonName node-problem-detector-plus` |

## 操作步骤

### 1. 安装 NodeProblemDetectorPlus 组件（控制面）

```bash
tccli tke InstallAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName node-problem-detector-plus
# expected: 返回 RequestId，组件开始安装
```

### 2. 查看检测结果（数据面）

安装后，节点对象上会新增 Condition 条目。

> **注意**：kubectl 数据面当前因公网端点被 CAM 策略 strategyId:240463971 拒绝而不可达。以下 kubectl 命令为参考格式，需在内网/VPN 环境执行。

#### 查看节点 Condition

```bash
kubectl describe node <NodeName> | grep -A1 Conditions -m1
```

```text
Name:         ...
Status:       Running
...
```

#### 查看特定节点上的 NPD 事件

```bash
kubectl get event --field-selector involvedObject.kind=Node,involvedObject.name=<NodeName>
```

```text
NAME  STATUS  AGE
...
```

#### 查看指定 Condition（例如 KernelDeadlock）

```bash
kubectl get node <NodeName> -o jsonpath='{.status.conditions[?(@.type=="KernelDeadlock")]}'
# expected: {"type":"KernelDeadlock","status":"False"}
```

```text
NAME  STATUS  AGE
...
```

## 验证

### 控制面 (tccli)

```bash
tccli tke DescribeAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName node-problem-detector-plus
# expected: 组件状态为 Succeeded，AddonName 为 node-problem-detector-plus
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

### 数据面 (kubectl)

> **注意**：kubectl 数据面当前因公网端点被 CAM 策略 strategyId:240463971 拒绝而不可达。以下 kubectl 命令为参考格式，需在内网/VPN 环境执行。

验证 DaemonSet 运行状态：

```bash
kubectl get ds -n kube-system node-problem-detector
# expected: DESIRED = CURRENT = READY，每个 worker 节点一个 Pod
```

```text
NAME  STATUS  AGE
...
```

查看节点 Condition 确认 NPD 已注入检测项：

```bash
kubectl get node <NodeName> -o jsonpath='{.status.conditions[?(@.type=="KernelDeadlock")]}'
# expected: 包含 "type":"KernelDeadlock" 的 JSON 对象，status 为 False（正常节点）
```

```text
NAME  STATUS  AGE
...
```

查看 NPD Pod 日志：

```bash
kubectl logs -n kube-system ds/node-problem-detector --tail=20
# expected: 日志中包含各检测器启动信息，无持续报错
```

```text
NAME  STATUS  AGE
...
```

## 清理

### 控制面 (tccli)

```bash
tccli tke DeleteAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName node-problem-detector-plus
# expected: 返回 RequestId，组件开始卸载
```

> **说明：** 卸载 NPD Plus 后节点 Condition 中的健康检测项将不再更新。如果同时安装了 Node-Healing 组件且依赖 NPD 检测结果，请先处理 Node-Healing 配置。

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| DaemonSet Pod 处于 Pending | `kubectl describe pod -n kube-system <npd-pod>` 查看 Events | 节点资源不足，无法分配 200M 内存 | 释放节点资源或降低 `resources.requests.memory`；扩容节点后重试 |
| 节点 Condition 未出现预期检测项 | `kubectl get pod -n kube-system` 检查 NPD Pod | Pod 未调度到该节点或 Pod 异常 | 检查 DaemonSet `nodeSelector` 或 tolerations 是否覆盖所有节点；确认 Pod 日志无报错 |
| SerfFailed 为 True | 检查集群网络插件与节点间连通性 | 集群网络通信异常（Serf 基于 gossip 协议） | 排查节点网络（安全组、路由、CNI） |
| FrequentKubeletRestart 频繁触发 | `journalctl -u kubelet` 查看 kubelet 日志 | kubelet 因配置错误或资源不足频繁重启 | 修复 kubelet 配置，确保资源充足 |
| Unable to connect to server / 公网端点不可达 | 实际集群 cls-xxxxxxxx，`tccli tke DescribeClusters` 查看公网端点 | 公网端点被 CAM 策略 strategyId:240463971 拒绝（tke:clusterExtranetEndpoint=true） | 在内网/VPN 环境执行 kubectl 命令；或为 CAM 策略放行 tke:clusterExtranetEndpoint |

## 下一步

- [Node-Healing 说明](../Node-Healing 说明/tccli 操作.md) — 基于 NPD 检测结果实现节点自动修复
- [组件的生命周期管理](../组件的生命周期管理/tccli 操作.md) — 组件安装、升级与卸载
- [OOMGuard 说明](../OOMGuard 说明/tccli 操作.md) — 内存 OOM 防护
- [GPU 故障检测和自愈](https://cloud.tencent.com/document/product/457/127503) — GPU 节点的专项检测与自愈

## 控制台替代

通过 [TKE 控制台安装 NodeProblemDetectorPlus 组件](https://cloud.tencent.com/document/product/457/49422) 操作。
