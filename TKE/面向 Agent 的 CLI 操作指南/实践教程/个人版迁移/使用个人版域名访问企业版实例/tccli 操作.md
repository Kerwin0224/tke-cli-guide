# 使用个人版域名访问企业版实例（tccli）

> 对照官方：[使用个人版域名访问企业版实例](https://cloud.tencent.com/document/product/1141/82855) · page_id `82855`

## 概述

使用个人版服务域名（`ccr.ccs.tencentyun.com`）访问企业版实例内镜像，无需变更 TKE 集群、CI/CD 平台内已有镜像仓库地址及访问凭证配置。通过在私有网络内将个人版域名解析至企业版实例的内网访问入口，客户端使用个人版域名 + 个人版凭证即可拉取/推送企业版实例内镜像。

支持**智能回源**：企业版实例内无对应镜像时，自动回源请求个人版服务内对应镜像（需同命名空间、同仓库名且同 tag 完全不存在于企业版实例才回源）。

## 前置条件

- [环境准备](../../../环境准备.md)
- 已 [购买企业版实例](../../../操作指南/创建企业版实例/tccli%20操作.md)，实例状态为 `Running`
- 已 [将个人版数据迁移至企业版](../个人版迁移至企业版完全指南/tccli%20操作.md)
- 已配置[内网访问控制](../../../操作指南/访问配置/访问网络控制/配置内网访问控制/tccli%20操作.md)，目标 VPC 已接入企业版实例
- 已开通 [私有域解析 PrivateDNS](https://console.cloud.tencent.com/privatedns) 服务（免费，不会在 PrivateDNS 产品侧产生额外费用）
- 已有 TKE 集群（含 TKE Serverless 集群），集群所在 VPC 已接入企业版实例

验证基础环境：

```bash
tccli tcr DescribeInstances --Registryids '["<RegistryId>"]' --region ap-guangzhou --output json
```

```json
{
  "TotalCount": 0,
  "Registries": [],
  "RegistryId": "<RegistryId>",
  "RegistryName": "<RegistryName>",
  "RegistryType": "<RegistryType>",
  "Status": "<Status>",
  "PublicDomain": "<PublicDomain>",
  "CreatedAt": "<CreatedAt>"
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 查看企业版实例域名 | `tccli tcr DescribeInstances` | 是 |
| 查看内网访问链路及 AccessIp | `tccli tcr DescribeInternalEndpoints --RegistryId <RegistryId>` | 是 |
| 查看内网 DNS 解析状态 | `tccli tcr DescribeInternalEndpointDnsStatus --cli-input-json file://dns-status.json` | 是 |
| 开启内网自动解析（默认域名） | `tccli tcr CreateInternalEndpointDns --UsePublicDomain true` | 是 |
| 新建私有域（PrivateDNS） | PrivateDNS 控制台操作；无 `tccli` 直接等价接口 | — |
| 配置 CNAME 解析记录 | PrivateDNS 控制台操作；无 `tccli` 直接等价接口 | — |
| 获取临时登录指令 | `tccli tcr CreateInstanceToken --TokenType temp --ExpTime 1` | 否（每次生成新 Token） |
| 验证镜像拉取 | `docker pull ccr.ccs.tencentyun.com/<namespace>/<repo>:<tag>` | 是 |
| 验证镜像推送 | `docker push ccr.ccs.tencentyun.com/<namespace>/<repo>:<tag>` | 否 |

## 操作步骤

### 基础知识

个人版与企业版域名对比：

| 版本 | 域名类型 | 示例 |
|------|---------|------|
| 个人版（主服务地域） | 公网/内网共用 | `ccr.ccs.tencentyun.com` |
| 个人版（其他地域） | 独立域名 | `hkccr.ccs.tencentyun.com`（中国香港） |
| 企业版默认域名 | 公网域名 | `<实例名>.tencentcloudcr.com` |
| 企业版内网专用域名 | VPC 内网 | `<实例名>-vpc.tencentcloudcr.com` |
| 企业版自定义域名 | 自定义 | `xxx-company.com` |

以广州地域个人版命名空间 `team-a` 下 `nginx:latest` 迁移至企业版实例 `company-a` 为例：

- 个人版访问地址：`ccr.ccs.tencentyun.com/team-a/nginx:latest`
- 企业版访问地址：`company-a.tencentcloudcr.com/team-a/nginx:latest`

### 原理介绍

新建企业版实例时会默认下发支持个人版域名的证书，并支持处理个人版的认证鉴权请求。

在私有网络内将个人版域名解析至企业版内网访问入口后：
1. 客户端访问 `ccr.ccs.tencentyun.com` → DNS 

```json
{
  "TotalCount": 0,
  "Registries": [],
  "RegistryId": "<RegistryId>",
  "RegistryName": "<RegistryName>",
  "RegistryType": "<RegistryType>",
  "Status": "<Status>",
  "PublicDomain": "<PublicDomain>",
  "CreatedAt": "<CreatedAt>"
}
```解析至企业版内网 IP
2. 请求到达企业版实例 → 自动处理个人版认证鉴权
3. 企业版实例查找对应命名空间/仓库 → 返回镜像；若不存在则回源至个人版查询

取消私有网络内的强制解析后，恢复正常访问个人版服务。

### 确认基础环境

```bash
# 1. 确认实例状态 Running
tccli tcr DescribeInstances --Registryids '["<RegistryId>"]' --region ap-guangzhou --output json
```

Output:

```json
{
    "Response": {
        "Registries": [
            {
                "RegistryId": "tcr-example",
                "RegistryName": "tcr-example",
                "Status": "Running",
                "PublicDomain": "tcr-example.tencentcloudcr.com",
                "InternalEndpoint": "10.0.0.100"
            }
        ],
        "TotalCount": 1,
        "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
    }
}
```

```bash
# 2. 确认内网访问链路正常
tccli tcr DescribeInternalEndpoints --RegistryId <RegistryId> --region ap-guangzhou --output json
```

Output:

```json
{
    "Response": {
        "AccessVpcSet": [
            {
                "AccessIp": "10.0.1.5",
                "VpcId": "vpc-example",
                "SubnetId": "subnet-example",
                "Status": "Running"
            }
        ],
        "TotalCount": 1,
        "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
    }
}
```

记录 `AccessIp`，后续配置 PrivateDNS 解析时需作为 CNAME 记录值的目标。

### 配置私有域解析

> **说明**：PrivateDNS 私有域及解析记录的创建/管理需通过 [PrivateDNS 控制台](https://console.cloud.tencent.com/privatedns) 操作，无对应的 `tccli` 接口。

**步骤 1：创建私有域**

1. 前往 [私有域解析 PrivateDNS](https://console.cloud.tencent.com/privatedns) 控制台
2. 单击**新建私有域**：
   - **域名**：`tencentyun.com`
   - **关联 VPC**：选择企业版实例已接入的 VPC
   - **子域名递归解析**：保持开启（开启后未显式配置的 `*.tencentyun.com` 子域名仍走公网 DNS 解析，避免影响其他腾讯云服务）
   - 其他选项保持默认

**步骤 2：配置解析记录**

1. 进入新建的私有域 `tencentyun.com` 详情页
2. 在**解析记录**中单击**添加记录**：

| 字段 | 值（主服务地域） | 值（其他地域，如中国香港） |
|------|----------------|--------------------------|
| 主机记录 | `ccr.ccs` | `hkccr.ccs` |
| 记录类型 | `CNAME` | `CNAME` |
| 记录值 | 企业版实例域名（如 `tcr-example.tencentcloudcr.com`） | 企业版实例域名 |

> 记录值使用企业版实例的默认域名或自定义域名均可，需确认该域名在产品控制台内已配置自动解析。

3. 其他选项保持默认，保存。

### 确认 DNS 解析状态（企业版侧）

企业版实例侧也需确认内网自动解析已开启：

```json title="examples/tcr-dns-status.json"
{
    "VpcSet": [
        {
            "InstanceId": "<RegistryId>",
            "VpcId": "<VpcId>",
            "EniLBIp": "<AccessIp>",
            "UsePublicDomain": true
        }
    ]
}
```

```bash
tccli tcr DescribeInternalEndpointDnsStatus \
  --cli-input-json file://examples/tcr-dns-status.json \
  --region ap-guangzhou \
  --output json
```

Output:

```json
{
    "Response": {
        "VpcSet": [
            {
                "VpcId": "vpc-example",
                "EniLBIp": "10.0.1.5",
                "Status": "Enabled",
                "UsePublicDomain": true
            }
        ],
        "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
    }
}
```

若 `Status` 为 `Disabled`，开启自动内网解析：

```bash
tccli tcr CreateInternalEndpointDns \
  --InstanceId <RegistryId> \
  --VpcId <VpcId> \
  --EniLBIp "<AccessIp>" \
  --UsePublicDomain true \
  --region ap-guangzhou \
  --output json
```

### 验证访问效果

#### 场景 1：拉取已迁移至企业版实例内镜像

1. 确认镜像已迁移至企业版实例：`tcr-example.tencentcloudcr.com/team-a/nginx:latest`
2. 使用个人版域名拉取：

```bash
docker login ccr.ccs.tencentyun.com
# 用户名: <UIN>, 密码: <个人版密码>

docker pull ccr.ccs.tencentyun.com/team-a/nginx:latest
```

Output:

```text
latest: Pulling from team-a/nginx
Digest: sha256:e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855
Status: Image is up to date for ccr.ccs.tencentyun.com/team-a/nginx:latest
```

3. 集群中通过工作负载拉取：
   - 镜像地址：`ccr.ccs.tencentyun.com/team-a/nginx:latest`
   - 访问凭证：`qcloudregistrykey`（个人版凭证）

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-test
  template:
    metadata:
      labels:
        app: nginx-test
    spec:
      imagePullSecrets:
      - name: qcloudregistrykey
      containers:
      - name: nginx
        image: ccr.ccs.tencentyun.com/team-a/nginx:latest
```

```bash
kubectl apply -f nginx-test.yaml
```

Output:

```text
deployment.apps/nginx-test created
```

```bash
kubectl get pods
```

Output:

```text
NAME                          READY   STATUS    RESTARTS   AGE
nginx-test-xxxxxxxxxx-xxxxx   1/1     Running   0          30s
```

#### 场景 2：拉取尚未迁移至企业版实例内镜像（智能回源）

个人版已有镜像 `ccr.ccs.tencentyun.com/team-b/apache:latest`，尚未同步至企业版实例。

使用个人版域名拉取时，企业版实例无对应仓库（`team-b/apache`），自动回源至个人版拉取：

```bash
docker pull ccr.ccs.tencentyun.com/team-b/apache:latest
```

Output:

```text
latest: Pulling from team-b/apache
...
Status: Downloaded newer image for ccr.ccs.tencentyun.com/team-b/apache:latest
```

> **限制**：如果企业版仓库已存在但某个具体 tag 不存在，**不会**回源至个人版拉取。回源仅在仓库名完全不存在于企业版实例时生效。

#### 场景 3：推送镜像至企业版实例

使用个人版域名推送镜像至企业版实例：

```bash
docker tag nginx:latest ccr.ccs.tencentyun.com/team-a/nginx:v2
docker push ccr.ccs.tencentyun.com/team-a/nginx:v2
```

Output:

```text
The push refers to repository [ccr.ccs.tencentyun.com/team-a/nginx]
v2: digest: sha256:... size: 528
```

> 若企业版实例内已有 `team-a` 命名空间则可推送成功，否则将直接报错失败（个人版域名模式**不会**回源创建命名空间）。

## 验证

### Control plane (tccli)

```bash
# 确认实例状态 Running
tccli tcr DescribeInstances --Registryids '["<RegistryId>"]' --region ap-guangzhou --output json

# 确认内网链路 Running
tccli tcr DescribeInternalEndpoints --RegistryId <RegistryId> --region ap-guangzhou --output json

# 确认 DNS 解析已启用
tccli tcr DescribeInternalEndpointDnsStatus \
  --cli-input-json file://examples/tcr-dns-status.json \
  --region ap-guangzhou --output json
```

### Data plane

```bash
# 验证 Docker 可拉取已迁移镜像
docker pull ccr.ccs.tencentyun.com/<namespace>/<repo>:<tag>

# 验证集群 Pod 可正常运行
kubectl get pods -n <Namespace>

# 验证从集群节点可解析个人版域名
kubectl run dns-test --image=busybox:1.28 --rm -it --restart=Never -- nslookup ccr.ccs.tencentyun.com
# 期望: 返回企业版实例内网 IP (AccessIp)
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl` 连接超时/拒绝 | `tccli tke DescribeClusters` 返回集群 Running 但 kubectl 不可达 | CAM 策略 240463971 拒绝公网访问 APIServer 端点 | 通过 VPN/IOA 接入内网后执行 kubectl 命令 |
| `docker pull ccr.ccs.tencentyun.com/...` 超时 | 节点执行 `nslookup ccr.ccs.tencentyun.com` 验证解析 | PrivateDNS 未创建 `tencentyun.com` 私有域或 CNAME 指向错误 | 创建私有域 `tencentyun.com` 并关联正确 VPC；`ccr.ccs` 主机记录 CNAME 指向企业版实例域名 |
| `docker pull` 返回个人版镜像而非企业版 | `tccli tcr DescribeRepositories` 确认企业版仓库是否存在 | 企业版实例内存在对应仓库但 tag 不同，不触发回源 | 确认企业版仓库存在且 tag 一致；回源仅在仓库名完全不存在时生效 |
| 集群 Pod 仍从公网拉取个人版镜像 | 节点执行 `nslookup ccr.ccs.tencentyun.com` 验证解析 | 集群节点 VPC 未关联私有域或子域名递归解析未开启 | 确认集群节点 VPC 已关联 `tencentyun.com` 私有域；开启子域名递归解析 |
| Push 报 `denied` / `unauthorized` | `docker login ccr.ccs.tencentyun.com` 验证凭证 | 企业版实例内无对应命名空间或凭证错误 | 确认企业版实例内已有对应命名空间；使用正确的个人版用户名（UIN）和密码 |

## 清理

### Data plane (kubectl)

```

```text
# command executed successfully
```bash
kubectl delete deployment nginx-test -n <Namespace>
kubectl delete pod dns-test -n <Namespace> --ignore-not-found
```

### Control plane (tccli)

无企业版实例侧资源需清理（仅 DNS 配置变更）。若需恢复个人版直接访问，在 PrivateDNS 控制台删除 `ccr.ccs` 解析记录或删除整个私有域 `tencentyun.com`。

## 下一步

- [个人版迁移至企业版完全指南](../个人版迁移至企业版完全指南/tccli%20操作.md)（page_id `52292`） — 完整的迁移路径
- [TKE 集群使用 TCR 插件内网免密拉取容器镜像](https://cloud.tencent.com/document/product/1141/48184)
- [配置内网访问控制](../../../操作指南/访问配置/访问网络控制/配置内网访问控制/tccli%20操作.md)

## 控制台替代

登录 [容器镜像服务控制台](https://console.cloud.tencent.com/tcr) 完成企业版实例创建与数据迁移 → 前往 [PrivateDNS 控制台](https://console.cloud.tencent.com/privatedns) 创建 `tencentyun.com` 私有域并添加 `ccr.ccs` CNAME 记录指向企业版实例域名 → 在 TKE 集群内保持已有 `ccr.ccs.tencentyun.com` 镜像地址与 `qcloudregistrykey` 凭证即可访问企业版实例。
