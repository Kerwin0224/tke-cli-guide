---
doc_type: How-to
subtype: 6A
fused: true
---
# 管理集群访问端点

> 开启/关闭集群的公网或内网访问端点。端点是 kubectl/API 访问 API Server 的入口。异步操作。

## 概述

集群创建后默认无外网端点（`DescribeClusterEndpointStatus` 返 `NotFound`）。需显式开启才能从公网或 VPC 内网访问 API Server。

| 端点 | 接口 | 作用 | 安全 |
|:-----|:-----|:-----|:-----|
| 公网端点 | `CreateClusterEndpoint --IsExtranet true` | 公网访问 API Server | 需 ACL 白名单 |
| 内网端点 | `CreateClusterEndpoint --IsExtranet false` | VPC 内网访问 | VPC 隔离，更安全 |

操作是**异步**的：`CreateClusterEndpoint` 返回即提交，端点就绪需轮询 `DescribeClusterEndpointStatus` 直到 `Running`。

## 准备工作

### 环境检查

```bash
tccli --version
# expected: tccli 版本号

tccli tke DescribeClusterStatus --region ap-guangzhou --ClusterIds '["<CLUSTER_ID>"]' \
  --filter "ClusterStatusSet[0].ClusterState"
# expected: "Running"
```

### 资源检查

```bash
# 确认当前无端点或端点状态
tccli tke DescribeClusterEndpointStatus --region ap-guangzhou --ClusterId "<CLUSTER_ID>"
# expected: Status="NotFound"（未开启）或 "Running"（已开启）

# 内网端点需子网
tccli vpc DescribeSubnets --region <REGION> --Filters '[{"Name":"vpc-id","Values":["<VPC_ID>"]}]' \
  --filter "SubnetSet[].{id:SubnetId,name:SubnetName,cidr:CidrBlock}" --output text
# expected: 子网列表（内网端点需指定 SubnetId）
```

## 关键字段

> 来源：`tccli tke CreateClusterEndpoint --generate-cli-skeleton`。

| 字段 | 类型 | 必填 | 约束 | 填错时的错误 |
|:------|------|:--------:|------------|---------------|
| ClusterId | string | 是 | `cls-xxxxxxxx` | `ResourceNotFound` |
| IsExtranet | boolean | 是 | `true`（公网）/ `false`（内网） | `InvalidParameterValue` |
| SubnetId | string | 内网必填 | VPC 内子网 ID | `ResourceNotFound.SubnetId` |
| SecurityGroup | string | 否 | 安全组 ID，控制访问源 | `ResourceNotFound.SecurityGroup` |
| Domain | string | 否 | 自定义域名 | `InvalidParameterValue` |
| ExistedLoadBalancerId | string | 否 | 复用已有 CLB | `ResourceNotFound` |

> 公网端点用 CLB 暴露，内网端点用子网内 CLB/VIP。`SecurityGroup` 限定可访问的源 IP（白名单）。

## 操作步骤

### 步骤 1：决策 — 公网 vs 内网

#### 为什么选内网端点

- **公网端点**: 本地开发、CI/CD 从公网访问。需 ACL 白名单限定源 IP，否则暴露 API Server 到公网
- **内网端点（推荐）**: 同 VPC 的 CVM/kubectl 访问。VPC 隔离，不暴露公网，低延迟
- **默认推荐**: 生产用内网端点；本地开发临时用公网端点 + 白名单
- **能切换吗?**: 能。`SwitchClusterEndpoint` 切换公网/内网，或先 `DeleteClusterEndpoint` 再建

### 步骤 2：开启公网端点 — 最小化

```bash
tccli tke CreateClusterEndpoint --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" --IsExtranet true
# expected: exit 0, 返回 RequestId
```

| 占位符 | 含义 | 约束 | 如何获取 |
|:------------|:-----|:-----|:---------|
| `<CLUSTER_ID>` | 集群 ID | `cls-xxxxxxxx` | `tccli tke DescribeClusters` → `Clusters[].ClusterId` |

### 步骤 3：开启内网端点 — 增强

```bash
tccli tke CreateClusterEndpoint --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" --IsExtranet false \
  --SubnetId "<SUBNET_ID>" --SecurityGroup "<SECURITY_GROUP_ID>"
# expected: exit 0
```

> 内网端点必须指定 `SubnetId`（端点 CLB 所在子网）。建议同时指定 `SecurityGroup` 限定访问源。

### 步骤 4：配置 ACL 白名单（公网端点）

```bash
# 修改公网端点的安全策略（白名单源 IP）
tccli tke ModifyClusterEndpointSP --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" --SecurityPolicies '[{"Action":"accept","CidrBlock":"<YOUR_IP>/32"}]'
# expected: exit 0
```

> 公网端点未配白名单时默认拒绝所有。`Action=accept` 放行指定 CIDR。

### 步骤 5：验证

异步操作，检查 ≥4 个维度：

```bash
tccli tke DescribeClusterEndpointStatus --region ap-guangzhou --ClusterId "<CLUSTER_ID>"
# expected: Status="Running"，ErrorMsg 为空
```

| 维度 | 命令 | 预期 |
|:-----|:-----|:-----|
| 端点状态 | `DescribeClusterEndpointStatus` → `Status` | `Running` |
| 端点地址 | `DescribeClusterEndpoints` → `ClusterExternalEndpoint`/`ClusterIntranetEndpoint` | 非空 IP |
| ACL 生效 | `DescribeClusterEndpoints` → `SecurityPolicy` | 含白名单规则 |
| kubeconfig 可用 | `kubectl get nodes`（用 kubeconfig） | 节点列表返回 |

### 独立 VIP 端点

> 除普通端点外，可创建独立 VIP 端点（`CreateClusterEndpointVip`），用独立 CLB VIP 暴露 API Server，配 `SecurityPolicies` 白名单。区别于普通端点的 `IsExtranet` 模式。

```bash
# 创建 VIP 端点 (配白名单源 IP)
tccli tke CreateClusterEndpointVip --region <REGION> \
  --ClusterId "<CLUSTER_ID>" --SecurityPolicies '["<YOUR_IP>/32"]'
# expected: exit 0

# 查询 VIP 端点状态
tccli tke DescribeClusterEndpointVipStatus --ClusterId "<CLUSTER_ID>" --region <REGION>
# expected: exit 0, Status=Running (未创建时 NotFound)
```
```json
{"Status": "NotFound", "ErrorMsg": "NotFound", "RequestId": "..."}
```

```bash
# 删除 VIP 端点
tccli tke DeleteClusterEndpointVip --ClusterId "<CLUSTER_ID>" --region <REGION>
# expected: exit 0
```

> VIP 端点 `Status` 状态机：`NotFound`（未创建）→ `Creating` → `Running`。`DeleteClusterEndpointVip` 仅需 `ClusterId`。

### 切换公网/内网端点

> 已开启的端点可在公网/内网间切换，避免先删再建。`SwitchClusterEndpoint` 的 `IsExtranet` 指定目标类型，`Rollback` 控制切换失败是否回滚。参数以 `--generate-cli-skeleton` 为准。

```bash
# 切换端点类型（IsExtranet=true 切公网，false 切内网；Rollback=true 失败自动回滚）
tccli tke SwitchClusterEndpoint --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" --IsExtranet false --Rollback true
# expected: exit 0 返回 RequestId; 集群类型/端点类型不支持报 FailedOperation.SwitchClusterEndpoint: SWITCH_CLUSTER_ENDPOINT_ERROR
```

| 占位符 | 含义 | 约束 |
|:-------|:-----|:-----|
| `--IsExtranet` | 目标端点类型 | `true`=公网，`false`=内网 |
| `--Rollback` | 切换失败是否回滚到原端点 | `true`（推荐，避免切换失败导致无端点可用）/ `false` |

> `SwitchClusterEndpoint` 切换是异步操作，提交后用 `DescribeClusterEndpointStatus` 轮询到 `Running`。切换到内网端点需集群 VPC 内有可用子网（同 `CreateClusterEndpoint --IsExtranet false` 的子网要求）。

## 清理

> **副作用警告**：关闭端点会断开所有通过该端点的 kubectl/API 访问。生产环境关闭内网端点前确认无业务依赖。

```bash
# 1. 关闭端点
tccli tke DeleteClusterEndpoint --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" --IsExtranet true
# expected: exit 0

# 2. 验证已关闭
tccli tke DescribeClusterEndpointStatus --region ap-guangzhou --ClusterId "<CLUSTER_ID>"
# expected: Status="NotFound"
```

## 故障恢复

### 命令返回错误 (exit ≠ 0)

| 现象 | 诊断 | 根因 | 修复 |
|:--------|:----------|:------------|:-----|
| `ResourceNotFound.SubnetId` | `tccli vpc DescribeSubnets` | 内网端点未指定子网或子网不在集群 VPC | 用集群 VPC 内的子网 |
| `ResourceNotFound.SecurityGroup` | `tccli vpc DescribeSecurityGroups` | 安全组不存在 | 重建安全组或换一个 |
| `FailedOperation` | `DescribeClusterEndpointStatus` → `ErrorMsg` | 端点创建中或 CLB 资源不足 | 等待；超时查 ErrorMsg |
| `UnsupportedOperation` | `DescribeClusterStatus` 看状态 | 集群非 Running | 等集群 Running 后重试 |
| `ResourceInUse` | `DescribeClusterEndpointStatus` | 端点已存在 | 先 `DeleteClusterEndpoint` 再建 |

### 命令成功但状态不对 (exit = 0)

| 现象 | 诊断 | 根因 | 修复 |
|:--------|:----------|:------------|:-----|
| 长时间停在 `Creating` | `DescribeClusterEndpointStatus` → `ErrorMsg` | CLB 创建失败或子网 IP 不足 | 查 ErrorMsg，换子网或提工单 |
| 端点 `Running` 但 kubectl 连不上 | `kubectl get nodes --v=6` | ACL 白名单未放行你的 IP | `ModifyClusterEndpointSP` 加你的 IP |
| 公网端点 `Running` 但 `ClusterExternalEndpoint` 为空 | `DescribeClusterEndpoints` | CLB VIP 分配延迟 | 等 1-2 分钟重查 |

## 下一步

- [查询集群](../clusters/query.md) — `DescribeClusterEndpoints` 看端点地址
- [认证配置](../security/auth.md) — 用端点 + kubeconfig 配置 kubectl
- [配置 VPC-CNI](vpc-cni.md) — Pod 网络模型
- [故障排查](../troubleshooting.md) — 端点不通的诊断

## 控制台替代方案

[容器服务控制台 - 集群访问地址](https://console.cloud.tencent.com/tke2/cluster)
