# OOMGuard 说明（tccli）

> 对照官方：[OOMGuard 说明](https://cloud.tencent.com/document/product/457/49220) · page_id `49220`

## 概述

OOMGuard 是一个在用户态处理容器 cgroup OOM 的 TKE 组件。核心思路：在内核 cgroup OOM killer 触发之前，OOMGuard 从用户态终止超限容器，降低命中因 cgroup 内存回收失败导致内核代码路径进而引发节点故障（宕机、重启、进程卡死）的概率。

当 cgroup OOM 发生且 OOMGuard 杀死容器时，会上报 `OomGuardKillContainer` 事件至 Kubernetes，可通过 `kubectl get event` 查看。

### 工作原理

1. OOMGuard 在 memory cgroup 上设置 **threshold notify**，当内存使用量越过阈值时收到内核通知（机制参考：[threshold notify](https://lwn.net/Articles/529927/)）。
2. 触发阈值前，OOMGuard 写入 `memory.force_empty` 触发对应 cgroup 的内存回收。
3. 若达到阈值时 `memory.stat` 仍显示大量 cache，则**不触发**后续 kill 策略，由内核自然 OOM 处理。

**阈值计算示例：**

Pod memory limit = 1000M：
- `margin = 1000M × margin_ratio(0.02) = 20M`
- margin 钳位：最小值 `min_margin(1M)`，最大值 `max_margin(50M)`
- `threshold = limit - margin = 1000M - 20M = 980M`

当 Pod 内存达到 980M，OOMGuard 收到通知并执行相应策略。

### 关键约束

- **仅适用于 CentOS 7.2/7.6 自带的内核（尤其是 3.10 低版本内核）**，其他 OS/镜像版本**不需要安装**此组件。
- Containerd socket 路径不可从 TKE 默认值修改：
  - Docker 运行时：`/run/docker/containerd/docker-containerd.sock`
  - Containerd 运行时：`/run/containerd/containerd.sock`
- Cgroup memory 子系统挂载点不可从默认值修改：`/sys/fs/cgroup/memory`

### 部署的 Kubernetes 对象

| 对象名称 | 类型 | 默认资源 | 命名空间 |
|---------|------|---------|---------|
| oomguard | ServiceAccount | — | kube-system |
| system:oomguard | ClusterRoleBinding | — | — |
| system:oomguard | ClusterRole | — | — |
| oom-guard | DaemonSet | 0.02C / 120MB | kube-system |

## 前置条件

- [环境准备](../../../../环境准备.md)
- 已 [连接集群](../../../../集群配置/集群管理/连接集群/tccli 操作.md)，kubectl 可正常访问 APIServer
- 确认集群节点使用 CentOS 7.2/7.6 自带内核（其他 OS 请勿安装）
- 确认 containerd socket 路径和 cgroup memory 挂载点均为 TKE 默认值
- 集群状态 `Running`

```bash
# 检查集群状态
tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'
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

- 需要 CAM 权限：`tke:DescribeClusters`、`tke:DescribeAddon`、`tke:InstallAddon`、`tke:UpdateAddon`、`tke:DeleteAddon`

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 查看集群信息 | `tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'` | 是 |
| 查看 OOMGuard 组件详情 | `tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName oomguard` | 是 |
| 安装 OOMGuard 组件 | `tccli tke InstallAddon --region <Region> --ClusterId <ClusterId> --AddonName oomguard` | 否 |
| 升级 OOMGuard 组件 | `tccli tke UpdateAddon --region <Region> --ClusterId <ClusterId> --AddonName oomguard --AddonVersion <AddonVersion>` | 否 |
| 卸载 OOMGuard 组件 | `tccli tke DeleteAddon --region <Region> --ClusterId <ClusterId> --AddonName oomguard` | 是 |
| 查看 OOMGuard 事件 | `kubectl get event --field-selector reason=OomGuardKillContainer` | 是 |
| 修改策略参数 | `kubectl edit ds -n kube-system oom-guard` | 是 |

## 操作步骤

### 步骤 1：安装 OOMGuard 组件（控制面）

```bash
tccli tke InstallAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName oomguard
# expected: exit 0，组件安装请求已提交
```

### 步骤 2：配置 OOMGuard 策略参数（数据面）

OOMGuard DaemonSet 支持以下启动参数，可通过编辑 DaemonSet 调整：

```bash
kubectl edit ds -n kube-system oom-guard
# expected: 进入编辑模式
```

关键参数：

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `--policy` | `container` | 达到阈值后的处理策略 |
| `--margin_ratio` | `0.02` | 用于计算 margin 的 limit 比例 |
| `--min_margin` | `1M` | margin 最小值 |
| `--max_margin` | `50M` | margin 最大值 |

#### `--policy` 策略说明

| 策略值 | 行为 |
|--------|------|
| **container**（默认） | 选择 cgroup 下的一个 Docker 容器，终止整个容器 |
| **process** | 与内核 cgroup OOM killer 策略相同——选择 cgroup 内 `oom_score` 最高的进程，发送 SIGKILL |
| **noop** | 仅记录日志，不执行任何操作 |

> **说明：** 修改 `--policy` 参数后需重启 DaemonSet Pod 生效。

> **注意**：kubectl 数据面当前因公网端点被 CAM 策略 strategyId:240463971 拒绝而不可达。以下 kubectl 命令为参考格式，需在内网/VPN 环境执行。

### 步骤 3：查看 OOMGuard 事件（数据面）

当 OOMGuard 触发容器终止时会生成 `OomGuardKillContainer` 事件：

```bash
kubectl get event --all-namespaces --field-selector reason=OomGuardKillContainer
# expected: 返回 OomGuardKillContainer 事件列表
```

```text
NAME  STATUS  AGE
...
```

### 组件 RBAC 权限（控制面已自动创建）

组件安装时自动创建 ClusterRole `system:oomguard`：

| API Group | 资源 | 操作 | 用途 |
|-----------|------|------|------|
| `""` (core) | events | create, patch, update | 上报 OomGuardKillContainer 事件 |

## 验证

### 控制面（tccli）

```bash
tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName oomguard \
    | jq '.Addons[0] | {AddonName, Status, AddonVersion}'
# expected: Status "Running"，AddonName "oomguard"
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

### 数据面（kubectl）

```bash
kubectl get ds -n kube-system oom-guard
# expected: DESIRED = CURRENT = READY，所有节点均已部署

kubectl logs -n kube-system ds/oom-guard --tail=20
# expected: 日志包含 Starting OOMGuard，无报错

kubectl describe clusterrole system:oomguard
# expected: Rules 包含对 events 的 create, patch, update 权限
```

```text
NAME  STATUS  AGE
...
```

> **注意**：kubectl 数据面当前因公网端点被 CAM 策略 strategyId:240463971 拒绝而不可达。以下 kubectl 命令为参考格式，需在内网/VPN 环境执行。

## 清理

### 控制面（tccli）

```bash
tccli tke DeleteAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName oomguard
# expected: exit 0，组件卸载请求已提交

# 验证已卸载
tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName oomguard \
    | jq '.Addons[] | select(.AddonName == "oomguard")'
# expected: 空结果
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

> **计费提醒：** OOMGuard 为免费组件，不产生额外计费。卸载后节点将恢复由内核 cgroup OOM killer 直接处理内存超限。

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| OOMGuard Pod 处于 `CrashLoopBackOff` | `kubectl describe pod -n kube-system -l app=oom-guard` 查看 Events | containerd socket 路径被修改（非 TKE 默认值） | 检查 containerd socket 路径是否为 TKE 默认值 `/run/docker/containerd/docker-containerd.sock` 或 `/run/containerd/containerd.sock` |
| OOMGuard 不触发 kill | `kubectl get ds -n kube-system oom-guard -o yaml` 检查 `--policy` 参数 | `--policy=noop` 被设置或 cgroup memory 子系统挂载点异常 | 检查 DaemonSet `--policy` 参数；确认 `/sys/fs/cgroup/memory` 存在且可写 |
| 容器 OOM 后节点仍出现卡死/重启 | 检查节点 OS 与内核版本 | 内核 3.10 缺陷导致 memory reclamation 失败路径被命中 | OOMGuard 无法 100% 覆盖所有内核 OOM 路径，建议升级节点 OS 至 CentOS 7.4+ 或更换内核版本 |
| 安装后大量 `OomGuardKillContainer` 事件 | `kubectl get event --field-selector reason=OomGuardKillContainer` 查看事件频率 | 集群内业务容器内存限制过低 | 调高受影响 Pod 的 memory limit，或优化应用内存使用 |
| `Unable to connect to the server` | `kubectl cluster-info` 检查连通性 | 公网端点被 CAM 策略 strategyId:240463971 以 `tke:clusterExtranetEndpoint=true` 条件拒绝 | 通过 IOA/VPN 连接内网，或使用同 VPC CVM 执行 kubectl 命令 |

## 下一步

- [资源利用率分析和优化](https://cloud.tencent.com/document/product/457/32957) — 优化集群资源使用，减少 OOM 发生
- [组件的生命周期管理](../组件的生命周期管理/tccli 操作.md) — 组件安装、升级与卸载
- [NodeProblemDetectorPlus 说明](../NodeProblemDetectorPlus 说明/tccli 操作.md) — 节点健康检测组件
- [Node-Healing 说明](../Node-Healing 说明/tccli 操作.md) — 节点自愈组件

## 控制台替代

通过 [TKE 控制台安装 OOMGuard 组件](https://cloud.tencent.com/document/product/457/49220) 操作。
