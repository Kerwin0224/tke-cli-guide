# nvidia-gpu 说明（tccli）

> 对照官方：https://cloud.tencent.com/document/product/457/104593

## 概述

nvidia-gpu 组件自动发现节点上的 NVIDIA GPU（`nvidia.com/gpu`）作为可调度资源，帮助运行依赖 GPU 的容器。同时自动部署监控 Exporter，提供 GPU 卡级别、Pod 级别和容器级别的监控数据。

### 部署对象

| Kubernetes 对象名称 | 类型 | 默认占用资源 | 所属 Namespaces |
| --- | --- | --- | --- |
| nvidia-device-plugin-daemonset | DaemonSet | 0.1 核 CPU, 100MB 内存 | kube-system |
| nvidia-gpu-exporter | DaemonSet | 0.1 核 CPU, 100MB 内存 | kube-system |
| nvidia-gpu-exporter | ServiceAccount | — | kube-system |
| nvidia-gpu-exporter | ClusterRole | — | — |
| nvidia-gpu-exporter | ClusterRoleBinding | — | — |
| nvidia-gpu-exporter | Service | — | kube-system |

### 限制条件

- 支持 Kubernetes 1.16 及以上版本。

### 重要提示

对于已有 GPU 节点的集群，首次安装时需检查是否已存在 DaemonSet `nvidia-device-plugin-daemonset`。如存在，需先将其删除，再安装 nvidia-gpu。删除该 DaemonSet 不会影响已运行的 GPU 业务 Pod，安装期间新建的 GPU Pod 会在安装完成后自动运行。

### 监控依赖

查看 GPU 监控数据需要 monitor-agent 组件版本 ≥ 1.3.11。

## 前置条件

环境准备请参考 [环境准备](../../../../环境准备.md)，连接集群请参考 [连接集群（tccli）](../../../../集群配置/集群管理/连接集群/tccli 操作.md)。

先确认集群状态正常：

```bash
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

### CAM 权限

| 操作 | API 权限 |
| --- | --- |
| 安装组件 | `tke:InstallAddon` |
| 查询组件状态 | `tke:DescribeAddon` |
| 查询组件配置 | `tke:DescribeAddonValues` |
| 更新组件 | `tke:UpdateAddon` |
| 删除组件 | `tke:DeleteAddon` |

## 控制台与 CLI 参数映射

控制台「组件管理」页面对应的 tccli 核心命令：

| 控制台操作 | tccli 命令 |
| --- | --- |
| 查询集群组件列表 | `tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'`（含 Addons 字段） |
| 查询组件详情 | `tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName nvidia-gpu` |
| 安装组件 | `tccli tke InstallAddon --region <Region> --ClusterId <ClusterId> --AddonName nvidia-gpu --AddonVersion <AddonVersion>` |
| 更新组件 | `tccli tke UpdateAddon --region <Region> --ClusterId <ClusterId> --AddonName nvidia-gpu --AddonVersion <AddonVersion>` |
| 删除组件 | `tccli tke DeleteAddon --region <Region> --ClusterId <ClusterId> --AddonName nvidia-gpu` |

## 操作步骤

### 1. 检查已有 nvidia-device-plugin DaemonSet（存量 GPU 节点集群）

对于已有 GPU 节点的集群，首次安装前需确认是否已存在旧的 device-plugin DaemonSet。

> **注意**：kubectl 数据面当前因公网端点被 CAM 策略 strategyId:240463971 拒绝而不可达。以下 kubectl 命令为参考格式，需在内网/VPN 环境执行。

```bash
kubectl get daemonset nvidia-device-plugin-daemonset -n kube-system 2>/dev/null
# expected: 无输出（不存在）或显示已有 DaemonSet
```

```text
NAME  STATUS  AGE
...
```

如有输出，先删除：

```bash
kubectl delete daemonset nvidia-device-plugin-daemonset -n kube-system
# expected: daemonset.apps "nvidia-device-plugin-daemonset" deleted
```

### 2. 安装 nvidia-gpu 组件

通过 InstallAddon 安装组件：

```bash
tccli tke InstallAddon --region <Region> --ClusterId <ClusterId> --AddonName nvidia-gpu --AddonVersion <AddonVersion>
# expected: 返回安装任务 RequestId
```

## 验证

查询组件安装状态：

```bash
tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName nvidia-gpu
# expected: Phase "Running"
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

> **注意**：kubectl 数据面当前因公网端点被 CAM 策略 strategyId:240463971 拒绝而不可达。以下 kubectl 命令为参考格式，需在内网/VPN 环境执行。

检查 DaemonSet 运行状态：

```bash
kubectl get daemonset -n kube-system nvidia-device-plugin-daemonset nvidia-gpu-exporter
# expected: DESIRED 与 READY 数量一致
```

```text
NAME  STATUS  AGE
...
```

验证 GPU 资源已注册到节点：

```bash
kubectl describe node <NodeName> | grep nvidia.com/gpu
# expected: 资源数量大于 0（如 nvidia.com/gpu: 1）
```

```text
NAME  STATUS  AGE
...
```

查看 GPU 监控指标（需 monitor-agent ≥ 1.3.11）：

```bash
kubectl get --raw /apis/metrics.k8s.io/v1beta1 | jq '.resources[] | select(.name | startswith("gpu"))'
# expected: 返回 gpu 相关 metrics 资源
```

```text
NAME  STATUS  AGE
...
```

## 清理

> **计费提醒**：nvidia-gpu 组件本身免费，卸载后仅停止 GPU 监控数据采集，不影响 GPU 资源使用和计费。

通过 DeleteAddon 卸载组件：

```bash
tccli tke DeleteAddon --region <Region> --ClusterId <ClusterId> --AddonName nvidia-gpu
# expected: 返回删除任务
```

验证已卸载：

```bash
tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName nvidia-gpu
# expected: Phase "NotFound" 或返回 NotFound 错误
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

## 排障

| 现象 | 诊断 | 根因 | 修复 |
| --- | --- | --- | --- |
| 安装后 GPU 节点未注册 `nvidia.com/gpu` 资源 | `kubectl describe daemonset nvidia-device-plugin-daemonset -n kube-system` 检查 Pod 调度和启动事件 | DaemonSet Pod 未成功启动 | 确认节点存在 GPU 硬件并检查 Pod 调度事件，修复调度问题后重启 Pod |
| GPU 监控数据为空 | 查询 monitor-agent 版本 | monitor-agent 版本过低（低于 1.3.11） | 通过 UpdateAddon 升级 monitor-agent 至 1.3.11 及以上版本 |
| 存量集群首次安装报错 | 检查是否存在旧 `nvidia-device-plugin-daemonset` | 旧 DaemonSet 与组件安装冲突 | `kubectl delete daemonset nvidia-device-plugin-daemonset -n kube-system` 后重试安装 |
| DaemonSet Pod CrashLoopBackOff | `kubectl logs -n kube-system -l name=nvidia-device-plugin-daemonset` 查看日志 | 节点 NVIDIA 驱动未安装或版本不兼容 | 安装或更新节点 NVIDIA 驱动，确认驱动版本与组件兼容 |
| tccli/kubectl Unable to connect | 检查公网端点访问策略 | 公网端点被 CAM 策略 strategyId:240463971（`tke:clusterExtranetEndpoint=true`）拒绝 | 在内网/VPN 环境执行命令，或调整 CAM 策略放行公网端点 |

## 下一步

- [查看监控数据](https://cloud.tencent.com/document/product/457/xxxxx) — 了解 GPU 监控指标和告警配置
- [tke-monitor-agent 说明](https://cloud.tencent.com/document/product/457/xxxxx) — 监控组件版本要求说明
- [NVIDIA MPS 特性介绍](https://cloud.tencent.com/document/product/457/xxxxx) — GPU 多进程服务
- [使用 GPU 节点](https://cloud.tencent.com/document/product/457/xxxxx) — 创建和管理 GPU 工作负载

## 控制台替代

如需通过控制台操作，登录 [腾讯云容器服务控制台](https://console.cloud.tencent.com/tke2/cluster)，进入集群详情 →「组件管理」→ 找到 nvidia-gpu → 进行安装、更新或卸载。
