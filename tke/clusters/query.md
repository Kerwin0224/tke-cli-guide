---
doc_type: How-to
subtype: 6A
fused: true
---
# 查询和过滤集群

> 查询集群列表与详情。只读操作，无副作用。支持两种粒度：列表查询（`DescribeClusters`）与单集群全貌（`DescribeClusterStatus` + `DescribeClusterEndpoints`）。

## 概述

查询集群有两种入口，用途不同：

| 查询 | 接口 | 用途 | 返回 |
|:-----|:-----|:-----|:-----|
| 列表查询 | `DescribeClusters` | 看账号下所有集群，按状态/类型过滤 | 集群基本信息（ID/名称/版本/状态/类型） |
| 单集群健康 | `DescribeClusterStatus` | 看某集群运行全貌 | 状态 + 节点计数 + 删除保护 + 审计开关 |
| 单集群访问 | `DescribeClusterEndpoints` / `DescribeClusterSecurity` | 看访问地址与凭证 | 端点 + kubeconfig + 密码 |

操作是**同步**的，命令返回即完成。

## 准备工作

### 环境检查

```bash
tccli --version
# expected: tccli 版本号

tccli cvm DescribeRegions --region ap-guangzhou
# expected: RegionSet 列表返回 → 凭证有效
```

### 资源检查

```bash
# 确认至少有一个集群可查
tccli tke DescribeClusters --region ap-guangzhou --output text --filter "TotalCount"
# expected: 数字 ≥ 1
```

## 关键字段

> 来源：`tccli tke DescribeClusters --generate-cli-skeleton`（实测）。`DescribeClusters` 是两版同名 Action，**入参两版一致**（5 字段，可跨版本传参），但**响应不同**（同名≠同契约，D30）：旧版（2018-05-25，本文所用）响应顶层无 `Errors`，`Clusters[]` 字段更丰富（含 `ClusterNetworkSettings`/`ClusterNodeNum`/`Property` 等 28 字段）；新版（2022-05-01）响应顶层多 `Errors`，`Clusters[]` 字段精简（10 字段）。**用 `--filter` 取字段时按所调版本的响应结构写**，跨版本套用会 `KeyError`——期待 `Errors` 在旧版踩空，期待 `ClusterNetworkSettings` 在新版丢失。本文示例均按旧版响应结构。

| 字段 | 类型 | 必填 | 约束 | 填错时的错误 |
|:------|------|:--------:|------------|---------------|
| ClusterIds | list | 否 | `["cls-xxx"]`，为空查全部 | `ResourceNotFound` |
| Filters | list | 否 | `Name`/`Values` 对，支持 ClusterName/ClusterType/ClusterStatus/vpc-id/tag-key | `InvalidParameterValue` |
| Limit | int | 否 | 1-100，默认 20 | `InvalidParameterValue` |
| Offset | int | 否 | 默认 0，分页偏移 | `InvalidParameterValue` |
| ClusterType | string | 否 | `MANAGED_CLUSTER` / `INDEPENDENT_CLUSTER` | `InvalidParameterValue` |

> Filter 的 `ClusterStatus` 值用 `Running`（首字母大写），见 [状态机](../reference/states.md)。

## 操作步骤

### Minimal — 列表查询（默认全部）

```bash
tccli tke DescribeClusters --region ap-guangzhou --Limit 10
# expected: TotalCount + Clusters 列表
```

```json
{
    "TotalCount": 2,
    "Clusters": [
        {"ClusterId": "cls-example1", "ClusterName": "prod", "ClusterStatus": "Running", "ClusterVersion": "1.34.1", "ClusterType": "MANAGED_CLUSTER"},
        {"ClusterId": "cls-example2", "ClusterName": "test", "ClusterStatus": "Running", "ClusterVersion": "1.30.0", "ClusterType": "INDEPENDENT_CLUSTER"}
    ],
    "RequestId": "xxx"
}
```

### Enhanced: JMESPath 投影（省 token）

用 `--filter` 在 CLI 侧裁剪字段，只输出关心的列（实测字段名 `ClusterId`/`ClusterName`/`ClusterVersion`/`ClusterType`）：

```bash
tccli tke DescribeClusters --region ap-guangzhou \
  --filter "Clusters[?ClusterStatus=='Running'].{id:ClusterId,name:ClusterName,ver:ClusterVersion,type:ClusterType}" \
  --output text
# expected: 每行一个集群，制表符分隔 id/name/ver/type
```

```text
cls-example1	prod	1.34.1	MANAGED_CLUSTER
cls-example2	test	1.30.0	INDEPENDENT_CLUSTER
```

> `--filter`（JMESPath）和 `--Filters`（API 入参）是两回事：`--Filters` 让服务端按条件返回，`--filter` 让 CLI 本地裁剪。两者可叠加——先 `--Filters` 服务端过滤，再 `--filter` 本地投影。

### Enhanced: 服务端过滤 + 分页

```bash
# 按状态过滤（服务端）
tccli tke DescribeClusters --region ap-guangzhou \
  --Filters '[{"Name":"ClusterStatus","Values":["Running"]}]' \
  --Limit 10 --Offset 0
# expected: TotalCount = Running 集群数

# 翻页（超过 Limit 时）
tccli tke DescribeClusters --region ap-guangzhou --Limit 10 --Offset 10
# expected: 第 11-20 个集群；不足时 TotalCount 不变但 Clusters 变少
```

### 单集群健康全貌

```bash
tccli tke DescribeClusterStatus --region ap-guangzhou --ClusterIds '["<CLUSTER_ID>"]'
# expected: ClusterState="Running", ClusterInstanceState="AllNormal"（取值：`-` 空集群无节点 / `AllNormal` 健康 / 异常时为问题描述，见 [故障排查](../troubleshooting.md)）
```

```json
{
    "ClusterStatusSet": [
        {
            "ClusterId": "cls-example",
            "ClusterState": "Running",
            "ClusterInstanceState": "AllNormal",
            "ClusterRunningNodeNum": 2,
            "ClusterFailedNodeNum": 0,
            "ClusterDeletionProtection": true,
            "ClusterAuditEnabled": true
        }
    ],
    "TotalCount": 1,
    "RequestId": "xxx"
}
```

### 单集群访问地址与凭证

```bash
# 访问端点（公网/内网地址）
tccli tke DescribeClusterEndpoints --region ap-guangzhou --ClusterId "<CLUSTER_ID>"
# expected: ClusterDomain = "cls-xxx.ccs.tencent-cloud.com"
```

```json
{
    "ClusterExternalEndpoint": "",
    "ClusterIntranetEndpoint": "",
    "ClusterDomain": "cls-example.ccs.tencent-cloud.com",
    "ClusterExternalACL": null,
    "ClusterExternalDomain": "cls-example.ccs.tencent-cloud.com",
    "ClusterIntranetDomain": "cls-example.ccs.tencent-cloud.com",
    "SecurityGroup": "",
    "ClusterIntranetSubnetId": "",
    "CertificationAuthority": "-----BEGIN CERTIFICATE-----\n...\n-----END CERTIFICATE-----\n",
    "RequestId": "xxx"
}
```

> `ClusterExternalEndpoint`/`ClusterIntranetEndpoint` 为空表示未开启外网/内网访问端点，见 [管理端点](../networking/endpoints.md)。`ClusterExternalACL`（非 `SecurityPolicy`）是外网访问白名单。`DescribeClusterSecurity` 返回完整访问凭证（实测字段：`UserName`/`Password`/`CertificationAuthority`/`Kubeconfig`/`Domain`/`PgwEndpoint`/`JnsGwEndpoint`/`SecurityPolicy`/`ClusterExternalEndpoint`），用于配置 kubectl，见 [认证配置](../security/auth.md)。

### 集群配置查询

> 集群等级、控制器状态、组件额外参数的查询（只读）。

```bash
# 集群等级属性 (可用等级 L5~L100, 含节点/Pod/CRD 上限)
tccli tke DescribeClusterLevelAttribute --ClusterID "<CLUSTER_ID>" --region <REGION>
# expected: exit 0, Items[] 含 Name/Alias/NodeCount/PodCount/Enable
```
```json
{
    "TotalCount": 9,
    "Items": [{"Name": "5节点", "Alias": "L5", "NodeCount": 5, "PodCount": 150, "Enable": true}]
}
```

> ⚠️ `DescribeClusterLevelAttribute`/`DescribeClusterLevelChangeRecords` 用 `--ClusterID`（大写 ID），与其他集群接口的 `--ClusterId`（小写 d）不同——大小写写错报 `Unknown options`。

```bash
# 集群控制器状态 (route-controller / node-ipam-controller 等)
tccli tke DescribeClusterControllers --ClusterId "<CLUSTER_ID>" --region <REGION>
# expected: exit 0, ControllerStatusSet[] 含 Name/Enabled

# 当前组件额外参数 (Etcd/KubeAPIServer/KubeControllerManager/KubeScheduler)
tccli tke DescribeClusterExtraArgs --ClusterId "<CLUSTER_ID>" --region <REGION>
# expected: exit 0, ClusterExtraArgs 四组件参数

# 可用额外参数 (按版本+类型查, 不绑集群)
tccli tke DescribeClusterAvailableExtraArgs --ClusterVersion "<VERSION>" --ClusterType MANAGED_CLUSTER --region <REGION>
# expected: exit 0, AvailableExtraArgs 含各组件可配参数 Name/Type/Usage/Constraint

# 集群等级变更记录
tccli tke DescribeClusterLevelChangeRecords --ClusterID "<CLUSTER_ID>" --region <REGION>
# expected: exit 0, Items[] 变更记录 (无变更则空)
```

> 集群属性变更（ModifyClusterAttribute/Tags/Image/ExtraArgs、AddClusterCIDR、AcquireClusterAdminRole）与运行时配置（ModifyClusterRuntimeConfig/ModifyMasterComponent）是写操作，见 [配置集群属性与运行时](configure.md)。

| 占位符 | 含义 | 约束 | 获取方式 |
|:-------|:-----|:-----|:---------|
| `<CLUSTER_ID>` | 集群 ID | `cls-xxxxxxxx` | `tccli tke DescribeClusters --region <REGION>` → `Clusters[].ClusterId` |
| `<REGION>` | 地域 | 如 `ap-guangzhou` | `tccli tke DescribeRegions` |
| `<VERSION>` | K8s 版本 | 如 `1.34.1` | `tccli tke DescribeClusters --ClusterIds '["<CLUSTER_ID>"]'` → `ClusterVersion` |

### 计费与 RBAC 查询

> 集群等级价格、子账号 RBAC 关系、标签批量修改状态查询。

```bash
# 查询集群等级价格 (不绑集群, 按 ClusterLevel; 用真实枚举 L5/L20/L50/L100/L200/L500)
tccli tke GetClusterLevelPrice --ClusterLevel L20 --region <REGION>
# expected: exit 0, Cost/TotalCost/Policy (实测 L5→13/L20→37/L50→47/L100→83; 传不存在的 L10 触发 FailedOperation.TradeCommon)
```

> `GetClusterLevelPrice` 属计费查询（P8 cost transparency），仅需 `ClusterLevel`（真实枚举 L5/L20/L50/L100/L200/L500；**无 L10**，传 L10 稳定触发 `FailedOperation.TradeCommon`）。等级属性见上文 [DescribeClusterLevelAttribute](#集群配置查询)。

```bash
# 查询集群 CommonName (RBAC 子账号/角色关系)
tccli tke DescribeClusterCommonNames --ClusterId "<CLUSTER_ID>" --region <REGION> \
  --SubaccountUins '["<SUB_UIN>"]' --RoleIds '["<ROLE_ID>"]'
# expected: exit 0, CommonNames[] 含子账号与角色绑定

# 查询标签批量修改状态 (异步任务结果)
tccli tke DescribeBatchModifyTagsStatus --ClusterId "<CLUSTER_ID>" --region <REGION>
# expected: exit 0, Status=done + FailedResources[]
```
```json
{"FailedResources": [], "Status": "done", "SyncSubresource": false, "Tags": [], "RequestId": "..."}
```

> `DescribeClusterCommonNames` 查 RBAC 子账号（SubaccountUins）/角色（RoleIds）与集群的绑定关系。`DescribeBatchModifyTagsStatus` 查标签批量修改异步任务状态（`Status=done` 完成，`FailedResources[]` 失败资源）。

## 验证

> 只读操作，验证即确认输出结构正确、字段名匹配。

| 维度 | 命令 | 预期 |
|:-----|:-----|:-----|
| 列表查询 | `DescribeClusters --filter "TotalCount"` | 数字 ≥ 1 |
| filter 字段名 | `DescribeClusters --filter "Clusters[0].ClusterId"` | 返回首个集群 ID，无 JMESPath 错误 |
| 单集群状态 | `DescribeClusterStatus` | `ClusterState` 字段存在 |
| 访问地址 | `DescribeClusterEndpoints` | `ClusterDomain` 字段存在 |

## 清理

> 只读操作，无副作用，无需清理（P8 显式标注）。

## 故障恢复

### 命令返回错误 (exit ≠ 0)

| 现象 | 诊断 | 根因 | 修复 |
|:--------|:----------|:------------|:-----|
| `RegionNotFound` | `tccli tke DescribeClusters --region <REGION>` | 地域不支持 | 换有效地域，如 `ap-guangzhou` |
| `AuthFailure.SecretIdNotFound` | `tccli cvm DescribeRegions` | 凭证失效 | `tccli configure` 重新配置 |
| `InvalidParameterValue` | 检查 `--Filters` 的 Name/Values 格式 | Filter 名拼写错或值类型不对 | 用支持的 Filter 名（ClusterName/ClusterType/ClusterStatus/vpc-id/tag-key/tag-value/Tags） |
| `ResourceNotFound` | `tccli tke DescribeClusters` 核对 ID | `ClusterIds` 里的集群不存在 | 确认集群 ID 格式 `cls-xxxxxxxx` |

### 命令成功但状态不对 (exit = 0)

| 现象 | 诊断 | 根因 | 修复 |
|:--------|:----------|:------------|:-----|
| `TotalCount: 0` 但集群存在 | 换地域查 `tccli tke DescribeClusters --region <其他地域>` | region 不对，集群在别的地域 | 用正确地域重查 |
| `--filter` 返回空但 `TotalCount > 0` | 先去掉 `--filter` 看 Clusters 字段名 | JMESPath 字段名拼错（如 `ClusterState` 写成 `cluster_state`） | 用响应实际字段名，区分大小写 |
| 分页遗漏集群 | 检查 Offset/Limit | 只读了前 N 条，未翻页 | 循环 `Offset += Limit` 直到 Clusters 为空 |
| `DescribeClusterStatus` 返回空 ClusterStatusSet | 核对 ClusterIds 参数格式 | `ClusterIds` 未用 JSON 数组 | 传 `--ClusterIds '["cls-xxx"]'`（JSON 字符串数组） |

> ⚠️ `--filter` 字段名必须匹配 API 实际响应键名。响应是 `Clusters` 就不能写 `clusters`，是 `ClusterState` 就不能写 `cluster_state`。首次用某接口时，先 `--Limit 1` 看响应结构，再构造 `--filter`。

## 下一步

- [创建集群](create.md) — 查询的逆操作，新建集群
- [配置集群属性与运行时](configure.md) — 改等级/标签/镜像/运行时（写操作）
- [单集群健康全貌](../reference/states.md) — 状态字段含义
- [管理端点](../networking/endpoints.md) — `ClusterExternalEndpoint` 为空时如何开启
- [认证配置](../security/auth.md) — 用 `DescribeClusterSecurity` 的 kubeconfig 配置 kubectl
- [故障排查](../troubleshooting.md) — `ClusterInstanceState` 非 AllNormal 的诊断

## 控制台替代方案

[容器服务控制台 - 集群列表](https://console.cloud.tencent.com/tke2/cluster)

## Action 清单

| Action | 类型 | 版本 | 说明 |
|:-------|:-----|:-----|:-----|
| `AcquireClusterAdminRole` | 主操作 | 2018-05-25 | 获取集群管理员角色（权限） |
| `AddClusterCIDR` | 主操作 | 2018-05-25 | 添加集群 CIDR |
| `ModifyClusterAttribute` | 主操作 | 2018-05-25 | 修改集群属性 |
| `ModifyClusterRuntimeConfig` | 主操作 | 2018-05-25 | 修改运行时配置 |
| `ModifyMasterComponent` | 主操作 | 2018-05-25 | 修改 Master 组件参数 |
| `DescribeClusters` | 验证 | 2018-05-25 | 列表查询（入参两版一致，响应不同见关键字字段段 D30） |
| `DescribeClusterStatus` | 验证 | 2018-05-25 | 单集群健康全貌 |
| `DescribeClusterSecurity` | 验证 | 2018-05-25 | kubeconfig + 密码 + CA |
| `DescribeClusterEndpoints` | 验证 | 2018-05-25 | 访问端点地址 |
| `DescribeClusterExtraArgs` | 验证 | 2018-05-25 | 当前组件额外参数 |
| `DescribeClusterAvailableExtraArgs` | 验证 | 2018-05-25 | 可用额外参数（按版本+类型） |
| `DescribeClusterLevelAttribute` | 验证 | 2018-05-25 | 集群等级属性（用 `--ClusterID` 大写） |
| `DescribeClusterLevelChangeRecords` | 验证 | 2018-05-25 | 等级变更记录（用 `--ClusterID` 大写） |
| `DescribeClusterCommonNames` | 验证 | 2018-05-25 | RBAC 子账号/角色绑定 |
| `DescribeClusterControllers` | 验证 | 2018-05-25 | 控制器状态 |
| `DescribeBatchModifyTagsStatus` | 验证 | 2018-05-25 | 标签批量修改异步状态 |
| `GetClusterLevelPrice` | 验证 | 2018-05-25 | 集群等级价格（计费域） |
| `DescribeRegions` | 验证 | 2018-05-25 | 凭证有效性检查 |
