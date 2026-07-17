---
doc_type: How-to
subtype: 6A
fused: true
---
# 查询和过滤集群

> 控制台: [容器服务控制台 - 集群列表](https://console.cloud.tencent.com/tke2/cluster)
> 查询集群列表与详情。只读操作，无副作用。支持两种粒度：列表查询（`DescribeClusters`）与单集群全貌（`DescribeClusterStatus` + `DescribeClusterEndpoints`）。

> 官方文档：[基本概念](https://cloud.tencent.com/document/product/457/45598) · [连接集群](https://cloud.tencent.com/document/product/457/32191)

## 概述

查询集群有三类入口，用途不同：

| 查询 | 接口 | 用途 | 返回 |
|:-----|:-----|:-----|:-----|
| 列表查询 | `DescribeClusters` | 看账号下所有集群，按状态/类型过滤 | 集群基本信息（ID/名称/版本/状态/类型） |
| 单集群健康 | `DescribeClusterStatus` | 看某集群运行全貌 | 状态 + 节点计数 + 删除保护 + 审计开关 |
| 单集群访问 | `DescribeClusterEndpoints` / `DescribeClusterSecurity` | 看访问地址与凭证 | 端点 + kubeconfig + 密码 |

操作是**同步**的，命令返回即完成。

> 配额：单地域集群默认上限 **20**，查询可核对当前用量。[配额说明](https://cloud.tencent.com/document/product/457/9087)

## 触发条件

- 需要查看账号下有哪些集群、某集群的状态/版本/类型（创建后核对、日常巡检）— 用本文列表查询或单集群健康查询
- 写操作（删除/升级/配置）前需确认目标集群 ID 与状态 — 用 `DescribeClusters`/`DescribeClusterStatus` 锁定目标
- `--filter` 取字段返回空或 None，怀疑跨版本字段缺失 — 跳到 [§字段缺失](#跨版本字段缺失的静默返回) 核对版本响应结构

## 准备工作

### 环境检查

```bash
tccli --version
# expected: tccli 版本号

tccli tke DescribeRegions
# expected: RegionInstanceSet 列表返回 → 凭证有效 + TKE 域可达（顶层键 RegionInstanceSet，非 RegionSet；鉴权探针无需先假定 --region）
```

### 资源检查

```bash
# 确认至少有一个集群可查
tccli tke DescribeClusters --region ap-guangzhou --version 2022-05-01 --output text --filter "TotalCount"
# expected: 数字 ≥ 1
```

## 两版同名 Action：DescribeClusters

> `DescribeClusters` 在 TKE 两个 API 版本中**同名存在**，**入参两版完全一致**（5 字段：`ClusterIds`/`ClusterType`/`Filters`/`Limit`/`Offset`，可跨版本传参），但**响应结构不同**——同名≠同契约。调用时用 `--version` 显式指定版本，避免静默走默认版与意图错位。

| 版本 | `--version` | 顶层字段 | `Clusters[]` 字段数 | 定位 |
|:-----|:-----|:-----|:-----|:-----|
| 2018-05-25（TCCLI 默认） | `--version 2018-05-25` | `TotalCount`/`Clusters`/`RequestId`（**无 `Errors`**） | **28**（含网络/节点数/运行时等） | 字段丰富，一次查询拿全部信息 |
| 2022-05-01（官方当前版） | `--version 2022-05-01` | `TotalCount`/`Clusters`/**`Errors`**/`RequestId` | **10**（精简） | 官方当前版本，长期维护方向 |

> **本文主示例走新版（2022-05-01，官方当前版）**。需要网络配置/节点数/运行时等丰富字段时走旧版（见 [§取丰富字段（旧版独有）](#取丰富字段旧版独有)）。两版 `Clusters[]` 共有 9 字段：`ClusterId`/`ClusterName`/`ClusterDescription`/`ClusterVersion`/`ClusterType`/`ClusterStatus`/`ClusterLevel`/`CreatedTime`/`TagSpecification`。

### 跨版本字段缺失的静默返回

> `--filter`（JMESPath）按所调版本的响应结构取字段。**跨版本套用 `--filter` 表达式会取不到字段**——期待的字段在另一版不存在，JMESPath 不报错但返回 `None`（空），据此判断会误以为“集群无此属性”。

| 跨版本套用 | 旧版取新版独有 | 新版取旧版独有 |
|:-----|:-----|:-----|
| 字段 | `Errors`（顶层） | `ClusterNetworkSettings`/`ClusterNodeNum`/`ContainerRuntime`/`DeletionProtection`/`ClusterOs`/`ImageId`/`RuntimeVersion`/`Property`/`ClusterMaterNodeNum`/`ClusterEtcdNodeNum` 等 19 个 |
| 结果 | 返回 `None` | 返回 `None` |
| 根因 | 旧版响应顶层无 `Errors` | 新版 `Clusters[]` 精简掉这些字段 |

双版本对照（ap-guangzhou，同一集群）：

```bash
# 旧版取 Errors → None（旧版顶层无此字段）
tccli tke DescribeClusters --region ap-guangzhou --version 2018-05-25 --Limit 1 \
  --filter "Errors" --output text
# expected: None

# 新版取 ClusterNetworkSettings → None（新版 Clusters[] 无此字段）
tccli tke DescribeClusters --region ap-guangzhou --version 2022-05-01 --Limit 1 \
  --filter "Clusters[0].ClusterNetworkSettings" --output text
# expected: None
```

> **判据**：写 `--filter` 前，先确认所调版本的响应结构。首次用某版本时，先 `--Limit 1` 看顶层与 `Clusters[0]` 的实际字段名，再构造表达式。两版共有的 9 字段（ID/名称/状态/类型/版本/等级等）跨版本安全，其余按版本写。

## 关键字段

> `DescribeClusters` 入参两版一致（5 字段）：

| 字段 | 类型 | 必填 | 约束 | 填错时的错误 |
|:------|------|:--------:|------------|---------------|
| ClusterIds | list | 否 | `["cls-xxx"]`，为空查全部；不存在的 ID 不报错，返回 `TotalCount=0` | — |
| Filters | list | 否 | `Name`/`Values` 对，支持 ClusterName/ClusterType/ClusterStatus/vpc-id/tag-key/tag-value/Tags | `InvalidParameter.Param`（`invalid filter name`） |
| Limit | int | 否 | 默认 20；超出范围被服务端截断，不报错 | — |
| Offset | int | 否 | 默认 0，分页偏移 | — |
| ClusterType | string | 否 | `MANAGED_CLUSTER` / `INDEPENDENT_CLUSTER` | `InvalidParameter` |

> Filter 的 `ClusterStatus` 值用 `Running`（首字母大写），见 [状态机](../reference/states.md)。Filter 结构与多 Filter AND/OR 语义见 [共享字段](../reference/shared-fields.md#filter-查询过滤)。

## 操作步骤

> ⚠️ **高危操作**：查询无写副作用，但 `--ClusterIds` 暴露集群 ID 需保密；`--filter` 字段名拼错静默返回 `None` 而非报错，可能误导判断。[常见高危操作](https://cloud.tencent.com/document/product/457/39539)

### 最小化 — 列表查询（新版，官方当前版）

```bash
tccli tke DescribeClusters --region ap-guangzhou --version 2022-05-01 --Limit 10
# expected: TotalCount + Clusters 列表（9 字段精简结构，顶层含 Errors）
```

```json
{
    "TotalCount": 2,
    "Clusters": [
        {"ClusterId": "cls-example1", "ClusterName": "prod", "ClusterStatus": "Running", "ClusterVersion": "1.34.1", "ClusterType": "MANAGED_CLUSTER", "ClusterLevel": "L20", "VpcId": "vpc-example", "CreatedTime": "2026-01-01 00:00:00", "TagSpecification": null},
        {"ClusterId": "cls-example2", "ClusterName": "test", "ClusterStatus": "Running", "ClusterVersion": "1.30.0", "ClusterType": "INDEPENDENT_CLUSTER", "ClusterLevel": "L5", "VpcId": "vpc-example", "CreatedTime": "2026-02-01 00:00:00", "TagSpecification": null}
    ],
    "Errors": [],
    "RequestId": "xxx"
}
```

> 新版 `Clusters[]` 仅 10 字段（含 `VpcId`，旧版的 `VpcId` 在 `ClusterNetworkSettings.VpcId` 里）。顶层多 `Errors`（正常为空数组 `[]`，部分集群查询出错时填错误明细）。

### 增强：JMESPath 投影（省 token）

用 `--filter` 在 CLI 侧裁剪字段，只输出关心的列。两版共有的 9 字段跨版本安全：

```bash
tccli tke DescribeClusters --region ap-guangzhou --version 2022-05-01 \
  --filter "Clusters[?ClusterStatus=='Running'].{id:ClusterId,name:ClusterName,ver:ClusterVersion,type:ClusterType}" \
  --output text
# expected: 每行一个集群，制表符分隔
```

```text
cls-example1	prod	MANAGED_CLUSTER	1.34.1
cls-example2	test	INDEPENDENT_CLUSTER	1.30.0
```

> ⚠️ `--output text` 的列序按投影 key **名字母序**，**非书写序**。声明 `{id,name,ver,type}` 与 `{ver,type,id,name}` 输出列序相同（均为 `id,name,type,ver`）。若需固定列顺序，改用 `--output json`；多字段 text 适合人眼看 tab 分隔，不适合按列号机器解析。

> `--filter`（JMESPath）和 `--Filters`（API 入参）是两回事：`--Filters` 让服务端按条件返回，`--filter` 让 CLI 本地裁剪。两者可叠加——先 `--Filters` 服务端过滤，再 `--filter` 本地投影。

### 增强：服务端过滤 + 分页

```bash
# 按状态过滤（服务端）
tccli tke DescribeClusters --region ap-guangzhou --version 2022-05-01 \
  --Filters '[{"Name":"ClusterStatus","Values":["Running"]}]' \
  --Limit 10 --Offset 0
# expected: TotalCount = Running 集群数

# 翻页（超过 Limit 时）
tccli tke DescribeClusters --region ap-guangzhou --version 2022-05-01 --Limit 10 --Offset 10
# expected: 第 11-20 个集群；不足时 TotalCount 不变但 Clusters 变少
```

### 取丰富字段（旧版独有）

> 需要网络配置、节点数、容器运行时、删除保护等字段时走旧版——这些字段新版 `Clusters[]` 已精简（19 个旧版独有字段在新版丢失）。旧版是 TCCLI 默认版，可省略 `--version`；为明确意图建议显式标注。

```bash
tccli tke DescribeClusters --region ap-guangzhou --version 2018-05-25 \
  --filter "Clusters[?ClusterStatus=='Running'].{id:ClusterId,name:ClusterName,net:ClusterNetworkSettings.VpcId,nodes:ClusterNodeNum,rt:ContainerRuntime,del:DeletionProtection}" \
  --output text
# expected: 含网络/节点数/运行时/删除保护（这些字段新版取会返回 None）
```

```text
True	cls-example1	prod	vpc-example	4	containerd
False	cls-example2	test	vpc-example	0	containerd
```

> 此投影的 text 列序按 key 名字母序固定为 `del,id,name,net,nodes,rt`，不是对象中的书写顺序；上例两行依次解释为删除保护、集群 ID、名称、VPC、节点数、运行时。若需按字段名稳定读取，改用 `--output json`。

> 旧版 `Clusters[]` 28 字段中，19 个新版没有：`ClusterNetworkSettings`（含 VpcId/子网/CIDR）、`ClusterNodeNum`/`ClusterMaterNodeNum`/`ClusterEtcdNodeNum`（节点计数）、`ContainerRuntime`/`RuntimeVersion`（运行时）、`DeletionProtection`（删除保护）、`ClusterOs`/`ImageId`/`OsCustomizeType`（镜像）、`Property`/`ProjectId`/`CdcId`/`IsHighAvailability`/`ClusterCategory`/`SecurityModeConfig`/`EnableExternalNode`/`AutoUpgradeClusterLevel`/`QGPUShareEnable`。查这些字段必须走旧版。

### 单集群健康全貌

> `DescribeClusterStatus` 返回单集群运行状态（非列表查询），字段与 `DescribeClusters` 不同：状态字段叫 `ClusterState`（非 `ClusterStatus`）。

```bash
tccli tke DescribeClusterStatus --region ap-guangzhou --ClusterIds '["<CLUSTER_ID>"]'
# expected: ClusterState="Running", ClusterInstanceState="AllNormal"（取值：`-` 空集群无节点 / `AllNormal` 健康 / 异常时为问题描述）
```

异常问题描述含义见 [故障排查](../troubleshooting.md)

```json
{
    "ClusterStatusSet": [
        {
            "ClusterId": "cls-example",
            "ClusterState": "Running",
            "ClusterInstanceState": "-",
            "ClusterBMonitor": false,
            "ClusterInitNodeNum": 0,
            "ClusterRunningNodeNum": 0,
            "ClusterFailedNodeNum": 0,
            "ClusterClosedNodeNum": 0,
            "ClusterClosingNodeNum": 0,
            "ClusterDeletionProtection": false,
            "ClusterAuditEnabled": false
        }
    ],
    "TotalCount": 1,
    "RequestId": "xxx"
}
```

> 字段说明（2018 默认版）：上表键齐全。`ClusterInstanceState` 为 `-` 表示空集群（`ClusterRunningNodeNum=0`）；有节点且健康为 `AllNormal`。新建托管空集群默认 `ClusterDeletionProtection=false`、`ClusterAuditEnabled=false`（需显式 `Enable*` 才为 `true`）。

### 单集群访问地址与凭证

```bash
# 访问端点（公网/内网地址）
tccli tke DescribeClusterEndpoints --region ap-guangzhou --ClusterId "<CLUSTER_ID>"
# expected: ClusterDomain = "cls-xxx.ccs.tencent-cloud.com"
```

```json
{
    "CertificationAuthority": "-----BEGIN CERTIFICATE-----\n...\n-----END CERTIFICATE-----\n",
    "ClusterExternalEndpoint": "",
    "ClusterIntranetEndpoint": "",
    "ClusterDomain": "cls-example.ccs.tencent-cloud.com",
    "ClusterExternalACL": null,
    "ClusterExternalDomain": "cls-example.ccs.tencent-cloud.com",
    "ClusterIntranetDomain": "cls-example.ccs.tencent-cloud.com",
    "SecurityGroup": "",
    "ClusterIntranetSubnetId": "",
    "IntranetSecurityGroup": "",
    "RequestId": "xxx"
}
```

> `ClusterExternalEndpoint`/`ClusterIntranetEndpoint` 为空表示未开启外网/内网访问端点，见 [管理端点](../networking/endpoints.md)。`ClusterExternalACL`（非 `SecurityPolicy`）是外网访问白名单；`SecurityGroup`/`IntranetSecurityGroup` 分别是外网/内网端点的安全组。

### 集群访问凭证

`DescribeClusterSecurity` 返回完整访问凭证（kubeconfig/密码/CA），用于配置 kubectl，见 [认证配置](../security/auth.md)。

```bash
tccli tke DescribeClusterSecurity --region ap-guangzhou --ClusterId "<CLUSTER_ID>"
# expected: exit 0, 返回 Kubeconfig/Password/UserName/CertificationAuthority/Domain/PgwEndpoint/JnsGwEndpoint/SecurityPolicy/ClusterExternalEndpoint
```

```json
{
    "UserName": "admin",
    "Password": "********",
    "CertificationAuthority": "-----BEGIN CERTIFICATE-----\n...\n-----END CERTIFICATE-----\n",
    "ClusterExternalEndpoint": "",
    "Domain": "cls-example.ccs.tencent-cloud.com",
    "PgwEndpoint": "",
    "SecurityPolicy": null,
    "Kubeconfig": "apiVersion: v1\nkind: Config\n...",
    "JnsGwEndpoint": null,
    "RequestId": "xxx"
}
```

> `Kubeconfig` 是可直接写入文件的完整 kubeconfig；`Password` 是集群访问密码；`SecurityPolicy` 是集群访问策略组（注意：外网访问白名单是 `DescribeClusterEndpoints` 的 `ClusterExternalACL`，非此字段）。配置 kubectl 见 [认证配置](../security/auth.md)。

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
    "Items": [{"Name": "5节点", "Alias": "L5", "NodeCount": 5, "PodCount": 150, "ConfigMapCount": 128, "RSCount": 900, "CRDCount": 150, "OtherCount": 150, "Enable": true}]
}
```

> ⚠️ `DescribeClusterLevelAttribute`/`DescribeClusterLevelChangeRecords` 用 `--ClusterID`（大写 ID），与其他集群接口的 `--ClusterId`（小写 d）不同——大小写写错报 `Unknown options`。`Items[]` 每项含各资源上限：`NodeCount`/`PodCount`/`ConfigMapCount`/`RSCount`/`CRDCount`/`OtherCount`，`Enable=false` 的等级（如 L1000+）需工单开通。

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

> `GetClusterLevelPrice`（集群等级价格查询，按 `ClusterLevel` 真实枚举 L5/L20/L50/L100/L200/L500/L1000/L3000/L5000；L5 可询价（实际返回 Cost=13），传不存在的等级（如 L10）触发 `FailedOperation.TradeCommon`）属计费查询，主命令见 [配置集群属性 — 选等级决策](configure.md#为什么选这个等级)，本文不再重复。Pod 计费/预留实例查询见 [配额和限制](../reference/quotas.md)。

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

| 占位符 | 含义 | 约束 | 获取方式 |
|:-------|:-----|:-----|:---------|
| `<SUB_UIN>` | 子账号 UIN | 数字串 | `tccli cam ListUsers` → 子账号 Uin；或控制台「访问管理 - 用户列表」 |
| `<ROLE_ID>` | 角色 ID | `cam:role` 角色 ID | `tccli cam DescribeRoleList` → 角色的 RoleId |

## 验证

> 只读操作，验证即确认输出结构正确、字段名匹配。

| 维度 | 命令 | 预期 |
|:-----|:-----|:-----|
| 列表查询 | `DescribeClusters --filter "TotalCount"` | 数字 ≥ 1 |
| filter 字段名 | `DescribeClusters --filter "Clusters[0].ClusterId"` | 返回首个集群 ID，无 JMESPath 错误 |
| 单集群状态 | `DescribeClusterStatus` | `ClusterState` 字段存在 |
| 访问地址 | `DescribeClusterEndpoints` | `ClusterDomain` 字段存在 |
| 版本一致性 | `DescribeClusters` 两版 `--filter "Clusters[0].ClusterId"` | 两版返回相同集群 ID（入参一致） |

## 故障恢复

### 命令返回错误 (exit ≠ 0)

| 现象 | 诊断 | 根因 | 修复 |
|:--------|:----------|:------------|:-----|
| `RegionNotFound` | `tccli tke DescribeClusters --region <REGION>` | 地域不支持 | 换等地域，如 `ap-guangzhou` |
| `AuthFailure.SecretIdNotFound` | `tccli tke DescribeRegions` | 凭证失效 | 见 [配置凭证](../../getting-started/credentials.md) 重新配置 |
| `InvalidParameterValue` | 检查 `--Filters` 的 Name/Values 格式 | Filter 名拼写错或值类型不对 | 用支持的 Filter 名（ClusterName/ClusterType/ClusterStatus/vpc-id/tag-key/tag-value/Tags） |
| `ResourceNotFound` | `tccli tke DescribeClusters` 核对 ID | `ClusterIds` 里的集群不存在 | 确认集群 ID 格式 `cls-xxxxxxxx` |

### 命令成功但状态不对 (exit = 0)

| 现象 | 诊断 | 根因 | 修复 |
|:--------|:----------|:------------|:-----|
| `TotalCount: 0` 但集群存在 | 换地域查 `tccli tke DescribeClusters --region <其他地域>` | region 不对，集群在别的地域 | 用正确地域重查 |
| `--filter` 返回空但 `TotalCount > 0` | 先去掉 `--filter` 看 Clusters 字段名 | JMESPath 字段名拼错（如 `ClusterState` 写成 `cluster_state`），或跨版本取了不存在的字段（见 [§字段缺失](#跨版本字段缺失的静默返回)） | 用所调版本响应的实际字段名，区分大小写；查丰富字段走旧版 |
| 分页遗漏集群 | 检查 Offset/Limit | 只读了前 N 条，未翻页 | 循环 `Offset += Limit` 直到 Clusters 为空 |
| `DescribeClusterStatus` 返回空 ClusterStatusSet | 核对 ClusterIds 参数格式 | `ClusterIds` 未用 JSON 数组 | 传 `--ClusterIds '["cls-xxx"]'`（JSON 字符串数组） |

> ⚠️ `--filter` 字段名必须匹配 API 实际响应键名，且须与所调版本一致。响应是 `Clusters` 就不能写 `clusters`，是 `ClusterState` 就不能写 `cluster_state`。跨版本时，旧版独有字段（`ClusterNetworkSettings` 等）在新版取会返回 `None` 而非报错——这是静默返回 None，最易误导。首次用某接口/版本时，先 `--Limit 1` 看响应结构，再构造 `--filter`。

## 收尾确认

```bash
# 核对查询通道可用 + 写操作前目标集群状态（只读操作无残留资源维度；此处确认后续写操作可依赖本查询）
tccli tke DescribeClusters --region <REGION> --filter "TotalCount" --output text
# expected: 数字 ≥ 0 → 列表查询通道正常

tccli tke DescribeClusterStatus --region <REGION> --filter "ClusterStatusSet[?ClusterId=='<CLUSTER_ID>'] | [0].{state:ClusterState,id:ClusterId}" --output text
# expected: state=Running → 目标集群健康，可进入写操作（删除/升级/配置）前的目标核对
```

> 查询通道可用 + 目标集群 `Running` = 只读查询完成，可进入写操作（[删除](delete.md)/[升级](upgrade.md)/[配置](configure.md)）前用本文核对目标集群 ID 与状态。只读操作无残留资源维度，此处确认查询通道就绪后再进入下一步写操作。

## 下一步

- [创建集群](create.md) — 查询的逆操作，新建集群
- [配置集群属性与运行时](configure.md) — 改等级/标签/镜像/运行时（写操作）
- [单集群健康全貌](../reference/states.md) — 状态字段含义
- [管理端点](../networking/endpoints.md) — `ClusterExternalEndpoint` 为空时如何开启
- [认证配置](../security/auth.md) — 用 `DescribeClusterSecurity` 的 kubeconfig 配置 kubectl
- [故障排查](../troubleshooting.md) — `ClusterInstanceState` 非 AllNormal 的诊断
