# tke-karpenter 说明（tccli）

> 对照官方：https://cloud.tencent.com/document/product/457/111805

## 概述

tke-karpenter 是基于 Karpenter 的节点弹性伸缩组件，相比 CA（Cluster Autoscaler）具有更快扩容速度和更精细资源匹配能力。组件支持按实例类型、可用区、容量类型（竞价/按量）等策略维度配置节点弹性策略，适用于对扩容延迟和资源利用率有更高要求的场景。

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
- 集群版本满足 tke-karpenter 组件兼容性要求。
- 当前 CAM 子账号具备 TKE 组件读写权限，包括 `tke:DescribeAddon`、`tke:InstallAddon`、`tke:UpdateAddon`、`tke:DeleteAddon`。

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 关键参数 |
| --- | --- | --- |
| 查看集群列表 | `tccli tke DescribeClusters` | `--region <Region>` `--ClusterIds '[\"<ClusterId>\"]'` |
| 查看组件信息 | `tccli tke DescribeAddon` | `--AddonName tke-karpenter` |
| 安装组件 | `tccli tke InstallAddon` | `--AddonName tke-karpenter` `--ClusterId <ClusterId>` `--AddonVersion <AddonVersion>` |
| 更新组件 | `tccli tke UpdateAddon` | `--AddonName tke-karpenter` `--ClusterId <ClusterId>` `--AddonVersion <AddonVersion>` |
| 卸载组件 | `tccli tke DeleteAddon` | `--AddonName tke-karpenter` `--ClusterId <ClusterId>` |

## 操作步骤

### 1. 查看组件信息

```bash
tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName tke-karpenter
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
tccli tke InstallAddon --region <Region> --ClusterId <ClusterId> --AddonName tke-karpenter --AddonVersion <AddonVersion>
```

安装为异步过程，返回 RequestId 后需通过 `DescribeAddon` 轮询组件状态。

### 3. 查看运行状态（数据面）

> **注意**：kubectl 数据面当前因公网端点被 CAM 策略 strategyId:240463971 拒绝而不可达。以下 kubectl 命令为参考格式，需在内网/VPN 环境执行。

```bash
kubectl get pods -n kube-system -l app.kubernetes.io/name=karpenter
```

```text
NAME  STATUS  AGE
...
```

### 4. 卸载组件

```bash
tccli tke DeleteAddon --region <Region> --ClusterId <ClusterId> --AddonName tke-karpenter
```

## 验证

```bash
tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName tke-karpenter
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
kubectl get pods -n kube-system -l app.kubernetes.io/name=karpenter
# expected: karpenter Pod Running
```

```text
NAME  STATUS  AGE
...
```

## 清理

```bash
tccli tke DeleteAddon --region <Region> --ClusterId <ClusterId> --AddonName tke-karpenter
```

卸载后再次查询应返回空结果：

```bash
tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName tke-karpenter
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
| InstallAddon 返回 InvalidParameter.AddonName | 组件名不匹配 | 组件名不存在 | 通过 `DescribeAddon` 查看支持组件列表，确认正确的 AddonName |
| 组件安装后 NotReady | `kubectl describe pod` 排查 | 资源不足或版本不兼容 | 调整集群资源或升级集群版本后重试 |
| Unable to connect to the cluster / kubectl 连接失败 | 检查公网端点访问策略 | 公网端点被 CAM 策略 strategyId:240463971（`tke:clusterExtranetEndpoint=true`）拒绝 | 切换至内网/VPN 环境执行 kubectl，或调整 CAM 策略放通公网端点 |

## 下一步

- [组件的生命周期管理](../组件的生命周期管理/tccli 操作.md)
- [扩展组件概述](../扩展组件概述/tccli 操作.md)
- [Cluster Autoscaler 说明](../Cluster Autoscaler 说明/tccli 操作.md)

## 控制台替代

对应控制台路径：登录容器服务 TKE 控制台 → 集群管理 → 选择目标集群 → 组件管理 → tke-karpenter，依次执行查询、安装、更新、卸载操作。以上 tccli 命令与控制台操作一一对应，可在自动化场景中替代控制台交互。
