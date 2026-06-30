# TKE 集群使用 TCR 插件内网免密拉取容器镜像（tccli）

> 对照官方：[TKE 集群使用 TCR 插件内网免密拉取容器镜像](https://cloud.tencent.com/document/product/1141/48184) · page_id `48184`

## 概述

在 TKE 集群中安装 TCR 插件，实现内网免密拉取 TCR 企业版实例内的容器镜像并创建工作负载。TCR 插件自动为集群内节点配置关联 TCR 实例的内网解析，部署工作负载时无需显式配置 `imagePullSecrets`。

## 前置条件

- [环境准备](../../环境准备.md)
- 已 [连接集群](../../../../TKE/面向%20Agent%20的%20CLI%20操作指南/集群配置/集群管理/连接集群/tccli%20操作.md)，kubectl 可正常访问 APIServer
- 已创建 [TCR 企业版实例](../../操作指南/创建企业版实例/tccli%20操作.md)
- 已创建 [TKE 集群](https://cloud.tencent.com/document/product/457/32189)
- TKE 集群 Kubernetes 版本 ≥ 1.14（TCR 组件最低要求）
- 如使用子账号，需提前授予 TCR 实例操作权限和 TKE 集群操作权限

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 查看 TCR 实例列表 | `tccli tcr DescribeInstances` | 是 |
| 创建 TCR 命名空间 | `tccli tcr CreateNamespace` | 否 |
| 创建镜像仓库 | `tccli tcr CreateRepository` | 否 |
| 获取登录凭证并推送镜像 | `docker login` / `docker push` | — |
| 查看 TCR 内网访问链路 | `tccli tcr DescribeInternalEndpoints` | 是 |
| 关联 VPC 至 TCR 实例 | `tccli tcr ManageInternalEndpoint --Operation Create` | 否 |
| 配置内网域名解析 | `tccli tcr CreateInternalEndpointDns` | 否 |
| 查看 TKE 集群列表 | `tccli tke DescribeClusters` | 是 |
| 查看集群已安装组件 | `tccli tke DescribeAddon` | 是 |
| 安装 TCR 组件 | `tccli tke InstallAddon --AddonName TCR` | 否 |
| 创建工作负载（Deployment） | `kubectl apply -f deployment.yaml` | 否 |

## 操作步骤

### 准备容器镜像

#### 1. 查询 TCR 实例

```bash
tccli tcr DescribeInstances --region ap-guangzhou
```

```json
{
  "TotalCount": 1,
  "Registries": [
    {
      "RegistryId": "tcr-example",
      "RegistryName": "demo-tcr",
      "RegistryType": "basic",
      "Status": "Running",
      "PublicDomain": "demo-tcr.tencentcloudcr.com",
      "RegionName": "ap-guangzhou",
      "RegistryChargeType": 1,
      "ExpiredAt": "2027-01-01 00:00:00"
    }
  ],
  "RequestId": "req-example-001"
}
```

#### 2. 创建命名空间

```bash
tccli tcr CreateNamespace --RegistryId <RegistryId> --NamespaceName docker --region ap-guangzhou
```

#### 3. 推送容器镜像

登录 TCR 实例并推送镜像。获取短期访问凭证（参考 [服务级账号管理](../../操作指南/访问配置/访问凭证配置/服务级账号管理/tccli%20操作.md)），执行 docker login 后推送：

```bash
docker login demo-tcr.tencentcloudcr.com --username=<临时用户名> --password=<临时密码>
docker tag getting-started:latest demo-tcr.tencentcloudcr.com/docker/getting-started:latest
docker push demo-tcr.tencentcloudcr.com/docker/getting-started:latest
```

```text
The push refers to repository [demo-tcr.tencentcloudcr.com/docker/getting-started]
latest: digest: sha256:a1b2c3d4... size: 1234
```

推送成功后可在 [镜像仓库](https://console.cloud.tencent.com/tcr/repository) 控制台查看。

### 配置 TKE 集群访问 TCR 实例

TCR 企业版默认拒绝全部来源的外部访问。需将 TKE 集群所在的 VPC 关联至 TCR 实例，并配置内网域名解析。

#### 步骤 1：查询集群 VPC 信息

```bash
tccli tke DescribeClusters --ClusterIds '["<ClusterId>"]' --region ap-guangzhou
```

```json
{
  "TotalCount": 1,
  "Clusters": [
    {
      "ClusterId": "cls-example",
      "ClusterName": "tke-demo",
      "ClusterVersion": "1.28.3",
      "ClusterStatus": "Running",
      "ClusterNetworkSettings": {
        "VpcId": "vpc-example",
        "ClusterCIDR": "172.16.0.0/16"
      }
    }
  ],
  "RequestId": "req-example-002"
}
```

从输出中获取 `VpcId`（如 `vpc-example`），用于下一步关联 TCR。

#### 步骤 2：关联集群 VPC 至 TCR 实例

将集群 VPC 关联到 TCR 实例的内网访问链路：

```bash
tccli tcr ManageInternalEndpoint \
  --RegistryId <RegistryId> \
  --Operation Create \
  --VpcId <VpcId> \
  --SubnetId <SubnetId> \
  --region ap-guangzhou
```

查看已关联的 VPC：

```bash
tccli tcr DescribeInternalEndpoints --RegistryId <RegistryId> --region ap-guangzhou
```

```json
{
  "AccessVpcSet": [
    {
      "VpcId": "vpc-example",
      "SubnetId": "subnet-example",
      "Status": "Running",
      "AccessIp": "10.0.0.10",
      "EniLBIp": "10.0.0.11"
    }
  ],
  "TotalCount": 1,
  "RequestId": "req-example-003"
}
```

#### 步骤 3：配置内网域名解析

为 TCR 实例创建内网域名解析，使集群内 Pod 能通过内网域名访问 TCR：

```bash
tccli tcr CreateInternalEndpointDns \
  --InstanceId <RegistryId> \
  --VpcId <VpcId> \
  --EniLBIp <EniLBIp> \
  --UsePublicDomain true \
  --RegionName ap-guangzhou \
  --region ap-guangzhou
```

```json
{
  "RequestId": "req-example-004"
}
```

> **说明**：若所在地域暂不支持自动解析，可在 TKE 集群节点自定义脚本中添加 `echo '<内网解析IP> <TCR实例域名>' >> /etc/hosts` 手动配置。

#### 步骤 4：安装 TCR 组件

在 TKE 集群中安装 TCR 扩展组件。组件安装完成后，集群内 Pod 可内网免密拉取关联实例的镜像。

```bash
tccli tke InstallAddon --cli-input-json file://install-tcr-addon.json --region ap-guangzhou
```

`install-tcr-addon.json`：

```json
{
  "ClusterId": "<ClusterId>",
  "AddonName": "TCR",
  "RawValues": "{\"registryId\":\"<RegistryId>\"}"
}
```

```json
{
  "RequestId": "req-example-005"
}
```

确认组件安装状态：

```bash
tccli tke DescribeAddon --ClusterId <ClusterId> --AddonName TCR --region ap-guangzhou
```

```json
{
  "Addons": [
    {
      "AddonName": "TCR",
      "AddonVersion": "1.0.0",
      "AddonStatus": "Running",
      "RawValues": "{\"registryId\":\"<RegistryId>\"}"
    }
  ],
  "RequestId": "req-example-006"
}
```

`AddonStatus` 为 `Running` 表示组件已正常运行，集群具备内网免密拉取能力。

### 使用 TCR 实例内容器镜像创建工作负载

组件安装完成后创建工作负载，无需指定 `imagePullSecrets`：

`tcr-deployment.yaml`：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tcr-getting-started
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: tcr-getting-started
  template:
    metadata:
      labels:
        app: tcr-getting-started
    spec:
      containers:
      - name: getting-started
        image: demo-tcr.tencentcloudcr.com/docker/getting-started:latest
```

```bash
kubectl apply -f tcr-deployment.yaml
```

```text
deployment.apps/tcr-getting-started created
```

```bash
kubectl get deployment tcr-getting-started
```

```text
NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
tcr-getting-started   1/1     1            1           30s
```

> **注意**：请避免在工作负载中手动配置其他 `imagePullSecrets`，否则将导致无法加载 TCR 插件的免密拉取配置。

## 验证

### Control plane (tccli)

验证 TCR 组件运行正常：

```bash
tccli tke DescribeAddon --ClusterId <ClusterId> --AddonName TCR --region ap-guangzhou
```

```json
{
  "Addons": [],
  "AddonName": "<AddonName>",
  "AddonVersion": "<AddonVersion>",
  "RawValues": "

```json
{
  "AccessVpcSet": [],
  "VpcId": "<VpcId>",
  "SubnetId": "<SubnetId>",
  "Status": "<Status>",
  "AccessIp": "<AccessIp>",
  "TotalCount": 0,
  "RequestId": "<RequestId>"
}
```<RawValues>",
  "Phase": "<Phase>",
  "Reason": "<Reason>",
  "CreateTime": "<CreateTime>",
  "RequestId": "<RequestId>"
}
```

验证内网访问链路正常：

```bash
tccli tcr DescribeInternalEndpoints --RegistryId <RegistryId> --region ap-guangzhou
```

### Data plane (kubectl)

验证 Pod 运行状态和镜像拉取正常：

```bash
kubectl get pods -l app=tcr-getting-started
```

```text
NAME                                  READY   STATUS    RESTARTS   AGE
tcr-getting-started-7d8f9c6b4-x9q2r   1/1     Running   0          60s
```

```bash
kubectl describe pod -l app=tcr-getting-started | grep -E "Image:|ImagePullSecrets:"
```

```text
    Image:          demo-tcr.tencentcloudcr.com/docker/getting-started:latest
    ImagePullSecrets:  <none>
```

确认镜像拉取成功（`ImagePullSecrets` 为 `<none>`，表明免密拉取生效）。

## 清理

### Data plane (kubectl)

```bash
kubectl delete deployment tcr-getting-started
```

### Control plane (tccli)

卸载 TCR 组件：

```bash
tccli tke DeleteAddon --ClusterId <ClusterId> --AddonName TCR --region ap-guangzhou
```

删除 TCR 内网域名解析（可选）：

```bash
tccli tcr DeleteInternalEndpointDns \
  --InstanceId <RegistryId> \
  --VpcId <VpcId> \
  --EniLBIp <EniLBIp> \
  --region ap-guangzhou
```

移除 VPC 关联（可选）：

```bash
tccli tcr ManageInternalEndpoint \
  --RegistryId <RegistryId> \
  --Operation Delete \
  --VpcId <VpcId> \
  --SubnetId <SubnetId> \
  --region ap-guangzhou
```

> 不建议删除 TCR 实例及命名空间，除非确认不再使用。

## 排障

| 现象 | 处理 |
|------|------|
| `ImagePullBackOff` 且 Pod Events 显示认证失败 | 检查 `InstallAddon` 中 `RawValues` 的 `registryId` 是否正确；确认 `ImagePullSecrets` 为空 |
| TCR 组件安装失败 | 确认集群 Kubernetes 版本 ≥ 1.14；检查集群与 TCR 实例是否在同一地域 |
| `CreateInternalEndpointDns` 报错 | 确认 `VpcId` 已通过 `ManageInternalEndpoint` 关联至 TCR 实例；检查 `EniLBIp` 是否为正确的接入 IP |
| `ManageInternalEndpoint` 报 VPC 冲突 | 该 VPC 已关联至当前 TCR 实例，先 `DescribeInternalEndpoints` 确认后跳过 |
| Pod 无法解析 TCR 内网域名 | 检查 `CreateInternalEndpointDns` 是否已成功创建；试用 `nslookup demo-tcr.tencentcloudcr.com` 在 Pod 内测试解析 |
| docker push 失败 | 确认已通过 `docker login` 登录 TCR；检查网络访问策略是否允许当前客户端 IP |

## 下一步

- [从自建 Harbor 同步镜像到 TCR 企业版](../从自建%20Harbor%20同步镜像到%20TCR%20企业版/tccli%20操作.md)
- [TCR 企业版实例管理 — 内网访问控制](../../操作指南/访问配置/访问网络控制/配置内网访问控制/tccli%20操作.md)
- [Deployment 管理 — 创建工作负载](https://cloud.tencent.com/document/product/457/31705)
- [TCR 组件说明](https://cloud.tencent.com/document/product/457/49225)

## 控制台替代

[控制台 → 容器镜像服务](https://console.cloud.tencent.com/tcr) 管理 TCR 实例的命名空间与镜像仓库；[控制台 → 容器服务](https://console.cloud.tencent.com/tke2/cluster) 进入集群 → 组件管理 → 安装 TCR 组件，勾选"启用内网解析功能"。
