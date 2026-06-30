# tccli-on-tke-tcr

> 腾讯云容器服务 (TKE) 和容器镜像服务 (TCR) 的 `tccli` 命令行操作指南 —— 面向 AI Agent 和运维人员，每条命令可复制执行。

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](../LICENSE)

这是文档源码仓库，通过 [GitBook Git Sync](https://gitbook.com/docs/getting-started/git-sync) 自动同步到在线文档站点。本文件是 **GitHub 仓库门面**；线上 GitBook 文档的首页是仓库根目录的 [`README.md`](../README.md)（两者刻意分离，互不干扰）。

## 内容

覆盖 `tccli tke` 和 `tccli tcr` 的全部运维操作，按版本发布：

| 版本 | 状态 | 说明 |
|------|------|------|
| **v2**（当前线上） | 🟢 live | 扁平结构：`tke/` `tcr/` `quickstart/` `cross-product/` `appendix/` |
| **v1** | 📦 存档 | 旧结构：`TKE/` `TCR/` 下嵌套「面向 Agent 的 CLI 操作指南」 |

## 版本管理

本仓库用 **分支 + 指针 + tag** 三层模型管理文档版本（免费版 GitBook 只同步 `master` 一个分支，所以版本切换 = 移动 `master` 指针）：

- **版本分支** `v1` / `v2`：版本内容真相之源，append-only，可继续优化
- **指针** `master`：GitBook 同步的分支，指向当前线上版本，用 `git reset --hard <版本分支> && git push -f origin master` 切换
- **发布快照** tag `v1-final` / `v2.0`：不可变发布存档

```bash
# 切换线上版本
git checkout master && git reset --hard v1 && git push -f origin master   # 切到 v1
git checkout master && git reset --hard v2 && git push -f origin master   # 切回 v2

# 继续优化某版本（不影响线上）
git checkout v1   # 编辑 → commit → push origin v1
```

## 环境准备

```bash
pip install tccli
tccli configure set secretId <SecretId> secretKey <SecretKey>
tccli configure set region ap-guangzhou
```

## 仓库结构

```
.
├── README.md              # GitBook 线上首页（v2 内容，勿改作门面）
├── SUMMARY.md             # GitBook 目录
├── .gitbook.yaml          # GitBook 配置（root: ./）
├── tke/  tcr/  quickstart/  cross-product/  appendix/   # v2 文档
└── .github/README.md      # ← 本文件，GitHub 仓库门面
```

## License

[MIT](../LICENSE) © 2026 Kerwin
