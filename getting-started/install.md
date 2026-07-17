---
doc_type: How-to
---
# 安装 TCCLI

> 安装腾讯云命令行工具 TCCLI，本指南所有命令的前置。已安装可跳到 [配置凭证](credentials.md)。
>
> 官方文档：[TCCLI 安装](https://cloud.tencent.com/document/product/440/6176)

## 概述

TCCLI 是腾讯云 API 的命令行客户端，用 Python 写成。本指南统一用 **uv** 安装与管理——uv 是一个跨平台的 Python 包管理器，自动为 TCCLI 建独立环境，不污染系统 Python，不会与系统包冲突。

本指南覆盖 TKE（容器服务）和 TCR（容器镜像服务）两个产品的 TCCLI 操作。

## 触发条件

- 终端执行 `tccli --version` 报 `command not found`，或版本过低（缺新接口/字段名不一致） — 用本文安装或升级
- 终端执行任意 `tccli tke`/`tccli tcr` 命令报 `command not found: tccli` — TCCLI 未装，用本文安装

## 决策依据

#### 为什么用 uv

| 需求 | uv 如何满足 |
|:-----|:-----|
| 跨平台 | macOS / Linux / Windows 同一套工具，安装命令一致 |
| 不污染系统 Python | `uv tool install` 把 TCCLI 装进独立环境，只暴露一个 `tccli` 命令到 PATH |
| 规避系统包冲突 | 现代 macOS / 部分 Linux 的系统 Python 受保护（PEP 668），直接 `pip install` 会被拒；uv 自带独立环境，不受此限制 |
| 升级与卸载干净 | `uv tool upgrade tccli` / `uv tool uninstall tccli` 一条命令，无残留 |

> uv 可在任何机器上用同一条命令装好 TCCLI，且不影响系统 Python。

> ⚠️ **保持 TCCLI 最新**：TCCLI 持续新增/调整 Action 与字段，旧版本可能缺新接口或字段名不同。开始使用前与定期维护时执行：
>
> ```bash
> uv tool upgrade tccli
> # expected: exit 0（已是最新则显示 Nothing to upgrade）
> tccli --version
> # expected: 输出当前已安装的 TCCLI 版本号
> ```
>
> 本指南命令示例基于近期 `tccli` 版本；TCCLI 持续新增/调整 Action 与字段，若你的版本不同，优先以 `tccli <service> help` / `help --detail` 的实时契约为准。

## 准备工作

### 安装 uv

uv 一次性安装，之后所有 Python 工具都用它管理。按平台选一种：

| 平台 | 命令 |
|:-----|:-----|
| macOS / Linux | `curl -LsSf https://astral.sh/uv/install.sh \| sh` |
| Windows（PowerShell） | `powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 \| iex"` |
| macOS（Homebrew） | `brew install uv` |
| Windows（WinGet） | `winget install astral-sh.uv` |

> 上述脚本地址必须保持为 uv 官方域名 `astral.sh`。安全敏感或受管环境不要直接执行未审阅的网络内容：先下载脚本并审阅，再由组织内部镜像、受管包管理器或固定版本流程安装。

验证 uv 已就绪：

```bash
uv --version
# expected: uv 0.11.x 或更高
```

> 安装后若 `uv` 命令找不到，重新打开终端，或确认 `~/.local/bin`（macOS/Linux）在 PATH 中。

## 操作步骤

### 安装 TCCLI

```bash
uv tool install tccli
# expected: exit 0，输出 Installed tccli 3.x.x
```

uv 会下载 TCCLI 及其依赖到独立环境，并把 `tccli` 命令链接到 PATH。

## 验证

```bash
tccli --version
# expected: 输出当前已安装的 TCCLI 版本号
```
```text
3.1.126.1
```

```bash
which tccli
# expected: tccli 可执行路径（如 ~/.local/bin/tccli）
```

## 故障恢复

| 现象 | 诊断命令 | 根因 | 修复 |
|:-----|:---------|:-----|:-----|
| `command not found: uv` | `which uv` | uv 未安装或未在 PATH | 按上文"安装 uv"步骤重装；macOS/Linux 确认 `~/.local/bin` 在 PATH（`echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zshrc` 后 `source ~/.zshrc`） |
| `command not found: tccli`（安装成功后） | `ls ~/.local/bin/tccli` | PATH 未含 `~/.local/bin` | 同上加入 PATH；Windows 将 `%USERPROFILE%\.local\bin` 加入 PATH |
| 版本过旧，缺少新接口 | `tccli --version` 确认是否最新 | TCCLI 版本低 | `uv tool upgrade tccli` |
| 升级后仍报旧命令不存在 | `tccli help \| grep <Action>` | 升级未生效或装了多个 TCCLI | `uv tool uninstall tccli && uv tool install tccli` 重装 |

## 更新与卸载

```bash
# 更新到最新版
uv tool upgrade tccli
# expected: exit 0

# 卸载（连同独立环境一起清除，无残留）
uv tool uninstall tccli
# expected: exit 0
```

## 收尾确认

```bash
# 衔接下一步：tccli 在 PATH 且可进产品域（安装闭环；凭证下一步再配）
tccli --version
# expected: 输出当前已安装的 TCCLI 版本号

tccli tke help 2>&1 | head -3
# expected: AVAILABLE VERSIONS 含 2018-05-25 / 2022-05-01 → 可进入 [配置凭证](credentials.md)
```

## 下一步

### 新手操作顺序

官方新手指引顺序：**注册实名 → 服务角色授权 → 创建标准集群 → 部署工作负载 → 运维（连接/升级/节点/网络/日志/监控/TCR）**。本指南对应：

| 序 | 意图 | 文档 |
|:--:|:-----|:-----|
| 1 | 装 TCCLI | 本文 |
| 2 | 配凭证 + 首次 TKE 服务授权（`TKE_QCSRole`；VPC-CNI 另需 `IPAMDofTKE_QCSRole`） | [配置凭证](credentials.md) · [服务角色总表](credentials.md#服务角色tke--ipamd--as--tcr--可观测) · [服务授权 43416](https://cloud.tencent.com/document/product/457/43416) |
| 3 | 备 VPC/子网 | [准备 VPC](prepare-vpc.md) |
| 4 | 建托管标准集群 | [TKE Quickstart](../quickstart/tke-first-cluster.md) · [创建集群](../tke/clusters/create.md) |
| 5 | 上线前检查 | [Quickstart — 上线前检查](../quickstart/tke-first-cluster.md#上线前检查) |

- [配置凭证](credentials.md) — 安装后配置 CAM 凭证
- [TKE 快速入门](../quickstart/tke-first-cluster.md) — 创建第一个集群
- [TCR 快速入门](../quickstart/tcr-first-registry.md) — 推送第一份镜像
