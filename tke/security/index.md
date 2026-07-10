---
doc_type: Overview
---
# 集群加固

> TKE 集群的安全防护：认证、审计、加密、删除保护。决定谁能访问集群、操作可追溯、防误删。本节是 TKE 集群内的安全配置；让 TCCLI 能调用 API 的 CAM 根凭证是产品之上的全局前置，见 [配置凭证](../../getting-started/credentials.md)。

## 是什么

TKE 集群加固分五个维度：认证（谁能连）、审计（操作可追溯）、加密（Secret 保护）、删除保护（防误删）、准入控制（OPA 强制合规规则）。

> 调度策略（SchedulerPolicy）控制 Pod 放置，属集群配置非安全，见 [调度策略](../clusters/scheduling.md)。

## 核心概念

| 概念 | 含义 | 为什么重要 |
|:-----|:-----|:-----|
| kubeconfig | 集群访问凭证（证书 + 端点） | kubectl 连集群的钥匙 |
| OIDC | 开放身份认证（企业 SSO 接入） | 用企业身份系统登录集群 |
| RBAC | 基于角色的访问控制 | 细粒度控制子账号权限 |
| 审计日志 | 所有 API 操作记录到 CLS | 合规审计、事故追溯 |
| 删除保护 | `DeletionProtection` 开关 | 防止误删集群 |
| 加密 | KMS 加密 Secret | 保护敏感数据 |
| 准入控制 | OPA/Gatekeeper 强制合规规则 | 防止违规操作（如删带节点的集群） |

## 安全能力对比

| 能力 | 接口 | 作用 | 默认 |
|:-----|:-----|:-----|:-----|
| 删除保护 | `EnableClusterDeletionProtection` | 删除集群前必须关闭 | 建议开启 |
| 审计 | `EnableClusterAudit` | API 操作落 CLS | 建议开启 |
| 认证 | `ModifyClusterAuthenticationOptions` | OIDC/ServiceAccount 配置 | TKE 默认 |
| RBAC | `GrantUserPermissions` | 子账号权限授予 | 按需 |
| 加密 | `EnableEncryptionProtection` | KMS 加密 Secret | 按需 |

## 不适用场景

- 不需审计（测试集群）→ 跳过 audit
- 用 CAM 而非 RBAC 管权限 → 跳过 auth 的 RBAC 段
- 无敏感 Secret → 不需加密

## 快速检查

```bash
# 查看集群安全开关状态
tccli tke DescribeClusterStatus --region <REGION> --filter "ClusterStatusSet[?ClusterId=='<CLUSTER_ID>'] | [0].{audit:ClusterAuditEnabled,protect:ClusterDeletionProtection}"
# expected: 新建托管空集群常见 audit=false, protect=false；生产建议两者都开（EnableClusterAudit / EnableClusterDeletionProtection）
```

## 文档

- [认证配置](auth.md) — kubeconfig 获取/轮转，OIDC，RBAC
- [审计日志](audit.md) — 开启/关闭审计，CLS 查询
- [集群保护策略](protection.md) — etcd 加密、删除保护、OPA 准入、事件持久化
- [删除集群](../clusters/delete.md) — 删除保护与清理
- [错误码](../reference/error-codes.md) — `UnauthorizedOperation.CamNoAuth` 诊断
