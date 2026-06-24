# tke-event-collector 说明（tccli）

> 对照官方：https://cloud.tencent.com/document/product/457/91049

## 概述

tke-event-collector 通过监听集群事件并上报到 CLS（日志服务）实现集群事件持久化，组件以 Deployment 形式运行在 TKE 集群中。启用后可在控制台 **集群 ID > 日志 > 事件日志** 查看事件总览和异常事件聚合检索，支持全局检索与自定义查询。

## 前置条件

环境准备与集群连通性请参见 [环境准备](../../../../环境准备.md) 与 [连接集群](../../../../集群配置/集群管理/连接集群/tccli 操作.md)。

先确认集群状态正常：

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

确认返回的集群 `Status.Phase` 为 `Running` 后再继续。

- 已开通日志服务（CLS）并完成授权。
- CLS 限制：每个日志集最多可创建 500 个日志主题和指标主题。

CAM 权限：调用 TKE 组件接口需具备相应权限，建议为操作账号授予 `QcloudTKEFullAccess` 或包含以下操作的策略：

| 操作 | 说明 |
| --- | --- |
| tke:DescribeClusters | 查询集群 |
| tke:DescribeAddon | 查询组件 |
| tke:InstallAddon | 安装组件 |
| tke:UpdateAddon | 更新组件 |
| tke:DeleteAddon | 删除组件 |

## 控制台与 CLI 参数映射

| 操作 | 控制台路径 | tccli 命令 |
| --- | --- | --- |
| 查询集群 | 集群管理 | `tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'` |
| 查询组件 | 集群 > 组件管理 | `tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName tke-event-collector` |
| 安装组件 | 集群 > 组件管理 > 新建 | `tccli tke InstallAddon --region <Region> --ClusterId <ClusterId> --AddonName tke-event-collector --AddonVersion <AddonVersion>` |
| 更新组件 | 集群 > 组件管理 > 更新 | `tccli tke UpdateAddon --region <Region> --ClusterId <ClusterId> --AddonName tke-event-collector --AddonVersion <AddonVersion>` |
| 删除组件 | 集群 > 组件管理 > 删除 | `tccli tke DeleteAddon --region <Region> --ClusterId <ClusterId> --AddonName tke-event-collector` |

## 操作步骤

### 1. 安装 tke-event-collector 组件（控制面）

```bash
tccli tke InstallAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName tke-event-collector \
    --AddonVersion <AddonVersion>
```

### 2. 配置 CLS 日志集和日志主题

在控制台开启事件日志功能：

1. 登录 TKE 控制台，选择目标集群 ID 进入详情页。
2. 选择 **日志** > **事件日志**，点击右侧 **开启**。
3. 配置参数：

| 配置项 | 说明 |
| --- | --- |
| 日志所在地域 | CLS 支持跨地域投递日志，点击修改可选择目标地域 |
| 日志集 | 选择已有日志集或前往 CLS 控制台新建 |
| 日志主题 | 支持「自动创建」和「选择已有」两种模式 |

> 重要限制：每个日志集最多创建 500 个日志主题及指标主题。

4. 点击 **确定** 完成开启。

### 3. 检索和分析事件日志（控制台）

开启后可在控制台检索：

1. 前往 **日志**，点击事件日志旁的 **检索分析**。
2. 在事件日志页面点击 **全局检索**。
3. 配置显示字段和检索条件进行搜索分析。

### 4. 通过 kubectl 查看集群原始事件（数据面）

> **注意**：kubectl 数据面当前因公网端点被 CAM 策略 strategyId:240463971 拒绝而不可达。以下 kubectl 命令为参考格式，需在内网/VPN 环境执行。

```bash
kubectl get event --all-namespaces --sort-by='.lastTimestamp'
```

```text
NAME  STATUS  AGE
...
```

查看特定命名空间事件：

```bash
kubectl get event -n default --s

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
```ort-by='.lastTimestamp'
```

查看 Warning 级别事件：

```bash
kubectl get event --all-namespaces --field-selector type=Warning
```

## 验证

### 控制面（tccli）

```bash
tccli tke DescribeAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName tke-event-collector
```

预期输出：组件状态为 `Succeeded`，AddonName 为 `tke-event-collector`。

### 数据面（kubectl）

> **注意**：kubectl 数据面当前因公网端点被 CAM 策略 strategyId:240463971 拒绝而不可达。以下 kubectl 命令为参考格式，需在内网/VPN 环境执行。

验证事件采集组件正常运行：

```bash
kubectl get deploy -n kube-system | grep event
```

预期输出：tke-event-collector 相关的 Deployment `READY` = `1/1`。

验证事件可正常获取：

```bash
kubectl get event --all-namespaces --sort-by='.lastTimestamp'
```

预期输出：列出最近的集群事件信息。

在控制台确认事件已上报至 CLS：导航至 **集群详情 > 日志 > 事件日志**，确认有事件数据展示。

## 清理

### 控制面（tccli）

```bash
tccli tke DeleteAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName tke-event-collector
```

> 计费提醒：卸载组件不会自动删除 CLS 中的日志主题和已采集事件数据。请前往 CLS 控制台手动删除不再使用的日志主题以避免持续计费。

## 排障

| 现象 | 诊断 | 根因 | 修复 |
| --- | --- | --- | --- |
| 控制台事件日志无数据 | 检查组件状态与 CLS 主题 | CLS 日志主题配置错误或组件未成功启动 | 确认组件 Running，检查 CLS 日志主题是否存在且未被删除 |
| 开启事件日志报错日志主题达上限 | 查看 CLS 日志集主题数 | CLS 日志集下日志主题数超过 500 个限制 | 清理不再使用的日志主题，或创建新的日志集 |
| 事件日志数据延迟较大 | 检查集群事件量与 CLS 写入 QPS | 集群事件量大，CLS 写入 QPS 达到限制 | 调整 CLS 日志主题的分区数（split 分区） |
| kubectl get event 显示大量 Warning | 分析 Warning 事件来源 | 集群组件或工作负载存在异常 | 按 Warning 来源排查对应组件或 Pod 状态 |
| kubectl Unable to connect to server | 检查 CAM 策略与公网端点 | 公网端点被 CAM 策略 strategyId:240463971 拒绝（tke:clusterExtranetEndpoint=true） | 在内网/VPN 环境执行，或调整 CAM 策略放行公网端点 |

## 下一步

- tke-log-agent 说明 — 容器日志采集组件
- tke-monitor-agent 说明 — 监控数据采集组件
- 开启日志采集 — CLS 日志采集完整指南
- 组件的生命周期管理 — 组件安装、升级与卸载

## 控制台替代

如需通过控制台操作，可在 **容器服务控制台 > 集群 > 组件管理** 中找到 tke-event-collector 组件，执行安装、更新、删除；并在 **集群 > 日志 > 事件日志** 中配置 CLS 日志集与日志主题、检索分析事件。对应的 tccli 命令见上文 [控制台与 CLI 参数映射](#控制台与-cli-参数映射)。
