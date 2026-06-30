# DeScheduler 说明（tccli）

> 对照官方：https://cloud.tencent.com/document/product/457/50921

## 概述

DeScheduler 是 TKE 基于 Kubernetes 社区 [DeScheduler](https://github.com/kubernetes-sigs/descheduler) 开发的插件，基于节点真实负载进行重调度。安装后与 kube-scheduler 配合，监控高负载节点并将低优先级 Pod 进行驱逐。TKE 推荐与 **Dynamic Scheduler** 配套使用，实现多维度的负载均衡。

> **废弃声明**：DeScheduler 和 DynamicScheduler 扩展组件已于 **2023 年 11 月 30 日** 下线。该日期后新安装不再支持，存量组件无法更新。请迁移至 **TKE 原生节点专用调度器**，该调度器覆盖了两个组件的功能，无需 Prometheus 依赖并支持节点虚拟超卖。

### 工作原理

- 基于 Prometheus 监控和 node_exporter 指标（CPU 利用率、内存利用率、网络 IO、系统负载），判断真实节点负载。
- 当节点 5 分钟平均 CPU 或内存利用率超过配置阈值时，DeScheduler 将其识别为高负载节点，执行 Pod 驱逐逻辑，尝试通过重调度将节点负载降至目标利用率以下。
- 社区策略（如 `LowNodeUtilization`）依赖 Pod request/limit 数据，仅从资源分配角度平衡，无法反映真实负载差异。

## 前置条件

环境准备：[环境准备](../../../../环境准备.md)；连接集群：[连接集群](../../../../集群配置/集群管理/连接集群/tccli 操作.md)。

集群状态检查：

```bash
# expected: 返回集群列表，确认目标集群 Status 为 Running
tccli tke DescribeClusters --region <Region>
```

### 集群与组件要求

- Kubernetes 版本 **≥ v1.10.x**。
- 集群至少 **5 个节点**，其中 **4 个及以上**节点的负载低于目标利用率（驱逐前提条件）。
- 已部署 **Prometheus 监控**（自建或托管）及相关规则配置。
- 已在目标工作负载（StatefulSet、Deployment 等）的 Pod 模板中添加驱逐注解 `descheduler.alpha.kubernetes.io/evictable: 'true'`（组件默认不驱逐任何 Pod）。

### CAM 权限

| 操作 | API 权限 |
|------|----------|
| 查询集群 | `tke:DescribeClusters` |
| 安装组件 | `tke:InstallAddon` |
| 查询组件信息 | `tke:DescribeAddon` |
| 查询组件参数 | `tke:DescribeAddonValues` |
| 更新组件 | `tke:UpdateAddon` |
| 删除组件 | `tke:DeleteAddon` |

### 部署的 Kubernetes 对象

| 对象名称 | 类型 | 资源占用 | 命名空间 |
|----------|------|----------|----------|
| descheduler | Deployment | CPU: 200m, 内存: 200Mi（1 实例） | kube-system |
| descheduler | ClusterRole | — | kube-system |
| descheduler | ClusterRoleBinding | — | kube-system |
| descheduler | ServiceAccount | — | kube-system |
| descheduler-policy | ConfigMap | — | kube-system |
| probe-prometheus | ConfigMap | — | kube-system |

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 说明 |
| --- | --- | --- |
| 查看集群列表 | `tccli tke DescribeClusters --region <Region>` | 获取 <ClusterId> |
| 查看组件详情 | `tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName DeScheduler` | 查询组件状态与参数 |
| 安装组件 | `tccli tke InstallAddon --region <Region> --ClusterId <ClusterId> --AddonName DeScheduler --AddonVersion <AddonVersion>` | 传入 Prometheus 地址、阈值等参数 |
| 更新组件 | `tccli tke UpdateAddon --region <Region> --ClusterId <ClusterId> --AddonName DeScheduler --Param <JSON>` | 修改组件配置（存量已废弃，无法更新） |
| 删除组件 | `tccli tke DeleteAddon --region <Region> --ClusterId <ClusterId> --AddonName DeScheduler` | 卸载组件 |

| Console 参数 | tccli 参数 | 幂等性 |
|-------------|-----------|--------|
| 组件选择（勾选 DeScheduler） | `AddonName: "DeScheduler"` | 是 |
| 组件版本 | `AddonVersion` | 否 |
| Prometheus 查询地址 | `RawValues` | 否 |
| 利用率阈值和目标利用率 | `RawValues` | 否 |

## 操作步骤

### Step 1: 部署 Prometheus 依赖

**方式一：自建 Prometheus**

部署 node-exporter 和 Prometheus，并配置聚合规则：

```yaml
groups:
   - name: cpu_mem_usage_active
     interval: 30s
     rules:
     - record: mem_usage_active
       expr: 100*(1-node_memory_MemAvailable_bytes/node_memory_MemTotal_bytes)
   - name: cpu-usage-1m
     interval: 1m
     rules:
     - record: cpu_usage_avg_5m
       expr: 100 - (avg by (instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
   - name: mem-usage-1m
     interval: 1m
     rules:
     - record: mem_usage_avg_5m
       expr: avg_over_time(mem_usage_active[5m])
```

> **注意**：若同时使用 DeScheduler 和 DynamicScheduler，请使用合并版规则（详见 DynamicScheduler 说明页），避免相互覆盖。

Prometheus 服务器配置：

```yaml
global:
   evaluation_interval: 30s
   scrape_interval: 30s
   external_labels:
rule_files:
- /etc/prometheus/rules/*.yml
```

将规则文件（如 `de-scheduler.yaml`）放入 Prometheus 容器的 `/etc/prometheus/rules/` 目录（通常通过 ConfigMap 挂载），然后重载 Prometheus 服务器。

**方式二：托管 Prometheus**

1. 在 TKE 控制台左侧导航进入 **Prometheus 监控**。
2. 创建与集群同 VPC 的 Prometheus 实例并关联集群。
3. 关联后 node-exporter 自动部署到各节点。
4. 配置 Prometheus 聚合规则（内容同自建方案）。规则保存后立即生效，无需重启。

### Step 2: 添加驱逐注解

在需要被驱逐的工作负载中添加注解：

```yaml
descheduler.alpha.kubernetes.io/evictable: 'true'
```

示例（Deployment Pod 模板）：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  template:
    metadata:
      annotations:
        descheduler.alpha.kubernetes.io/evictable: 'true'
```

### Step 3: 安装 DeScheduler 组件

```bash
# expected: 返回 RequestId，组件开始安装
tccli tke InstallAddon \
    --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName DeScheduler \
    --AddonVersion <AddonVersion>
```

安装后可配置参数（Prometheus 查询地址、利用率阈值、目标利用率等）。建议开启**事件持久化**以便监控和排查（驱逐事件的 reason 为 `"Descheduled"`）。

### 使用场景与风险控制

**适用场景**：解决社区 DeScheduler 仅基于 APIServer request/limit 数据调度导致的实际负载不均问题。

**风险控制**：
- 建议开启事件持久化以监控驱逐行为。
- 大量驱逐可能导致服务不可用。除 Kubernetes PDB 对象外，TKE DeScheduler 额外检查：调用驱逐 API 前验证工作负载的就绪 Pod 数是否超过副本数的一半，否则跳过驱逐。
- 在某些场景中 Pod 可能被反复调度到需要重调度的节点上，导致反复驱逐。解决方式：更换可调度节点或标记 Pod 不可驱逐。

## 验证

```bash
# expected: Phase 为 Running，组件已安装
tccli tke DescribeAddon \
    --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName DeScheduler
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

```bash
# expected: descheduler Deployment Ready 1/1
kubectl get deploy descheduler -n kube-system
```

```text
NAME  STATUS  AGE
...
```

```bash
# expected: 列出 reason=Descheduled 的驱逐事件
kubectl get events -A --field-selector reason=Descheduled
```

```text
NAME  STATUS  AGE
...
```

## 清理

```bash
# expected: 返回 RequestId，组件开始卸载
tccli tke DeleteAddon \
    --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName DeScheduler
```

> **注意**：kubectl 数据面当前因公网端点被 CAM 策略 strategyId:240463971 拒绝而不可达。以下 kubectl 命令为参考格式，需在内网/VPN 环境执行。

```bash
# expected: Error from server (NotFound): deployments.apps "descheduler" not found
kubectl get deploy descheduler -n kube-system
```

```text
NAME  STATUS  AGE
...
```

> **计费提醒**：DeScheduler 组件本身不产生额外费用。如使用了托管 Prometheus 服务，请注意相关计费。

## 排障

| 现象 | 诊断 | 根因 | 修复 |
| --- | --- | --- | --- |
| Pod 未被驱逐 | 检查工作负载 Pod 模板注解 | 未添加 `descheduler.alpha.kubernetes.io/evictable: 'true'` 注解，组件默认不驱逐任何 Pod | 为需驱逐的 Pod 模板添加 evictable 注解 |
| Pod 被反复驱逐 | 查看调度日志与节点负载 | Pod 持续被调度到高负载节点 | 更换可调度节点或标记 Pod 不可驱逐（移除注解）；检查节点亲和性和污点配置 |
| 组件不生效 | 检查 Prometheus 规则与关联状态 | Prometheus 聚合规则未配置或未关联集群 | 检查聚合规则是否正确加载；确认 Prometheus 已关联集群 |
| 安装/更新失败 | 确认组件是否已废弃 | DeScheduler 已于 2023-11-30 下线，新安装和更新均已停止 | 迁移至 TKE 原生节点专用调度器 |
| Unable to connect to the server | 检查公网端点与 CAM 策略 | 公网端点被 CAM 策略 strategyId:240463971（`tke:clusterExtranetEndpoint=true`）拒绝 | 通过内网/VPN 访问，或调整 CAM 策略 |

## 下一步

- [TKE 原生节点专用调度器](https://cloud.tencent.com/document/product/457)
- [DynamicScheduler 说明（tccli）](../DynamicScheduler%20说明/tccli%20操作.md)
- [Kubernetes Descheduler 社区文档](https://github.com/kubernetes-sigs/descheduler)
- [Prometheus 监控服务](https://cloud.tencent.com/document/product/1416)
- [TKE 组件管理概述](https://cloud.tencent.com/document/product/457/51082)

## 控制台替代

如需通过控制台操作：登录 [容器服务控制台 - 组件管理](https://console.cloud.tencent.com/tke2/cluster?rid=<Region>)，进入集群 → 组件管理 → 找到 DeScheduler → 安装/更新/卸载。组件已废弃，建议迁移至 TKE 原生节点专用调度器。
