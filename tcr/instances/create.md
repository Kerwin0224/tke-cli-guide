---
doc_type: How-to
subtype: 6A
fused: true
---
# 创建 TCR 实例

> 创建容器镜像服务 (TCR) 企业版实例，用于存储和管理 Docker/OCI 镜像。
> 控制台: [实例管理](https://console.cloud.tencent.com/tcr/?rid=1) →「新建」进入 [购买页](https://buy.cloud.tencent.com/tcr)（单页选购，非多步向导）

## 触发条件

- `tccli tcr DescribeInstances --region <REGION>` 返回 `TotalCount: 0` 或实例数已达 `LimitExceeded.InstanceQuota` 配额上限需扩容
- 需要独立镜像存储域名/VPC 内网访问/镜像签名能力，个人版 (`DescribeImagePersonal`) 已不满足
- `CheckInstanceName --RegistryName "<NAME>"` 返回 `IsValidated: true`（名称未被占用）且选定地域在 `DescribeRegions` 返回列表内


## 概述

TCR 实例是镜像存储的容器 —— 每个实例有独立的域名、存储、访问策略。创建实例后在实例内创建命名空间和仓库。

| 选项 | 最佳场景 | 配额（NS / 仓库 / Helm / VPC） | 升级路径 |
|------|---------|:---:|---------|
| basic (基础版) | 个人开发/小团队入门 | 50 / 1000 / 1000 / 5 | 可升级为 standard |
| standard (标准版) | 中小团队生产 | 100 / 3000 / 3000 / 10 | 可升级为 premium |
| premium (高级版) | 大规模 / 完整同步与企业能力 | 500 / 5000 / 5000 / 20 | 当前最高规格 |

镜像与 Chart 数据落在关联 COS 桶，按 COS 用量计费（非上表「存储配额」）。**规格选择**: 初次使用从 `basic` 开始，后续按需升级。

操作是**异步**的: `CreateInstance` 返回 `RegistryId` 后实例进入 `Deploying`，须轮询 `DescribeInstanceStatus` 直到 `Status: "Running"` 才可使用（约 30-60 秒，见 [实例状态](../reference/states.md)）。

## 准备工作

### 环境检查

```bash
tccli --version
# expected: 3.1.124.1 或更高

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
| RegistryName | string | 是 | 控制台：5–50 字符，小写字母/数字/`-`，不能以 `-` 开头或结尾；创建后不可改 | `InvalidParameter.RegistryName` |
| RegistryType | string | 是 | `basic` / `standard` / `premium` | `InvalidParameter.RegistryType` |
| RegistryChargeType | integer | 否 | `0` 按量计费（API 默认）/ `1` 预付费（包年包月） | `InvalidParameter.RegistryChargeType` |
| DeletionProtection | boolean | 否 | `true` / `false`，默认 `false` | — |
| RegistryChargePrepaid | object | 仅预付费 | `Period` (月数) + `RenewFlag` | `InvalidParameter.RegistryChargePrepaid` |
| TagSpecification | object | 否 | Tags 数组 | `InvalidParameter.TagSpecification` |
| SyncTag | boolean | 否 | 是否同步 COS Tag | — |
| EnableCosMAZ | boolean | 否 | 是否开启 COS 多 AZ；购买页默认勾选并建议开启 | — |

## 操作步骤

### 步骤 1：决策 — 选实例规格

#### 为什么先选 basic

- **basic vs standard**: basic 配额 NS 50 / 仓库 1000，适合入门；standard 为 100 / 3000，更适合生产；`basic` 不支持跨实例自动同步
- **按量 vs 包年包月**: API 默认按量（`RegistryChargeType: 0`）；购买页建议长期使用优先包年包月（防欠费回收）。文档同时保留两种路径
- **规格选择**: `basic` + 按量 —— 熟悉后再升级规格或转预付费
- **能改吗?**: 规格可以升级 (basic→standard→premium)，不能降级。计费模式从按量可转包年包月，反之需退订

### 步骤 2：创建实例

`CreateInstance` 必传 `RegistryName` + `RegistryType`。按场景**二选一**：A 最小化（basic 按量，入门测试）或 B 增强（premium 包年包月+删除保护+COS 多 AZ，生产）。

> ⚠️ **A 与 B 是二选一变体，不是先做 A 再做 B**——两者各调一次 `CreateInstance` 会建**两个实例**。实例创建后改配置（规格升级/续费/存储扩容）用 `ModifyInstance`/`ModifyInstanceStorage`/`RenewInstance`（见本文"实例生命周期管理"段），**禁用第二次 `CreateInstance` 改配置**。规格可升不可降（basic→standard→premium）。

#### 选项 A：最小化（basic 按量，入门测试）

```bash
tccli tcr CreateInstance \
  --region ap-guangzhou \
  --RegistryName "<INSTANCE_NAME>" \
  --RegistryType basic
# expected: { "RegistryId": "tcr-xxxxxxxx", "RequestId": "..." }
```

| 占位符 | 含义 | 约束 | 如何获取 |
|------------|------|------|---------|
| `<INSTANCE_NAME>` | 实例名称 | 5–50 字符，小写字母/数字/`-`，全局唯一 | 自己定义 |

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
  --EnableCosMAZ true
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

> ⚠️ 删除 TCR 实例会**级联删除**实例内所有命名空间/仓库/镜像（数据永久丢失）；`RegistryChargeType=1` 预付费实例删除不退费；自定义域名与 VPC 链接同时删除。TCR Instance 无独立删除章，本段是唯一删除入口。

> 若需保留某些镜像，先 `tccli tcr DuplicateImage` 复制到其他实例（见 [推送拉取镜像](../images/push-pull.md)）。

```bash
# 确认子资源（命名空间/仓库）+ 查删除保护状态
tccli tcr DescribeNamespaces --region ap-guangzhou --RegistryId "<REGISTRY_ID>"
tccli tcr DescribeRepositories --region ap-guangzhou --RegistryId "<REGISTRY_ID>"
tccli tcr DescribeInstances --region ap-guangzhou --Registryids '["<REGISTRY_ID>"]' --filter "Registries[0].DeletionProtection"
# expected: 子资源列表 + DeletionProtection 值（true 时需先关）

# 关删除保护（仅 DeletionProtection=true 时）+ 删除 + 验证
tccli tcr ModifyInstance --region ap-guangzhou --RegistryId "<REGISTRY_ID>" --DeletionProtection false
# expected: exit 0

tccli tcr DeleteInstance --region ap-guangzhou --RegistryId "<REGISTRY_ID>"
# expected: exit 0

tccli tcr DescribeInstances --region ap-guangzhou --Registryids '["<REGISTRY_ID>"]'
# expected: { "TotalCount": 0, "Registries": [] } → 已删（tccli 默认剥离 Response 包装层）
```

> 若 DescribeInstances 仍返回实例但状态为 `Deleting`，属删除中（异步），稍候再查。Deleting 状态诊断见 [实例状态](../reference/states.md)。

> **Billing warning**: 按量计费实例删除即停止计费。包年包月实例提前删除**不退费**。

## 故障恢复

### 命令返回错误（exit ≠ 0）

| 现象 | 诊断 | 根因 | 修复 |
|---------|----------|------------|-----|
| `AuthFailure.SecretIdNotFound` | `tccli tcr DescribeRegions` | 凭证未配置 | 见 [配置凭证](../../getting-started/credentials.md) |
| `InvalidParameter.RegistryName` | 检查名称长度和字符 | 名称格式错误 | 使用 5–50 字符小写字母/数字/`-` |
| `ResourceInUse.RegistryNameExists` | `tccli tcr CheckInstanceName --RegistryName "<NAME>"` | 名称已被占用 | 换一个名称 |
| `LimitExceeded.InstanceQuota` | `tccli tcr DescribeInstances` | 实例数达上限 (默认 10) | 删除闲置实例或提工单 |
| `UnsupportedRegion` | `tccli tcr DescribeRegions` | 所选地域不支持 TCR | 换一个支持的地域 |

### 命令成功但状态不对（exit = 0）

| 现象 | 诊断 | 根因 | 修复 |
|---------|----------|------------|-----|
| `Status` 不是 `Running` | `tccli tcr DescribeInstanceStatus --RegistryIds '["<REGISTRY_ID>"]'` | 初始化未完成 | 轮询 `DescribeInstanceStatus` 直到 `Running`（创建是异步操作。注意 `DescribeInstanceStatus` 用 `--RegistryIds` 大写 D，与 `DescribeInstances` 的 `--Registryids` 小写 d 不同） |
| 无法 `docker login` | `tccli tcr ManageExternalEndpoint --RegistryId "<ID>" --Operation Open` | 公网访问未开启 | 开启公网访问端点 |
| 创建成功但未出现在列表 | 检查 `--region` 是否与创建时一致 | 查看的地域错误 | 切换到创建时的地域 |

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
# 衔接下一步前置：实例 Running 可进入创建命名空间/仓库（Verify 查字段存在，这里查能否进入下一阶段）
tccli tcr DescribeInstances --region ap-guangzhou --Registryids '["<REGISTRY_ID>"]' \
  --filter "Registries[0].{status:Status,name:RegistryName,type:RegistryType,protect:DeletionProtection}"
# expected: status="Running", name/type 与创建参数一致, protect=true

# 业务可用性边界：公网端点开启后才能 docker login（Verify 查 PublicDomain 格式，这里查端点实际状态=Opened）
tccli tcr DescribeExternalEndpointStatus --region ap-guangzhou --RegistryId "<REGISTRY_ID>"
# expected: Status: "Opened"  → 公网可达，可 docker login（未开启则 ManageExternalEndpoint --Operation Open）
```

> 实例 Running + 删除保护已开 + 公网端点 Opened = 创建闭环完成，可进入创建命名空间/仓库（`DescribeNamespaces` 须在 Running 后调用，Deploying 中调用返回空）。空实例无镜像，须先建命名空间才能 push。

---

## 下一步

- [管理实例访问](manage-access.md) — 开启公网/内网访问、创建 Token
- [创建命名空间和仓库](../repositories/manage.md) — 实例内首项操作
- [推送镜像](../images/push-pull.md) — docker push 你的第一个镜像
