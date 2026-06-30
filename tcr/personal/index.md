---
doc_type: Overview
---
# TCR 个人版

> 适合个人开发者和轻量场景——免费使用，与企业版 API 形态完全不同。

## 这是什么

TCR 个人版是腾讯云容器镜像服务的免费版本，提供基础的镜像托管能力。与企业版不同，个人版**不需要先创建实例**，直接使用命名空间组织仓库，是共享服务（所有个人版用户共享后端）。

> ⚠️ 个人版 API 形态与企业版完全不同：所有 Action 带 `Personal` 后缀（如 `CreateNamespacePersonal` 而非 `CreateNamespace`），且个人版是全局服务（无 `--region` 概念，不传地域参数）。两版不能混用接口。

## 核心概念

| 概念 | 含义 | 为什么重要 |
|:-----|:-----|:-----|
| 个人版用户 | `CreateUserPersonal` 创建的登录账号 | docker login 的用户名密码 |
| 命名空间 | 镜像仓库的逻辑分组 | 组织仓库，个人版内唯一 |
| 仓库 | 存放同名称不同 tag 的镜像 | 相当于一个 Docker 镜像名 |
| 配额 | 命名空间/仓库数上限 | 超过需升级企业版 |

## 个人版 vs 企业版

| 维度 | 个人版 | 企业版 |
|:-----|:-----|:-----|
| 实例 | 无实例概念（共享服务） | 独立实例，有状态机 |
| API | `*Personal` 后缀，全局服务 | `Create*` 等，按地域 |
| 计费 | 免费 | 按量计费 |
| 配额 | 命名空间 2000、仓库 10000（实测） | basic 50/1000、standard 100/3000、premium 500/5000 |
| SLA | 无 | 99.9% |
| 安全 | 基础 | 镜像扫描/签名/VPC 访问/不可变 |

> 个人版配额实测（`DescribeUserQuotaPersonal`）：命名空间 2000、仓库 10000。配额高但共享后端，无 SLA、无安全特性。

## 不适用场景

- 企业生产环境 → 升级到 [TCR 企业版](../instances/create.md)，获得 VPC 内网、安全扫描、跨地域同步
- 大规模 CI/CD → 个人版有 QPS 限制且无 SLA
- 需要 VPC 内网拉取 → 个人版不支持，须企业版
- 需要镜像签名/不可变规则 → 个人版不支持，须企业版 premium/standard

## 快速检查

```bash
# 查看个人版配额（实测返回 Data.LimitInfo[]）
tccli tcr DescribeUserQuotaPersonal
# expected: Data.LimitInfo 含 namespace/repo 配额

# 查看个人版命名空间（必填 Namespace/Limit/Offset）
tccli tcr DescribeNamespacePersonal --Namespace "" --Limit 10 --Offset 0
# expected: Data.NamespaceList（空则无命名空间）
```

> 个人版是全局服务，**不传 `--region`**。`DescribeNamespacePersonal` 的 `--Namespace`/`--Limit`/`--Offset` 全部必填。

## 文档

- [个人版全功能操作](manage.md) — 创建用户、命名空间、仓库、推送镜像
- [TCR 企业版概览](../instances/index.md) — 对比与升级路径
- [推送拉取镜像](../images/push-pull.md) — 企业版推送（个人版参考 manage.md）
