---
doc_type: Reference
subtype: 8C
---
# TCR 配额与限制

> 企业版按实例规格（basic / standard / premium）区分配额；个人版为共享服务、另有限额。数值以官方「产品服务层级与容量限制」为准，调额后以官网最新表为准。超限时常见 `LimitExceeded` / `FailedOperation.QuotaExceed`（以实际 `Error.Code` 为准）。

## 企业版实例配额（按规格）

| 限制项 | 基础版 (basic) | 标准版 (standard) | 高级版 (premium) | 作用域 | 超限信号 |
|:-------|:-------------:|:-----------------:|:----------------:|:-------|:---------|
| 命名空间 | **50** | **100** | **500** | 每实例 | `LimitExceeded` |
| 镜像仓库 | **1000** | **3000** | **5000** | 每实例 | `LimitExceeded` |
| 镜像版本 (tag) | 无限制 | 无限制 | 无限制 | 每仓库 | — |
| VPC 接入数 | **5** | **10** | **20** | 每实例 | `LimitExceeded` |
| 服务级账号 | **10** | **20** | **100** | 每实例 | `LimitExceeded` |
| Helm 仓库 | **1000** | **3000** | **5000** | 每实例 | `LimitExceeded` |
| 地域级独立实例 | 默认 **10**（各规格同） | 同左 | 同左 | 每地域每账号 | `LimitExceeded` |

> 规格在创建时由 `RegistryType` 选定（`basic` / `standard` / `premium`），创建后升配见官方购买/变配说明。功能差异（签名验签、按需加载、多地域复制等）随规格开放，见官方功能差异表——本页只列**可判定配额数字**。

## 个人版限额

个人版无独立企业版实例，限额由 `DescribeUserQuotaPersonal` 返回（账号级）。`DescribeUserQuotaPersonal` 返回字段 `Data.LimitInfo[].Type` / `Value`：

| Type | 含义 | 示例账号 Value（以 `DescribeUserQuotaPersonal` 实时返回为准） |
|:-----|:-----|:---------------------:|
| `namespace` | 命名空间上限 | 10 |
| `repo` | 仓库上限 | 500 |
| `tag` | 版本上限 | 100 |
| `trigger` | 触发器上限 | 10 |

> 个人版适合临时测试；生产与独立存储/SLA 选企业版。控制台入口已与企业版合并，API 面仍用 `*Personal` Action。

## 查询配额用量

```bash
# 企业版：本地域实例数（对照「地域级默认 10」）
tccli tcr DescribeInstances --region <REGION> --output text --filter "TotalCount"
# expected: 数字（如 1）≤ 地域实例上限

# 企业版：某实例命名空间存量（对照规格表；响应顶层 TotalCount，与 NamespaceList 同级）
tccli tcr DescribeNamespaces --region <REGION> --RegistryId "<REGISTRY_ID>" \
  --output text --filter "TotalCount"
# expected: 数字 ≤ 该规格命名空间配额（basic=50）

# 企业版：某实例仓库存量（顶层 TotalCount，与 RepositoryList 同级）
tccli tcr DescribeRepositories --region <REGION> --RegistryId "<REGISTRY_ID>" \
  --output text --filter "TotalCount"
# expected: 数字 ≤ 该规格仓库配额（basic=1000）

# 企业版：服务级账号存量（顶层 TotalCount）
tccli tcr DescribeServiceAccounts --region <REGION> --RegistryId "<REGISTRY_ID>" \
  --output text --filter "TotalCount"
# expected: 数字 ≤ 该规格 SA 配额（basic=10）；无 SA 时为 0

# 个人版：账号限额表
tccli tcr DescribeUserQuotaPersonal --output json
# expected: Data.LimitInfo[] 含 Type=namespace/repo/tag/trigger 与 Value
```

| 占位符 | 含义 | 获取方式 |
|:-------|:-----|:---------|
| `<REGION>` | 地域 | `tccli tcr DescribeRegions`（如 `ap-guangzhou`） |
| `<REGISTRY_ID>` | 企业版实例 ID | `tccli tcr DescribeInstances --region <REGION> --filter "Registries[].{id:RegistryId,name:RegistryName}"` |

## API 限频

> 腾讯云 API 默认约 **10 次/秒**（部分接口更低）。批量 Describe/推送脚本须串行或加间隔，超限见 `RequestLimitExceeded`。

| 场景 | 建议 |
|:-----|:-----|
| 列举实例/命名空间/仓库 | `--filter` + `--output text`，避免反复全量 JSON |
| 批量打 tag / 同步 | 控制并发；同步任务并发受规格限制（标准/高级约最大 10） |

## 下一步

- [创建实例](../instances/create.md) — `RegistryType` 选型与删除保护
- [访问控制](../access/manage.md) — VPC 接入 / 服务级账号占用配额
- [个人版](../personal/index.md) — 个人版限额与企业版差异
- [故障排查](../troubleshooting.md) — 配额类错误码汇总
