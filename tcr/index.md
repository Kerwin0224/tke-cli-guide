---
doc_type: Overview
---
# TCR 容器镜像服务

> 腾讯云容器镜像服务 (Tencent Container Registry) — 安全、高性能的 Docker/OCI 镜像仓库。
>
> 官方文档：[产品服务层级与容量限制](https://cloud.tencent.com/document/product/1141/104731) · [个人版快速入门](https://cloud.tencent.com/document/product/1141/63910)

## 是什么

TCR 是腾讯云的容器镜像托管服务：**企业版**提供独享 Registry 实例（独立域名、关联 COS、VPC/公网访问控制、扫描/签名/生命周期/跨地域同步）；**个人版**是共享后端的免费基础托管，限额使用、**不承诺 SLA**，面向个人或临时测试。

核心对象关系:

```mermaid
graph TD
    Registry[Registry 实例] --> Namespace[Namespace 命名空间]
    Registry --> Token[Token 访问凭证]
    Registry --> Policy[SecurityPolicy 白名单]
    Registry --> Rule[TagRetentionRule 保留规则]
    Namespace --> Repository[Repository 仓库]
    Repository --> Image[Image 镜像版本]
```

## 适合谁

| 角色 | 目标 | 主要文档 |
|------|------|-------------|
| 开发者 | 推送/拉取镜像 | [快速入门](../quickstart/tcr-first-registry.md) |
| DevOps | CI/CD 流水线集成 | [访问控制](access/manage.md) |
| SRE | 镜像安全、生命周期管理 | [生命周期](lifecycle/index.md) |
| 运维 | 多地域同步、跨账号同步 | [实例同步](replication/manage.md) |

## 何时阅读
- 你要在腾讯云上管理容器镜像仓库（推送/拉取/删除镜像）— 本域是入口
- 你已读完 [快速入门](../quickstart/tcr-first-registry.md)，需查阅某个具体操作（访问控制/生命周期/镜像安全）— 看下方导航或 [实例管理](instances/index.md)
- 你在 docker login/push/pull 遇到权限或网络问题 — 直接看 [访问控制](access/manage.md) 或 [故障排查](troubleshooting.md)

> Agent：调用 `tccli tcr` 前先掌握 [Agent 操作手册](../appendix/agent-optimization.md) 的 flag 组合（压缩管道 / 模板入参 / waiter）。

## 核心概念

| 概念 | 含义 | 为什么重要 |
|------|------|-----------|
| Registry | TCR 企业版实例（API 术语为 Registry，tccli 命令用 `DescribeInstances` 返回 `Registries` 字段） | 镜像的顶级容器，每个实例有独立域名和存储 |
| Namespace | 仓库的逻辑分组，与 Kubernetes Namespace 是彼此独立的资源（可按团队约定同名，但不会自动关联） | 组织镜像（如 `backend` / `frontend`） |
| Repository | 单个镜像的存储单元 | 如 `backend/api-server`，包含多个 Tag |
| Image Tag | 镜像版本标识 | `v1.0.0` / `latest`，Tag 可被覆盖（除非开启不可变） |
| Token | Registry 访问凭证，可按任务创建临时或长期类型 | 作为 docker login 密码；须按密钥管理，泄露后立即轮换 |

## 企业版 vs 个人版

| 维度 | 企业版 | 个人版 |
|------|--------|--------|
| 隔离 | 独享实例 + 独享存储后端 | 共享服务后端与存储 |
| 存储 | 关联账号 COS（按实际用量计费） | 免费额度内托管 |
| 功能 | 访问控制、同步/复制、GC、签名、不可变 Tag 等 | 基本（仓库+Tag 管理） |
| SLA | 企业版各规格承诺 SLA（见层级说明） | **不支持 / 不承诺** |
| 费用 | 托管服务费（按量/包年包月）+ COS | 免费 |

> 企业版三档配额（命名空间 / 镜像仓库 / Helm / VPC 接入）：`basic` 50/1000/1000/5、`standard` 100/3000/3000/10、`premium` 500/5000/5000/20。跨实例自动同步等能力随规格开放（`basic` 不支持实例同步）。
>
> 个人版默认配额（不可调额，不够则升企业版）：命名空间 **10**/地域；镜像仓库 **广州 500 / 其他地域 100**；单镜像 Tag **100**。详见 [个人版](personal/index.md)。

## 控制台决策流（企业版）

按控制台左栏分桶选型，再进对应 How-to：

| 分桶（控制台左栏） | 先决决策 | 文档 |
|------|----------|------|
| 实例管理 | 规格 `basic`/`standard`/`premium` + 计费（API 默认按量；长期使用可优先包年包月） | [创建实例](instances/create.md) · [实例管理](instances/index.md) |
| 资源中心 | 命名空间 → 镜像仓库 / Helm Chart / **AI 模型、AI Skills** | [管理仓库](repositories/manage.md) · [AI 模型与 Skill](ai/index.md) |
| 运维中心 · 访问 | **优先内网（VPC）**，公网作补充；用户级账号 vs 服务级账号 | [访问控制](access/manage.md) · [访问管理](instances/manage-access.md) |
| 运维中心 · 版本管理 | 版本保留（删 Tag）与制品清理/GC（回收层数据）分工 | [生命周期](lifecycle/index.md) |
| 运维中心 · 同步复制 | 实例同步 vs 实例复制；**基础版不支持**实例同步 | [实例同步](replication/manage.md) |
| 运维中心 · 镜像安全 | 镜像签名（侧栏当前仅此叶子）；CVE 白名单走 API | [镜像签名](security/signing.md) · [CVE 白名单](security/cve-whitelist.md) |
| 运维中心 · 其他 | 触发器、镜像加速、域名管理、制品清理 | [镜像加速](images/acceleration.md) · [自定义域名](instances/custom-domain.md) |
| 交付中心 | 交付流水线（控制台能力） | — |

## 不适用场景

- 只有几个公开镜像、无合规要求 → [Docker Hub](https://hub.docker.com/) 免费即可
- 只在本地开发 → 自建 `docker run registry:2`
- 需要 SLA / VPC 内网 / 扫描签名 / 不可变 Tag → **禁止用个人版**，走企业版 [创建实例](instances/create.md)
- 个人版配额不够（NS>10 或仓库超地域上限）→ 升企业版；个人版**不支持调额**
- 需要 OCI 兼容但不限于镜像、且要自建全栈 → 考虑 Harbor 自建

## 快速检查

```bash
tccli tcr DescribeInstances --region ap-guangzhou
# expected: { "TotalCount": ..., "Registries": [...] }
```

## 导航

- [创建实例](instances/create.md) — 创建你的第一个 TCR 实例
- [管理访问](instances/manage-access.md) — 开启公网/内网、创建 Token
- [自定义域名](instances/custom-domain.md) — 绑定自有域名 + 证书
- [管理仓库](repositories/manage.md) — 命名空间和仓库 CRUD
- [推送镜像](images/push-pull.md) — docker push 你的第一个镜像
- [镜像加速](images/acceleration.md) — P2P 大规模拉取加速
- [AI 模型与 Skill](ai/index.md) — 模型仓库与 Skill 查询
- [访问控制](access/manage.md) — Token、公网白名单、VPC 链接
- [生命周期](lifecycle/index.md) — 版本保留、不可变标签、自动清理
- [实例同步](replication/manage.md) — 跨地域/跨账号同步镜像
- [镜像签名](security/signing.md) — 镜像签名与防篡改
- [CVE 白名单](security/cve-whitelist.md) — 漏洞扫描阻断白名单
- [个人版](personal/index.md) — 免费、共享的轻量镜像仓库
- [参考](reference/states.md) — 实例状态、错误码、共享字段
