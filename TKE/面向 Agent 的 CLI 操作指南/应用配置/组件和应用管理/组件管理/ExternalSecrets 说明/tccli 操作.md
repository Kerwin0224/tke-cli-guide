# ExternalSecrets 说明（tccli）

> 对照官方：[ExternalSecrets 说明](https://cloud.tencent.com/document/product/457/130974) · page_id `130974`

## 概述

External Secrets 组件用于在 TKE 集群中实现 Kubernetes Secret 的外部管理。它对接外部 Secret 管理系统（腾讯云 SSM、HashiCorp Vault、AWS Secrets Manager 等），自动将外部 Secret 同步到 Kubernetes Secret，避免敏感数据直接存储在代码仓库或配置文件中。

该组件基于 [external-secrets](https://external-secrets.io/) 开源项目构建，使用 CRD 描述 Secret 来源和同步策略，Controller 负责监听和同步。

### 组件子模块

| 子模块 | 类型 | 说明 |
|--------|------|------|
| external-secrets | Deployment | 核心 Controller，协调和同步 ExternalSecret、ClusterExternalSecret、PushSecret 资源 |
| webhook | Deployment | Validating Webhook Server，对 CRD 资源进行准入校验 |
| cert-controller | Deployment | 证书控制器，处理 Webhook 所需 TLS 证书的自动轮换 |

### 支持的 CRD 资源

| CRD | 说明 |
|-----|------|
| SecretStore | 命名空间级别的 Secret 存储后端配置 |
| ClusterSecretStore | 集群级别的 Secret 存储后端配置 |
| ExternalSecret | 命名空间级别的外部 Secret 同步对象 |
| ClusterExternalSecret | 集群级别的外部 Secret 同步对象，可跨命名空间创建 Secret |
| PushSecret | 将 Kubernetes Secret 推送到外部 Secret 管理系统 |

## 前置条件

- [环境准备](../../../../环境准备.md)
- 已创建 TKE 集群且状态 `Running`

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
| 查看 ExternalSecrets 组件详情 | `tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName external-secrets` | 是 |
| 查看组件可配置参数 | `tccli tke DescribeAddonValues --region <Region> --ClusterId <ClusterId> --AddonName external-secrets` | 是 |
| 安装 ExternalSecrets 组件 | `tccli tke InstallAddon --region <Region> --ClusterId <ClusterId> --AddonName external-secrets` | 否 |
| 修改组件配置 | `tccli tke UpdateAddon --region <Region> --ClusterId <ClusterId> --AddonName external-secrets --RawValues <json>` | 否 |
| 卸载组件 | `tccli tke DeleteAddon --region <Region> --ClusterId <ClusterId> --AddonName external-secrets` | 是 |
| 创建 SecretStore | `kubectl apply -f secretstore.yaml` | 否（同名报错） |
| 创建 ExternalSecret | `kubectl apply -f externalsecret.yaml` | 否（同名报错） |

## 操作步骤

### 步骤 1：查看可配置参数（控制面）

```bash
tccli tke DescribeAddonValues --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName external-secrets
# expected: 返回可配置参数列表（replicaCount、leaderElect、image 等）
```

```json
{
  "Values": "<Values>",
  "DefaultValues": "<DefaultValues>",
  "RequestId": "<RequestId>"
}
```

### 步骤 2：安装组件（控制面）

```bash
tccli tke InstallAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName external-secrets \
    --AddonVersion <AddonVersion>
# expected: exit 0，组件安装请求已提交
```

生产环境建议配置高可用（≥2 副本 + leader election），通过 `--RawValues` 传入：

```bash
tccli tke InstallAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName external-secrets \
    --AddonVersion <AddonVersion> \
    --RawValues '{"replicaCount":2,"leaderElect":true,"resources":{"requests":{"cpu":"50m","memory":"128Mi"},"limits":{"cpu":"200m","memory":"256Mi"}},"webhook":{"replicaCount":2},"certController":{"replicaCount":1}}'
# expected: exit 0
```

### 步骤 3：创建 SecretStore（数据面，SSM 示例）

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: tencent-ssm
spec:
  provider:
    ssm:
      region: ap-guangzhou
      auth:
        secretRef:
          accessKeyIDSecretRef:
            name: tcloud-credentials
            key: SecretId
          accessKeySecretSecretRef:
            name: tcloud-credentials
            key: SecretKey
```

```bash
kubectl apply -f secretstore-ssm.yaml
# expected: secretstore.external-secrets.io/tencent-ssm created
```

> **注意**：kubectl 数据面当前因公网端点被 CAM 策略 strategyId:240463971 拒绝而不可达。以下 kubectl 命令为参考格式，需在内网/VPN 环境执行。

### 步骤 4：同步外部 Secret（数据面）

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: my-external-secret
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: tencent-ssm
    kind: SecretStore
  target:
    name: my-k8s-secret
  data:
    - secretKey: password
      remoteRef:
        key: my-ssm-secret-name
```

```bash
kubectl apply -f externalsecret.yaml
# expected: externalsecret.external-secrets.io/my-external-secret created
```

## 验证

### 控制面（tccli）

```bash
tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName external-secrets \
    | jq '.Addons[0] | {AddonName, AddonVersion, Status}'
# expected: Status "Running"，AddonName "external-secrets"
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
kubectl get pods -n kube-system -l app.kubernetes.io/name=external-secrets
# expected: external-secrets、cert-controller、webhook Pod 均 Running

kubectl get crd | grep external-secrets.io
# expected: 列出 clustersecretstores、externalsecrets、pushsecrets、secretstores 等 CRD
```

```text
NAME  STATUS  AGE
...
```

> **注意**：kubectl 数据面当前因公网端点被 CAM 策略 strategyId:240463971 拒绝而不可达。以下 kubectl 命令为参考格式，需在内网/VPN 环境执行。

## 清理

> **注意：** 卸载 ExternalSecrets 组件前，需先删除所有 ExternalSecret、SecretStore 等 CRD 资源，否则 CRD 管理器可能阻止卸载。

### 数据面（kubectl）

```bash
kubectl delete externalsecrets --all --all-namespaces
kubectl delete secretstores --all --all-namespaces
kubectl delete clustersecretstores --all
kubectl delete clusterexternalsecrets --all
# expected: CRD 资源已删除
```

### 控制面（tccli）

```bash
tccli tke DeleteAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName external-secrets
# expected: exit 0，组件卸载请求已提交

# 验证已卸载
tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName external-secrets \
    | jq '.Addons[] | select(.AddonName == "external-secrets")'
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

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| ExternalSecret 状态不是 Ready | `kubectl describe externalsecret <name>` 查看 Status 中的错误信息 | SecretStore 配置的凭证无效或过期 | 检查 SecretStore 引用的凭证 Secret 是否有效 |
| Webhook 校验失败 | `kubectl describe <crd-resource>` 查看校验拒绝原因 | CRD 资源格式不正确 | 检查 CRD 资源格式是否正确，按报错信息修正 |
| `kubectl get crd` 无 external-secrets 相关 CRD | `tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName external-secrets` 检查组件状态 | 组件未正确安装或 CRD 未注册 | 重新安装组件 |
| `Unable to connect to the server` | `kubectl cluster-info` 检查连通性 | 公网端点被 CAM 策略 strategyId:240463971 以 `tke:clusterExtranetEndpoint=true` 条件拒绝 | 通过 IOA/VPN 连接内网，或使用同 VPC CVM 执行 kubectl 命令 |

## 下一步

- [组件的生命周期管理](../组件的生命周期管理/tccli 操作.md) — 组件的安装、升级与卸载
- [external-secrets 官方文档](https://external-secrets.io/) — 支持的 Provider 和详细配置
- [腾讯云 SSM 文档](https://cloud.tencent.com/document/product/1140) — 凭据管理系统使用指南

## 控制台替代

访问 [TKE 控制台 - 组件管理](https://console.cloud.tencent.com/tke2/cluster)，点击 **新建**，勾选 **external-secrets** 完成安装。
