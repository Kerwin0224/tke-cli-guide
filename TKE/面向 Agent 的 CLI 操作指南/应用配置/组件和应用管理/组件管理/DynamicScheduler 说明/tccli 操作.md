# DynamicScheduler 说明（tccli）

> 对照官方：https://cloud.tencent.com/document/product/457/50843

## 概述

Dynamic Scheduler 是 TKE 基于 Kubernetes Kube-scheduler Extender 机制开发的插件，基于**节点真实负载**进行调度，与默认 kube-scheduler 协作，避免仅基于资源 request/limit 值调度导致的负载不均。

> **废弃声明**：DynamicScheduler 和 DeScheduler 扩展组件已于 **2023 年 11 月 30 日** 下线。该日期后新安装不再支持，存量组件无法更新。请迁移至 **TKE 原生节点专用调度器**，该调度器覆盖了两个组件的功能，无需 Prometheus 依赖并支持节点虚拟超卖（超过 100% 装箱率）。

### 架构组成

- **node-annotator**（Deployment × 1）：定期从 Prometheus 拉取节点负载指标并写入 Node annotations。
  - 删除组件不会自动清除其写入的 annotations，需手动清理。
- **dynamic-scheduler**（Deployment × 3）：作为 scheduler-extender，读取 Node annotations 中的负载数据进行调度干预。
  - Predicate（预选策略）：过滤高负载节点，超过阈值的节点被排除。
  - Priority（优选策略）：按节点负载反向评分，负载越低评分越高，权重可配置。

### 防热点策略

- 节点过去 1 分钟内调度了 **> 2 个 Pod** → 优先级评分 **减 1**。
- 节点过去 5 分钟内调度了 **> 5 个 Pod** → 优先级评分 **减 1**。

### 卸载说明

卸载仅移除动态调度逻辑，原生 kube-scheduler 继续正常工作。

## 前置条件

环境准备：[环境准备](../../../../环境准备.md)；连接集群：[连接集群](../../../../集群配置/集群管理/连接集群/tccli 操作.md)。

集群状态检查：

```bash
# expected: 返回集群列表，确认目标集群 Status 为 Running
tccli tke DescribeClusters --region <Region>
```

### 集群与组件要求

- Kubernetes 版本 **≥ v1.10.x**。
- 已部署 **Prometheus 监控**（自建或托管）及录制规则。
- Prometheus 需提供以下指标：`cpu_usage_avg_5m`、`cpu_usage_max_avg_1h`、`cpu_usage_max_avg_1d`、`mem_usage_avg_5m`、`mem_usage_max_avg_1h`、`mem_usage_max_avg_1d`。

### Kubernetes 主版本升级注意

- **托管集群**：无需重新配置此插件。
- **独立集群**：Master 升级会重置所有 master 组件配置，破坏 Scheduler Extender 配置。升级后**必须先卸载再重新安装**此插件。

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
| node-annotator | Deployment | CPU: 100m, 内存: 100Mi × 1 实例 | kube-system |
| dynamic-scheduler | Deployment | CPU: 400m, 内存: 200Mi × 3 实例 | kube-system |
| dynamic-scheduler | Service | — | kube-system |
| node-annotator | ClusterRole | — | kube-system |
| node-annotator | ClusterRoleBinding | — | kube-system |
| node-annotator | ServiceAccount | — | kube-system |
| dynamic-scheduler-policy | ConfigMap | — | kube-system |
| restart-kube-scheduler | ConfigMap | — | kube-system |
| probe-prometheus | ConfigMap | — | kube-system |

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 说明 |
| --- | --- | --- |
| 查看集群列表 | `tccli tke DescribeClusters --region <Region>` | 获取 <ClusterId> |
| 查看组件详情 | `tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName DynamicScheduler` | 查询组件状态与参数 |
| 安装组件 | `tccli tke InstallAddon --region <Region> --ClusterId <ClusterId> --AddonName DynamicScheduler --AddonVersion <AddonVersion>` | 传入 Predicate/Priority 参数 |
| 更新组件 | `tccli tke UpdateAddon --region <Region> --ClusterId <ClusterId> --AddonName DynamicScheduler --Param <JSON>` | 修改组件配置（存量已废弃，无法更新） |
| 删除组件 | `tccli tke DeleteAddon --region <Region> --ClusterId <ClusterId> --AddonName DynamicScheduler` | 卸载组件 |

| Console 参数 | tccli 参数 | 幂等性 |
|-------------|-----------|--------|
| 组件选择（勾选 DynamicScheduler） | `AddonName: "DynamicScheduler"` | 是 |
| 组件版本 | `AddonVersion` | 否 |
| Prometheus 数据查询地址 | `RawValues` | 否 |
| Predicate 参数（预选阈值） | `RawValues` | 否 |
| Priority 参数（优选权重） | `RawValues` | 否 |

### 配置参数说明

**Predicate（预选）参数**：

| 参数 | 说明 |
|------|------|
| 5 分钟平均 **CPU** 利用率阈值 | 超过此阈值的节点在预选阶段被过滤 |
| 1 小时最大 **CPU** 利用率阈值 | 超过此阈值的节点在预选阶段被过滤 |
| 5 分钟平均 **内存** 利用率阈值 | 超过此阈值的节点在预选阶段被过滤 |
| 1 小时最大 **内存** 利用率阈值 | 超过此阈值的节点在预选阶段被过滤 |

**Priority（优选）参数**：

| 参数 | 说明 |
|------|------|
| 5 分钟平均 CPU 利用率权重 | 权重越大，5 分钟平均 CPU 对评分影响越大 |
| 1 小时最大 CPU 利用率权重 | 权重越大，1 小时最大 CPU 对评分影响越大 |
| 1 天最大 CPU 利用率权重 | 权重越大，1 天最大 CPU 对评分影响越大 |
| 5 分钟平均内存利用率权重 | 权重越大，5 分钟平均内存对评分影响越大 |
| 1 小时最大内存利用率权重 | 权重越大，1 小时最大内存对评分影响越大 |
| 1 天最大内存利用率权重 | 权重越大，1 天最大内存对评分影响越大 |

## 操作步骤

### Step 1: 部署 Prometheus 录制规则

**自建 Prometheus**：配置 PrometheusRule 并确保规则文件挂载到容器。

**托管 Prometheus**：在 TKE 控制台的 Prometheus 监控页面创建实例、关联集群后配置规则。

核心录制规则 YAML：

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: example-record
spec:
  groups:
    - name: cpu_mem_usage_active
      interval: 30s
      rules:
        - record: cpu_usage_active
          expr: 100 - (avg by (instance) (irate(node_cpu_seconds_total{mode="idle"}[30s])) * 100)
        - record: mem_usage_active
          expr: 100*(1-node_memory_MemAvailable_bytes/node_memory_MemTotal_bytes)
    - name: cpu-usage-5m
      interval: 5m
      rules:
        - record: cpu_usage_max_avg_1h
          expr: max_over_time(cpu_usage_avg_5m[1h])
        - record: cpu_usage_max_avg_1d
          expr: max_over_time(cpu_usage_avg_5m[1d])
    - name: cpu-usage-1m
      interval: 1m
      rules:
        - record: cpu_usage_avg_5m
          expr: avg_over_time(cpu_usage_active[5m])
    - name: mem-usage-5m
      interval: 5m
      rules:
        - record: mem_usage_max_avg_1h
          expr: max_over_time(mem_usage_avg_5m[1h])
        - record: mem_usage_max_avg_1d
          expr: max_over_time(mem_usage_avg_5m[1d])
    - name: mem-usage-1m
      interval: 1m
      rules:
        - record: mem_usage_avg_5m
          expr: avg_over_time(mem_usage_active[5m])
```

自建 Prometheus 服务器配置：

```yaml
global:
  evaluation_interval: 30s
  scrape_interval: 30s
  external_labels:
rule_files:
- /etc/prometheus/rules/*.yml
```

> **注意**：若同时使用 DeScheduler 和 DynamicScheduler，需使用合并版规则。规则文件互相独立，避免直接覆盖。合并版规则参见 DeScheduler 说明页。

### Step 2: 安装 DynamicScheduler 组件

```bash
# expected: 返回 RequestId，组件开始安装
tccli tke InstallAddon \
    --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName DynamicScheduler \
    --AddonVersion <AddonVersion>
```

安装后组件自动运行，无需额外配置。建议开启**事件持久化**以便异常监控和故障诊断。

## 验证

```bash
# expected: Phase 为 Running，组件已安装
tccli tke DescribeAddon \
    --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName DynamicScheduler
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
# expected: node-annotator 1/1 与 dynamic-scheduler 3/3 均 Ready
kubectl get deploy -n kube-system node-annotator dynamic-scheduler
```

```text
NAME  STATUS  AGE
...
```

```bash
# expected: Node annotations 中出现 cpu_usage_avg_5m / mem_usage_avg_5m 等负载字段
kubectl get node -o yaml | grep usage
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
    --AddonName DynamicScheduler
```

> **注意**：kubectl 数据面当前因公网端点被 CAM 策略 strategyId:240463971 拒绝而不可达。以下 kubectl 命令为参考格式，需在内网/VPN 环境执行。

```bash
# expected: Error from server (NotFound): deployments.apps "node-annotator" not found
# expected: Error from server (NotFound): deployments.apps "dynamic-scheduler" not found
kubectl get deploy -n kube-system node-annotator dynamic-scheduler
```

```text
NAME  STATUS  AGE
...
```

> **注意**：卸载组件不会自动清除 node-annotator 写入的 Node annotations。如需清理，请手动删除。卸载仅移除动态调度逻辑，不影响原生 kube-scheduler。

> **计费提醒**：DynamicScheduler 组件本身不产生额外费用。如使用了托管 Prometheus 服务，请注意相关计费。

## 排障

| 现象 | 诊断 | 根因 | 修复 |
| --- | --- | --- | --- |
| 组件不生效 | 检查 Prometheus 录制规则与指标 | PrometheusRule 未配置或所需 6 个指标缺失 | 检查 PrometheusRule 是否正确创建；确认 `cpu_usage_avg_5m`、`cpu_usage_max_avg_1h`、`cpu_usage_max_avg_1d`、`mem_usage_avg_5m`、`mem_usage_max_avg_1h`、`mem_usage_max_avg_1d` 均可查询 |
| 独立集群升级后调度异常 | 确认 Master 是否升级 | Master 升级重置了 Extender 配置 | 卸载组件后重新安装：`tccli tke DeleteAddon` 后 `tccli tke InstallAddon` |
| 节点 annotations 无负载数据 | 检查 node-annotator 日志 | node-annotator 无法连接 Prometheus | 检查 Prometheus 查询 URL 是否正确配置；确认 Prometheus 与集群在同一 VPC |
| 安装/更新失败 | 确认组件是否已废弃 | DynamicScheduler 已于 2023-11-30 下线，新安装和更新均已停止 | 迁移至 TKE 原生节点专用调度器 |
| Unable to connect to the server | 检查公网端点与 CAM 策略 | 公网端点被 CAM 策略 strategyId:240463971（`tke:clusterExtranetEndpoint=true`）拒绝 | 通过内网/VPN 访问，或调整 CAM 策略 |

## 下一步

- [TKE 原生节点专用调度器](https://cloud.tencent.com/document/product/457)
- [DeScheduler 说明（tccli）](../DeScheduler%20说明/tccli%20操作.md)
- [Kubernetes Scheduler Extender](https://github.com/kubernetes/kubernetes/tree/master/pkg/scheduler)
- [Prometheus 监控服务](https://cloud.tencent.com/document/product/1416)
- [TKE 组件管理概述](https://cloud.tencent.com/document/product/457/51082)

## 控制台替代

如需通过控制台操作：登录 [容器服务控制台 - 组件管理](https://console.cloud.tencent.com/tke2/cluster?rid=<Region>)，进入集群 → 组件管理 → 找到 DynamicScheduler → 安装/更新/卸载。组件已废弃，建议迁移至 TKE 原生节点专用调度器。
