# TKE · tccli 操作手册

**别让 Agent 对着 `tccli` 盲试。**  
这里是可复制的 TKE 命令手册；文档站带只读 MCP，Agent 能先查页再执行。人也照样能打开手册逐步做。

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](../LICENSE)
[![Docs](https://img.shields.io/badge/docs-GitBook-blue)](https://tccli-agent.gitbook.io/tccli)
[![MCP](https://img.shields.io/badge/MCP-ready-green)](https://tccli-agent.gitbook.io/tccli/~gitbook/mcp)

**打开手册** → [tccli-agent.gitbook.io/tccli](https://tccli-agent.gitbook.io/tccli)

---

## 你能用来做什么

| 你要… | 这里给… |
|--------|---------|
| 复制粘贴就能跑的 `tccli tke` | [在线手册](https://tccli-agent.gitbook.io/tccli) 全文 |
| 让 Agent 查文档再调命令 | 下方 **1 条命令接 MCP** |
| 固定「先查手册 / 怎么调 tccli / 有问题回本仓」 | 两个可选 skill |

不覆盖：控制台点选、Terraform、纯 kubectl。

### 接上之后，Agent 少盲试多少？

真实腾讯云上做过对照：同一模型、同一套 tccli skill、同一任务  
（托管空集群 → 1 个节点 → 公网 kubectl → nginx Running），唯一差别是 **能不能查本手册**。

| 模型 | 能查本手册 | 不能查 | 完成任务 |
|------|------------|--------|----------|
| **GLM** | **92** 次命令 | 119 次 | 都能完成 |
| **DeepSeek Flash** | **69** 次命令 | 114 次 | 都能完成 |
| Grok 4.5 | 27 次 | 19 次 | 都能完成 |

GLM、DeepSeek Flash 在能查手册时，完成同一件事用的命令更少。  
Grok 本身就强，有时不查文档也会更快——手册的价值是**可查、可遵循**，不是万能加速器。

---

## 开始用（Agent）

### 1. 接入手册 MCP

只读文档：搜索、打开页面、问答、反馈文档问题。  
**不会**用你的云账号调 API。

```
https://tccli-agent.gitbook.io/tccli/~gitbook/mcp
```

**Claude Code**

```sh
claude mcp add --transport http tccli-agent-docs \
  https://tccli-agent.gitbook.io/tccli/~gitbook/mcp
```

接入后执行 `/mcp` 确认，然后让 Agent 查 TKE 相关页再执行命令。

**Cursor / VS Code**（`mcp.json`）：

```json
{
  "mcpServers": {
    "tccli-agent-docs": {
      "url": "https://tccli-agent.gitbook.io/tccli/~gitbook/mcp"
    }
  }
}
```

**Codex**

```sh
codex mcp add tccli-agent-docs \
  https://tccli-agent.gitbook.io/tccli/~gitbook/mcp
```

### 2. 安装 skill（推荐）

```sh
npx skills add Kerwin0224/tccli-agent-sop -g -y
npx skills add Kerwin0224/tencentcloud-tccli-skill -g -y
```

| skill | 做什么 |
|-------|--------|
| [tccli-agent-sop](https://github.com/Kerwin0224/tccli-agent-sop) | 先查手册再执行；问题开到本仓 Issues |
| [tencentcloud-tccli-skill](https://github.com/Kerwin0224/tencentcloud-tccli-skill) | tccli 怎么调（filter / skeleton / waiter 等） |

### 3. 丢给 Agent 一句

> 用手册 MCP 查清步骤，在 `ap-guangzhou` 创建一个空的托管 TKE 集群；每条命令要能对应到手册页。

---

## 安全

- MCP 只读文档、可反馈文档问题，**改不了**你的云资源  
- 真正调 `tccli` 时，用你本机自己的凭证  
- 不要把 SecretId / SecretKey 贴进不可信对话  

本机尚未安装 tccli 时：`uv tool install tccli` 或 `pip install tccli`，再 `tccli configure`。步骤见手册 [安装](https://tccli-agent.gitbook.io/tccli/getting-started/install) / [凭证](https://tccli-agent.gitbook.io/tccli/getting-started/credentials)。

---

## 人读手册

导航、边界说明、全部命令正文：  
**[tccli-agent.gitbook.io/tccli](https://tccli-agent.gitbook.io/tccli)**

本仓库是上述手册的源码。

---

## License

[MIT](../LICENSE) © 2026 Kerwin
