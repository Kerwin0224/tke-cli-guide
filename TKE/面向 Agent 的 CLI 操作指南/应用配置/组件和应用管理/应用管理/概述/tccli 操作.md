# 应用管理概述（tccli）

> 对照官方：[应用管理概述](https://cloud.tencent.com/document/product/457/32729) · page_id `32729`

## 概述

TKE 应用管理提供 Helm Chart 托管、应用市场和应用生命周期管理能力。可通过控制台应用市场一键部署常用中间件，或通过本地 Helm 客户端连接集群进行 Chart 管理。

## 前置条件

- [环境准备](../../../../环境准备.md)：`tccli` 已配置，`kubectl` ≥ v1.28
- 已 [连接集群](../../../../集群配置/集群管理/连接集群/tccli 操作.md)，kubectl 可正常访问 APIServer
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

- 需要 CAM 权限：`tke:DescribeClusters`、`tke:DescribeClusterKubeconfig`

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 查看集群信息 | `tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'` | 是 |
| 获取集群 Kubeconfig | `tccli tke DescribeClusterKubeconfig --region <Region> --ClusterId <ClusterId>` | 是 |
| 查看应用列表 | `helm list -A` | 是 |
| 创建应用 | `helm install <release> <chart> -n <namespace>` | 否（重复创建报错） |
| 更新应用 | `helm upgrade <release> <chart> -n <namespace>` | 是 |
| 删除应用 | `helm uninstall <release> -n <namespace>` | 是 |

## 操作步骤

本页面为概念性说明，无操作步骤。具体操作指引请参考：

- 创建和管理应用：参见 [使用应用](../使用应用/tccli 操作.md)
- TKE 应用市场：参见 [应用市场](../应用市场/tccli 操作.md)
- 本地 Helm 客户端连接集群：参见 [本地 Helm 客户端连接集群](../本地 Helm 客户端连接集群/tccli 操作.md)

## 验证

不适用（概述页面）。

## 清理

不适用（概述页面）。

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `helm list` 报 `Kubernetes cluster unreachable` | `kubectl cluster-info` 检查连通性 | kubeconfig 未配置或公网端点不可达 | 使用 `tccli tke DescribeClusterKubeconfig` 获取 kubeconfig；公网端点被 CAM 策略 strategyId:240463971 以 `tke:clusterExtranetEndpoint=true` 条件拒绝时，通过 IOA/VPN 连接内网 |
| `Unable to connect to the server` | `kubectl cluster-info` 检查连通性 | 公网端点被 CAM 策略 strategyId:240463971 拒绝 | 通过 IOA/VPN 连接内网，或使用同 VPC CVM 执行 kubectl 命令 |

## 下一步

- [使用应用](../使用应用/tccli 操作.md) — 创建和管理应用
- [应用市场](../应用市场/tccli 操作.md) — TKE 应用市场
- [本地 Helm 客户端连接集群](../本地 Helm 客户端连接集群/tccli 操作.md) — helm CLI 操作

## 控制台替代

控制台 **运维中心 → 应用市场** 浏览和管理应用。详情参见 [应用管理概述（控制台）](https://cloud.tencent.com/document/product/457/32729)。
