# 组件与应用概述（tccli）

> 对照官方：[组件与应用概述](https://cloud.tencent.com/document/product/457/81234) · page_id `81234`

## 概述

TKE 为集群提供了多种组件以丰富集群功能、提升性能与稳定性。组件分为三种类型：

| 组件类型 | 说明 | 相关文档 |
|---------|------|---------|
| 系统组件 | 集群默认安装的核心组件（如 CBS-CSI、IPAMD、monitoragent 等），异常可能导致集群故障 | — |
| 增强组件 | TKE 提供的扩展组件包，用户可按需选择部署 | [扩展组件概述](https://cloud.tencent.com/document/product/457/39048) |
| 应用市场 | 集成 Helm 3.0，提供 Helm Chart、容器镜像及软件服务 | [应用市场](./应用管理/应用市场/tccli 操作.md) |

对于标记为"预览版"的功能，TKE 不提供服务等级协议（SLA）保障。

## 前置条件

- 了解 TKE 集群的组件管理模型：系统组件由 TKE 负责维护与自动更新；增强组件由用户自行安装、升级和卸载
- 应用市场中的应用仅保证在 TKE 支持的集群类型和 Kubernetes 版本内正常安装部署，不提供运行时调试或 SLA 保障
- [环境准备](../../环境准备.md)：`tccli` 已配置

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

- 需要 CAM 权限：`tke:DescribeClusters`、`tke:DescribeAddon`、`tke:InstallAddon`、`tke:DeleteAddon`

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 查看集群信息 | `tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'` | 是 |
| 查看组件列表 | `tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId>` | 是 |
| 安装组件 | `tccli tke InstallAddon --region <Region> --ClusterId <ClusterId> --AddonName <AddonName>` | 否 |
| 升级组件 | `tccli tke UpdateAddon --region <Region> --ClusterId <ClusterId> --AddonName <AddonName> --AddonVersion <AddonVersion>` | 否 |
| 卸载组件 | `tccli tke DeleteAddon --region <Region> --ClusterId <ClusterId> --AddonName <AddonName>` | 是 |
| 查看应用列表 | `helm list -A` | 是 |

## 操作步骤

本页面为概念性说明，无操作步骤。具体操作指引请参考：

- 增强组件的安装、升级、卸载：参见 [组件的生命周期管理](./组件管理/组件的生命周期管理/tccli 操作.md)
- 组件高可用配置：参见 [组件高可用](./组件管理/组件高可用/tccli 操作.md)
- 各具体组件说明：参见 [扩展组件概述](./组件管理/扩展组件概述/tccli 操作.md)
- 应用市场与 Helm 应用：参见 [应用管理概述](./应用管理/概述/tccli 操作.md)

## 验证

不适用（概述页面）。

## 清理

不适用（概述页面）。

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 组件安装后异常 | `tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName <AddonName>` 查看 Status 与 reason | 组件版本与集群 Kubernetes 版本不兼容 | 检查组件版本兼容性，参见 [组件版本维护说明](https://cloud.tencent.com/document/product/457/71800) |
| 应用市场中安装的应用运行异常 | `helm status <release-name> -n <namespace>` 查看失败原因 | 应用本身配置或依赖问题 | 参考应用详情中的官方文档链接；TKE 不提供应用运行时调试支持 |
| `Unable to connect to the server` | `kubectl cluster-info` 检查连通性 | 公网端点被 CAM 策略 strategyId:240463971 以 `tke:clusterExtranetEndpoint=true` 条件拒绝 | 通过 IOA/VPN 连接内网，或使用同 VPC CVM 执行 kubectl 命令 |

## 下一步

- [扩展组件概述](./组件管理/扩展组件概述/tccli 操作.md) — 了解所有可用增强组件
- [组件的生命周期管理](./组件管理/组件的生命周期管理/tccli 操作.md) — 安装、升级、卸载组件
- [组件高可用](./组件管理/组件高可用/tccli 操作.md) — 配置组件跨可用区高可用
- [应用管理概述](./应用管理/概述/tccli 操作.md) — 使用 Helm 应用
- [组件版本维护说明](https://cloud.tencent.com/document/product/457/71800) — 了解组件版本策略

## 控制台替代

访问 [TKE 控制台 - 组件管理](https://console.cloud.tencent.com/tke2/cluster) 查看和管理集群组件。
