# 组件与应用概述（tccli）

> 对照官方：[组件与应用概述](https://cloud.tencent.com/document/product/457/39048) · page_id `39048`

## 概述

TKE 组件与应用管理提供扩展组件和应用市场两大能力。组件是集群级扩展功能，应用是基于 Helm Chart 的业务应用部署。

- **组件管理**：通过 `tccli tke` Addon 系列接口管理扩展组件的安装、升级、卸载
- **应用管理**：基于 Helm 的应用市场，支持一键部署常用中间件和框架、Helm Chart 版本管理、应用升级和回滚

## 前置条件

- [环境准备](../../../../环境准备.md)
- 熟悉 Kubernetes 基本概念和 TKE 集群架构
- 已创建集群且状态 `Running`

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
| 查看指定组件详情 | `tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName <AddonName>` | 是 |
| 安装组件 | `tccli tke InstallAddon --region <Region> --ClusterId <ClusterId> --AddonName <AddonName>` | 否 |
| 升级组件 | `tccli tke UpdateAddon --region <Region> --ClusterId <ClusterId> --AddonName <AddonName> --AddonVersion <AddonVersion>` | 否 |
| 卸载组件 | `tccli tke DeleteAddon --region <Region> --ClusterId <ClusterId> --AddonName <AddonName>` | 是 |
| 查看应用列表 | `helm list -A` | 是 |

## 操作步骤

本页面为概念性说明，无操作步骤。具体操作指引请参考：

- 组件管理（安装、升级、卸载）：参见 [组件的生命周期管理](../组件的生命周期管理/tccli 操作.md)
- 组件高可用配置：参见 [组件高可用](../组件高可用/tccli 操作.md)
- 各具体组件说明：参见 [扩展组件概述](../扩展组件概述/tccli 操作.md)
- 应用管理：参见 [应用管理概述](../../应用管理/概述/tccli 操作.md)

## 验证

不适用（概述页面）。

## 清理

不适用（概述页面）。

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `InstallAddon` 返回 `FailedOperation.AddonAlreadyInstalled` | `tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId>` 检查已安装组件 | 组件已安装 | 无需重复安装；如需重装请先 `DeleteAddon` |
| 组件安装后状态 `Failed` | `tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName <AddonName>` 查看 Status 与 reason | 组件版本与集群版本不兼容或资源不足 | 检查组件版本兼容性，扩容节点后重试 |
| 应用市场中安装的应用运行异常 | `helm status <release-name> -n <namespace>` 查看失败原因 | 应用本身配置或依赖问题 | 参考应用详情中的官方文档链接；TKE 不提供应用运行时调试支持 |
| `Unable to connect to the server` | `kubectl cluster-info` 检查连通性 | 公网端点被 CAM 策略 strategyId:240463971 以 `tke:clusterExtranetEndpoint=true` 条件拒绝 | 通过 IOA/VPN 连接内网，或使用同 VPC CVM 执行 kubectl 命令 |

## 下一步

- [扩展组件概述](../扩展组件概述/tccli 操作.md) — 了解所有可用增强组件
- [组件的生命周期管理](../组件的生命周期管理/tccli 操作.md) — 组件安装、升级与卸载
- [应用管理概述](../../应用管理/概述/tccli 操作.md) — 使用 Helm 应用

## 控制台替代

[容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) → 集群 → 组件管理 / 运维中心 → 应用市场。
