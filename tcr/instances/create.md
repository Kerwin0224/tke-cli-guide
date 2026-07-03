---
doc_type: How-to
subtype: 6A
fused: true
---
# 创建 TCR 实例

> 创建容器镜像服务 (TCR) 企业版实例，用于存储和管理 Docker/OCI 镜像。
> 控制台: [容器镜像服务 - 实例列表](https://console.cloud.tencent.com/tcr/instance) | page_id: `tcr-instance-create`

## 概述

TCR 实例是镜像存储的容器 —— 每个实例有独立的域名、存储、访问策略。创建实例后在实例内创建命名空间和仓库。

| 选项 | 最佳场景 | 关键限制 | 升级路径 |
|------|---------|:---:|---------|
| basic (基础版) | 个人开发/小团队 | 5 个仓库/NS，存储 100GB | 可升级为 standard |
| standard (标准版) | 中小团队生产 | 50 个仓库/NS，存储 500GB | 可升级为 premium |
| premium (高级版) | 企业级大规模使用 | 100 个仓库/NS，存储 1TB+ | 当前最高规格 |

**默认推荐**: basic —— 初次使用从基础版开始，后续按需升级。

操作是**同步**的: 创建命令返回即实例就绪（通常 < 30 秒）。

## 准备工作

### 环境检查

```bash
tccli --version
# expected: 版本号 (如 1.2.0+)

tccli tcr DescribeInstances --region ap-guangzhou
# expected: { "TotalCount": ..., "Registries": [...] }  → 凭证有效
```

### 资源检查

```bash
# 确认实例名未被占用
tccli tcr CheckInstanceName --region ap-guangzhou \
  --RegistryName "<INSTANCE_NAME>"
# expected: { "IsValid": true }  → 名称可用

# 确认地域支持 TCR
tccli tcr DescribeRegions
# expected: Regions 列表包含目标地域
```

## 关键字段

> 来源: `tccli tcr CreateInstance --generate-cli-skeleton`

| 字段 | 类型 | 必填 | 约束 | 填错时的错误 |
|-------|------|:--------:|------------|---------------|
| RegistryName | string | 是 | 2-30 字符，全局唯一 | `InvalidParameter.RegistryName` |
| RegistryType | string | 是 | `basic` / `standard` / `premium` | `InvalidParameter.RegistryType` |
| RegistryChargeType | integer | 否 | `1` (按量计费) / `2` (包年包月)，默认 `1` | `InvalidParameter.RegistryChargeType` |
| DeletionProtection | boolean | 否 | `true` / `false`，默认 `false` | — |
| RegistryChargePrepaid | object | 仅包年包月 | `Period` (月数) + `RenewFlag` | `InvalidParameter.RegistryChargePrepaid` |
| TagSpecification | object | 否 | Tags 数组 | `InvalidParameter.TagSpecification` |
| SyncTag | boolean | 否 | 是否同步 COS Tag | — |
| EnableCosMAZ | boolean | 否 | 是否开启 COS 多 AZ | — |

## 操作步骤

### 步骤 1：决策 — 选实例规格

#### 为什么先选 basic

- **basic vs standard**: basic 存储 100GB、5 仓库/NS，适合入门；standard 存储 500GB、50 仓库/NS，适合生产
- **按量 vs 包年包月**: 按量计费灵活但单价比包年包月高 ~30%；包年包月有折扣但退订复杂
- **默认推荐**: basic + 按量计费 —— 熟悉后再升级
- **能改吗?**: 规格可以升级 (basic→standard→premium)，不能降级。计费模式从按量可转包年包月，反之需退订

### 步骤 2：创建 — 最小化

```bash
tccli tcr CreateInstance \
  --region ap-guangzhou \
  --RegistryName "<INSTANCE_NAME>" \
  --RegistryType basic
# expected: { "RegistryId": "tcr-xxxxxxxx", "RequestId": "..." }
```

| 占位符 | 含义 | 约束 | 如何获取 |
|------------|------|------|---------|
| `<INSTANCE_NAME>` | 实例名称 | 2-30 字符，全局唯一，字母数字连字符下划线 | 自己定义 |

### 步骤 3：创建 — 增强：生产就绪

生产环境: premium 规格 + 包年包月 + 删除保护 + COS 多 AZ。

```bash
tccli tcr CreateInstance \
  --region ap-guangzhou \
  --RegistryName "<INSTANCE_NAME>" \
  --RegistryType premium \
  --RegistryChargeType 2 \
  --RegistryChargePrepaid '{
    "Period": 1,
    "RenewFlag": 1
  }' \
  --DeletionProtection true \
  --EnableCosMAZ true
# expected: { "RegistryId": "tcr-xxxxxxxx", "RequestId": "..." }
```

包年包月 1 个月 (`Period: 1`)，自动续费 (`RenewFlag: 1`)。COS 多 AZ 提升存储可靠性。

### 步骤 4：验证

```bash
tccli tcr DescribeInstances \
  --region ap-guangzhou \
  --RegistryIds '["<REGISTRY_ID>"]'
# expected: { "TotalCount": 1, "Registries": [{ "Status": "Running" }] }
```

| 维度 | 命令 | 预期 |
|-----------|---------|----------|
| Status | `DescribeInstances` | `Status: "Running"` |
| 实例名 | `DescribeInstances` → `Registries[].RegistryName` | 与创建参数一致 |
| 实例类型 | `DescribeInstances` → `Registries[].RegistryType` | 与创建参数一致 |
| 删除保护 | `DescribeInstances` → `Registries[].DeletionProtection` | 与创建参数一致 |
| 公网域名 | `DescribeInstances` → `Registries[].RegistryDomain` 格式 | `xxx.tencentcloudcr.com` |

## 清理

> ⚠️ **Side-effect 警告**: 删除 TCR 实例会:
> - 删除实例内**所有**命名空间、仓库和镜像 —— 数据永久丢失
> - `RegistryChargeType=2` (包年包月) 的实例删除不退费
> - 实例绑定的自定义域名、VPC 链接同时删除

### 1. 清理前检查

```bash
# 确认实例内的子资源（命名空间/仓库/镜像）
tccli tcr DescribeNamespaces --region ap-guangzhou --RegistryId "<REGISTRY_ID>"
tccli tcr DescribeRepositories --region ap-guangzhou --RegistryId "<REGISTRY_ID>"
# expected: 列出实例内命名空间与仓库（确认是否需保留镜像）
```

> 若需保留某些镜像，先 `tccli tcr DuplicateImage` 复制到其他实例（见 [推送拉取镜像](../images/push-pull.md)）。DeleteInstance 会**级联删除**实例内所有命名空间/仓库/镜像，不可恢复。

### 2. 关闭删除保护（先查后改）

```bash
# 先查 DeletionProtection 当前状态
tccli tcr DescribeInstances --region ap-guangzhou --RegistryIds '["<REGISTRY_ID>"]' \
  --filter "Registries[0].DeletionProtection"
# expected: true 或 false（仅 true 时才需关）

# 仅当上一步返回 true 时执行
tccli tcr ModifyInstance --region ap-guangzhou \
  --RegistryId "<REGISTRY_ID>" \
  --DeletionProtection false
# expected: exit 0
```

### 3. 删除

```bash
tccli tcr DeleteInstance --region ap-guangzhou --RegistryId "<REGISTRY_ID>"
# expected: exit 0
```

### 4. 验证已删除

```bash
tccli tcr DescribeInstances --region ap-guangzhou --RegistryIds '["<REGISTRY_ID>"]'
# expected: { "TotalCount": 0, "Registries": [] }（tccli 默认剥离 Response 包装层）
```

> 若 DescribeInstances 仍返回实例但状态为 `Deleting`，属删除中（异步），稍候再查。Deleting 状态诊断见 [实例状态](../reference/states.md)。

> **Billing warning**: 按量计费实例删除即停止计费。包年包月实例提前删除**不退费**。

## 故障恢复

### 命令返回错误（exit ≠ 0）

| 现象 | 诊断 | 根因 | 修复 |
|---------|----------|------------|-----|
| `AuthFailure.SecretIdNotFound` | `tccli tcr DescribeRegions` | 凭证未配置 | 见 [配置凭证](../../getting-started/credentials.md) |
| `InvalidParameter.RegistryName` | 检查名称长度和字符 | 名称格式错误 | 使用 2-30 字符字母数字连字符下划线 |
| `ResourceInUse.RegistryNameExists` | `tccli tcr CheckInstanceName --RegistryName "<NAME>"` | 名称已被占用 | 换一个名称 |
| `LimitExceeded.InstanceQuota` | `tccli tcr DescribeInstances` | 实例数达上限 (默认 10) | 删除闲置实例或提工单 |
| `UnsupportedRegion` | `tccli tcr DescribeRegions` | 所选地域不支持 TCR | 换一个支持的地域 |

### 命令成功但状态不对（exit = 0）

| 现象 | 诊断 | 根因 | 修复 |
|---------|----------|------------|-----|
| `Status` 不是 `Running` | `tccli tcr DescribeInstanceStatus --RegistryId "<ID>"` | 初始化未完成 | 等待 30 秒后重试查询 |
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

## 下一步

- [管理实例访问](manage-access.md) — 开启公网/内网访问、创建 Token
- [创建命名空间和仓库](../repositories/manage.md) — 实例内的第一步
- [推送镜像](../images/push-pull.md) — docker push 你的第一个镜像

## 控制台替代方案

[TCR 控制台 - 创建实例](https://console.cloud.tencent.com/tcr/instance/create)
