# TCR 说明（tccli）

> 对照官方：[TCR 说明](https://cloud.tencent.com/document/product/457/49225) · page_id `49225`

## 概述

TCR Addon 是腾讯云容器镜像服务（Tencent Container Registry）推出的官方组件，用于在 TKE 集群中实现**内网免密拉取容器镜像**。安装后，集群节点可通过内网免密拉取 TCR 企业版实例中的容器镜像，无需在资源 YAML 中显式配置 ImagePullSecret，提升拉取速度和配置便利性。

### 核心原理

TCR Assistant 是标准的 Kubernetes Operator，自动将 `imagePullSecret` 部署到集群的各个 Namespace，并关联到 ServiceAccount。当工作负载未显式指定 `imagePullSecret` 或 `serviceAccount` 时，Kubernetes 回退到当前命名空间的 `default` ServiceAccount，使用其上关联的 `imagePullSecret` 完成认证。

### 核心概念

| 名称 | 别名 | 说明 |
|------|------|------|
| ImagePullSecret | ips / ipss | TCR Assistant 的 CRD，存储镜像仓库凭证、目标 Namespace 和 ServiceAccount 规则 |

### 部署的 Kubernetes 对象

| 对象名称 | 类型 | 数量 | 命名空间 |
|---------|------|------|---------|
| tcr-example-system | Namespace | 1 | — |
| tcr-example-manager-role | ClusterRole | 1 | — |
| tcr-example-manager-rolebinding | ClusterRoleBinding | 1 | — |
| tcr-example-leader-election-role | Role | 1 | tcr-example-system |
| tcr-example-leader-election-rolebinding | RoleBinding | 1 | tcr-example-system |
| tcr-example-webhook-server-cert | Secret | 1 | tcr-example-system |
| tcr-example-webhook-service | Service | 1 | tcr-example-system |
| tcr-example-validating-webhook-configuration | ValidatingWebhookConfiguration | 1 | — |
| imagepullsecrets.tcr.tencentcloudcr.com | CRD | 1 | — |
| tcr.ips* | ImagePullSecret CR | 2~3 | tcr-example-system |
| tcr.ips* | Secret | (2~3) × 命名空间数 | 各命名空间 |
| tcr-example-controller-manager | Deployment | 1 | tcr-example-system |
| updater-config | ConfigMap | 1 | tcr-example-system |
| hosts-updater | DaemonSet | 节点数 | tcr-example-system |

### 资源消耗

| 组件 | CPU | 内存 | 实例数 |
|------|-----|------|--------|
| tcr-example-controller-manager | 500m | 512Mi | 1 |
| hosts-updater | 100m | 100Mi | worker 节点数 |

## 前置条件

- [环境准备](../../../../环境准备.md)
- 已 [连接集群](../../../../集群配置/集群管理/连接集群/tccli 操作.md)，kubectl 可正常访问 APIServer
- TKE 集群版本 **v1.10.x 及以上**，推荐 **v1.12.x 及以上**
- Kubernetes `controller manager` 必须包含启动参数 `authentication-kubeconfig` 和 `authorization-kubeconfig`（TKE v1.12.x+ 默认启用）
- 已开通腾讯云容器镜像服务（TCR）并完成服务授权
- 已创建 TCR 企业版实例，且实例地域与集群相同
- 用户需具备 TCR 实例的 `CreateInstanceToken` API 权限（由 TCR 管理员配置）

```bash
# 检查集群状态
tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'
# expected: ClusterStatus "Running"
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

- 需要 CAM 权限：`tke:DescribeClusters`、`tke:DescribeAddon`、`tke:InstallAddon`、`tke:UpdateAddon`、`tke:DeleteAddon`

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 查看集群信息 | `tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'` | 是 |
| 查看 TCR 组件详情 | `tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName tcr` | 是 |
| 安装 TCR 组件（含参数配置） | `tccli tke InstallAddon --region <Region> --ClusterId <ClusterId> --AddonName tcr --RawValues '<json>'` | 否 |
| 升级 TCR 组件 | `tccli tke UpdateAddon --region <Region> --ClusterId <ClusterId> --AddonName tcr --AddonVersion <AddonVersion>` | 否 |
| 卸载 TCR 组件 | `tccli tke DeleteAddon --region <Region> --ClusterId <ClusterId> --AddonName tcr` | 是 |
| 创建 ImagePullSecret CR | `kubectl apply -f <yaml>` | 否（同名报错） |
| 查看 TCR 组件凭证状态 | `kubectl get ipss` | 是 |
| 更新镜像拉取凭证 | `kubectl edit ipss <name>` | 是 |

## 操作步骤

### 步骤 1：获取 TCR 访问凭证

在 [容器镜像服务控制台](https://console.cloud.tencent.com/tcr) 获取：

- **实例域名**（如 `<InstanceName>.tencentcloudcr.com`）
- **长期访问凭证**（推荐生产使用）：
  - 用户名：腾讯云账号 ID（如 `100012345678`）或长期凭证用户名
  - 密码/Token：在 TCR 实例 → 访问凭证 → 创建长期访问令牌

> **注意：** 控制台临时登录命令生成的 token 有效期约为 1 小时，仅适合测试。生产环境请使用长期访问凭证。

### 步骤 2：安装 TCR 组件（控制面）

安装 TCR 组件时需要指定关联的 TCR 企业版实例和免密配置：

```bash
tccli tke InstallAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName tcr \
    --RawValues '{
  "global": {
    "imagePullSecrets": [
      {
        "name": "tcr-example",
        "namespaces": "*",
        "serviceAccounts": "*",
        "docker": {
          "server": "<InstanceDomain>",
          "username": "<AccountId>",
          "password": "<LongTermToken>"
        }
      }
    ],
    "hosts": []
  }
}'
# expected: exit 0，组件安装请求已提交
```

> **说明：** `hosts` 配置用于内网免密解析（测试用途），生产环境建议使用 TCR 内网自动解析、PrivateDNS 或自建 DNS。测试场景可填 `["<InstanceDomain>"]` 启用 Host 配置。

### 步骤 3：创建 ImagePullSecret CR（数据面，替代安装时 RawValues）

若不通过 RawValues 安装时配置，也可以在安装组件后通过 kubectl 创建 CR：

```yaml
apiVersion: tcr.tencentcloudcr.com/v1
kind: ImagePullSecret
metadata:
  name: imagepullsecret-sample
spec:
  namespaces: "*"
  serviceAccounts: "*"
  docker:
    username: "<AccountId>"
    password: "<LongTermToken>"
    server: <InstanceDomain>
```

| 字段 | 说明 | 备注 |
|------|------|------|
| `namespaces` | 命名空间匹配规则 | `"*"` 或空 = 匹配所有；多个用逗号分隔；不支持正则，需精确指定名称 |
| `serviceAccounts` | ServiceAccount 匹配规则 | `"*"` 或空 = 匹配所有；多个用逗号分隔 |
| `docker.server` | 镜像仓库域名 | 仅填域名，不含协议头 |
| `docker.username` | 镜像仓库用户名 | 确保有足够访问权限 |
| `docker.password` | 对应密码 | 建议使用长期访问令牌 |

```bash
kubectl apply -f imagepullsecret.yaml
# expected: imagepullsecret.tcr.tencentcloudcr.com/imagepullsecret-sample created
```

> **注意**：kubectl 数据面当前因公网端点被 CAM 策略 strategyId:240463971 拒绝而不可达。以下 kubectl 命令为参考格式，需在内网/VPN 环境执行。

### 步骤 4：查看 ImagePullSecret 状态（数据面）

```bash
kubectl get ipss
# expected: 包含 SECRETS-DESIRED 和 SECRETS-SUCCESS 等列

kubectl describe ipss imagepullsecret-sample
# expected: Events 中可见 Secret Update Successful 和 Service Accounts Modify Successful 条目
```

### 步骤 5：验证免密拉取（数据面）

创建测试工作负载（不指定 imagePullSecret）：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tcr-example-app
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: tcr-example-app
  template:
    metadata:
      labels:
        app: tcr-example-app
    spec:
      containers:
        - name: app
          image: <InstanceDomain>/<namespace>/<repo>:<tag>
```

```bash
kubectl apply -f tcr-example-app.yaml
kubectl get pod -n default -l app=tcr-example-app
# expected: Pod Running，Events 中无 ErrImagePull 或 ImagePullBackOff
```

```text
NAME  STATUS  AGE
...
```

### 步骤 6：更新镜像拉取凭证（数据面）

当 Token 过期或需要更换凭证时，直接编辑 ImagePullSecret CR 即可，无需删除重建：

```bash
kubectl edit ipss imagepullsecret-sample
# expected: 修改 spec.docker.username 和 spec.docker.password 后保存
```

修改后 TCR Assistant 会自动更新所有关联的 Secret 和 ServiceAccount。

### 组件 RBAC 权限（控制面已自动创建）

安装时自动在 `tcr-example-system` 命名空间创建 Role，以及 ClusterRole `tcr-example-manager-role`：

| API Group | 资源 | 操作 | 用途 |
|-----------|------|------|------|
| `""` (core) | secrets | create, delete, patch, update, watch | 管理镜像拉取凭证 |
| `""` (core) | namespaces, serviceaccounts | get, list, watch, patch, update | 监控命名空间和 SA 变化自动注入 |
| `""` (core) | configmaps | get, list, watch, create, update, patch, delete | 选主与配置管理 |
| `tcr.tencentcloudcr.com` | imagepullsecrets | 完整 CRUD + watch | 管理 IPS CRD |

## 验证

### 控制面（tccli）

```bash
tccli tke DescribeAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName tcr \
    | jq '.Addons[0] | {AddonName, Status, AddonVersion}'
# expected: Status "Running"，AddonName "tcr"
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

### 数据面（kubectl）

```bash
kubectl get deploy -n tcr-example-system tcr-example-controller-manager
# expected: READY 1/1

kubectl get ds -n tcr-example-system hosts-updater
# expected: 每个 worker 节点一个 READY Pod

kubectl get ipss
# expected: SECRETS-DESIRED = SECRETS-SUCCESS

kubectl get sa default -n default -o yaml | grep imagePullSecrets -A3
# expected: imagePullSecrets 列表中包含 tcr.ips... 形式的 Secret
```

```text
NAME  STATUS  AGE
...
```

> **注意**：kubectl 数据面当前因公网端点被 CAM 策略 strategyId:240463971 拒绝而不可达。以下 kubectl 命令为参考格式，需在内网/VPN 环境执行。

## 清理

### 数据面（kubectl）

```bash
kubectl delete deploy tcr-example-app -n default
kubectl delete ipss imagepullsecret-sample
kubectl delete ns test-tcr
# expected: 资源已删除
```

### 控制面（tccli）

```bash
tccli tke DeleteAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName tcr
# expected: exit 0，组件卸载请求已提交

# 验证已卸载
tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName tcr \
    | jq '.Addons[] | select(.AddonName == "tcr")'
# expected: 空结果
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

> **重要提醒：** 删除 TCR 组件**不会**自动删除在 TCR 控制台创建的专用访问凭证。请前往 [容器镜像服务控制台](https://console.cloud.tencent.com/tcr) 手动禁用或删除不再使用的长期访问令牌。
>
> **计费提醒：** TCR 企业版实例按规格计费，删除集群中的 TCR 组件不会影响 TCR 实例本身。如不再需要 TCR 实例，请在 TCR 控制台销毁。

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Pod 拉取镜像报 `ImagePullBackOff`，TCR 组件已安装 | `kubectl get ipss` 检查 `SECRETS-SUCCESS` 是否与 `SECRETS-DESIRED` 一致 | ImagePullSecret CR 凭证错误或过期 | `kubectl edit ipss <name>` 更新凭证 |
| 新命名空间未自动获得 Secret | `kubectl get ipss -o yaml` 检查 `spec.namespaces` 字段 | `namespaces` 字段未匹配到新命名空间 | 将 `spec.namespaces` 设为 `"*"` 或包含该命名空间名 |
| `kubectl get ipss` 无输出 | `tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName tcr` 检查组件状态 | TCR 组件未正确安装或 CRD 未注册 | 重新安装组件；验证 CRD `kubectl get crd imagepullsecrets.tcr.tencentcloudcr.com` |
| Deployment `CrashLoopBackOff` | `kubectl describe pod` 查看 Events | 凭证 username 为腾讯云账号 ID 但密码为临时 token（已过期） | 在 TCR 控制台创建长期访问令牌，更新 `docker.password` |
| 免密拉取正常但 Pod 启动慢 | 检查 hosts-updater DaemonSet 是否启用 | hosts-updater 内网解析仅适用于测试 | 生产环境改用 TCR 内网自动解析或配置 PrivateDNS |
| `Unable to connect to the server` | `kubectl cluster-info` 检查连通性 | 公网端点被 CAM 策略 strategyId:240463971 以 `tke:clusterExtranetEndpoint=true` 条件拒绝 | 通过 IOA/VPN 连接内网，或使用同 VPC CVM 执行 kubectl 命令 |

## 下一步

- [TKE 集群使用 TCR 插件内网免密拉取容器镜像](https://cloud.tencent.com/document/product/457/49225) — 完整操作指南
- [TCR 企业版产品概述](https://cloud.tencent.com/document/product/1141/39287) — TCR 企业版功能介绍
- [TCR 访问凭证管理](https://cloud.tencent.com/document/product/1141/41830) — 创建和管理长期访问令牌
- [组件的生命周期管理](../组件的生命周期管理/tccli 操作.md) — 组件安装、升级与卸载

## 控制台替代

通过 [TKE 控制台安装 TCR 组件](https://cloud.tencent.com/document/product/457/49225) 操作。
