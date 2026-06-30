# Cluster Autoscaler 说明（tccli）

> 对照官方：https://cloud.tencent.com/document/product/457/84661

## 概述

Cluster Autoscaler（CA）是 Kubernetes 集群自动扩缩容组件，根据 Pod 调度状态自动增减节点数量实现弹性伸缩。

工作原理：

1. 监控 Pending 状态 Pod（因资源不足无法调度）。
2. 计算需新增节点数量和规格。
3. 调用云 API 增加节点（扩容）。
4. 监控空闲节点，利用率低时缩容。

配置说明：

- 安装后自动生效。
- 核心配置项：最大/最小节点数范围、扩容冷却时间（默认 10 分钟）、缩容冷却时间（默认 10 分钟）、节点利用率阈值。

## 前置条件

环境准备：参见 [环境准备](../../../../环境准备.md)；连接集群：参见 [连接集群](../../../../集群配置/集群管理/连接集群/tccli 操作.md)。

先确认目标集群状态正常：

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

- 已创建 TKE 集群且集群处于 Running 状态。
- 集群已配置节点池，且节点池最大/最小节点数满足预期弹性范围。
- 集群版本满足 ClusterAutoscaler 组件兼容性要求。
- 当前 CAM 子账号具备 TKE 组件读写权限，包括 `tke:DescribeAddon`、`tke:InstallAddon`、`tke:UpdateAddon`、`tke:DeleteAddon`。

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 关键参数 |
| --- | --- | --- |
| 查看集群列表 | `tccli tke DescribeClusters` | `--region <Region>` `--ClusterIds '[\"<ClusterId>\"]'` |
| 查看组件信息 | `tccli tke DescribeAddon` | `--AddonName ClusterAutoscaler` |
| 安装组件 | `tccli tke InstallAddon` | `--AddonName ClusterAutoscaler` `--ClusterId <ClusterId>` `--AddonVersion <AddonVersion>` |
| 更新组件 | `tccli tke UpdateAddon` | `--AddonName ClusterAutoscaler` `--ClusterId <ClusterId>` `--AddonVersion <AddonVersion>` |
| 卸载组件 | `tccli tke DeleteAddon` | `--AddonName ClusterAutoscaler` `--ClusterId <ClusterId>` |

## 操作步骤

### 1. 查看组件信息

```bash
tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName ClusterAutoscaler
```

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
```

返回结果包含组件版本、状态、所需参数等。记录可用 `AddonVersion` 以备安装使用。

### 2. 安装组件

```bash
tccli tke InstallAddon --region <Region> --ClusterId <ClusterId> --AddonName ClusterAutoscaler --AddonVersion <AddonVersion>
```

安装为异步过程，返回 RequestId 后需通过 `DescribeAddon` 轮询组件状态。

### 3. 查看运行状态（数据面）

> **注意**：kubectl 数据面当前因公网端点被 CAM 策略 strategyId:240463971 拒绝而不可达。以下 kubectl 命令为参考格式，需在内网/VPN 环境执行。

```bash
kubectl get deploy -n kube-system cluster-autoscaler
```

```text
NAME  STATUS  AGE
...
```

### 4. 卸载组件

```bash
tccli tke DeleteAddon --region <Region> --ClusterId <ClusterId> --AddonName ClusterAutoscaler
```

## 验证

```bash
tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName ClusterAutoscaler
# expected: Status Running
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

数据面验证（需内网/VPN 环境）：

```bash
kubectl get deploy -n kube-system cluster-autoscaler
# expected: READY 1/1
```

```text
NAME  STATUS  AGE
...
```

## 清理

```bash
tccli tke DeleteAddon --region <Region> --ClusterId <ClusterId> --AddonName ClusterAutoscaler
```

卸载后再次查询应返回空结果：

```bash
tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName ClusterAutoscaler
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

## 排障

| 现象 | 诊断 | 根因 | 修复 |
| --- | --- | --- | --- |
| InstallAddon 返回 FailedOperation.AddonAlreadyInstalled | 已安装同版本组件 | 组件已存在 | 先通过 `DescribeAddon` 检查当前状态，确认无需重复安装 |
| 组件安装失败 | `DescribeAddon` 查看 reason 字段 | 集群版本不兼容或资源不足 | 升级集群版本或调整资源后重试 |
| 节点未自动扩容 | 检查节点池配置及 max 节点数限制 | 节点池配置错误或已达最大节点数 | 校正节点池配置，提升 max 节点数上限 |
| 节点未自动缩容 | 检查冷却时间与 Pod 驱逐状态 | 仍在冷却时间内或 Pod 无法驱逐（PDB 限制） | 等待冷却时间结束，检查 PDB 配置 |
| Unable to connect to the cluster / kubectl 连接失败 | 检查公网端点访问策略 | 公网端点被 CAM 策略 strategyId:240463971（`tke:clusterExtranetEndpoint=true`）拒绝 | 切换至内网/VPN 环境执行 kubectl，或调整 CAM 策略放通公网端点 |

## 下一步

- [组件的生命周期管理](../组件的生命周期管理/tccli 操作.md)
- [扩展组件概述](../扩展组件概述/tccli 操作.md)
- [tke-karpenter 说明](../tke-karpenter 说明/tccli 操作.md)

## 控制台替代

对应控制台路径：登录容器服务 TKE 控制台 → 集群管理 → 选择目标集群 → 组件管理 → Cluster Autoscaler，依次执行查询、安装、更新、卸载操作。以上 tccli 命令与控制台操作一一对应，可在自动化场景中替代控制台交互。
