---
doc_type: How-to
subtype: 6B
fused: false
---
# 配置集群认证

> 控制台: [容器服务控制台 - 集群认证](https://console.cloud.tencent.com/tke2/cluster)
> 获取/轮转 kubeconfig、配置 OIDC 企业 SSO、授予 RBAC 权限。配置型操作（改变认证行为，不创建资源）。
>
> **凭证分层**: 本文讲的是 **kubeconfig**（kubectl 连集群的凭证，TKE 产品内）。让 TCCLI 能调用 API 的 **CAM 根凭证**（SecretId/SecretKey）是全局前置，见 [配置凭证](../../getting-started/credentials.md)。

> 官方文档：[身份验证和授权概述](https://cloud.tencent.com/document/product/457/11542) · [服务授权相关角色权限说明](https://cloud.tencent.com/document/product/457/43416) · [常见高危操作](https://cloud.tencent.com/document/product/457/39539)
> 配额：单集群 RBAC 授权条目数受集群规格限制；子账号权限策略数按 CAM 配额。[配额限制](https://cloud.tencent.com/document/product/457/9087)
> ⚠️ **高危操作**：误授 `cluster-admin`（`tke:admin`）角色致全集群失控；kubeconfig 泄露致集群凭据外泄。[常见高危操作](https://cloud.tencent.com/document/product/457/39539)

## 触发条件

- `DescribeClusterKubeconfig` 返回的 kubeconfig 用 `kubectl get nodes` 报 `certificate expired`，需轮转证书
- 多团队需企业 SSO 登录集群，`DescribeClusterAuthenticationOptions` → `OIDCConfig` 为空，需配 OIDC
- 子账号 `kubectl` 连集群报 `Unauthorized`，`DescribeUserPermissions` 返回空，需授予 RBAC 权限 — 看 [故障恢复](#故障恢复)


## 概述

认证决定谁能连集群、以什么身份连。三种认证方式：

| 方式 | 作用 | 适用 |
|:-----|:-----|:-----|
| kubeconfig | 证书 + 端点，kubectl 直连 | 个人开发、运维 |
| OIDC | 企业身份系统 SSO | 企业多团队 |
| RBAC | 子账号细粒度权限 | 多人协作 |

## 决策依据

#### kubeconfig 与 OIDC

- **kubeconfig**: 使用证书认证。适合个人或 CI/CD
- **OIDC**: 接企业 SSO（如腾讯云账号、第三方 IdP），身份集中管理。适合多团队
- **默认推荐**: 小团队用 kubeconfig；企业多团队用 OIDC
- **可否切换**: 可。OIDC 与 kubeconfig 可共存

## 配置项

### kubeconfig 获取与轮转

```bash
# 获取 kubeconfig（响应含 Kubeconfig 与 RequestId，须提取 YAML 叶字段）
tccli tke DescribeClusterKubeconfig --region <REGION> --ClusterId "<CLUSTER_ID>" \
  --filter "Kubeconfig" --output text > kubeconfig.yaml
# expected: kubeconfig 文件生成，首个顶层键为 apiVersion，并含 clusters/contexts/users
kubectl --kubeconfig kubeconfig.yaml get nodes
# expected: 返回节点列表
```

```bash
# 轮转证书（kubeconfig 过期或泄露时）
tccli tke UpdateClusterKubeconfig --region <REGION> --ClusterId "<CLUSTER_ID>"
# expected: exit 0, 重新 DescribeClusterKubeconfig 获取新凭证
```

> `DescribeClusterKubeconfig` 响应同时包含 `Kubeconfig` 与 `RequestId`；必须用 `--filter "Kubeconfig" --output text` 只提取完整 YAML，再重定向供 `kubectl --kubeconfig kubeconfig.yaml get nodes` 使用。

### 集群访问 Token 轮转

> 完整入参以 `tccli tke RotateClusterToken help --detail` 与实际 CLI 行为为准（2018-05-25）。

**当前 TCCLI 能力边界**：

| 渠道 | 结果 |
|:-----|:-----|
| `tccli tke RotateClusterToken --generate-cli-skeleton` | `{}`（无业务字段） |
| `help --detail` → AVAILABLE PARAMETERS | **无** |
| 官方 Example | `tccli tke RotateClusterToken --cli-unfold-argument`（**无** `--ClusterId`） |
| 不传业务参调用 | 服务端 `MissingParameter`：请求缺少必传参数 `ClusterId` |
| 传 `--ClusterId cls-xxx` | TCCLI **本地** `Unknown options: --ClusterId`（该 Action 未暴露此入参） |
| `--cli-input-json file://` 写入 `{"ClusterId":"..."}` | 仍 `MissingParameter`（JSON 中未在 CLI 入参模型暴露的字段被丢弃/不生效） |

结论：**服务端要求 `ClusterId`，当前 TCCLI 未暴露该入参**，按文档写 `--ClusterId` 的命令**不可执行**。凭证轮换请用已暴露 `ClusterId` 的路径：

```bash
# 可执行：轮转 kubeconfig 证书面（skeleton 含 ClusterId）
tccli tke UpdateClusterKubeconfig --region <REGION> --ClusterId "<CLUSTER_ID>"
# expected: exit 0；再 DescribeClusterKubeconfig 取新凭证

# RotateClusterToken：当前 TCCLI 该 Action 不接受 --ClusterId（传入 → Unknown options）；
# 不传 ClusterId 则 MissingParameter。待 TCCLI 暴露 ClusterId 入参后再用本 Action。
# tccli tke RotateClusterToken --region <REGION>
```

> `UpdateClusterKubeconfig` 与 `RotateClusterToken` 是不同凭证面；后者需当前 TCCLI 对该 Action 暴露 `ClusterId` 入参后才能在 CLI 闭环。权限上写操作仍受 CAM 约束（如资源标签 `billing`），未授权时常见 `AuthFailure.UnauthorizedOperation`。


### OIDC 配置

> 完整入参以 `tccli tke ModifyClusterAuthenticationOptions help --detail` 为准。

```bash
tccli tke ModifyClusterAuthenticationOptions --region <REGION> \
  --ClusterId "<CLUSTER_ID>" \
  --ServiceAccounts '{"UseTKEDefault":true}' \
  --OIDCConfig '{"AutoCreateOIDCConfig":true,"AutoInstallPodIdentityWebhookAddon":true}'
# expected: exit 0
```

| 字段 | 类型 | 作用 |
|:-----|:-----|:-----|
| `ServiceAccounts.UseTKEDefault` | boolean | 用 TKE 默认 ServiceAccount 签发 |
| `ServiceAccounts.Issuer` | string | 自定义 issuer URL |
| `ServiceAccounts.JWKSURI` | string | JWKS 公钥地址 |
| `OIDCConfig.AutoCreateOIDCConfig` | boolean | 自动创建 OIDC 配置 |
| `OIDCConfig.AutoCreateClientId` | list | 自动创建的 ClientId |
| `OIDCConfig.AutoInstallPodIdentityWebhookAddon` | boolean | 自动安装 Pod Identity Webhook 插件 |

### RBAC 权限授予

```bash
# 授予子账号集群权限
tccli tke GrantUserPermissions --region <REGION> \
  --TargetUin "<UIN>" \
  --Permissions '[{"ClusterId":"<CLUSTER_ID>","RoleName":"tke:dev","RoleType":"cluster","IsCustom":false,"Namespace":""}]'
# expected: exit 0
```

> ⚠️ **参数层级**: `GrantUserPermissions` 顶层参数是 `TargetUin`（非 `AccountUin`）+ `Permissions` 对象数组。`ClusterId`/`RoleName`/`RoleType`/`IsCustom`/`Namespace` 都在 `Permissions` 元素内（非顶层）。`RoleName` 取值见 [子账号权限管理](#子账号权限管理)。Permission 结构见 [共享字段](../reference/shared-fields.md)。完整入参以 `tccli tke GrantUserPermissions help --detail` 为准。

## 应用

> kubectl（K8s 原生命令，非 tccli；TCCLI 管 TKE 抽象层不提供 K8s 资源操作能力）

#### 1. 获取 kubeconfig（须先开访问端点）

```bash
tccli tke DescribeClusterKubeconfig --region ap-guangzhou --ClusterId "<CLUSTER_ID>" \
  --filter "Kubeconfig" --output text > ~/.kube/tke-config
# expected: YAML 含 apiVersion/clusters；server 为公网或内网 VIP
```

须先开访问端点 → [管理端点](../networking/endpoints.md)

#### 2. 必要时改写 server

若 Domain/server 为 `cls-*.ccs.tencent-cloud.com` 且 dig 失败：用 `ClusterExternalEndpoint` 改写 server（见 [管理端点](../networking/endpoints.md) 步骤 5）。

#### 3. 验证连通

本机须公网端点 Created + SecurityPolicies 含出口 IP；仅内网 VIP 时本机超时属网络路径，非证书问题。

```bash
kubectl --kubeconfig ~/.kube/tke-config get nodes --request-timeout=15s
# expected: 节点列表；超时/Unable to connect/no such host → 查端点、ACL、是否改写为 ClusterExternalEndpoint
```

连通失败排查 → [管理端点](../networking/endpoints.md)

## 验证

> kubectl（K8s 原生命令，非 tccli；TCCLI 管 TKE 抽象层不提供 K8s 资源操作能力）
```bash
# 查看当前认证配置
tccli tke DescribeClusterAuthenticationOptions --region <REGION> --ClusterId "<CLUSTER_ID>"
# expected: 返回 ServiceAccounts/OIDCConfig 配置

# kubectl 连通性（前置：公网端点 Created + SecurityPolicies 含本机出口 IP，或本机在 VPC 内）
kubectl --kubeconfig kubeconfig.yaml get nodes --request-timeout=15s
# expected: 节点列表；context deadline exceeded → 端点未开/仅内网/ACL 未放行
```

| 维度 | 命令 | 预期 |
|:-----|:-----|:-----|
| kubeconfig 有效 | `kubectl --kubeconfig <file> get nodes` | 节点列表返回 |
| OIDC 配置 | `DescribeClusterAuthenticationOptions` | OIDCConfig 非空 |
| 证书未过期 | `kubectl --kubeconfig <file> version` | 无认证错误 |

## 回滚

```bash
# OIDC 改回默认
tccli tke ModifyClusterAuthenticationOptions --region <REGION> \
  --ClusterId "<CLUSTER_ID>" --ServiceAccounts '{"UseTKEDefault":true}'
# expected: exit 0

# 获取轮转后的新 kubeconfig（旧凭证失效）
tccli tke DescribeClusterKubeconfig --region <REGION> --ClusterId "<CLUSTER_ID>" \
  --filter "Kubeconfig" --output text > kubeconfig.yaml
kubectl --kubeconfig kubeconfig.yaml get nodes
# expected: YAML 首个顶层键为 apiVersion，且返回节点列表
```

## 故障恢复 {#故障恢复}

### 命令返回错误 (exit ≠ 0)

| 现象 | 诊断 | 根因 | 修复 |
|:--------|:----------|:------------|:-----|
| `UnauthorizedOperation.CamNoAuth` | 查 CAM 策略 | 子账号无 `tke:DescribeClusterKubeconfig` 权限 | 授予 `tke:*` 相关权限 |
| `InvalidParameter` / `InvalidParameter.Param` | 查 OIDCConfig JSON 格式 | JSON 拼错或字段名错 | 用 `tccli tke ModifyClusterAuthenticationOptions --generate-cli-skeleton` 核对字段名 |
| `ResourceNotFound` | `DescribeClusters` 核对 ID | ClusterId 错 | 确认集群 ID |

### 命令成功但状态不对 (exit = 0)

| 现象 | 诊断 | 根因 | 修复 |
|:--------|:----------|:------------|:-----|
| kubectl 报 `certificate expired` | `kubectl version` | kubeconfig 证书过期 | `UpdateClusterKubeconfig` 轮转 |
| kubectl 报 `Unable to connect` | 端点状态 | 集群端点未开启或不通 | 见 [管理访问端点](../networking/endpoints.md) |
| OIDC 登录失败 | `DescribeClusterAuthenticationOptions` | issuer 不可达或 JWKSURI 错 | 确认 IdP 服务可达 |

## 子账号权限管理 {#子账号权限管理}

> 查询/删除子账号（CAM 子用户）的集群 RBAC 权限。`GrantUserPermissions` 授予权限（见上文），本段是查询与撤销。

```bash
# 查询子账号的集群权限 (TargetUin = 子账号 UIN)
tccli tke DescribeUserPermissions --TargetUin "<SUB_UIN>" --region <REGION>
# expected: exit 0, Permissions[] 含 ClusterId/RoleName/RoleType/Namespace
```
```json
{"TargetUin": "1000000000", "Permissions": [], "RequestId": "..."}
```

```bash
# 删除子账号的指定权限 (按 ClusterId + RoleName 定位)
tccli tke DeleteUserPermissions --TargetUin "<SUB_UIN>" --region <REGION> \
  --Permissions '[{"ClusterId":"<CLUSTER_ID>","RoleName":"<ROLE>","RoleType":"cls","Namespace":""}]'
# expected: exit 0
```

> `TargetUin` 是子账号 UIN（非主账号）。`DeleteUserPermissions` 的 `Permissions[]` 精确定位要撤销的权限（ClusterId+RoleName+Namespace），`RoleType` 如 `cls`（集群级）/ `ns`（命名空间级）。

`RoleName` 预置角色枚举：

| RoleName | 角色 | 作用域 | 适用 |
|:---------|:-----|:-------|:-----|
| `tke:admin` | 集群管理员 | 集群级 (`RoleType=cls`) | 全部操作，生产集群慎授 |
| `tke:ops` | 运维人员 | 集群级 | 集群运维操作，无敏感删除 |
| `tke:dev` | 开发人员 | 集群级 | 工作负载读写，无集群管理 |
| `tke:ro` | 只读用户 | 集群级 | 只读，审计/排查用 |
| `tke:ns:dev` | 命名空间开发人员 | 命名空间级 (`RoleType=ns`，须配 `Namespace`) | 单命名空间读写 |
| `tke:ns:ro` | 命名空间只读用户 | 命名空间级 | 单命名空间只读 |

> 传非预置值（如自定义角色名）也可，但须是集群内已存在的 Role/ClusterRole。命名空间级角色（`tke:ns:*`）必须同时传 `Namespace`，否则报 `InvalidParameter`。

## 收尾确认

> kubectl（K8s 原生命令，非 tccli；TCCLI 管 TKE 抽象层不提供 K8s 资源操作能力）

汇总核对三项：kubeconfig 可用 + OIDC 配置生效 + RBAC 权限授予。

#### 1. kubeconfig 端到端可用

OIDC 配置核对后，再确认 kubeconfig 可连通集群。

```bash
kubectl --kubeconfig kubeconfig.yaml get nodes
# expected: 节点列表返回
```

#### 2. OIDC 配置生效

```bash
tccli tke DescribeClusterAuthenticationOptions --region <REGION> --ClusterId "<CLUSTER_ID>" \
  --filter "{oidc:OIDCConfig}"
# expected: OIDCConfig 与设置一致
```

#### 3. RBAC 权限授予生效（子账号可连集群）

```bash
tccli tke DescribeUserPermissions --TargetUin "<SUB_UIN>" --region <REGION> \
  --filter "Permissions[].{cluster:ClusterId,role:RoleName}"
# expected: 含目标集群与角色 → 认证配置闭环完成
```

> kubeconfig 可连通 + OIDC 配置生效 + RBAC 权限授予 = 跨步骤闭环。除各配置项字段存在外，还须汇总三类认证方式（kubeconfig/OIDC/RBAC）端到端可用，是进下一阶段（部署应用/开启审计）的前置。

---

## 下一步

- [管理访问端点](../networking/endpoints.md) — kubeconfig 需配合端点
- [审计日志](audit.md) — 认证后开启操作审计
- [查询集群](../clusters/query.md) — `DescribeClusterSecurity` 取凭证
- [故障排查](../troubleshooting.md) — kubeconfig 不可用诊断
