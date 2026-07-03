---
doc_type: Overview
---
# 应用发布

> 用 Helm Release 在 TKE 集群内部署应用。Release 是 TKE 封装的 Helm 部署实例，管理应用从部署到卸载的完整生命周期。

## 是什么

Release 把 Helm Chart 部署成一个集群内可管理的实例：创建（部署 Chart）→ 升级（改版本/Values）→ 回滚（退历史版本）→ 卸载（移除）。一个 Release = 一个 Chart 的某次部署。

| 概念 | 含义 |
|:-----|:-----|
| Chart | Helm 包，含 K8s 资源模板 + 默认值 |
| Release | Chart 的一次部署实例，有独立版本历史 |
| Values | 覆盖 Chart 默认值的配置 |

> Release 属 **TKE 2018-05-25**（新版无应用发布 Action）。

## 何时用

| 场景 | 用 Release？ |
|:-----|:-----|
| 部署带版本管理的复杂应用（如 Redis、MySQL） | ✅ Release 管理升级/回滚 |
| 一次性部署简单 YAML | 可直接 kubectl apply，无需 Release |
| CI/CD 流水线部署 | ✅ Release 配合流水线升级 |

## 何时不用

- 只用 kubectl 管理 K8s 原生资源 → 用 kubectl，不走 Release
- 用 ArgoCD/Flux 等 GitOps 工具 → 用对应工具，Release 是 TKE 内置方案

## 快速检查

```bash
tccli tke DescribeClusterReleases --region <REGION> --ClusterId "<CLUSTER_ID>"
# expected: 返回 ReleaseSet，含已部署 Release 列表
```

## 文档

- [管理应用发布](manage.md) — 创建/升级/回滚/卸载 Release 完整闭环

## 下一步

- [管理应用发布](manage.md) — 部署第一个 Release
- [插件管理](../addons/manage.md) — 部署前装必要插件（如 cbs-csi 存储）
