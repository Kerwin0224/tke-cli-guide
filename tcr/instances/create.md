---
doc_type: How-to
subtype: 6A
fused: true
---
# 创建 TCR 实例

> 创建容器镜像服务 (TCR) 企业版实例，用于存储和管理 Docker/OCI 镜像。
> 控制台: [实例管理](https://console.cloud.tencent.com/tcr/?rid=1) →「新建」进入 [购买页](https://buy.cloud.tencent.com/tcr)（单页选购，非多步向导）
> 官方文档: [产品概述](https://cloud.tencent.com/document/product/1141/39278) · [产品服务层级与容量限制](https://cloud.tencent.com/document/product/1141/104731) · [企业版快速入门](https://cloud.tencent.com/document/product/1141/39287) · [创建企业版实例](https://cloud.tencent.com/document/product/1141/51110) · [退费说明](https://cloud.tencent.com/document/product/1141/53319)

## 触发条件

- `tccli tcr DescribeInstances --region <REGION>` 返回 `TotalCount: 0`，需要创建首个企业版实例；若实例数已达地域级配额上限（官方默认 10），须先清理闲置实例或申请提高配额
- 需要独立镜像存储域名/VPC 内网访问/镜像签名能力，个人版 (`DescribeImagePersonal`) 已不满足
- `CheckInstanceName --RegistryName "<NAME>"` 返回 `IsValidated: true`（名称未被占用）且选定地域在 `DescribeRegions` 返回列表内


## 概述

TCR 实例是镜像存储的容器 —— 每个实例有独立的域名、存储、访问策略。创建实例后在实例内创建命名空间和仓库。

| 选项 | 最佳场景 | 配额（NS / 仓库 / Helm / VPC） | 升级路径 |
|------|---------|:---:|---------|
| basic (基础版) | 个人开发/小团队入门 | 50 / 1000 / 1000 / 5 | 可升级为 standard |
| standard (标准版) | 中小团队生产 | 100 / 3000 / 3000 / 10 | 可升级为 premium |
| premium (高级版) | 大规模 / 完整同步与企业能力 | 500 / 5000 / 5000 / 20 | 当前最高规格 |

> 配额与功能差异以官方 [产品服务层级与容量限制](https://cloud.tencent.com/document/product/1141/104731) 为准；配额数字汇总见 [配额与限制](../reference/quotas.md)。地域级默认最多 **10** 个企业版实例（超额常见 `LimitExceeded` 类错误码，以实际响应 `Error.Code` 为准）。

镜像与 Chart 数据落在关联 COS 桶，按 COS 用量计费（非上表「存储配额」）。**规格选择**：先按必需功能确定最低规格，再核对容量配额和计费方式；只有所需功能在 `basic` 可用且容量满足时，才从 `basic` 做最小验证，后续可按需升级。

操作是**异步**的: `CreateInstance` 返回 `RegistryId` 后实例进入创建过渡态（`Pending`→`Deploying`，官方 `Registry.Status` 无 `Creating` 字面值），须轮询 `DescribeInstanceStatus` 直到 `Status: "Running"` 才可使用（约 3-5 分钟，见 [实例状态](../reference/states.md)）。

## 准备工作

### 环境检查

```bash
tccli --version
# expected: 最新版本或更高

tccli tcr DescribeInstances --region ap-guangzhou
# expected: { "TotalCount": ..., "Registries": [...] }  → 凭证有效
```

### 资源检查

```bash
# 确认实例名未被占用
tccli tcr CheckInstanceName --region ap-guangzhou \
  --RegistryName "<INSTANCE_NAME>"
# expected: { "IsValidated": true, "DetailCode": 0 }  → 名称可用（IsValidated=true 即可用，DetailCode=0 无冲突）

# 确认地域支持 TCR
tccli tcr DescribeRegions
# expected: Regions 列表包含目标地域
```

### 创建后改不了（须在创建时定）

| 决策项 | 约束 | 错了怎么办 |
|:-------|:-----|:-----------|
| **实例名** `RegistryName` | 全局唯一；直接用于 Registry 访问域名；**创建后不可修改** | 只能删实例重建（数据在关联 COS，误删桶不可找回） |
| **实例地域** `--region` | **创建后地域无法更改**；按容器集群所在地选择 | 在目标地域新建实例 + 同步/迁移镜像 |
| **实例域名** | 前缀与实例名一致，自动生成；**创建后无法修改** | 同实例名——重建；或用 [自定义域名](custom-domain.md) 作补充入口（不替代原域名） |
| **关联 COS 桶** | 创建时自动关联；镜像数据落桶 | **勿误删 COS 桶**，否则实例内镜像无法找回 |

## 关键字段

> 完整入参以 `tccli tcr CreateInstance help --detail` 为准。

| 字段 | 类型 | 必填 | 约束 | 填错时的错误 |
|-------|------|:--------:|------------|---------------|
| RegistryName | string | 是 | 控制台：5–50 字符，小写字母/数字/`-`，不能以 `-` 开头或结尾；创建后不可改。短名/大写等非法格式 → `InvalidParameter.ErrorNameIllegal`；`CheckInstanceName` 返回 `IsValidated: false` | `InvalidParameter.ErrorNameIllegal` / `InvalidParameter.ErrorRegistryName` / `InvalidParameter.ErrorNameExists` |
| RegistryType | string | 是 | `basic` / `standard` / `premium`（大小写敏感，须小写） | `InvalidParameter`（`not support registry type`） |
| RegistryChargeType | integer | 否 | `0` 按量计费（API 默认）/ `1` 预付费（包年包月） | `InvalidParameter` |
| DeletionProtection | boolean | 否 | `true` / `false`，默认 `false` | — |
| RegistryChargePrepaid | CreateInstance | 条件 | 仅预付费实例适用；对象含 `Period`（月数）和 `RenewFlag` | `InvalidParameter` |
| RegistryChargePrepaid | RenewInstance | 是 | 预付费续费标识与购买时长 | `InvalidParameter` |
| TagSpecification | object | 否 | Tags 数组 | `InvalidParameter.ErrorTagOverLimit` |
| SyncTag | boolean | 否 | 是否同步 COS Tag | — |
| EnableCosMAZ | boolean | 否 | 是否开启 COS 多 AZ；购买页默认勾选并建议开启 | — |
| EnableCosVersioning | boolean | 否 | 是否开启关联 COS 桶多版本控制 | — |

| 字段 | 所属 Action | 必填 | 说明 |
|:---|:---|:---:|:---|
| `RegistryId` | `DeleteInstance` | 是 | 待删除实例 ID |
| `RegistryId` | `ModifyInstance` | 是 | 待修改实例 ID |
| `RegistryChargePrepaid` | `RenewInstance` | 是 | 预付费续费标识与购买时长 |

## 操作步骤

### 步骤 1：决策 — 选实例规格 {#步骤-1决策-选实例规格}

#### 先按功能门槛选最低规格

- **先核必需功能**：镜像部署阻断至少选 `standard`；镜像签名验签、按需加载容器镜像或同实例多地域就近访问选 `premium`；跨实例自定义规则同步和跨账号实例间镜像同步至少选 `standard`
- **再核容量**：确认命名空间、仓库、Helm、VPC 接入与服务级账号配额；若功能无更高规格门槛且 `basic` 容量足够，可用 `basic` 做最小验证
- **最后选计费**：API 默认按量（`RegistryChargeType: 0`）；购买页建议长期使用优先包年包月（防欠费回收）。按量与包年包月两条路径均可执行，按场景二选一
- **可修改**：规格可以升级（basic→standard→premium），不能降级。计费模式从按量可转包年包月；包年包月退还须满足当前退费规则

### 步骤 2：创建实例

`CreateInstance` 必传 `RegistryName` + `RegistryType`。按场景**二选一**：A 最小化（basic 按量，入门测试）或 B 增强（premium 包年包月+删除保护+COS 多 AZ，生产）。

> ⚠️ **A 与 B 是二选一变体，不是先做 A 再做 B**——两者各调一次 `CreateInstance` 会建**两个实例**。实例创建后改配置（规格升级/续费/存储扩容）用 `ModifyInstance`/`ModifyInstanceStorage`/`RenewInstance`（见本页「实例生命周期管理」段），**禁用第二次 `CreateInstance` 改配置**。规格可升不可降（basic→standard→premium）。

#### 选项 A：最小化（basic 按量，入门测试）

```bash
tccli tcr CreateInstance \
  --region ap-guangzhou \
  --RegistryName "<INSTANCE_NAME>" \
  --RegistryType basic \
  --RegistryChargeType 0 \
  --TagSpecification '{"ResourceType":"instance","Tags":[{"Key":"billing","Value":"<BILLING_OWNER>"}]}'
# expected: { "RegistryId": "tcr-xxxxxxxx", "RequestId": "..." }
```

| 占位符 | 含义 | 约束 | 如何获取 |
|------------|------|------|---------|
| `<INSTANCE_NAME>` | 实例名称 | 5–50 字符，小写字母/数字/`-`，全局唯一 | 自己定义 |
| `<BILLING_OWNER>` | 计费标签值 | 用于标识资源负责人或成本归属 | 按团队标签规范填写 |

#### 选项 B：增强（premium 包年包月+删除保护+COS 多 AZ，生产）

> **与 A 二选一，非在 A 之后执行**。生产环境: premium 规格 + 包年包月 + 删除保护 + COS 多 AZ。

```bash
tccli tcr CreateInstance \
  --region ap-guangzhou \
  --RegistryName "<INSTANCE_NAME>" \
  --RegistryType premium \
  --RegistryChargeType 1 \
  --RegistryChargePrepaid '{
    "Period": 1,
    "RenewFlag": 1
  }' \
  --DeletionProtection true \
  --EnableCosMAZ true \
  --TagSpecification '{"ResourceType":"instance","Tags":[{"Key":"billing","Value":"<BILLING_OWNER>"}]}'
# expected: { "RegistryId": "tcr-xxxxxxxx", "RequestId": "..." }
```

包年包月 1 个月 (`Period: 1`)，自动续费 (`RenewFlag: 1`)。COS 多 AZ 提升存储可靠性。

### 步骤 3：验证

```bash
tccli tcr DescribeInstances \
  --region ap-guangzhou \
  --Registryids '["<REGISTRY_ID>"]'
# expected: { "TotalCount": 1, "Registries": [{ "Status": "Running" }] }
```

| 维度 | 命令 | 预期 |
|-----------|---------|----------|
| Status | `DescribeInstances` | `Status: "Running"` |
| 实例名 | `DescribeInstances` → `Registries[].RegistryName` | 与创建参数一致 |
| 实例类型 | `DescribeInstances` → `Registries[].RegistryType` | 与创建参数一致 |
| 删除保护 | `DescribeInstances` → `Registries[].DeletionProtection` | 与创建参数一致 |
| 公网域名 | `DescribeInstances` → `Registries[].PublicDomain` 格式 | `xxx.tencentcloudcr.com` |

## 清理

> ⚠️ 删除 TCR 实例会**级联删除**实例内所有命名空间/仓库/镜像（数据永久丢失，不可恢复）；自定义域名与 VPC 链接同时删除。包年包月实例应先在控制台实例列表确认是否满足五天无理由或普通自助退还条件，并以 [退费说明](https://cloud.tencent.com/document/product/1141/53319) 和控制台试算金额为准；特殊地域、特殊配置或部分活动资源可能不支持自助退还。本段是删除 TCR 实例的唯一操作入口。

> 若需保留某些镜像，先 `tccli tcr DuplicateImage` 复制到其他实例（见 [推送拉取镜像](../images/push-pull.md)）。

```bash
# 确认子资源（命名空间/仓库）+ 查删除保护状态
tccli tcr DescribeNamespaces --region ap-guangzhou --RegistryId "<REGISTRY_ID>"
# expected: NamespaceList 可空；有残留命名空间时先清理再删实例

tccli tcr DescribeRepositories --region ap-guangzhou --RegistryId "<REGISTRY_ID>"
# expected: RepositoryList 可空；有残留仓库时先清理再删实例

tccli tcr DescribeInstances --region ap-guangzhou --Registryids '["<REGISTRY_ID>"]' --filter "Registries[0].DeletionProtection"
# expected: 子资源列表 + DeletionProtection 值（true 时需先关）

# 关删除保护（仅 DeletionProtection=true 时）+ 删除 + 验证
tccli tcr ModifyInstance --region ap-guangzhou --RegistryId "<REGISTRY_ID>" --DeletionProtection false
# expected: exit 0

tccli tcr DeleteInstance --region ap-guangzhou --RegistryId "<REGISTRY_ID>"
# expected: exit 0
# 可选：--DeleteBucket true 同时删关联 COS 桶（默认 false，仅删实例；桶内镜像数据策略按账号需要选择）
# 可选：--DryRun true 仅预检不真正删除

tccli tcr DescribeInstances --region ap-guangzhou --Registryids '["<REGISTRY_ID>"]'
# expected: { "TotalCount": 0, "Registries": [] } → 已删（tccli 默认剥离 Response 包装层）
```

> 若 DescribeInstances 仍返回实例但状态为 `Deleting`（或删除失败相关 `DeleteFailed`/`DeleteBucketFailed`），属删除中或删除异常，稍候再查或见 [实例状态](../reference/states.md)。

> **计费警告**：按量计费实例删除即停止计费。包年包月实例如需提前结束，先从 [TCR 控制台实例管理](https://console.cloud.tencent.com/tcr/) 发起自助退还并查看资格与退款试算；退款金额、代金券处理、实例和 COS 的处置以当前 [退费说明](https://cloud.tencent.com/document/product/1141/53319) 及控制台结果为准，不要先用 `DeleteInstance` 代替退还流程。

## 故障恢复

### 命令返回错误（exit ≠ 0）

| 现象 | 诊断 | 根因 | 修复 |
|---------|----------|------------|-----|
| `AuthFailure.SecretIdNotFound` | `tccli tcr DescribeRegions` | 凭证未配置 | 见 [配置凭证](../../getting-started/credentials.md) |
| `InvalidParameter.ErrorNameIllegal` / `InvalidParameter.ErrorRegistryName` | `CheckInstanceName`（`IsValidated: false`，`DetailCode` 非 0） | 名称长度/字符非法（短名、大写、`-` 首尾等） | 使用 5–50 字符小写字母/数字/`-`，不能以 `-` 开头或结尾 |
| `InvalidParameter.ErrorNameExists` | `tccli tcr CheckInstanceName --RegistryName "<NAME>"` | 名称已被占用 | 换一个名称 |
| `InvalidParameter.ErrorNameReserved` | 换名重试 | 名称被保留 | 换一个名称 |
| `InvalidParameter`（`not support registry type`） | 检查 `RegistryType` 字面值 | 非 `basic`/`standard`/`premium` 或大小写错误 | 用小写规格枚举 |
| `InvalidParameter.UnsupportedRegion` | `tccli tcr DescribeRegions` | 所选地域不支持创建实例 | 换支持的地域 |
| `FailedOperation.ValidateRegistryNameFail` | `CheckInstanceName` | 名称校验失败 | 按控制台规则改名后再建 |
| `FailedOperation.TradeFailed` | 保留完整 `Error.Message` 和 `RequestId`；检查购买页、计费状态与服务角色授权状态 | 交易链路校验失败；若消息明确提到 `TCR_QCSRole`，需进一步检查该服务角色，不能仅凭错误码断定根因 | 按完整消息处理对应前置；消息明确要求角色时在控制台检查/补充授权，否则携 `RequestId` 查询交易失败原因或提交工单 |
| `LimitExceeded`（实例配额，默认地域级 10） | `tccli tcr DescribeInstances` / `--AllRegion true` | 实例数达上限 | 删除闲置实例或提工单 |

### 命令成功但状态不对（exit = 0）

| 现象 | 诊断 | 根因 | 修复 |
|---------|----------|------------|-----|
| `Status` 不是 `Running` | `tccli tcr DescribeInstanceStatus --RegistryIds '["<REGISTRY_ID>"]'` | 初始化/部署未完成或异常 | 轮询 `DescribeInstanceStatus` 直到 `Running`（创建异步；过渡态为 `Pending`/`Deploying`，非 `Creating`。注意 `DescribeInstanceStatus` 用 `--RegistryIds` 大写 D，与 `DescribeInstances` 的 `--Registryids` 小写 d 不同）。若为 `FailedCreated`/`Bucket-Error`/`Unhealthy` 等，见 [状态机](../reference/states.md) |
| 无法 `docker login` | `DescribeInternalEndpoints` / `DescribeExternalEndpointStatus` | 访问端点未开（默认全拒绝） | 优先内网 `ManageInternalEndpoint`；本地/外网再 `ManageExternalEndpoint --Operation Create`，见 [访问管理](manage-access.md) |
| 创建成功但未出现在列表 | 检查 `--region` 是否与创建时一致；或 `DescribeInstances --AllRegion true` | 查看的地域错误 | 切换到创建时的地域 |

## 实例生命周期管理

> 创建后的存储扩容、续费、校验、命名空间查询。

```bash
# 校验实例名可用性 (创建前/改名前用, 仅 RegistryId)
tccli tcr CheckInstance --RegistryId "<REGISTRY_ID>" --region <REGION>
# expected: exit 0, IsValidated=true (有效)
```
```json
{"IsValidated": true, "RegionId": 1, "RequestId": "..."}
```

```bash
# 存储扩容 (跨地域, TargetRegion + TargetStorageName)
tccli tcr ModifyInstanceStorage --RegistryId "<REGISTRY_ID>" --region <REGION> \
  --TargetRegion "<TARGET_REGION>" --TargetStorageName "<STORAGE_NAME>"
# expected: exit 0

# 续费 (嵌套 RegistryChargePrepaid: Period/RenewFlag, Flag 控制续费方式)
tccli tcr RenewInstance --RegistryId "<REGISTRY_ID>" --region <REGION> \
  --RegistryChargePrepaid '{"Period":12,"RenewFlag":1}' --Flag 0
# expected: exit 0

# 查询所有实例的全部命名空间 (不绑实例, 账号级)
tccli tcr DescribeInstanceAllNamespaces --Limit 50 --region <REGION>
# expected: exit 0, 返回各实例命名空间汇总
```

> `CheckInstance` 仅需 `RegistryId`，返回 `IsValidated`+`RegionId`。`ModifyInstanceStorage` 的 `TargetStorageName` 是目标存储规格名。`RenewInstance` 的 `RegistryChargePrepaid.Period` 是月数，`RenewFlag` 是续费标志（0/1/2），`Flag` 控制续费方式。`DescribeInstanceAllNamespaces` 不传 RegistryId（账号级查询）。

## 收尾确认

```bash
# 确认实例 Running，可进入创建命名空间/仓库
tccli tcr DescribeInstances --region ap-guangzhou --Registryids '["<REGISTRY_ID>"]' \
  --filter "Registries[0].{status:Status,name:RegistryName,type:RegistryType,protect:DeletionProtection}"
# expected: status="Running", name/type 与创建参数一致, protect 与创建参数一致

# 确认新实例尚无命名空间，下一步从创建命名空间开始
tccli tcr DescribeNamespaces --region ap-guangzhou --RegistryId "<REGISTRY_ID>" --filter "TotalCount"
# expected: 0
```

> 实例 `Running` = 创建完成，可进入 [访问管理](manage-access.md) 与 [命名空间/仓库](../repositories/manage.md)。`DescribeNamespaces` 须在 `Running` 后调用（`Pending`/`Deploying` 过渡态中调用通常返回空）。空实例无镜像，须先建命名空间才能 push。
>
> 访问端点**默认不强制开启**：默认拒绝全部公网/内网访问。生产优先内网 VPC；本地或外网 CI 再开公网并配白名单——见 [访问管理](manage-access.md)。docker login/push/pull 属 docker CLI（非 tccli；TCCLI 无镜像传输能力），在端点+Token 配好后于 [推送拉取镜像](../images/push-pull.md) 执行。
>
> **账号侧诊断**：若企业版 `DescribeInstances`（含 `--AllRegion true`）`TotalCount: 0` 且 `CreateInstance` 返回 `FailedOperation.TradeFailed`，完整消息可能提到 `TCR_QCSRole` / 商品下单校验——这只是诊断线索，不能证明该角色是所有 `TradeFailed` 的确定根因。先保存完整 `Error.Message` 与 `RequestId`：消息明确要求 `TCR_QCSRole` 时检查控制台服务角色授权；否则按消息检查计费/购买前置，并携 `RequestId` 查询原因或提交工单。

---

## 下一步

- [管理实例访问](manage-access.md) — **优先内网**再按需公网、创建 Token
- [创建命名空间和仓库](../repositories/manage.md) — 实例内首项操作
- [推送镜像](../images/push-pull.md) — docker push 你的第一个镜像
- [配额与限制](../reference/quotas.md) — 规格配额与 `LimitExceeded` 对照
