# 使用自定义域名及云联网实现跨地域内网访问（tccli）

> 对照官方：[使用自定义域名及云联网实现跨地域内网访问](https://cloud.tencent.com/document/product/1141/76084) · page_id `76084`

## 概述

容器镜像服务（TCR）企业版支持网络访问控制，允许接入指定私有网络 VPC，使该 VPC 内 Docker 客户端通过内网访问镜像数据。在多云/分布式云场景中，用户需要在多地域、多私有网络同时接入 TCR 企业版单个实例，并实现正常的内网推送、拉取镜像。

本文介绍如何使用自定义域名，配合云联网（CCN）、Private DNS 产品，实现多私有网络 VPC 同时接入 TCR 实例并通过内网分发容器镜像。

## 前置条件

- [环境准备](../../环境准备.md)
- 已完成 [创建企业版实例](../../操作指南/创建企业版实例/tccli%20操作.md)，实例 `Status` 为 `Running`
- 已拥有合法域名（可通过 [域名注册服务](https://console.cloud.tencent.com/domain) 注册）
  - 若在公网环境内使用自定义域名，域名需已完成 [ICP 备案](https://cloud.tencent.com/product/ba)
- 已为域名签发 SSL 证书（可通过 [SSL 证书服务](https://cloud.tencent.com/product/ssl) 购买），并确认证书已绑定需使用的自定义域名
- 已开通 [云联网](https://console.cloud.tencent.com/vpc/ccn) 服务，并将多地域 VPC 接入同一云联网实例
- 已开通 [Private DNS](https://console.cloud.tencent.com/privatedns) 服务
- 如使用子账号操作，需为其授予对应实例的操作权限，参见 [企业版授权方案示例](https://cloud.tencent.com/document/product/1141/41417)

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 购买企业版实例 | `tccli tcr CreateInstance` | 否 |
| 查看实例详情 | `tccli tcr DescribeInstances` | 是 |
| 查看自定义域名列表 | `tccli tcr DescribeInstanceCustomizedDomain` | 是 |
| 添加自定义域名 | `tccli tcr CreateInstanceCustomizedDomain` | 否（重名报错） |
| 删除自定义域名 | `tccli tcr DeleteInstanceCustomizedDomain` | 是 |
| 配置内网访问（接入 VPC） | `tccli tcr ManageInternalEndpoint` | 否 |
| 查看内网访问链路 | `tccli tcr DescribeInternalEndpoints` | 是 |
| 创建云联网并关联 VPC | 跨产品控制台操作（VPC/CCN），无直接 TCR CLI | — |
| 配置 Private DNS 私有域解析 | 跨产品控制台操作（Private DNS），无直接 TCR CLI | — |

## 操作步骤

### 创建 TCR 企业版实例，并绑定自定义域名

#### 创建企业版实例

在容器业务部署地域购买企业版实例：

```bash
tccli tcr CreateInstance \
  --RegistryName <your-instance-name> \
  --RegistryType basic \
  --region ap-guangzhou \
  --output json
```

```json
{
  "RegistryId": "tcr-xxxxxxxx",
  "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

| 参数 | 必填 | 说明 |
|------|------|------|
| `--RegistryName` | 是 | 实例名称，需全局唯一 |
| `--RegistryType` | 是 | 实例规格：`basic`（基础版）/ `standard`（标准版）/ `premium`（高级版） |

> **注意**：`CreateInstance` 为预付费购买，创建后即开始计费。建议按需选择规格。

查看实例状态，确认 `Status` 为 `Running`：

```bash
tccli tcr DescribeInstances \
  --RegistryIds "[\"tcr-xxxxxxxx\"]" \
  --region ap-guangzhou \
  --output json
```

```json
{
  "Registries": [
    {
      "RegistryId": "tcr-xxxxxxxx",
      "RegistryName": "my-registry",
      "RegistryType": "basic",
      "Status": "Running",
      "PublicDomain": "tcr-xxxxxxxx.tencentcloudcr.com",
      "RegionName": "ap-guangzhou"
    }
  ],
  "TotalCount": 1,
  "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

#### 初始化实例并推送首个镜像

初次使用需初始化实例并上传首个镜像。使用 Docker 客户端推送镜像：

```bash
docker login tcr-xxxxxxxx.tencentcloudcr.com --username=<临时登录用户名>
```

```bash
docker tag nginx:latest tcr-xxxxxxxx.tencentcloudcr.com/<namespace>/nginx:latest
```

```bash
docker push tcr-xxxxxxxx.tencentcloudcr.com/<namespace>/nginx:latest
```

> 在此步骤中同时接入指定私有网络（如 vpc-gz-01），通过内网推送镜像。内网访问配置见下一步。

#### 配置内网访问（接入 VPC）

为实例接入指定私有网络 VPC，使该 VPC 内容器客户端可通过内网拉取/推送镜像：

```bash
tccli tcr ManageInternalEndpoint \
  --RegistryId tcr-xxxxxxxx \
  --Operation Create \
  --VpcId vpc-gz01xxxx \
  --SubnetId subnet-gz01xxxx \
  --region ap-guangzhou \
  --output json
```

```json
{
  "RegistryId": "tcr-xxxxxxxx",
  "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

| 参数 | 必填 | 说明 |
|------|------|------|
| `--RegistryId` | 是 | TCR 企业版实例 ID |
| `--Operation` | 是 | 操作类型：`Create`（接入）/ `Delete`（移除） |
| `--VpcId` | 是 | 待接入的私有网络 VPC ID |
| `--SubnetId` | 是 | 私有网络的子网 ID |

查看已接入的 VPC 列表：

```bash
tccli tcr DescribeInternalEndpoints \
  --RegistryId tcr-xxxxxxxx \
  --region ap-guangzhou \
  --output json
```

```json
{
  "AccessVpcSet": [
    {
      "VpcId": "vpc-gz01xxxx",
      "SubnetId": "subnet-gz01xxxx",
      "Status": "Running",
      "AccessIp": "10.0.0.10",
      "EniLbIp": "10.0.0.11"
    }
  ],
  "TotalCount": 1,
  "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

#### 配置自定义域名

参考 [配置自定义域名](../../操作指南/访问配置/访问域名配置/配置自定义域名/tccli%20操作.md) 完成。简述如下：

查看当前自定义域名列表：

```bash
tccli tcr DescribeInstanceCustomizedDomain \
  --RegistryId tcr-xxxxxxxx \
  --region ap-guangzhou \
  --output json
```

```json
{
  "DomainInfoList": [],
  "RegistryId": "<RegistryId>",
  "CertId": "<CertId>",
  "DomainName": "<DomainName>",
  "Status": "<Status>",
  "TotalCount": 0,
  "RequestId": "<RequestId>"
}
```

添加自定义域名：

```bash
tccli tcr CreateInstanceCustomizedDomain \
  --RegistryId tcr-xxxxxxxx \
  --DomainName <your-custom-domain> \
  --CertificateId <CertificateId> \
  --region ap-guangzhou \
  --output json
```

```json
{
  "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

| 参数 | 必填 | 说明 |
|------|------|------|
| `--RegistryId` | 是 | TCR 企业版实例 ID |
| `--DomainName` | 是 | 自定义域名，如 `registry.example.com` |
| `--CertificateId` | 是 | 已绑定自定义域名的 SSL 证书 ID（格式 `cert-xxxxx`） |

### 使用云联网关联多个私有网络 VPC

云联网（CCN）用于打通广州与上海等多个地域的私有网络 VPC，使不同地域的容器集群能跨地域互访。

> **注意**：云联网的创建和 VPC 关联为跨产品操作（VPC 产品线），通过 [私有网络控制台](https://console.cloud.tencent.com/vpc/ccn) 或 `tccli vpc` 完成，本节不再展开 TCR CLI。

简要步骤：

1. 前往 [云联网控制台](https://console.cloud.tencent.com/vpc/ccn) 新建云联网实例。
2. 将广州和上海等地域的多 VPC 关联至该云联网。
3. 确认路由已自动学习，所有关联 VPC 间网络互通。

> 可选使用 [对等连接](https://console.cloud.tencent.com/vpc/conn) 替代云联网关联 VPC。

### 配置自定义域名的私有域解析

1. 前往 [Private DNS 控制台](https://console.cloud.tencent.com/privatedns)，使用已绑定的自定义域名新建 Private Zone。
2. 将该 Private Zone 关联至已接入的多个私有网络 VPC（广州、上海等）。
3. 配置解析记录：
   - **A 记录**：记录值填写已接入私有网络的内网访问 IP（可通过 `tccli tcr DescribeInternalEndpoints` 获取）
   - **CNAME 记录**：记录值为实例的标准访问域名（如 `tcr-xxxxxxxx.tencentcloudcr.com`）

> **注意**：Private DNS 是

```json
{
  "DomainInfoList": [],
  "RegistryId": "<RegistryId>",
  "CertId": "<CertId>",
  "DomainName": "<DomainName>",
  "Status": "<Status>",
  "TotalCount": 0,
  "RequestId": "<RequestId>"
}
```独立于 TCR 的产品，配置操作在控制台完成。

## 验证

### Control plane (tccli)

确认自定义域名绑定成功：

```bash
tccli tcr DescribeInstanceCustomizedDomain \
  --RegistryId tcr-xxxxxxxx \
  --region ap-guangzhou \
  --output json
```

```json
{
  "DomainInfoList": [
    {
      "DomainName": "registry.example.com",
      "CertificateId": "cert-xxxxx",
      "Status": "Active"
    }
  ],
  "TotalCount": 1,
  "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

确认 VPC 内网访问链路正常：

```bash
tccli tcr DescribeInternalEndpoints \
  --RegistryId tcr-xxxxxxxx \
  --region ap-guangzhou \
  --output json
```

```json
{
  "RequestId": "..."
}
```

### Data plane (docker)

#### 验证已接入实例的私有网络 VPC（广州）

在位于广州的已接入 VPC 内云服务器上执行：

```bash
docker login <your-custom-domain> --username=<临时登录用户名>
```

```bash
docker pull <your-custom-domain>/<namespace>/nginx:latest
```

镜像拉取成功则说明私有网络接入、自定义域名、私有域解析配置正常。

#### 验证已接入云联网的其他私有网络 VPC（上海）

在位于上海的已接入云联网的 VPC 内云服务器上执行：

```bash
docker login <your-custom-domain> --username=<临时登录用户名>
```

```bash
docker pull <your-custom-domain>/<namespace>/nginx:latest
```

镜像拉取成功则说明云联网配置正常，上海 VPC 内可跨地域通过内网拉取镜像。

## 清理

### Control plane (tccli)

```bash
tccli tcr DeleteInstanceCustomizedDomain \
  --RegistryId tcr-xxxxxxxx \
  --DomainName <your-custom-domain> \
  --region ap-guangzhou \
  --output json
```

在 Private DNS 控制台删除为该域名创建的 Private Zone 及解析记录。

在云联网控制台解除 VPC 关联（按需）。

```bash
tccli tcr ManageInternalEndpoint \
  --RegistryId tcr-xxxxxxxx \
  --Operation Delete \
  --VpcId vpc-gz01xxxx \
  --SubnetId subnet-gz01xxxx \
  --region ap-guangzhou \
  --output json
```

> **警告**：删除自定义域名可能造成容器集群内已有镜像拉取配置无法使用，请谨慎操作。

## 排障

| 现象 | 处理 |
|------|------|
| `CreateInstanceCustomizedDomain` 返回 `InvalidParameter` | 检查 `--CertificateId` 是否有效且已绑定该域名 |
| 自定义域名添加成功但无法访问 | 检查 DNS 解析记录（A/CNAME）是否正确；确认 VPC 已接入实例 |
| 云联网关联后上海 VPC 无法拉取镜像 | 检查云联网路由是否正确传播；确认 Private Zone 已关联上海 VPC |
| Private DNS 解析不生效 | DNS 有 TTL 缓存，等待过期或手动刷新；检查 A 记录 IP 是否为 `DescribeInternalEndpoints` 返回的 `AccessIp` |
| Docker 拉取返回 401 | 确认已 `docker login` 且访问凭证未过期 |

## 下一步

- [配置自定义域名](../../操作指南/访问配置/访问域名配置/配置自定义域名/tccli%20操作.md)（page_id `53879`）
- [配置内网访问控制](../../操作指南/访问配置/访问网络控制/配置内网访问控制/tccli%20操作.md)（page_id `41838`）
- [同实例多地域复制镜像](../../操作指南/镜像分发/同实例多地域复制镜像/tccli%20操作.md)（page_id `52095`）
- [混合云下的多平台镜像数据同步复制](https://cloud.tencent.com/document/product/1141/60740)
- [全球多地域间同步镜像实现就近访问](https://cloud.tencent.com/document/product/1141/61458)

## 控制台替代

登录 [容器镜像服务控制台](https://console.cloud.tencent.com/tcr) → 选择**实例管理** → 购买企业版实例并初始化 → 选择左侧**域名管理** → 添加自定义域名及证书 → 选择左侧**访问控制** → 接入私有网络 VPC → 在 [云联网控制台](https://console.cloud.tencent.com/vpc/ccn) 关联 VPC → 在 [Private DNS 控制台](https://console.cloud.tencent.com/privatedns) 配置私有域解析。
