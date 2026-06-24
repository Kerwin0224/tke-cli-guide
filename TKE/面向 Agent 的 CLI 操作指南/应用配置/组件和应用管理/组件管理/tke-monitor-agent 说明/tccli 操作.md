# tke-monitor-agent 说明（tccli）

> 对照官方：[tke-monitor-agent 说明](https://cloud.tencent.com/document/product/457/66815) · page_id `66815`

## 概述

tke-monitor-agent 是腾讯云容器服务升级基础监控架构后的新一代监控数据采集组件。它作为 DaemonSet 部署在每个节点上，负责采集容器、Pod、节点及官方组件的监控数据。

该组件的数据为以下功能提供支撑：
- 控制台基础监控指标展示
- 指标告警
- 基于基础指标的 HPA（水平伸缩）

部署后能极大改善之前因基础监控运行不稳定导致的监控数据无法正常获取的问题。

### 部署的 Kubernetes 对象

| 对象名称 | 类型 | 命名空间 |
|---------|------|---------|
| tke-monitor-agent | DaemonSet | kube-system |
| tke-monitor-agent | ServiceAccount | kube-system |
| tke-monitor-agent | ClusterRole | 集群级 |
| tke-monitor-agent | ClusterRoleBinding | 集群级 |

## 前置条件

- [环境准备](../../../../环境准备.md)
- 已 [连接集群](../../../../集群配置/集群管理/连接集群/tccli 操作.md)，kubectl 可正常访问 APIServer

确认集群状态正常：

```bash
tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'
```

```json
{
  "TotalCount": 0,
  "Clusters": [],
  "ClusterId": "<ClusterId>",
  "ClusterName": "<ClusterName>",
  "ClusterDescription": "<ClusterDescription>",
  "ClusterVersion": "<ClusterVersion>",
  "ClusterOs": "<ClusterOs>",
  "ClusterType": "<ClusterType>"
}
```

**CAM 权限要求：**

| 操作 | 所需权限 |
|------|---------|
| 安装组件 | `tke:InstallAddon` |
| 查看组件状态 | `tke:DescribeAddon` |
| 查看组件配置 | `tke:DescribeAddonValues` |
| 卸载组件 | `tke:DeleteAddon` |

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 查看集群列表 | `tccli tke DescribeClusters` | 是 |
| 查看组件详情 | `tccli tke DescribeAddon --AddonName tke-monitor-agent` | 是 |
| 安装组件 | `tccli tke InstallAddon --AddonName tke-monitor-agent` | 否（重复安装报错） |
| 更新组件 | `tccli tke UpdateAddon --AddonName tke-monitor-agent` | 否 |
| 查看组件列表 | `tccli tke DescribeAddonValues --AddonName tke-monitor-agent` | 是 |
| 查看监控数据 | 控制台图表（无直接 tccli API） | 是 |
| 卸载组件 | `tccli tke DeleteAddon --AddonName tke-monitor-agent` | 是 |

## 操作步骤

### 1. 安装 tke-monitor-agent 组件（控制面）

```bash
tccli tke InstallAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName tke-monitor-agent
```

### 2. 查看监控数据（控制台）

安装后，可通过控制台查看监控：

1. 导航至 **集群详情 > 工作负载 > DaemonSet > kube-system > tke-monitor-agent > 详情**。
2. 通过 **监控** 页签查看 Agent 自身的 CPU/内存消耗。
3. 通过各工作负载、节点、Pod 详情页的 **监控** 页签查看采集到的业务监控数据。

### 3. 手动查看采集源数据（数据面）

> **注意**：kubectl 数据面当前因公网端点被 CAM 策略 strategyId:240463971 拒绝而不可达。以下 kubectl 命令为参考格式，需在内网/VPN 环境执行。

Agent 通过 kubelet `/metrics` 端点获取 cAdvisor 指标。可手动验证端点可达性：

```bash
kubectl get --raw /api/v1/nodes/<NodeName>/proxy/metrics/cadvisor | head -50
```

```text
NAME  STATUS  AGE
...
```

查看 tke-monitor-agent Pod 日志：

```bash
kubectl logs -n kube-system ds/tke-monitor-agent --tail=30
```

### 组件 RBAC 权限（控制面已自动创建）

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: tke-monitor-agent
rules:
  - apiGroups: ["apps"]
    resources: ["replicasets"]
    verbs: ["list", "watch"]
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["list", "watch"]
  - apiGroups: [""]
    resources: ["nodes", "nodes/proxy", "nodes/metrics"]
    verbs: ["list", "watch", "get"]
  - apiGroups: [""]
    resources: ["services"]
    verbs: ["list", "watch"]
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["list", "watch"]
  - apiGroups: ["monitor.tencent.io"]
    resources: ["custommetrics"]
    verbs: ["update"]
```

| 功能 | 涉及对象 | 操作 |
|------|---------|------|
| 采集 Pod 数量和 Pod 相关信息 | replicasets, deployments, pods | list, watch |
| 访问 kubelet /metrics 获取 cAdvisor 指标 | nodes, nodes/proxy, nodes/metrics | list, watch, get |


```json
{
  "Addons": [],
  "AddonName": "<AddonName>",
  "AddonVersion": "<AddonVersion>",
  "RawValues": "<RawValues>",
  "Phase": "<Phase>",
  "Reason": "<Reason>",
  "CreateTime": "<CreateTime>",
  "RequestId": "<RequestId>"
}
```| 传递指标到 cluster-monitor | services | list, watch |
| 上报指标到 hpa-metrics-server | custommetrics（monitor.tencent.io） | update |

## 验证

### 控制面 (tccli)

```bash
tccli tke DescribeAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName tke-monitor-agent
```

```json
{
  "RequestId": "..."
}
```

**预期输出：** 组件状态为 `Succeeded`，AddonName 为 `tke-monitor-agent`。

### 数据面 (kubectl)

> **注意**：kubectl 数据面当前因公网端点被 CAM 策略 strategyId:240463971 拒绝而不可达。以下 kubectl 命令为参考格式，需在内网/VPN 环境执行。

验证 DaemonSet 运行状态：

```bash
kubectl get ds -n kube-system tke-monitor-agent
```

```text
NAME  STATUS  AGE
...
```

**预期输出：** `DESIRED` = `CURRENT` = `READY`，每个节点一个 Pod。

检查 Pod 状态：

```bash
kubectl get pod -n kube-system -l name=tke-monitor-agent
```

```text
NAME  STATUS  AGE
...
```

**预期输出：** 所有 Pod 状态为 `Running`。

验证基本监控功能（控制台）：
- 前往任意节点详情页的 **监控** 页签，确认 CPU、内存、磁盘等指标有数据显示。

### 资源消耗参考

资源消耗与节点上运行的 Pod 和容器数量成正比。压力测试参考（每节点 220 Pod × 3 容器）：
- **内存峰值：** ~40MiB
- **CPU 峰值：** 0.01 core

## 清理

### 控制面 (tccli)

```bash
tccli tke DeleteAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName tke-monitor-agent
```

> **说明：** 卸载后，控制台基础监控指标将不再更新，基于基础监控的 HPA 和告警规则可能失效。建议在业务低峰期操作。

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Pod 处于 `Pending` | `kubectl describe pod -n kube-system <podName>` 查看 Events | 节点资源不足无法调度（默认有资源请求） | 将 `resources.requests` 中的 CPU/内存值设为 `0` 以允许调度 |
| Pod 处于 `Evicted`（已驱逐） | `kubectl describe pod -n kube-system <podName>` 查看 Message 和 Events 字段 | 节点资源短缺或负载过高 | 释放节点资源 |
| Pod 处于 `CrashLoopBackOff` 或 `OOMKilled` | `kubectl logs -n kube-system <podName>` 查看日志 | 内存不足被杀死 | 增大内存限制（最大不超过 100MiB）；若 100M 仍 OOM 则提工单 |
| Pod 长期处于 `ContainerCreating` | `kubectl describe pod -n kube-system <podName>` 在 Events 中查找 `no space left on device` | 容器数据盘空间已满 | 清理节点数据盘 |
| 控制台监控图表无数据 | `kubectl get ds -n kube-system tke-monitor-agent` 确认所有节点就绪；检查 kubelet 是否正常运行 | Agent 未成功启动或 kubelet `/metrics` 不可达 | 重新安装组件；确认 kubelet 正常运行 |
| `Unable to connect to the server` | `kubectl cluster-info` 检查连通性 | 公网端点被 CAM 策略 strategyId:240463971 以 `tke:clusterExtranetEndpoint=true` 条件拒绝 | 通过 IOA/VPN 连接内网，或使用同 VPC CVM 执行 kubectl 命令 |

## 下一步

- [tke-event-collector 说明](../tke-event-collector 说明/tccli 操作.md) — 集群事件采集组件
- [tke-log-agent 说明](../tke-log-agent 说明/tccli 操作.md) — 容器日志采集组件
- [组件的生命周期管理](../组件的生命周期管理/tccli 操作.md) — 组件安装、升级与卸载
- [HPA 基于基础指标配置](https://cloud.tencent.com/document/product/457/37384) — 使用监控指标驱动自动伸缩

## 控制台替代

通过 [TKE 控制台安装 tke-monitor-agent 组件](https://cloud.tencent.com/document/product/457/66815) 操作。
