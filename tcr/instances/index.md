---
doc_type: Overview
---
# 实例管理

> TCR 企业版实例的创建与访问配置。实例是 TCR 的顶级资源，承载命名空间、仓库、镜像。

## 是什么

TCR 实例是独立的镜像仓库服务后端。企业版实例有独立存储与访问域名，个人版是共享服务。实例创建后产生计费（按量计费），承载所有镜像数据。

## 触发条件

- 你要创建企业版镜像仓库实例（选 `RegistryType` 规格/地域/后端存储）— 用 `CreateInstance`，见 [创建实例](create.md)
- 你要配置 `docker login` 访问（**先内网 VPC，再按需公网**、Token、白名单）— 见 [访问管理](manage-access.md)
- 你要绑定自有域名 + 证书访问实例 — 见 [自定义域名](custom-domain.md)
- 你遇到实例状态非 `Running` 无法 push/pull — 看 [实例状态](#实例状态)

## 控制台运维面

控制台实例管理是**列表折叠面板**（不同于 TKE 集群的「点进实例 → 左侧栏多页」）：

| 面 | 控制台行为 | 文档落点 |
|:---|:---|:---|
| 列表展开 | 实例 ID / 计费 / 标签 / 访问域名 / 后端存储 / 创建时间 / 配额使用 / 监控 | 本页快速检查 + [配额](../reference/quotas.md) |
| 点实例名 | 跳转镜像仓库页并选中该实例 | [管理仓库](../repositories/manage.md) |
| 功能页上下文 | 各运维页顶部「地域 + 实例」下拉（跨仓库/命名空间/凭证/同步/版本保留） | 各 How-to 的 `--RegistryId` |
| 行「更多」 | 复制实例信息 / 按量转包年包月 / 调整实例规格 / 开启销毁保护 / 销毁退还 | [创建实例](create.md)（规格/计费）· 销毁保护见 `DeletionProtection` |

## 核心概念

| 概念 | 含义 | 为什么重要 |
|:-----|:-----|:-----|
| Registry | 实例，独立镜像仓库服务 | 顶级资源，决定访问域名与存储 |
| RegistryType | `basic` / `standard` / `premium` | 决定配额、功能与价格 |
| Status | 实例状态（Creating/Running/Deleting） | push/pull 前必须 Running |
| PublicDomain | 公网访问域名 | docker login/push/pull 的地址 |
| 访问凭证 | Token（Username + Token） | docker login 的用户名密码 |

## 企业版规格对比

| 规格 | 命名空间配额 | 仓库配额 | VPC 接入 | 特性 | 适用 |
|:-----|:-----------|:---------|:---------|:-----|:-----|
| basic（基础） | 50 | 1000 | 5 | 独立存储 + 访问控制 | 小团队、测试 |
| standard（标准） | 100 | 3000 | 10 | + 镜像部署阻断 | 中型企业 |
| premium（高级） | 500 | 5000 | 20 | + 签名验签 + 按需加载 | 大规模、安全敏感 |

> 个人版免费但限额（命名空间 **10**/地域；仓库 **广州 500 / 其他 100**；Tag **100**），无 SLA，不可调额，无独立后端。生产环境用企业版。官方权威表：[产品服务层级](https://cloud.tencent.com/document/product/1141/104731)；本仓数字与用量查询见 [配额与限制](../reference/quotas.md)。地域级企业版实例默认最多 **10** 个。

## 企业版 vs 个人版

| 维度 | 企业版 | 个人版 |
|:-----|:-----|:-----|
| 实例 | 独立实例，有状态机 | 共享服务，无实例生命周期 |
| 计费 | 按量计费 | 免费 |
| 访问 | 独立域名 + Token | 共享域名 + 腾讯云账号 |
| 配额 | 按规格分层 | 固定限额 |
| API | `CreateInstance` 等实例管理 API | 无实例管理 API |

> API 形态完全不同：企业版有实例/命名空间/仓库 CRUD，个人版只有命名空间/仓库操作（无实例概念）。两者文档分开，见 [个人版](../personal/index.md)。

## 实例状态

| 状态 | 含义 | 触发 | 用户可执行操作 |
|:-----|:-----|:-----|:--------------|
| Creating | 创建中 | `CreateInstance` | 等待（3-5 分钟） |
| Running | 正常运行 | 创建完成 | 全部操作 |
| Deleting | 删除中 | `DeleteInstance` | 等待 |

> 完整状态机见 [状态机参考](../reference/states.md)。push/pull 前必须确认 `Running`。

## 不适用场景

- 个人开发者、临时测试 → [个人版](../personal/index.md)（免费）
- 只需存几个镜像，不需要独立后端 → 个人版即可
- 需要镜像签名验签 → 必须企业版 premium

## 快速检查

```bash
# 查看账号下的 TCR 实例
tccli tcr DescribeInstances --region <REGION> \
  --filter "Registries[].{id:RegistryId,name:RegistryName,status:Status,type:RegistryType,domain:PublicDomain}"
# expected: 实例列表，status 含 Running
```

## 文档

- [创建实例](create.md) — 规格选择、地域、后端存储；配额见 [quotas](../reference/quotas.md) / 官方 104731
- [访问管理](manage-access.md) — **先内网后公网**、Token、白名单
- [自定义域名](custom-domain.md) — 绑定自有域名 + 证书
- [状态机](../reference/states.md) — 实例状态枚举（Creating/Running/Deleting）
- [错误码](../reference/error-codes.md) — docker login/push 失败码
- [配额与限制](../reference/quotas.md) — 规格配额与用量查询
- [个人版](../personal/index.md) — 免费个人版（API 形态不同）
