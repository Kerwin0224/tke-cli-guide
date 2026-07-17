---
doc_type: How-to
subtype: 6A
fused: true
---
> 官方文档：[节点概述](https://cloud.tencent.com/document/product/457/32201) · [注册节点公网版](https://cloud.tencent.com/document/product/457/57916) · [注册节点常见问题](https://cloud.tencent.com/document/product/457/79751) · [注册节点价格和配额说明](https://cloud.tencent.com/document/product/457/79747) · [容器服务安全组设置](https://cloud.tencent.com/document/product/457/9084)
>
> 配额：公网带宽配额、安全组规则数量限制。[配额说明](https://cloud.tencent.com/document/product/457/9087)
>
> ⚠️ **高危操作**：安全组不严密致公网节点被非授权访问；DNS 配置不当致 apiserver 连接失败；脚本过期致节点失联。[常见高危操作](https://cloud.tencent.com/document/product/457/39539)

# 创建注册节点（公网版）

> 通过公网将自备机器接入已有标准集群，部署简单但存在安全风险。
> 控制台: [容器服务 - 节点池 - 注册节点](https://console.cloud.tencent.com/tke2/nodepool)

公网版与专线版共用同一套 tccli 接入动作（`EnableExternalNodeSupport` → `CreateExternalNodePool` → `DescribeExternalNodeScript` → 执行脚本）。区别在接入网络：公网版目标机器跨公网连接集群，专线版走专线 / 内网 / VPN。

> 公网接入有安全风险，生产环境优先使用[专线版](dedicated-line.md)（内网 / VPN）。公网版当前不支持 CLB 流量接入与 Prometheus 监控接入。

## 触发条件

- 集群内已有云上节点（注册节点特性要求系统组件运行在云上节点）。
- 目标机器可跨公网访问集群接入地址，操作系统为 TencentOS Server 3.1 / 2.4（TK4）。
- 你要在无专线 / VPN 的环境下将机器注册为节点。

## 准备工作

- 已创建 TKE 标准集群（见[创建集群](../../clusters/create.md)）。
- 已配置 tccli 凭证（见[配置凭证](../../../getting-started/credentials.md)）。
- 目标机器可访问公网，且操作系统符合要求。

## 与专线版的差异

| 维度 | 专线版 | 公网版 |
|:-----|:-----:|:-----:|
| 网络 | 专线 / 内网 / VPN | 公网 |
| CLB 流量接入 | 支持 | 不支持 |
| Prometheus 监控 | 支持 | 不支持 |
| 安全 | 生产推荐 | 谨慎使用 |

公网连接开关（`EnabledPublicConnect` / `PublicConnectUrl` / `PublicCustomDomain`）需在控制台注册节点配置中开启。`EnableExternalNodeSupport` 的 `ClusterExternalConfig` 不提供公网连接开关字段，因此 TCCLI 负责开启外部节点支持、建池、取脚本和回读状态，不能代替此控制台步骤。

> 公网版还有以下专项边界：仅适用于 Kubernetes 1.20 及以上、Global Router 容器网络的集群；云上与云下 Pod 默认隔离；目标节点需能访问公网接入所用 CLB 的 TCP 443、9000 端口及注册脚本依赖的镜像站点。公网版不支持 LoadBalancer Service、CLB/Nginx Ingress、Prometheus、云产品监控和 CLS 日志对接。

## 操作步骤

公网版的 tccli 步骤与专线版相同：

1. **控制台开启公网连接**：进入目标集群的注册节点配置，开启公网连接。
2. 查询支持现状：`DescribeExternalNodeSupportConfig`，确认 `EnabledPublicConnect=true` 且 `PublicConnectUrl` 非空；外部节点支持状态以 `Status` 为准（`Disabled`/`Initializing`/`Enabled`/`InitFailed`）。
3. 若 `Status=Disabled`，用 `EnableExternalNodeSupport` 开启外部节点支持（`NetworkType` 仅 `HostNetwork`/`CiliumBGP`）；该 Action 不开启公网连接。
4. 创建节点池：`CreateExternalNodePool`。
5. 获取脚本：`DescribeExternalNodeScript`（响应 `Command`/`Link`/`Token`）。
6. 在目标机器执行 `Command`，节点上线。

完整命令、字段表、占位符与验证见[创建注册节点（专线版）](dedicated-line.md)。本篇不重复命令，仅说明公网相关的差异与约束。

## 验证

| 维度 | 命令 | 期望 |
|:-----|:-----|:-----|
| 支持已开启（含公网） | `DescribeExternalNodeSupportConfig` | `Status=Enabled`，`EnabledPublicConnect=true` |
| 节点已注册 | `DescribeExternalNode --ClusterId <CLUSTER_ID> --NodePoolId <NODEPOOL_ID>` | 返回节点对象 |

## 故障恢复

| 现象 | 根因 | 修复 |
|:-----|:-----|:-----|
| 脚本执行后节点未上线 | 目标机器公网不可达集群接入地址 | 检查 `PublicConnectUrl` 可达性与安全组出方向 |
| 节点上线后无法被监控 | 公网版不支持 Prometheus | 改用专线版接入 |

## 收尾确认

```bash
tccli tke DescribeExternalNodeSupportConfig --ClusterId "<CLUSTER_ID>" --region ap-guangzhou \
  --filter "{status:Status,public:EnabledPublicConnect}"
# expected: Status=Enabled, EnabledPublicConnect=true
```

## 下一步

- 专线版接入：[创建注册节点（专线版）](dedicated-line.md)
- 对外暴露服务：[流量接入](traffic-access.md)
- 常见问题：[注册节点常见问题](faq.md)
