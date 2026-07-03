---
doc_type: How-to
---
# 安装 tccli

> 安装腾讯云命令行工具 tccli，本指南所有命令的前置。已安装可跳到 [配置凭证](credentials.md)。

## 概述

tccli 是腾讯云 API 的命令行客户端，用 Python 写成。本指南统一用 **uv** 安装与管理——uv 是一个跨平台的 Python 包管理器，自动为 tccli 建独立环境，不污染系统 Python，不会与系统包冲突。

本指南覆盖 TKE（容器服务）和 TCR（容器镜像服务）两个产品的 tccli 操作。

## 触发条件

- 终端执行 `tccli --version` 报 `command not found`，或版本低于 `3.1.117.1` — 用本文安装或升级
- 首次在新机器上使用本指南（任何 `tccli tke`/`tccli tcr` 命令都依赖 tccli 已装）

## 决策依据

#### 为什么用 uv

| 需求 | uv 如何满足 |
|:-----|:-----|
| 跨平台 | macOS / Linux / Windows 同一套工具，安装命令一致 |
| 不污染系统 Python | `uv tool install` 把 tccli 装进独立环境，只暴露一个 `tccli` 命令到 PATH |
| 规避系统包冲突 | 现代 macOS / 部分 Linux 的系统 Python 受保护（PEP 668），直接 `pip install` 会被拒；uv 自带独立环境，不受此限制 |
| 升级与卸载干净 | `uv tool upgrade tccli` / `uv tool uninstall tccli` 一条命令，无残留 |

> 一句话：uv 让你在任何机器上用同一条命令装好 tccli，且不影响系统 Python。

## 准备工作

### 安装 uv

uv 一次性安装，之后所有 Python 工具都用它管理。按平台选一种：

| 平台 | 命令 |
|:-----|:-----|
| macOS / Linux | `curl -LsSf https://astral.sh/uv/install.sh \| sh` |
| Windows（PowerShell） | `powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 \| iex"` |
| macOS（Homebrew） | `brew install uv` |
| Windows（WinGet） | `winget install astral-sh.uv` |

验证 uv 已就绪：

```bash
uv --version
# expected: uv 0.11.x 或更高
```

> 安装后若 `uv` 命令找不到，重新打开终端，或确认 `~/.local/bin`（macOS/Linux）在 PATH 中。

## 操作步骤

### 安装 tccli

```bash
uv tool install tccli
# expected: exit 0，输出 Installed tccli 3.x.x
```

uv 会下载 tccli 及其依赖到独立环境，并把 `tccli` 命令链接到 PATH。

## 验证

```bash
tccli --version
# expected: tccli 3.1.117.1 或更高
```
```text
3.1.117.1
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
| 版本过旧，缺少新接口 | `tccli --version` 对照 3.1.117.1 | tccli 版本低 | `uv tool upgrade tccli` |
| 升级后仍报旧命令不存在 | `tccli help \| grep <Action>` | 升级未生效或装了多个 tccli | `uv tool uninstall tccli && uv tool install tccli` 重装 |

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
tccli --version && which tccli
# expected: 打印 tccli 3.1.117.1（或更高）+ 可执行路径，两者齐出即安装闭环完成
```

## 下一步

- [配置凭证](credentials.md) — 安装后配置 CAM 凭证
- [TKE 快速入门](../quickstart/tke-first-cluster.md) — 创建第一个集群
- [TCR 快速入门](../quickstart/tcr-first-registry.md) — 推送第一份镜像
