---
doc_type: Quickstart
fused: true
---
# TKE 集群拉取 TCR 镜像

> 3 步完成 TKE 集群从 TCR 镜像仓库拉取镜像并部署。
> TKE 控制台: [容器服务](https://console.cloud.tencent.com/tke2) | TCR 控制台: [镜像仓库](https://console.cloud.tencent.com/tcr)

## 触发条件

- TKE 集群中 Pod 报 `ImagePullBackOff` 且镜像地址指向 TCR 私有仓库 — 用本文配置拉取链路
- 需让 TKE 集群从 TCR 拉取私有镜像部署工作负载（VPC 内网拉取场景）— 从 [步骤 1](#步骤-1创建-tcr-访问凭证) 开始


## 概述

TKE 集群拉取 TCR 镜像的三种典型场景：

| 场景 | 网络路径 | 适用 | 延迟 |
|:-----|:---------|:-----|:-----|
| VPC 内网拉取（推荐） | TKE → TCR InternalEndpoint（VPC 内 IP） | 同 VPC 的 TKE 与 TCR | 低 |
| 公网拉取 | TKE → TCR PublicDomain（公网 + 白名单） | TKE 与 TCR 不同 VPC | 高 |
| CI/CD 自动部署 | CI 推 TCR → TKE 拉取（imagePullSecret） | 持续部署流水线 | 中 |

> 本 Quickstart 走 **VPC 内网拉取** + `imagePullSecret` 场景。跨产品操作：TCCLI（取凭证）+ kubectl（配 Secret + 部署）。
>
> **免密替代路径**：TKE 集群可用 **TCR 插件**做内网免密拉取（无需本篇 Secret）；见 [TKE 使用 TCR 插件免密拉取](https://cloud.tencent.com/document/product/1141/48184)。公网拉取产生 COS 公网流量费；同 VPC 内网不计该费。`ImagePullBackOff` / `unauthorized` 时先核：内网端点是否接入、白名单/凭证是否有效、镜像地址是否用对域名。

## 准备工作

> 本篇主路径两个 CLI：TCCLI（管 TCR 凭证/端点）+ kubectl（配 Secret、部署并验证拉取；K8s 原生，TCCLI 做不到 Pod 级拉取确认）。本地 `docker tag/push` 见 [推送拉取镜像](../tcr/images/push-pull.md)，不在本 Quickstart 主线。

> kubectl（K8s 原生命令，非 tccli；TCCLI 管 TKE 抽象层不提供 K8s 资源操作能力）
```bash
# 1. tccli 可用
tccli --version
# expected: tccli 版本号

# 2. kubectl 可用且已配置 TKE 集群 kubeconfig
kubectl get nodes
# expected: 节点列表返回

# 3. TCR 实例 Running
tccli tcr DescribeInstanceStatus --region ap-guangzhou --RegistryIds '["<REGISTRY_ID>"]' \
  --filter "RegistryStatusSet[0].Status"
# expected: "Running"

# 4. TCR 内网端点已接入（VPC 内网拉取前提）
tccli tcr DescribeInternalEndpoints --region ap-guangzhou --RegistryId "<REGISTRY_ID>" \
  --filter "TotalCount"
# expected: ≥ 1（未接入见 [访问控制](../tcr/access/manage.md)）
```

## 步骤 1：创建 TCR 访问凭证

```bash
# 取临时 Token（返回 Username + Token + ExpTime）
tccli tcr CreateInstanceToken --region ap-guangzhou \
  --RegistryId "<REGISTRY_ID>" --TokenType temp --Desc "tke-pull"
# expected: {"Username":"<USERNAME>","Token":"<TOKEN>","ExpTime":<TS>}
```

```json
{
    "Username": "<USERNAME>",
    "Token": "<TOKEN>",
    "ExpTime": 1782702866551,
    "RequestId": "xxx"
}
```

> 临时 Token 1 小时过期。生产环境用长期凭证（服务账号），见 [访问控制](../tcr/access/manage.md)。

## 步骤 2：TKE 集群配置 imagePullSecret

> kubectl（K8s 原生命令，非 tccli；TCCLI 管 TKE 抽象层不提供 K8s 资源操作能力）
```bash
# 获取 TCR 访问地址
# PublicDomain 可直接作为公网镜像域名；InternalEndpoint 是内网接入地址，不等同于镜像域名

tccli tcr DescribeInstances --region ap-guangzhou --Registryids '["<REGISTRY_ID>"]' \
  --filter "Registries[0].{domain:PublicDomain,internalEndpoint:InternalEndpoint}"
# expected: domain=xxx.tencentcloudcr.com；internalEndpoint 为内网接入地址（已接入 VPC 时非空）

# 在 TKE 集群创建 imagePullSecret（用内网域名拉取，低延迟）
kubectl create secret docker-registry tcr-secret \
  --docker-server="<REGISTRY_INTERNAL_DOMAIN_OR_PUBLIC>" \
  --docker-username="<USERNAME>" \
  --docker-password="<TOKEN>" \
  --namespace=default
# expected: secret/tcr-secret created
```

| 占位符 | 含义 | 约束 | 如何获取 |
|:------------|:-----|:-----|:---------|
| `<REGISTRY_ID>` | TCR 实例 ID | `tcr-xxxxxxxx` | `tccli tcr DescribeInstances` |
| `<USERNAME>` | Token 用户名 | 数字账号 ID | 步骤 1 的 `Username` 字段 |
| `<TOKEN>` | Token 密码 | JWT 字符串 | 步骤 1 的 `Token` 字段 |
| `<REGISTRY_INTERNAL_DOMAIN_OR_PUBLIC>` | 拉取域名 | 公网用 `PublicDomain`；VPC 内网用开通内网访问后分配的专用域名 | TCR 控制台的实例访问凭证页或内网访问说明 |

> VPC 内网拉取时，`InternalEndpoint` 用于确认内网接入是否建立，不能直接作为 `--docker-server`。`--docker-server` 需填写 TCR 为实例分配的镜像访问域名；公网拉取填写 `PublicDomain`。

## 步骤 3：部署应用并验证拉取

> kubectl（K8s 原生命令，非 tccli；TCCLI 管 TKE 抽象层不提供 K8s 资源操作能力）
```bash
# 部署应用，引用 imagePullSecret（create deployment + patch 与 expected 资源类型一致）
kubectl create deployment my-app --image="<REGISTRY_DOMAIN>/<NAMESPACE>/<REPO>:<TAG>"
# expected: deployment.apps/my-app created

kubectl patch deployment my-app -p '{"spec":{"template":{"spec":{"imagePullSecrets":[{"name":"tcr-secret"}]}}}}'
# expected: deployment.apps/my-app patched（imagePullSecrets 生效）
```

### 验证

从两个维度确认镜像拉取链路通：

> kubectl（K8s 原生命令，非 tccli；TCCLI 管 TKE 抽象层不提供 K8s 资源操作能力）
```bash
# 维度 1: Pod Running，无 ImagePullBackOff
kubectl get pods -l app=my-app
# expected: Pod Running，STATUS 非 ImagePullBackOff/ErrImagePull
```

> kubectl（K8s 原生命令，非 tccli；TCCLI 管 TKE 抽象层不提供 K8s 资源操作能力）
```bash
# 维度 2: 确认镜像拉取来源
kubectl describe pod -l app=my-app | /usr/bin/grep -A2 "Events:"
# expected: Successfully pulled image "<REGISTRY_DOMAIN>/..."
```

> Pod Running + Events 含 "Successfully pulled image" = TKE→TCR 镜像拉取链路通。

---

## 收尾确认

> kubectl（K8s 原生命令，非 tccli；TCCLI 管 TKE 抽象层不提供 K8s 资源操作能力）
```bash
# 业务可用性：Secret 仍在时再部署一次，确认链路可复现（步骤验证已查首次 Running，此处不先删 Secret）
kubectl create deployment verify-app --image="<REGISTRY_DOMAIN>/<NAMESPACE>/<REPO>:<TAG>"
# expected: deployment.apps/verify-app created

kubectl patch deployment verify-app -p '{"spec":{"template":{"spec":{"imagePullSecrets":[{"name":"tcr-secret"}]}}}}'
# expected: deployment.apps/verify-app patched

kubectl get pods -l app=verify-app -o jsonpath='{.items[0].status.containerStatuses[0].ready}'
# expected: true → 镜像拉取+容器运行端到端成功

# 清理验证 Deployment（保留 Secret，供可选清理段处理）
kubectl delete deployment verify-app
# expected: deployment.apps "verify-app" deleted
```

> Secret 仍在时二次部署 Pod ready=true = TKE 拉取 TCR 镜像链路可复现，端到端配置完成。

## 清理

> kubectl（K8s 原生命令，非 tccli；TCCLI 管 TKE 抽象层不提供 K8s 资源操作能力）
```bash
# 1. 删除业务部署（若仍存在）
kubectl delete deployment my-app --ignore-not-found
# expected: deployment.apps "my-app" deleted 或 not found

# 2. 删除 imagePullSecret
kubectl delete secret tcr-secret -n default --ignore-not-found
# expected: secret "tcr-secret" deleted 或 not found

# 3. Token 清理：本篇步骤 1 用的是 TokenType=temp
# temp 创建时常返回 TokenId:""，且通常不出现在 DescribeInstanceToken 列表，约 1 小时自动过期，无需 DeleteInstanceToken
# 若改用 longterm：保存 Create 响应的 TokenId（或 DescribeInstanceToken → Tokens[].Id），再：
# tccli tcr DeleteInstanceToken --region ap-guangzhou --RegistryId "<REGISTRY_ID>" --TokenId "<TOKEN_ID>"
# expected (longterm): exit 0
```

> TCR 实例与镜像版本保留（不影响其他业务）。**temp Token 约 1 小时自动过期**；`DeleteInstanceToken`/`ModifyInstanceToken` 仅适用于 **longterm** 凭证。

---

## 下一步

- [TCR 访问控制](../tcr/access/manage.md) — 长期凭证与 VPC 内网配置
- [推送拉取镜像](../tcr/images/push-pull.md) — docker CLI 推送镜像到 TCR
- [TKE 快速入门](../quickstart/tke-first-cluster.md) — 创建 TKE 集群
- [故障排查](../tke/troubleshooting.md) — `ImagePullBackOff` 诊断
