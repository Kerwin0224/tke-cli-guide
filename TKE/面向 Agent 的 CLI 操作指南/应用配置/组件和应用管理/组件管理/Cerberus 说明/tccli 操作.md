# Cerberus 说明（tccli）

> 对照官方：https://cloud.tencent.com/document/product/457/80899

## 概述

Cerberus 是 TKE 提供的组件，支持对签名镜像进行可信验证。它可以确保只有经过授权的签名镜像才能在 TKE 集群中部署，从而降低容器环境中的镜像安全风险。

组件工作原理：在集群中部署 ValidatingWebhook，拦截 Pod 创建请求，调用 TCR 签名验证 API 检查镜像是否已签名且签名有效。验证通过则放行，验证不通过则拒绝部署。

### 部署的 Kubernetes 对象

| 对象名称 | 类型 | 资源占用 | 命名空间 |
|----------|------|----------|----------|
| cerberus-webhook | Deployment | CPU 0.5C, 内存 0.5G（1 实例） | tcr-example |
| cerberus-webhook | Service | — | tcr-example |
| cerberus-webhook | ConfigMap | — | tcr-example |
| cerberus-webhook | Secret | — | tcr-example |
| cerberus-webhook | ValidatingWebhookConfiguration | — | — |
| cerberus-rolebinding | ClusterRoleBinding | — | — |
| cerberus-role | ClusterRole | — | — |
| cerberus-controller | ServiceAccount | — | — |

### 组件权限（Kubernetes RBAC）

组件在集群中创建的最小权限 ClusterRole：

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  creationTimestamp: null
  name: cerberus-role
rules:
  - apiGroups:
      - tcr.tencentcloudcr.com
    resources:
      - verifications
    verbs:
      - create
      - delete
      - get
      - list
      - patch
      - update
      - watch
  - apiGroups:
      - tcr.tencentcloudcr.com
    resources:
      - verifications/finalizers
    verbs:
      - update
  - apiGroups:
      - tcr.tencentcloudcr.com
    resources:
      - verifications/status
    verbs:
      - get
      - patch
      - update
```

## 前置条件

环境准备参见 [环境准备](../../../../环境准备.md)，连接集群参见 [连接集群](../../../../集群配置/集群管理/连接集群/tccli 操作.md)。

先确认集群状态：

```bash
tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'
# expected: 返回集群信息，ClusterId 与 <ClusterId> 一致，ClusterStatus 为 Running
```

```json
{
  "TotalCount": "<TotalCount>",
  "Clusters": "<Clusters>",
  "ClusterId": "<ClusterId>",
  "ClusterName": "<ClusterName>",
  "ClusterDescription": "<ClusterDescription>",
  "ClusterVersion": "<ClusterVersion>"
}
```

其他前置条件：

- TKE 集群版本要求 **1.18**。
- 已在 TCR 企业版实例中开启**容器镜像签名**功能，并已对目标镜像完成签名。
- 部署前需授权容器服务角色使用 TCR 查询 API。

CAM 权限：

- `tke:InstallAddon`
- `tke:DescribeAddon`
- `tke:DescribeAddonValues`
- `tke:UpdateAddon`
- `tke:DeleteAddon`

## 控制台与 CLI 参数映射

| 操作 | 控制台路径 | tccli 命令 |
|---|---|---|
| 查看集群信息 | 集群管理 | `tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'` |
| 查看组件列表 | 组件管理 | `tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId>` |
| 查看组件详情 | 组件管理 → Cerberus | `tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName Cerberus` |
| 安装组件 | 组件管理 → 新建 | `tccli tke InstallAddon --region <Region> --ClusterId <ClusterId> --AddonName Cerberus` |
| 升级组件 | 组件管理 → 更新 | `tccli tke UpdateAddon --region <Region> --ClusterId <ClusterId> --AddonName Cerberus` |
| 删除组件 | 组件管理 → 删除 | `tccli tke DeleteAddon --region <Region> --ClusterId <ClusterId> --AddonName Cerberus` |

> **说明**：验证策略的详细配置（策略名称、命名空间、证明者、地域、TCR 实例、签名策略）目前仅支持通过控制台完成。tccli `InstallAddon` 仅负责安装组件本身。组件安装后可在控制台配置验证策略。

## 操作步骤

### 1. 授权容器服务使用 TCR 查询 API

1. 登录 [访问管理控制台](https://console.cloud.tencent.com/cam/role)。
2. 在角色页面，点击 **TCR_QCSRole**。
3. 在角色详情页，关联预设策略 **QcloudTCRReadOnlyAccess**。

### 2. 安装组件

```bash
tccli tke InstallAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName Cerberus
# expected: 返回 RequestId，组件开始安装
```

安装完成后确认状态：

```bash
tccli tke DescribeAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName Cerberus
# expected: AddonName 为 Cerberus，Phase 为 Running
```

```json
{
  "Addons": "<Addons>",
  "AddonName": "<AddonName>",
  "AddonVersion": "<AddonVersion>",
  "RawValues": "<RawValues>",
  "Phase": "<Phase>",
  "Reason": "<Reason>"
}
```

### 3. 配置验证策略（控制台）

1. 登录容器服务控制台，进入集群详情页。
2. 选择**组件管理**，找到已安装的 Cerberus 组件。
3. 配置验证策略参数：

| 参数 | 说明 |
|------|------|
| **策略名称** | 为验证策略设置合适的名称 |
| **命名空间** | 选择策略生效的 K8s 命名空间。每个策略仅作用于一个命名空间 |
| **证明者（Attestor）** | 映射镜像签名时使用的 KMS 密钥，确保签名来源可信 |
| **地域** | 选择 TCR 镜像仓库实例所在地域（与 TKE 集群地域无强制关联） |
| **TCR 实例** | 选择 TCR 镜像仓库实例 |
| **签名策略** | 选择已创建的签名策略（对应镜像签名使用的 KMS 密钥）。可选择多个签名策略 |

> **重要约束**：
> - 验证策略在组件创建后立即生效，验证不通过的镜像**不会被部署**。
> - 不要为包含 TKE 系统组件的命名空间（如 `kube-system`）设置验证策略。
> - 如需将不同 TCR 实例的签名策略绑定到同一命名空间：在已有策略中**新增证明者**。
> - 如需对多个命名空间设置验证：**新增策略**。
> - 镜像签名验证目前仅支持使用 **digest 格式**的镜像，否则会验证不通过。

## 验证

### 控制面 (tccli)

查询组件安装状态：

```bash
tccli tke DescribeAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName Cerberus
# expected: AddonName 为 Cerberus，Phase 为 Running
```

```json
{
  "Addons": "<Addons>",
  "AddonName": "<AddonName>",
  "AddonVersion": "<AddonVersion>",
  "RawValues": "<RawValues>",
  "Phase": "<Phase>",
  "Reason": "<Reason>"
}
```

### 数据面 (kubectl)

> **注意**：kubectl 数据面当前因公网端点被 CAM 策略 strategyId:240463971 拒绝而不可达。以下 kubectl 命令为参考格式，需在内网/VPN 环境执行。

验证组件工作负载是否正常运行：

```bash
kubectl get deploy cerberus-webhook -n tcr-example
# expected: READY 1/1，UP-TO-DATE 1，AVAILABLE 1
```

```text
NAME  STATUS  AGE
...
```

验证 ValidatingWebhookConfiguration 已注册：

```bash
kubectl get validatingwebhookconfigurations cerberus-webhook
# expected: WEBHOOKS 1
```

```text
NAME  STATUS  AGE
...
```

### 验证签名验证功能

在配置了验证策略的命名空间中创建一个引用已签名镜像的 Deployment：

```bash
kubectl create deploy test-signed \
    --image=<TCR实例域名>/<命名空间>/<镜像名>@sha256:<digest> \
    -n <目标命名空间>
```

- 验证通过：Pod 正常部署。
- 验证失败：Pod 部署被拦截，Event 中可见错误信息。

**三种失败场景**：

1. TCR 实例未开启镜像签名功能 - 错误提示签名功能未开启。
2. 镜像无签名信息 - 错误提示拉取的镜像无签名数据。
3. 签名验证不通过 - 错误提示镜像签名验证不通过。

## 清理

### 控制面 (tccli)

```bash
tccli tke DeleteAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName Cerberus
# expected: 返回 RequestId，组件开始卸载
```

### 数据面 (kubectl)

> **注意**：kubectl 数据面当前因公网端点被 CAM 策略 strategyId:240463971 拒绝而不可达。以下 kubectl 命令为参考格式，需在内网/VPN 环境执行。

验证清理后资源：

```bash
kubectl get deploy cerberus-webhook -n tcr-example
# expected: Error from server (NotFound): deployments.apps "cerberus-webhook" not found
```

```text
NAME  STATUS  AGE
...
```

> **计费提醒**：Cerberus 组件本身不产生额外费用，但 TCR 企业版实例按量计费，删除组件不会自动删除 TCR 实例，请按需清理。

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 镜像验证始终不通过 | 检查镜像引用格式 | 镜像未使用 digest 格式 | 使用 `image@sha256:<digest>` 格式重新部署 |
| 组件安装后 Pod 未启动 | 检查 TCR_QCSRole 授权 | TCR_QCSRole 未关联 QcloudTCRReadOnlyAccess | 在 CAM 控制台为 TCR_QCSRole 关联 QcloudTCRReadOnlyAccess 策略 |
| 系统命名空间中的 Pod 无法启动 | 检查是否对 kube-system 配置验证策略 | 系统组件被验证策略拦截 | 删除 kube-system 等系统命名空间的验证策略 |
| ValidatingWebhook 未注册 | `tccli tke DescribeAddon` 检查 Phase | 组件安装未完成 | 等待安装完成或重新安装 |
| Unable to connect to server / 公网端点不可达 | 实际集群 cls-xxxxxxxx，`tccli tke DescribeClusters` 查看公网端点 | 公网端点被 CAM 策略 strategyId:240463971 拒绝（tke:clusterExtranetEndpoint=true） | 在内网/VPN 环境执行 kubectl 命令；或为 CAM 策略放行 tke:clusterExtranetEndpoint |

## 下一步

- [TCR 容器镜像签名](https://cloud.tencent.com/document/product/1141/72467)
- [在 TKE 集群中使用 TCR 企业版](https://cloud.tencent.com/document/product/457/53955)
- [使用自定义策略管理 Kubernetes 权限](https://cloud.tencent.com/document/product/457/11528)
- [TKE 组件管理概述](https://cloud.tencent.com/document/product/457/51082)

## 控制台替代

通过 [TKE 控制台 - 组件管理 - Cerberus](https://cloud.tencent.com/document/product/457/80899) 操作，验证策略配置以控制台页面为准。
