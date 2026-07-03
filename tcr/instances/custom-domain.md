---
doc_type: How-to
subtype: 6A
fused: false
---
# 自定义域名

> 为 TCR 企业版实例绑定自定义域名，用公司自有域名访问镜像仓库。
> 控制台: [容器镜像服务 - 实例 - 自定义域名](https://console.cloud.tencent.com/tcr/repository)

## 概述

默认情况下 TCR 实例通过 `*.tencentcloudcr.com` 域名访问。绑定自定义域名后，可用 `registry.company.com` 等自有域名推拉镜像，便于企业内网 DNS 统一管理与证书管控。

自定义域名需配套 SSL 证书（HTTPS），证书从 SSL 证书服务获取。

## 决策依据

### 域名与证书匹配

| 项 | 要求 | 说明 |
|:---|:-----|:-----|
| 域名 | 已备案（中国大陆实例） | `DomainName` 必须是已备案的域名 |
| 证书 | SSL 证书，域名匹配 | `CertificateId` 对应的证书必须覆盖该域名 |
| 证书来源 | SSL 证书服务 | `tccli ssl DescribeCertificates` 获取 ID |

> ⚠️ `CertificateId` 必须是 SSL 服务中真实存在的证书，且证书域名需匹配 `DomainName`。传不存在的证书 ID 返回 `FailedOperation.CertificateNotFound 证书不存在`（此校验先于 CAM 权限发生）。

### 是否用自定义域名

若仅需内网访问，用默认域名 + VPC 内网端点即可（见 [访问管理](manage-access.md)）；需对外暴露或统一企业域名时才配自定义域名。

## 关键字段

| 参数 | 所属 Action | 必填 | 说明 |
|:-----|:-----------|:----:|:-----|
| `RegistryId` | 全部 | 是 | TCR 实例 ID |
| `DomainName` | Create/Delete | 是 | 自定义域名（已备案） |
| `CertificateId` | Create/Delete | 是 | SSL 证书 ID（SSL 服务） |
| `Limit`/`Offset` | Describe | 否 | 分页 |

## 操作步骤

### 步骤 1：查询可用 SSL 证书

```bash
tccli ssl DescribeCertificates --region <REGION> --filter "Certificates[].{id:CertificateId,name:Alias,domain:Domain}" --output table
# expected: exit 0, 列出所有证书及绑定域名
```
```
id            name                domain
cert-example  company-cert        registry.example.com
cert-example2 wildcard-cert       *.example.com
```

> 选一个域名匹配 `<DOMAIN_NAME>` 的证书，记下其 `CertificateId`。若证书不匹配，先在 SSL 服务申请/上传覆盖该域名的证书。

### 步骤 2：绑定自定义域名

```bash
tccli tcr CreateInstanceCustomizedDomain --region <REGION> \
  --RegistryId <REGISTRY_ID> \
  --DomainName <DOMAIN_NAME> \
  --CertificateId <CERTIFICATE_ID>
# expected: exit 0
```

### 步骤 3：查询确认绑定

```bash
tccli tcr DescribeInstanceCustomizedDomain --region <REGION> --RegistryId <REGISTRY_ID>
# expected: exit 0, DomainInfoList 含新域名
```
```json
{
    "DomainInfoList": [],
    "TotalCount": 0,
    "RequestId": "..."
}
```

> 上图为空结果示例（未绑定时）。绑定成功后 `DomainInfoList` 含域名对象，`TotalCount >= 1`。

### 步骤 4：配置 DNS 解析

在域名 DNS 服务商将 `<DOMAIN_NAME>` CNAME 到 TCR 实例的默认域名（`PublicDomain`，见 `DescribeInstances` 输出）。

## 验证

| 维度 | 命令 | 期望 |
|:-----|:-----|:-----|
| 域名已绑定 | `DescribeInstanceCustomizedDomain --RegistryId <REGISTRY_ID>` | `DomainInfoList` 含域名 |
| 证书有效 | `tccli ssl DescribeCertificates --filter "Certificates[?CertificateId=='<CERTIFICATE_ID>']"` | 证书未过期 |
| DNS 解析生效 | `nslookup <DOMAIN_NAME>` 或 `dig <DOMAIN_NAME>` | 解析到 TCR 实例 |
| 域名可访问 | `docker login <DOMAIN_NAME>` | 登录成功 |

| 占位符 | 含义 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<REGISTRY_ID>` | TCR 实例 ID | 企业版实例 | `tccli tcr DescribeInstances --region <REGION>` |
| `<REGION>` | 地域 | 如 `ap-guangzhou` | `tccli tcr DescribeRegions` |
| `<DOMAIN_NAME>` | 自定义域名 | 已备案，匹配证书 | 自有域名 |
| `<CERTIFICATE_ID>` | SSL 证书 ID | 覆盖该域名 | `tccli ssl DescribeCertificates` |

## 清理

```bash
tccli tcr DeleteInstanceCustomizedDomain --region <REGION> \
  --RegistryId <REGISTRY_ID> --DomainName <DOMAIN_NAME> --CertificateId <CERTIFICATE_ID>
# expected: exit 0
```

> `DeleteInstanceCustomizedDomain` 需同时传 `DomainName` 和 `CertificateId`（与 Create 同参）。

## 故障恢复

| 现象 | 根因 | 修复 |
|:-----|:-----|:-----|
| `FailedOperation.CertificateNotFound 证书不存在` | `CertificateId` 不是 SSL 服务中真实证书 | `tccli ssl DescribeCertificates` 查真实 ID |
| 证书域名不匹配 | 证书未覆盖 `<DOMAIN_NAME>` | 用泛域名证书或申请匹配域名的证书 |
| 域名访问超时 | DNS 未解析到 TCR 实例 | CNAME 到实例 `PublicDomain`，等 DNS 生效 |
| `docker login` 报证书错误 | 证书过期 / DNS 未生效 | 续期证书，重新绑定；确认 DNS 解析 |
| 域名无法备案 | 域名未在工信部备案（大陆实例） | 完成备案，或用境外地域实例 |

> 错误样本：传不存在的 `CertificateId` → `code:FailedOperation.DependenceError ... CertificateNotFound, Message:证书不存在`（校验先于 CAM 权限）。

## 下一步

- 实例访问凭证（Token/VPC）：[访问管理](manage-access.md)
- 创建企业版实例：[创建实例](create.md)
- 镜像推送拉取：[推送与拉取镜像](../images/push-pull.md)
