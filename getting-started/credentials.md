---
doc_type: How-to
subtype: 6A
fused: false
---
# 配置 TCCLI 凭证

> 获取腾讯云 CAM 凭证（SecretId/SecretKey）并配置到 TCCLI，让 TCCLI 能调用 TKE/TCR API。这是使用本指南所有命令的**唯一前置**——未配置凭证时，第一条 API 调用返回 `AuthFailure.SecretIdNotFound`。
> 控制台: [访问管理 CAM](https://console.cloud.tencent.com/cam)

> ⚠️ **控制台是固有边界**：CAM 根凭证（SecretId/SecretKey）的**首次获取**须经腾讯云控制台/浏览器——腾讯云不允许用 TCCLI 自举创建 API 密钥（`tccli auth login` 也触发浏览器登录）。本指南定位是"TCCLI 操作手册"，凭证首次获取这一步**必须**经控制台一次性操作，无法纯 CLI 闭环。这是腾讯云的固有边界，非文档缺陷。配好凭证后，后续所有 TKE/TCR 操作均可纯 CLI 完成。

> **前置**：你需有一个腾讯云账号。若没有，先在 [腾讯云首页](https://cloud.tencent.com/) 注册（注册需浏览器+手机验证，属一次性操作）。

## 概述

TCCLI 调用腾讯云 API 需要凭证。三类凭证作用域不同，本文只管第一类（CAM 根凭证，全局前置）；后两类在各自产品文档里配置：

| 凭证类型 | 作用 | 作用域 | 归属文档 |
|:---------|:-----|:-------|:---------|
| **CAM 根凭证**（SecretId/SecretKey） | 让 TCCLI 能调用腾讯云 API | 全局前置（产品之上） | 本文 |
| kubeconfig | kubectl 连 TKE 集群 | TKE 产品内 | [TKE 集群认证](../tke/security/auth.md) |

> 本文解决"TCCLI 怎么配凭证"。配好后所有 `tccli tke ...` / `tccli tcr ...` 命令才能工作。

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
2. **子账号路径（推荐）**: 用户 → 用户列表 → 新建用户（子用户）→ 授权策略 `QcloudTKEFullAccess` + `QcloudTCRFullAccess` → 该用户 API 密钥 → 新建密钥
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

> `tccli auth login` 触发 CAM 登录获取临时凭证，用系统默认浏览器打开授权页（源码 `login.py` 调 `webbrowser.open`，无 `--browser` 参数可选）。具体交互行为（浏览器/扫码）因环境而异——首次使用建议直接运行观察提示。临时凭证会过期，过期后重新 `tccli auth login`。适合本地交互、不想长期存密钥。

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

多 profile 验证（查看已配置的 profile 与 region）：

```bash
tccli configure list
# expected: 列出各 profile 的 region/output，凭证字段（secretId/secretKey）明文显示
```

> ⚠️ **安全红线**：`tccli configure list` **明文打印 secretId/secretKey**（源码 `configure.py` 的 `ConfigureListCommand._run_main` 直接输出 `cred[config]` 原值，未做脱敏；命令 `--help` 示例里的 `****` 仅为文档演示，非实际行为）。禁止在共享环境、截图、工单、会话记录中运行该命令；共享测试账号下严禁运行任何可能暴露凭证的命令。仅在自己的隔离终端核查 profile 配置时使用。

## 故障恢复

| 现象 | 诊断命令 | 根因 | 修复 |
|:-----|:---------|:-----|:-----|
| `AuthFailure.SecretIdNotFound` | `tccli configure list` 查 secretId 是否为空 | 凭证未配置或已过期 | `tccli configure` 重新配置 |
| `AuthFailure.SignatureFailure` | 检查 SecretKey 是否复制完整（含首尾空格） | SecretKey 错误 | 重新从 CAM 复制 SecretKey |
| `UnauthorizedOperation.CamNoAuth` | 查子账号授权策略 | 子账号无 TKE/TCR 权限 | CAM 控制台授权 `QcloudTKEFullAccess`/`QcloudTCRFullAccess` |
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
# 跨产品端到端：凭证对 TKE 和 TCR 两个产品域均生效（验证段仅验证 TKE 域）
tccli tcr DescribeRegions --filter "TotalCount" --output text
# expected: 非零数字（如 27）→ 凭证对 TCR 域同样可达，跨产品配置闭环完成

# profile 核查：确认当前用的是哪个 profile（避免误用主账号密钥）
tccli configure list | grep -E "^region|^output" 
# expected: region/output 行明文显示（凭证段不 grep，避免密钥入终端历史）
```

> TKE + TCR 双域均可调用 + 默认 profile 仅子账号密钥 = 凭证配置闭环完成，可进入任意 TKE/TCR 操作。

## 下一步

- [安装 TCCLI](install.md) — 若尚未安装
- [TKE 快速入门](../quickstart/tke-first-cluster.md) — 凭证配好后创建第一个集群
- [术语表](glossary.md) — VPC/CIDR/CAM 等术语释义
