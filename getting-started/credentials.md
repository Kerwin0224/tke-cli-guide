---
doc_type: How-to
subtype: 6A
fused: false
---
# 配置 TCCLI 凭证

> 获取腾讯云 CAM 凭证（SecretId/SecretKey）并配置到 TCCLI，让 TCCLI 能调用 TKE API。这是使用本指南所有命令的**唯一前置**——未配置凭证时，第一条 API 调用返回 `AuthFailure.SecretIdNotFound`。
> 控制台: [访问管理 CAM](https://console.cloud.tencent.com/cam)

> ⚠️ **控制台是固有边界**：CAM 根凭证（SecretId/SecretKey）的**首次获取**须经腾讯云控制台/浏览器——腾讯云不允许用 TCCLI 自举创建 API 密钥（`tccli auth login` 也触发浏览器登录）。本指南定位是"TCCLI 操作手册"，凭证首次获取这一步**必须**经控制台一次性操作，无法纯 CLI 闭环。这是腾讯云的固有边界，非文档缺陷。配好凭证后，后续所有 TKE 操作均可纯 CLI 完成。

> **前置**：你需有一个腾讯云账号。若没有，先在 [腾讯云首页](https://cloud.tencent.com/) 注册（注册需浏览器+手机验证，属一次性操作）。

## 概述

TCCLI 调用腾讯云 API 需要凭证。两类凭证作用域不同，本文只管第一类（CAM 根凭证，全局前置）；kubeconfig 在 TKE 安全文档里配置：

| 凭证类型 | 作用 | 作用域 | 归属文档 |
|:---------|:-----|:-------|:---------|
| **CAM 根凭证**（SecretId/SecretKey） | 让 TCCLI 能调用腾讯云 API | 全局前置（产品之上） | 本文 |
| kubeconfig | kubectl 连 TKE 集群 | TKE 产品内 | [TKE 集群认证](../tke/security/auth.md) |

> 本文解决"TCCLI 怎么配凭证"。配好后所有 `tccli tke ...` 命令才能工作。

## 触发条件

- 任意 `tccli <service> <Action>` 返回 `AuthFailure.SecretIdNotFound`（secretId is invalid）— 凭证未配或已失效，用本文配置
- 终端执行 `tccli configure list` 显示 secretId/secretKey 为空但调用报 `AuthFailure` — 凭证未配或已失效

## 决策依据

#### 主账号 vs 子账号

| 选项 | 最佳场景 | 风险 |
|:-----|:---------|:-----|
| 主账号 API 密钥 | 个人测试、快速验证 | 高（主账号密钥泄露=全部资源失守） |
| **子账号 API 密钥**（推荐） | 生产、CI/CD、团队 | 低（最小权限，可吊销） |

**默认推荐**: 子账号。主账号密钥只在隔离的测试环境用，且用完即删。

#### `tccli configure` vs `tccli auth login`

| 方式 | 机制 | 适用 |
|:-----|:-----|:-----|
| `tccli configure` | 交互式手填 SecretId/SecretKey/region/output | 已有 API 密钥、CI/CD（可 `tccli configure set` 非交互） |
| `tccli auth login` | CAM 登录获取临时凭证（浏览器交互） | 交互式登录、不想长期存密钥 |

> 两种方式二选一。`tccli auth login` 适合本地交互；`tccli configure` 适合自动化。本文以 `tccli configure` 为主（可复现、可脚本化）。

## 准备工作

### 1. 安装 TCCLI

若未安装，见 [安装 TCCLI](install.md)。验证：

```bash
tccli --version
# expected: 最新版本或更高
```

> 版本过低会缺新接口或字段名不一致。升级：`uv tool upgrade tccli`（uv 管理的 TCCLI）；非 uv 安装见 [安装 TCCLI](install.md)。

### 2. 获取 CAM API 密钥

在腾讯云控制台获取 SecretId/SecretKey：

1. 登录 [访问管理 CAM 控制台](https://console.cloud.tencent.com/cam)
2. **子账号路径（推荐）**: 用户 → 用户列表 → 新建用户（子用户）→ 授权策略 `QcloudTKEFullAccess` → 该用户 API 密钥 → 新建密钥
3. **主账号路径（仅测试）**: 访问管理 → API 密钥管理 → 新建密钥

| 值 | 含义 | 获取方式 |
|:---|:-----|:---------|
| SecretId | 密钥 ID（可公开标识） | CAM 控制台新建密钥 |
| SecretKey | 密钥（**切勿泄露**） | CAM 控制台新建密钥（仅创建时显示） |

> ⚠️ SecretKey 仅在创建时完整显示一次，须立即保存。泄露后须在 CAM 吊销重建。

## 操作步骤

### 方式 A: `tccli configure`（交互式，推荐首次使用）

```bash
tccli configure
# 交互式输入四项（方括号内为当前 profile 的默认值，新环境为 None）：
# TencentCloud API secretId[None]: <SECRET_ID>
# TencentCloud API secretKey[None]: <SECRET_KEY>
# Default region name[None]: <REGION>
# Default output format[json]: json
# expected: exit 0，凭证写入 ~/.tccli/default.credential，配置写入 ~/.tccli/default.configure
```

| 占位符 | 含义 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<SECRET_ID>` | CAM 密钥 ID | 以 `AKID` 开头 | CAM 控制台新建密钥 |
| `<SECRET_KEY>` | CAM 密钥 | 32 字符字符串 | CAM 控制台新建密钥 |
| `<REGION>` | 默认地域 | 如 `ap-guangzhou` | `tccli tke DescribeRegions` 查看 `RegionName` |

### 方式 B: `tccli configure set`（非交互，CI/CD）

```bash
tccli configure set secretId <SECRET_ID> --profile <PROFILE_NAME>
tccli configure set secretKey <SECRET_KEY> --profile <PROFILE_NAME>
tccli configure set region <REGION> --profile <PROFILE_NAME>
tccli configure set output json --profile <PROFILE_NAME>
# expected: exit 0
```

| 占位符 | 含义 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<PROFILE_NAME>` | 配置文件名 | 字母数字，如 `prod`/`dev` | 自定义 |

> `--profile` 支持多账号：不同 profile 存不同子账号凭证，用 `tccli --profile <NAME> <command>` 切换。

### 方式 C: `tccli auth login`（CAM 登录，临时凭证）

```bash
tccli auth login
# expected: exit 0，触发 CAM 登录流程（浏览器交互，仅接受 --profile 参数）
```

> `tccli auth login` 触发 CAM 登录获取临时凭证，用系统默认浏览器打开授权页（无 `--browser` 参数可选）。具体交互行为（浏览器/扫码）因环境而异——首次使用建议直接运行观察提示。临时凭证会过期，过期后重新 `tccli auth login`。适合本地交互、不想长期存密钥。

## 验证

凭证配置后，用轻量只读命令验证（不需预选地域，EXIT 0 即凭证有效）：

```bash
tccli tke DescribeRegions
# expected: exit 0，返回 TotalCount 与 RegionInstanceSet，无 Error 字段
```
```json
{
    "TotalCount": 42,
    "RegionInstanceSet": [
        {
            "RegionName": "ap-guangzhou",
            "RegionId": 1,
            "Status": "alluser"
        }
    ]
}
```

> ⚠️ **不要用 `tccli auth verify`**——该子命令**不存在**（`tccli auth` 仅有 login/logout/help，执行 verify 返回 exit 252 Invalid choice）。凭证验证用 `tccli tke DescribeRegions` 这类轻量只读调用。

## 服务角色（TKE / IPAMD / AS / 可观测）

> CAM **用户密钥**（SecretId/SecretKey）让 **你** 调 API；**服务角色**让 **云产品** 代你访问其他产品（TKE→CVM/CLB/CBS，IPAMD→ENI，节点池→AS，Prometheus 等）。密钥配好后仍可能因缺服务角色失败——错误码与用户侧 `CamNoAuth` 不同。  
> 官方角色/策略语义以 [服务授权相关角色权限说明](https://cloud.tencent.com/document/product/457/43416) 为准；下列 CLI 为 agent 可执行探测与补齐路径。

### 角色总表（按任务触发）

| 角色 | 服务载体（Principal） | 典型触发 | 缺了的表现 | 详文 |
|:-----|:----------------------|:---------|:-----------|:-----|
| `TKE_QCSRole` | `ccs.qcloud.com` | 首次建集群 / 代管 CVM·CLB·CBS | 创建中途失败 | 下节 + [创建集群](../tke/clusters/create.md) |
| `IPAMDofTKE_QCSRole` | `ccs.qcloud.com` | **VPC-CNI** 建集群 / 开 IPAMD | ENI/Pod IP 分配失败 | [VPC-CNI](../tke/networking/vpc-cni.md#ipamd-服务角色) |
| `AS_QCSRole` | `as.cloud.tencent.com` | `CreateClusterNodePool` / AS 节点池 | `UnauthorizedOperation.AutoScalingRoleUnauthorized` | [创建节点池](../tke/nodes/nodepool-create.md#as-服务角色节点池创建前) |
| `TKE_QCSLinkedRoleInPrometheusService` | 服务相关角色 | 首次 Prometheus 监控 | 建/关联监控失败 | [Prometheus](../tke/observability/prometheus.md#服务相关角色) |

`TKE_QCSRole` 上常挂的**策略**（功能级，角色已在仍可能缺策略）：

| 预设策略 | 触发功能 | 官方/说明 |
|:---------|:---------|:----------|
| `QcloudAccessForTKERole` | 默认：操作 CVM/CLB/CBS 等 | 43416 默认关联 |
| `QcloudAccessForTKERoleInOpsManagement` | 日志等运维 | 43416，常与主策略同开 |
| `QcloudAccessForTKERoleInCreatingCFSStorageclass` | CFS 扩展组件 | 首次装 CFS；[插件管理](../tke/addons/manage.md#cfs-服务授权) |
| `QcloudCVMFinanceAccess` | 包年包月云盘 PVC | 预付费盘支付权限 |
| `QcloudAccessFortkeRoleInMetricsbyLog` 等 | 成本洞察等扩展 | 功能页要求时再挂 |

> **不要**把用户策略 `QcloudTKEFullAccess` 与服务角色 `TKE_QCSRole` 混为一谈：前者授权给**用户**，后者授权给**服务**。

### 统一探测

```bash
# 列出 TKE / IPAMD / AS / Prometheus 相关角色
tccli cam DescribeRoleList --Page 1 --Rp 100 \
  --filter "List[?contains(RoleName,'TKE') || contains(RoleName,'IPAMD') || RoleName=='AS_QCSRole' || contains(RoleName,'Prometheus')].RoleName" \
  --output text
# expected: 已用过 TKE 时常含 TKE_QCSRole；VPC-CNI 前须含 IPAMDofTKE_QCSRole；节点池前须含 AS_QCSRole

# 查某角色已挂策略（例：TKE 主角色）
tccli cam ListAttachedRolePolicies --Page 1 --Rp 50 --RoleName TKE_QCSRole \
  --filter "List[].PolicyName" --output text
# expected: 至少含 QcloudAccessForTKERole（名称以实际返回为准）
```

| 任务 | 探测期望（角色名须在列表中） |
|:-----|:---------------------------|
| 任意建集群 | `TKE_QCSRole` |
| VPC-CNI / quickstart 默认网络 | `TKE_QCSRole` + `IPAMDofTKE_QCSRole` |
| 节点池 | 上列 + `AS_QCSRole` |
| Prometheus | `TKE_QCSLinkedRoleInPrometheusService` 或官方当前 Linked 角色名 |

### 补 TKE_QCSRole（主服务角色）

> **推荐主路径**：首次登录 [容器服务控制台](https://console.cloud.tencent.com/tke2) 弹窗 **同意授权**（官方 43416）。  
> **CLI 等价**（子账号须有 `cam:CreateRole` / `cam:AttachRolePolicy`；无 CAM 写权限则只能控制台主账号授权）：

```bash
# 1) 创建角色（Principal 必须是 ccs.qcloud.com）
tccli cam CreateRole \
  --RoleName TKE_QCSRole \
  --Description "TKE service role for accessing CVM CLB CBS and related resources" \
  --PolicyDocument '{"version":"2.0","statement":[{"effect":"allow","action":"sts:AssumeRole","principal":{"service":"ccs.qcloud.com"}}]}'
# expected: RoleId；角色已存在则跳过（Error 含已存在/Role 相关则复验 List）

# 2) 挂默认访问策略（建集群最低）
tccli cam AttachRolePolicy \
  --AttachRoleName TKE_QCSRole \
  --PolicyName QcloudAccessForTKERole
# expected: RequestId

# 3) 运维/日志（与控制台默认同批时常需要）
tccli cam AttachRolePolicy \
  --AttachRoleName TKE_QCSRole \
  --PolicyName QcloudAccessForTKERoleInOpsManagement
# expected: RequestId
```

### 补 IPAMDofTKE_QCSRole（VPC-CNI 前置）

> 首次使用 **VPC-CNI** 时官方要求授权 IPAMD，见 43416「IPAMDofTKE_QCSRole」。quickstart 默认 VPC-CNI，**建集群前应探测**。

```bash
tccli cam CreateRole \
  --RoleName IPAMDofTKE_QCSRole \
  --Description "TKE IPAMD service role for ENI and VPC resources" \
  --PolicyDocument '{"version":"2.0","statement":[{"effect":"allow","action":"sts:AssumeRole","principal":{"service":"ccs.qcloud.com"}}]}'
# expected: RoleId；已存在则跳过

tccli cam AttachRolePolicy \
  --AttachRoleName IPAMDofTKE_QCSRole \
  --PolicyName QcloudAccessForIPAMDofTKERole
# expected: RequestId

# 可选：弹性 IP 相关（部分账号控制台会挂）
# tccli cam AttachRolePolicy --AttachRoleName IPAMDofTKE_QCSRole --PolicyName QcloudAccessForIPAMDRoleInQcloudAllocateEIP
```

### 补 AS_QCSRole（节点池前置）

> 完整步骤见 [创建节点池 — AS 服务角色](../tke/nodes/nodepool-create.md#as-服务角色节点池创建前)。最短：

```bash
tccli cam CreateRole \
  --RoleName AS_QCSRole \
  --Description "Auto Scaling service role for TKE node pools" \
  --PolicyDocument '{"version":"2.0","statement":[{"action":"name/sts:AssumeRole","effect":"allow","principal":{"service":"as.cloud.tencent.com"}}]}'
# expected: RoleId；已存在则跳过

tccli cam AttachRolePolicy \
  --AttachRoleName AS_QCSRole \
  --PolicyName QcloudAccessForASRole
# expected: RequestId
```

### 服务相关角色（LinkedRole，如 Prometheus）

部分功能用 **服务相关角色**（非手写 `PolicyDocument`），用 `CreateServiceLinkedRole`：

```bash
# QCSServiceName 以官方产品页 / cam 角色载体文档为准；Prometheus 示例载体名以控制台授权页为准
tccli cam CreateServiceLinkedRole help --detail
# expected: 入参含 QCSServiceName[]

# 探测是否已有 Prometheus 相关 Linked 角色
tccli cam DescribeRoleList --Page 1 --Rp 100 \
  --filter "List[?contains(RoleName,'Prometheus')].RoleName" --output text
# expected: 含 TKE_QCSLinkedRoleInPrometheusService 或官方当前名；空则走控制台 Prometheus 首次授权或 CreateServiceLinkedRole
```

> `CreateServiceLinkedRole` 的 `QCSServiceName` 必须与 [角色载体](https://cloud.tencent.com/document/product/598/85165) 一致；不确定时用控制台该功能首次「服务授权」更稳。

### 功能策略补挂（角色已在、功能仍失败）

```bash
# CFS 组件
tccli cam AttachRolePolicy --AttachRoleName TKE_QCSRole \
  --PolicyName QcloudAccessForTKERoleInCreatingCFSStorageclass
# expected: RequestId

# 包年包月云盘支付
tccli cam AttachRolePolicy --AttachRoleName TKE_QCSRole \
  --PolicyName QcloudCVMFinanceAccess
# expected: RequestId
```

### 边界

- **控制台一次授权**：首次进 TKE 控制台「同意授权」仍是官方主路径（尤其主账号引导创建多策略）。
- **子账号**：可能无 `cam:CreateRole`，CLI 补角色会失败 → 主账号控制台授权或提升 CAM 权限。
- **策略名大小写**：以 `ListPolicies` / 控制台为准；账号侧可见 `QcloudAccessFortkeRoleInMetricsbyLog` 与 `QcloudAccessForTKERole` 混用大小写，挂载失败时用 `tccli cam ListPolicies` 搜精确名。

多 profile 验证（查看已配置的 profile 与 region）：

```bash
tccli configure list
# expected: 列出各 profile 的 region/output，凭证字段（secretId/secretKey）明文显示
```

> ⚠️ **安全红线**：`tccli configure list` **明文打印 secretId/secretKey**（不做脱敏；命令 `--help` 示例里的 `****` 仅为文档演示，非实际行为）。禁止在共享环境、截图、工单、会话记录中运行该命令；共享测试账号下严禁运行任何可能暴露凭证的命令。仅在自己的隔离终端核查 profile 配置时使用。

## 故障恢复

| 现象 | 诊断命令 | 根因 | 修复 |
|:-----|:---------|:-----|:-----|
| `AuthFailure.SecretIdNotFound` | `tccli configure list` 查 secretId 是否为空 | 凭证未配置或已过期 | `tccli configure` 重新配置 |
| `AuthFailure.SignatureFailure` | 检查 SecretKey 是否复制完整（含首尾空格） | SecretKey 错误 | 重新从 CAM 复制 SecretKey |
| `UnauthorizedOperation.CamNoAuth` | 查子账号授权策略 | 子账号无 TKE 权限 | CAM 控制台授权 `QcloudTKEFullAccess` |
| `UnauthorizedOperation.AutoScalingRoleUnauthorized` | `DescribeRoleList` 查 `AS_QCSRole` | 缺弹性伸缩服务角色 | [补 AS](#补-as_qcsrole节点池前置) → [创建节点池](../tke/nodes/nodepool-create.md#as-服务角色节点池创建前) |
| 创建集群中途失败（服务未授权） | `DescribeRoleList` 查 `TKE_QCSRole` | 缺 TKE 服务角色 | [补 TKE_QCSRole](#补-tke_qcsrole主服务角色) 或控制台服务授权 |
| VPC-CNI 创建/ENI 失败 | `DescribeRoleList` 查 `IPAMDofTKE_QCSRole` | 缺 IPAMD 服务角色 | [补 IPAMD](#补-ipamdoftke_qcsrolevpc-cni-前置) → [VPC-CNI](../tke/networking/vpc-cni.md#ipamd-服务角色) |
| `cam CreateRole` / `AttachRolePolicy` 被拒 | 查子账号 CAM | 无角色管理权限 | 主账号控制台授权，或给子账号 `cam:CreateRole`/`cam:AttachRolePolicy` |
| 地域相关报错（如 `UnsupportedRegion`/`InvalidRegion`） | `tccli tke DescribeRegions` 看地域状态 | region 不支持该产品 | 换 `ap-guangzhou`/`ap-shanghai` 等主地域 |
| 命令卡住无响应 | 检查网络/代理 | 防火墙拦截腾讯云 API | 配置 `--https-proxy` 或开放 `*.tencentcloudapi.com` |

## 清理

凭证本身不删除（删除=TCCLI 无法工作）。如需删除某 profile 的本地配置：

```bash
# 删除指定 profile 的本地配置（profile 须存在，否则报 "profile not exist"）
tccli configure remove --profile <PROFILE_NAME>
# expected: exit 0
```

| 占位符 | 含义 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<PROFILE_NAME>` | 要删除的 profile 名 | 须已存在 | `tccli configure list` 查看 |

> ⚠️ `tccli configure remove` 只删本地配置文件，**不吊销云端 API 密钥**。吊销密钥须在 [CAM 控制台](https://console.cloud.tencent.com/cam/capi) 操作（禁用或删除对应密钥）——这是不可逆的安全操作，删除后用该密钥的所有调用立即失败。

## 收尾确认

```bash
# TKE 域再确认（与验证段互补：filter 取 TotalCount）
tccli tke DescribeRegions --filter "TotalCount" --output text
# expected: 非零数字 → 凭证对 TKE 域可达

# profile 核查：确认当前用的是哪个 profile（避免误用主账号密钥）
tccli configure list | grep -E "^region|^output" 
# expected: region/output 行明文显示（凭证段不 grep，避免密钥入终端历史）
```

> TKE 域可调用 + 默认 profile 仅子账号密钥 = 凭证配置闭环完成，可进入 TKE 操作。

## 下一步

- [安装 TCCLI](install.md) — 若尚未安装
- [TKE 快速入门](../quickstart/tke-first-cluster.md) — 凭证配好后创建第一个集群
- [术语表](glossary.md) — VPC/CIDR/CAM 等术语释义
