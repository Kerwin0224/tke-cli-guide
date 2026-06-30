# 组件高可用（tccli）

> 对照官方：[组件高可用](https://cloud.tencent.com/document/product/457/131167) · page_id `131167`

## 概述

组件高可用是指集群中组件（如 CBS、CoreDNS 等）的工作负载 Pod 分布在两个及以上可用区（AZ）的节点上，单 AZ 故障时仍能保证组件服务连续可用。TKE 组件管理支持对满足条件的组件一键开启或关闭高可用。开启高可用时，系统将组件工作负载调整为跨可用区部署；关闭后解除约束，部署由调度器决定。

## 前置条件

- [环境准备](../../../../环境准备.md)
- 已 [连接集群](../../../../集群配置/集群管理/连接集群/tccli 操作.md)，kubectl 可正常访问 APIServer
- 集群已创建且状态 `Running`

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

- 需要 CAM 权限：`tke:DescribeClusters`、`tke:DescribeAddon`、`tke:UpdateAddon`、`tke:ModifyClusterAttribute`

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 查看集群信息 | `tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'` | 是 |
| 查看组件列表 | `tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId>` | 是 |
| 查看集群高可用属性 | `tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'`（查看 `Property`） | 是 |
| 修改集群高可用属性 | `tccli tke ModifyClusterAttribute --region <Region> --ClusterId <ClusterId> --Property '{"ClusterHAConfig":{"EnableHA":true}}'` | 否 |
| 触发组件重新部署（高可用模式） | `tccli tke UpdateAddon --region <Region> --ClusterId <ClusterId> --AddonName <AddonName> --AddonVersion <AddonVersion>` | 否 |

## 操作步骤

### 步骤 1：确认集群高可用属性（控制面）

```bash
tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]' \
    | jq '.Clusters[0].Property'
# expected: 返回集群属性，含 ClusterHAConfig
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

查看 `Property` 中的 `ClusterHAConfig.EnableHA` 字段。

### 步骤 2：开启集群高可用属性（控制面）

```bash
tccli tke ModifyClusterAttribute --region <Region> \
  --ClusterId <ClusterId> \
  --Property '{"ClusterHAConfig":{"EnableHA":true}}'
# expected: exit 0
```

### 步骤 3：确认多 AZ 节点（数据面）

集群需存在两个及以上可用区的 Ready 节点：

```bash
kubectl get nodes -L failure-domain.beta.kubernetes.io/zone -o custom-columns=NAME:.metadata.name,STATUS:.status.conditions[?(@.type=='Ready')].status,ZONE:.metadata.labels."failure-domain\.beta\.kubernetes\.io/zone"
# expected: 节点分布在 2 个及以上可用区，STATUS 均为 True
```

```text
NAME  STATUS  AGE
...
```

> **注意**：kubectl 数据面当前因公网端点被 CAM 策略 strategyId:240463971 拒绝而不可达。以下 kubectl 命令为参考格式，需在内网/VPN 环境执行。

### 步骤 4：开启组件高可用（控制面）

**前置条件：** 集群已开启高可用属性，且集群节点分布在 2 个及以上可用区。

开启高可用通过更新组件触发，组件会自动以高可用模式重新部署：

```bash
tccli tke UpdateAddon --region <Region> \
  --ClusterId <ClusterId> \
  --AddonName <AddonName> \
  --AddonVersion <CurrentVersion>
# expected: exit 0，组件以高可用模式重新部署
```

> **说明：** 开启高可用会修改组件工作负载的 `TopologySpreadConstraints`，将 Pod 分散调度到不同可用区。

支持高可用的组件及版本要求：

| 组件名称 | 版本要求 |
|---------|---------|
| cbs | 1.1.14 及以上 |
| eniipamd | 3.10.1 及以上 |
| coredns | 无版本限制 |
| kubernetes-proxy | 无版本限制 |

### 步骤 5：关闭组件高可用（控制面）

**前置条件：** 集群已关闭高可用属性。

```bash
tccli tke ModifyClusterAttribute --region <Region> \
  --ClusterId <ClusterId> \
  --Property '{"ClusterHAConfig":{"EnableHA":false}}'
# expected: exit 0
```

然后再次更新组件：

```bash
tccli tke UpdateAddon --region <Region> \
  --ClusterId <ClusterId> \
  --AddonName <AddonName> \
  --AddonVersion <CurrentVersion>
# expected: exit 0
```

> **注意：** 关闭组件高可用后，组件更新不再强制跨可用区部署，最终部署状态取决于 Kubernetes 调度器。

## 验证

### 控制面（tccli）

```bash
tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]' \
    | jq '.Clusters[0].Property.ClusterHAConfig'
# expected: EnableHA true/false 与预期一致

tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName <AddonName> \
    | jq '.Addons[0] | {AddonName, Status, AddonVersion}'
# expected: Status "Running"
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

### 数据面（kubectl）

```bash
kubectl get deploy,sts -n kube-system -o wide
# expected: 组件 Pod 分布在 2 个及以上可用区
```

```text
NAME  STATUS  AGE
...
```

> **注意**：kubectl 数据面当前因公网端点被 CAM 策略 strategyId:240463971 拒绝而不可达。以下 kubectl 命令为参考格式，需在内网/VPN 环境执行。

高可用状态判断：

| 状态 | 说明 |
|------|------|
| 高可用 | 组件所有 Deployment 和 StatefulSet 的 Running Pod 分布 ≥2 个可用区 |
| 非高可用 | 存在至少一个 Deployment 或 StatefulSet 的 Running Pod 均在同一可用区 |

## 清理

无需清理（属配置修改操作）。如需恢复非高可用状态，参考操作步骤第 5 步。

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 开启高可用后组件更新完成但状态仍为非高可用 | `kubectl get deploy,sts -n kube-system -o wide` 查看 Pod 可用区分布 | Kubernetes 版本过低，调度器不支持组件高可用调度特性 | 升级集群版本：v1.18 需 ≥1.18.4-tke.60，v1.20 需 ≥1.20.6-tke.64，v1.22 需 ≥1.22.5-tke.46，v1.24 需 ≥1.24.4-tke.36，v1.26 需 ≥1.26.1-tke.25，v1.28+ 均支持 |
| 开启高可用后组件更新失败 | `tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName <AddonName>` 查看 Status 与 reason | 组件版本或集群状态问题 | 参见 [组件升级常见错误和处理](https://cloud.tencent.com/document/product/457/130406) |
| 版本不满足要求的组件 | `tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName <AddonName>` 查看当前版本 | 组件版本低于高可用最低版本要求 | 先将组件升级到支持高可用的版本，再执行开启高可用操作 |
| 集群未开启高可用时关闭组件高可用 | `tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'` 查看 `Property.ClusterHAConfig.EnableHA` | 集群本身未开启高可用属性 | 先确认 `EnableHA` 为 `false` |

## 下一步

- [组件的生命周期管理](../组件的生命周期管理/tccli 操作.md)
- [CoreDNS 说明](../CoreDNS 说明/tccli 操作.md)

## 控制台替代

控制台 **运维中心 → 组件管理** 中，组件操作列单击 **开启高可用** / **关闭高可用**。
