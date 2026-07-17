---
doc_type: Overview
---
# 集群加固

> TKE 集群安全包含网络可达、身份认证、权限授权、审计、Secret 静态加密、删除保护和准入控制。TCCLI 调用腾讯云 API 使用 CAM 权限；kubectl/API Server 访问 Kubernetes 对象使用 Kubernetes RBAC。两者是不同权限平面，不能互相替代。
>
> 官方文档：[自定义 RBAC 授权](https://cloud.tencent.com/document/product/457/51683) · [使用集群审计排查问题](https://cloud.tencent.com/document/product/457/48980) · [使用 KMS 进行 ETCD 数据加密](https://cloud.tencent.com/document/product/457/45594) · [策略管理](https://cloud.tencent.com/document/product/457/103179)

## 是什么

集群加固按以下控制层处理：

1. **网络可达**：端点决定客户端能否连接 API Server。
2. **身份认证**：证书、OIDC 或 ServiceAccount 用于确认访问主体。
3. **权限授权**：CAM 控制腾讯云 API，Kubernetes RBAC 控制 Kubernetes 对象操作。
4. **审计**：记录 kube-apiserver 接口调用并写入 CLS，用于检索和追溯。
5. **数据保护**：KMS Encryption Provider 对 etcd 中的 Kubernetes Secret 进行信封加密。
6. **删除保护和准入控制**：集群级删除保护开关与 OPA/Gatekeeper 策略是两项独立能力。

> 调度策略（SchedulerPolicy）控制 Pod 放置，属于集群配置而非安全能力，见 [调度策略](../clusters/scheduling.md)。

## 触发条件

- 子账号调用 TKE 云 API 被拒绝：检查 CAM 权限和 TCCLI 凭证
- `kubectl` 返回 `Unauthorized`：检查 kubeconfig 凭据、OIDC/ServiceAccount 认证配置，再检查 Kubernetes RBAC 绑定
- 多团队需要企业 SSO：使用 `ModifyClusterAuthenticationOptions` 配置 OIDC
- 需要追溯 Kubernetes 对象操作：使用 `EnableClusterAudit` 将 kube-apiserver 调用审计数据写入 CLS
- 需要保护 etcd 中的 Secret：确认版本、KMS 授权和密钥生命周期后使用 `EnableEncryptionProtection`
- 需要防误删集群：使用 `EnableClusterDeletionProtection`
- 需要在准入阶段执行合规策略：先确认集群类型和版本支持 OPA/Gatekeeper

## 核心概念

| 概念 | 控制范围 | 为什么重要 |
|:-----|:---------|:-----------|
| 访问端点 | API Server 的网络可达性 | 决定客户端是否能到达集群，见[管理访问端点](../networking/endpoints.md) |
| kubeconfig | 访问端点及认证凭据等客户端配置 | 不等于授权；身份通过认证后仍受 RBAC 限制 |
| OIDC / ServiceAccount | Kubernetes 身份认证 | 确认访问主体，不直接授予对象权限 |
| CAM | 腾讯云 API 权限平面 | 控制子账号能否通过 TCCLI/云 API 管理 TKE 资源 |
| Kubernetes RBAC | Kubernetes 对象权限平面 | Role/ClusterRole 定义权限，Binding 将权限授予 User、Group 或 ServiceAccount |
| 集群审计 | kube-apiserver 接口调用记录 | 将 Kubernetes API 操作写入 CLS 供检索和追溯 |
| KMS Secret 加密 | etcd 中 Kubernetes Secret 的静态加密 | 降低 Secret 静态存储泄露风险，不表示整个 etcd 全量加密 |
| 删除保护 | `DeletionProtection` 集群级开关 | 删除集群前必须关闭 |
| OPA/Gatekeeper | Kubernetes 准入策略 | `dryrun` 只记录，`deny` 拦截命中请求；与集群级删除保护不同 |

## 两个权限平面

| 操作 | 权限平面 | 配置入口 |
|:-----|:---------|:---------|
| `tccli tke DescribeClusters`、创建或修改 TKE 资源 | CAM | 子账号策略、角色及 TCCLI 凭证 |
| `kubectl get pods`、创建或修改 Kubernetes 对象 | Kubernetes RBAC | Role/ClusterRole + RoleBinding/ClusterRoleBinding，或 TKE 授权能力 |

使用 CAM 管理云资源时，不能因此跳过 RBAC；只要主体需要访问 API Server 中的 Kubernetes 对象，就仍需相应 RBAC 权限。反过来，RBAC 也不能授予 TCCLI 调用腾讯云 API 的 CAM 权限。

## 安全能力对比

| 能力 | TCCLI 接口 | 作用 | 采用前检查 |
|:-----|:-----------|:-----|:-----------|
| 删除保护 | `EnableClusterDeletionProtection` | 防止直接删除集群 | 删除流程和运维授权 |
| 集群审计 | `EnableClusterAudit` | 将 kube-apiserver 调用审计数据写入 CLS | 集群类型、CLS 日志集/主题、地域和保留策略 |
| 认证配置 | `ModifyClusterAuthenticationOptions` | 配置 OIDC、ServiceAccount | 身份源及认证配置状态 |
| RBAC 授权 | `GrantUserPermissions` | 授予 Kubernetes 对象权限 | 最小权限范围；不替代 CAM |
| Secret 静态加密 | `EnableEncryptionProtection` | KMS 保护 etcd 中的 Kubernetes Secret | K8s ≥ 1.18、etcd ≥ 3.0、KMS 授权、费用和密钥生命周期 |
| OPA 策略 | `ModifyOpenPolicyList` | 修改策略的 `EnforcementAction` | K8s ≥ 1.16；标准集群或 Serverless；不支持注册集群和边缘集群 |

## 关键适用边界

### 集群审计

集群审计记录的是 **kube-apiserver 接口调用**，不是腾讯云管控面的“所有 API 操作”，也不代表记录所有系统行为。是否开启应结合合规、故障调查、成本和日志保留需求决定，不按“测试集群”直接豁免。

### KMS Secret 静态加密

- 适用于 Kubernetes ≥ 1.18、etcd ≥ 3.0 的独立集群和托管标准集群。
- 使用 KMS Encryption Provider 对 etcd 中的 Kubernetes **Secret** 做信封加密，不是整个 etcd 的全量加密。
- 开启前须开通 KMS、完成授权并提供 `KMSConfiguration`。
- **禁止随意禁用、删除或计划删除正在使用的 KMS 密钥**：API Server 可能不可用，Secret、ServiceAccount 等对象可能无法获取。
- 是否开启需综合数据敏感度、版本、费用、授权和密钥生命周期管理，不以“没有用户自建 Secret”作为跳过依据。

### OPA/Gatekeeper 准入控制

- 支持 Kubernetes ≥ 1.16 的 TKE 标准集群和 Serverless 集群。
- **不支持注册集群和边缘集群**。
- `ModifyOpenPolicyList` 当前仅修改策略的 `EnforcementAction`，不是任意编辑策略内容。
- `dryrun` 不拦截请求，`deny` 才会拦截命中请求。
- OPA 中的删除防护策略与 `EnableClusterDeletionProtection` 集群级开关是不同控制，可按需要组合使用。

## 快速检查

```bash
# 查看集群审计和删除保护字段；返回值按当前集群实际状态判断
tccli tke DescribeClusterStatus --region <REGION> \
  --filter "ClusterStatusSet[?ClusterId=='<CLUSTER_ID>'] | [0].{audit:ClusterAuditEnabled,protect:ClusterDeletionProtection}"
# expected: 返回 audit 和 protect 当前值

# 查看认证配置，不把认证配置误当成 RBAC 授权
tccli tke DescribeClusterAuthenticationOptions --region <REGION> --ClusterId "<CLUSTER_ID>"
# expected: 返回 ServiceAccount、OIDC 及认证配置状态
```

## 文档

- [认证配置](auth.md) — kubeconfig 获取/轮转、OIDC、ServiceAccount 和 RBAC 分层
- [审计日志](audit.md) — 开启/关闭集群审计及 CLS 查询
- [集群保护策略](protection.md) — Secret 静态加密、删除保护、OPA 准入和事件持久化
- [管理访问端点](../networking/endpoints.md) — API Server 网络可达性
- [删除集群](../clusters/delete.md) — 删除保护与清理
- [错误码](../reference/error-codes.md) — `UnauthorizedOperation.CamNoAuth` 诊断
