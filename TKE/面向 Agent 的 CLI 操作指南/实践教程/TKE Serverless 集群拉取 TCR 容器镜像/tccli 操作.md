# TKE Serverless 集群拉取 TCR 容器镜像（tccli）

> 对照官方：[TKE Serverless 集群拉取 TCR 容器镜像](https://cloud.tencent.com/document/product/1141/59029) · page_id `59029`

## 概述

在 TKE Serverless（弹性容器实例 EKS）集群中拉取 TCR 企业版实例内的私有容器镜像，并创建工作负载。

整体流程分为三步：(1) 将镜像推送至 TCR 企业版；(2) 打通 TKE Serverless 集群与 TCR 实例之间的内网访问；(3) 在集群中创建使用 TCR 镜像的工作负载。

> **适用场景：** TKE Serverless 集群（免运维 Pod）需要从 TCR 企业版拉取私有镜像来部署容器应用。若使用内网访问，需确保 TKE Serverless 集群 VPC 与 TCR 实例在同一地域。

## 前置条件

- [环境准备](../../环境准备.md)
- 已成功[购买企业版实例](https://cloud.tencent.com/document/product/1141/51110)（参见[企业版快速入门](../../快速入门/企业版快速入门/tccli%20操作.md)）。
- 已成功[创建 TKE Serverless 集群](https://cloud.tencent.com/document/product/457/39813)。
- 如果使用子账号操作，需提前授予对应实例的 `tcr:CreateNamespace`、`tcr:CreateRepository`、`tcr:ManageInternalEndpoint` 等权限，参见[企业版授权方案示例](https://cloud.tencent.com/document/product/1141/41417)。
- 推送镜像的客户端需在 TCR [网络访问策略](https://cloud.tencent.com/document/product/1141/41836) 允许范围内（公网或已接入 VPC）。

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|------|
| 查看 TCR 实例 | `DescribeInstances` | 是 |
| 创建命名空间 | `CreateNamespace` | 否（同名报错） |
| 创建镜像仓库 | `CreateRepository` | 否（同名报错） |
| 查看仓库列表 | `DescribeRepositories` | 是 |
| 查看命名空间列表 | `DescribeNamespaces` | 是 |
| 新建内网访问链路 | `ManageInternalEndpoint`（`Operation=Create`） | 否（重复创建报错） |
| 配置域名内网解析 | `CreateInternalEndpointDns` | 否 |
| 查看内网访问链路列表 | `DescribeInternalEndpoints` | 是 |
| 查看 TKE Serverless 集群 | `DescribeClusters`（tke） | 是 |
| 获取仓库域名 | `DescribeRepositories` 返回 `{RegistryName}.tencentcloudcr.com/{NamespaceName}/{RepositoryName}` | 是 |
| 获取访问凭证 | 参见[获取实例访问凭证](https://cloud.tencent.com/document/product/1141/41829) / `DescribeSecurityPolicies` | 是 |
| 创建工作负载 | kubectl（数据面） | — |

## 操作步骤

### 步骤1：准备容器镜像

#### 1.1 创建命名空间

新建的 TCR 企业版实例内无默认命名空间，需手动创建：

```bash
tccli tcr CreateNamespace \
    --RegistryId <RegistryId> \
    --NamespaceName <NamespaceName> \
    --IsPublic false \
    --region ap-guangzhou \
    --output json
```

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `RegistryId` | String | **是** | 实例 ID，如 `tcr-xxxxxxxx` |
| `NamespaceName` | String | **是** | 命名空间名称，建议使用项目或团队名 |
| `IsPublic` | Boolean | 否 | 是否公开，默认 `false`（私有） |

```json
{
    "Response": {
        "NamespaceId": 1,
        "RequestId": "00000000-0000-0000-0000-000000000000"
    }
}
```

验证：

```bash
tccli tcr DescribeNamespaces --RegistryId <RegistryId> --region ap-guangzhou --output json
```

```json
{
  "NamespaceList": [],
  "Name": "<Name>",
  "CreationTime": "<CreationTime>",
  "Public": true,
  "NamespaceId": 0,
  "TagSpecification": "<TagSpecification>",
  "ResourceType": "<ResourceType>",
  "Tags": []
}
```

#### 1.2 创建镜像仓库（可选）

> **说明：** 通过 `docker push` 推送镜像时，若目标仓库不存在将自动创建，无需提前手动创建。以下命令用于提前创建好镜像仓库。

```bash
tccli tcr CreateRepository \
    --RegistryId <RegistryId> \
    --NamespaceName <NamespaceName> \
    --RepositoryName <RepoName> \
    --BriefDescription "<描述>" \
    --region ap-guangzhou \
    --output json
```

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `RegistryId` | String | **是** | 实例 ID |
| `NamespaceName` | String | **是** | 命名空间名称 |
| `RepositoryName` | String | **是** | 镜像仓库名称 |
| `BriefDescription` | String | 否 | 仓库简要描述 |
| `DetailDescription` | String | 否 | 仓库详细描述 |

```json
{
    "Response": {
        "NamespaceName": "docker",
        "RepositoryName": "getting-started",
        "RequestId": "00000000-0000-0000-0000-000000000000"
    }
}
```

#### 1.3 推送容器镜像

使用 `docker` 将镜像推送至 TCR 实例。

**获取登录凭证：**

参考[获取实例访问凭证](https://cloud.tencent.com/document/product/1141/41829)，在 TCR 控制台获取临时或长期登录密码。

**登录 TCR 实例：**

```bash
docker login <RegistryName>.tencentcloudcr.com --username=<AccountId>
```

输入获取到的登录密码。

**推送镜像：**

```bash
docker tag getting-started:latest <RegistryName>.tencentcloudcr.com/<NamespaceName>/<RepoName>:latest

docker push <RegistryName>.tencentcloudcr.com/<NamespaceName>/<RepoName>:latest
```

将 `<RegistryName>`、`<NamespaceName>` 和 `<RepoName>` 替换为实际值。例如：

```bash
docker tag nginx:latest demo-tcr.tencentcloudcr.com/docker/getting-started:latest
docker push demo-tcr.tencentcloudcr.com/docker/getting-started:latest
```

### 步骤2：配置 TKE Serverless 集群访问 TCR 实例

TCR 企业版默认拒绝全部访问。需将 TKE Serverless 集群所在的 VPC 关联至 TCR 实例并配置内网域名解析，使 EKS Pod 可通过内网拉取镜像。

> 若 TKE Serverless 集群与 TCR 实例在同一地域，建议使用内网访问（免公网流量）。若需通过公网，参见[通过 NAT 网关访问

```json
{
  "TotalCount": 0,
  "Clusters": [],
  "ClusterId": "<ClusterId>",
  "ClusterName": "<ClusterName>",
  "ClusterDescription": "<ClusterDescription>",
  "ClusterVersion": "<ClusterVersion>",
  "ClusterOs": "<ClusterOs>",
  "ClusterType": "<ClusterType>"
}
```外网](https://cloud.tencent.com/document/product/457/48710)。

#### 2.1 在 TCR 实例中关联集群 VPC

**查看 TKE Serverless 集群所在的 VPC：**

```bash
tccli tke DescribeClusters --ClusterIds '["<ClusterId>"]' --region ap-guangzhou --output json
```

```json
{
  "RequestId": "..."
}
```

从返回的 `ClusterNetworkSettings.VpcId` 获取 VPC ID。

**新建内网访问链路：**

```bash
tccli tcr ManageInternalEndpoint \
    --RegistryId <RegistryId> \
    --Operation Create \
    --VpcId <VpcId> \
    --SubnetId <SubnetId> \
    --region ap-guangzhou \
    --output json
```

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `RegistryId` | String | **是** | TCR 实例 ID |
| `Operation` | String | **是** | 操作类型：`Create`（新建）/ `Delete`（删除） |
| `VpcId` | String | **是** | VPC ID，如 `vpc-xxxxxxxx` |
| `SubnetId` | String | **是** | 子网 ID，如 `subnet-xxxxxxxx` |

```json
{
    "Response": {
        "RegistryId": "tcr-xxxxxxxx",
        "RequestId": "00000000-0000-0000-0000-000000000000"
    }
}
```

**验证内网访问链路已创建：**

```bash
tccli tcr DescribeInternalEndpoints --RegistryId <RegistryId> --region ap-guangzhou --output json
```

```json
{
  "RequestId": "..."
}
```

确认返回的 `AccessVpcSet` 中包含目标 VPC 且 `Status` 为 `Running`。

#### 2.2 配置域名内网解析

创建内网访问链路后，需配置内网 DNS 解析使 VPC 内的 Pod 能以内网方式解析 TCR 域名：

```bash
tccli tcr CreateInternalEndpointDns \
    --RegistryId <RegistryId> \
    --InstanceId <InstanceId> \
    --VpcId <VpcId> \
    --EniLBIp <EniLBIp> \
    --UsePublicDomain true \
    --region ap-guangzhou \
    --output json
```

| 参数 | 说明 |
|------|------|
|

```json
{
  "VpcSet": [],
  "Region": "<Region>",
  "VpcId": "<VpcId>",
  "Status": "<Status>",
  "RequestId": "<RequestId>"
}
``` `RegistryId` | TCR 实例 ID |
| `InstanceId` | 实例 ID（与控制台实例 ID 一致） |
| `VpcId` | 关联的 VPC ID |
| `EniLBIp` | 内网访问链路的 ENI LB IP 地址（从 `DescribeInternalEndpoints` 获取） |
| `UsePublicDomain` | 是否使用公网域名解析。设为 `true` 表示内网解析到 TCR 标准域名 `*.tencentcloudcr.com` |

> **提示：** `EniLBIp` 可从 `DescribeInternalEndpoints` 返回的 `EniLBIp` 字段获取。

**验证内网 DNS 状态：**

```bash
tccli tcr DescribeInternalEndpointDnsStatus \
    --RegistryId <RegistryId> \
    --VpcId <VpcId> \
    --region ap-guangzhou \
    --output json
```

确认返回 `Status` 为 `SUCCESS`。

#### 2.3 获取 TCR 实例访问凭证

TKE Serverless 拉取 TCR 私有镜像需要 imagePullSecret。有两种方式：

**方式一：使用长期访问凭证（推荐）**

在 TCR 控制台生成[实例长期访问凭证](https://cloud.ten

```json
{
  "SecurityPolicySet": [],
  "PolicyIndex": 0,
  "Description": "<Description>",
  "CidrBlock": "<CidrBlock>",
  "PolicyVersion": "<PolicyVersion>",
  "RequestId": "<RequestId>"
}
```cent.com/document/product/1141/41829)，或在 API 侧创建：

```bash
tccli tcr CreateInstanceToken \
    --RegistryId <RegistryId> \
    --TokenType longterm \
    --Desc "<凭证描述>" \
    --region ap-guangzhou \
    --output json
```

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `RegistryId` | String | **是** | 实例 ID |
| `TokenType` | String | **是** | 凭证类型：`temp`（临时）/ `longterm`（长期） |
| `Desc` | String | 否 | 凭证描述 |

返回的 `Username` 和 `Token` 即用于 imagePullSecret。

> **注意：** 请妥善保管 `Token`，离开 API 返回体后无法再次查看。

**查看已有凭证：**

```bash
tccli tcr DescribeSecurityPolicies --RegistryId <RegistryId> --region ap-guangzhou --output json
```

```json
{
  "RequestId": "..."
}
```

### 步骤3：使用 TCR 镜像创建工作负载

#### 3.1 创建 Kubernetes imagePullSecret

在 TKE Serverless 集群中创建 Secret，用于拉取 TCR 私有镜像：

```bash
kubectl create secret docker-registry tcr-secret \
    --docker-server=<RegistryName>.tencentcloudcr.com \
    --docker-username=<Username> \
    --docker-password=<Token> \
    --namespace=<Namespace>
```

| 字段 | 说明 |
|------|------|
| `--docker-server` | TCR 仓库域名（`DescribeRepositories` 可查），格式 `<RegistryName>.tencentcloudcr.com` |
| `--docker-username` | 访问凭证用户名（腾讯云账号 ID，参见[账号信息](https://console.cloud.tencent.com/developer)） |
| `--docker-password` | 长期访问凭证或临时登录密码 |

或在 YAML 中定义：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: tcr-secret
  namespace: <Namespace>
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: <base64-encoded-docker-config>
```

#### 3.2 创建工作负载

创建 Deployment，使用 TCR 企业版内的镜像：

```bash
kubectl create deployment <DeploymentName> \
    --image=<RegistryName>.tencentcloudcr.com/<NamespaceName>/<RepoName>:<Tag> \
    --namespace=<Namespace>
```

为 Deployment 绑定 imagePullSecret：

```bash
kubectl patch deployment <DeploymentName> \
    -n <Namespace> \
    -p '{"spec":{"template":{"spec":{"imagePullSecrets":[{"name":"tcr-secret"}]}}}}'
```

或在 YAML 中定义：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: <DeploymentName>
  namespace: <Namespace>
spec:
  replicas: 1
  selector:
    matchLabels:
      app: <AppName>
  template:
    metadata:
      labels:
        app: <AppName>
    spec:
      imagePullSecrets:
      - name: tcr-secret
      containers:
      - name: <ContainerName>
        image: <RegistryName>.tencentcloudcr.com/<NamespaceName>/<RepoName>:<Tag>
        ports:
        - containerPort: 80
```

#### 3.3 验证工作负载

```bash
kubectl get deployment <DeploymentName> -n <Namespace>
kubectl get pods -n <Namespace> -l app=<AppName>
```

```text
NAME  STATUS  AGE
...
```

预期 `READY` 为 `1/1`，Pod `STATUS` 为 `Running`。

## 验证

### Control plane (tccli)

| 验证项 | 命令 | 期望结果 |
|--------|------|---------|
| TCR 实例已就绪 | `DescribeInstances --Registryids '["<RegistryId>"]'` | `Status: "Running"` |
| 命名空间已创建 | `DescribeNamespaces` | 目标命名空间存在 |
| 镜像仓库已就绪 | `DescribeRepositories` | 目标仓库存在 |
| 内网访问已启用 | `DescribeInternalEndpoints` | `AccessVpcSet` 包含目标 VPC，`Status: "Running"` |
| 内网 DNS 已配置 | `DescribeInternalEndpointDnsStatus` | `Status: "SUCCESS"` |

### Data plane (kubectl)

```bash
kubectl get deployment <DeploymentName> -n <Namespace> -o wide
kubectl describe pod -n <Namespace> -l app=<AppName>
```

```text
NAME  STATUS  AGE
...
```

检查 Pod Events 中无 `ImagePullBackOff` 或 `ErrImagePull` 错误。

## 排障

| 现象 | 处理 |
|------|------|
| Pod 报 `ImagePullBackOff` | 检查 `imagePullSecrets` 是否正确绑定：`kubectl get secret tcr-secret -n <Namespace>`；确认凭证未过期 |
| Pod 报 `ErrImagePull` | 确认镜像地址拼写正确：`<RegistryName>.tencentcloudcr.com/<NamespaceName>/<RepoName>:<Tag>`；确认镜像已推送成功 |
| 内网 DNS 解析失败 | 确认 `CreateInternalEndpointDns` 已执行；检查 `DescribeInternalEndpointDnsStatus` 返回状态 |
| `ManageInternalEndpoint` 报权限错误 | 确认子账号已授予 `tcr:ManageInternalEndpoint` 权限 |
| TKE Serverless 集群无法访问 TCR | 确认集群 VPC 已正确关联至 TCR 实例；若跨地域，需使用公网/NAT |
| docker push 报 `unauthorized` | 确认已执行 `docker login` 且凭证正确；确认网络策略允许当前客户端 IP |
| 推送时自动创建仓库失败 | TCR 企业版允许推送时自动创建仓库；若失败，手动执行 `CreateRepository` 再推送 |

## 清理

删除测试工作负载（避免持续计费）：

```bash
kubectl delete deployment <DeploymentName> -n <Namespace>
kubectl delete secret tcr-secret -n <Namespace>
```

## 控制台替代

[容器服务控制台 → Serverless 集群 → 工作负载 → Deployment → 新建](https://console.cloud.tencent.com/tke2/cluster)：在"实例内容器"中选择"容器镜像服务 企业版"，浏览选择地域、实例和镜像仓库，手动填写镜像访问凭证。
