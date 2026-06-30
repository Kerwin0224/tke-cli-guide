# 个人版迁移至企业版完全指南（tccli）

> 对照官方：[个人版迁移至企业版完全指南](https://cloud.tencent.com/document/product/1141/52292) · page_id `52292`

## 概述

将容器镜像数据及业务配置从 TCR 个人版服务（CCR）全量迁移至企业版服务。迁移分为两个阶段：

1. **数据迁移**：利用官方迁移工具（`ccr2tcr` Docker 镜像）将个人版内所有命名空间与镜像仓库导入企业版实例。
2. **业务迁移**：切换集群/CI-CD 平台中的镜像地址与访问凭证至企业版，或通过"个人版域名兼容功能"零配置切换。

个人版与企业版服务区别参见[产品服务层级与容量限制](https://cloud.tencent.com/document/product/1141/104731)。

## 前置条件

- [环境准备](../../../环境准备.md)
- 已开通并使用个人版服务，迁移操作账号具有拉取个人版镜像仓库内所有镜像的权限（参见[个人版授权方案示例](https://cloud.tencent.com/document/product/1141/41409)）
- 已 [购买企业版实例](../../../操作指南/创建企业版实例/tccli%20操作.md)，迁移操作账号具有推送镜像到该企业版实例的权限（参见[企业版授权方案示例](https://cloud.tencent.com/document/product/1141/41417)）
- 建议在私有网络 VPC 内执行迁移任务以提升迁移速度并避免公网流量成本
  - VPC 内运行：在目标企业版实例的[内网访问控制](../../../操作指南/访问配置/访问网络控制/配置内网访问控制/tccli%20操作.md)中添加迁移工具运行服务器所在的私有网络
  - 公网环境运行：开启目标企业版实例的公网访问入口并放通访问来源（参见[公网访问控制](https://cloud.tencent.com/document/product/1141/41837)）
- 准备一台 CPU/内存配置较高、内网带宽高的云服务器 CVM，安装最新 Docker 客户端（推荐选择已安装 Docker 的操作系统）
- 获取 API 调用密钥：前往 [访问管理控制台](https://console.cloud.tencent.com/cam/capi) 新建或查看已有密钥

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
| 查看企业版实例列表 | `tccli tcr DescribeInstances` | 是 |
| 查看企业版实例状态 | `tccli tcr DescribeInstanceStatus --RegistryIds '["<RegistryId>"]'` | 是 |
| 查看内网访问链路 | `tccli tcr DescribeInternalEndpoints --RegistryId <RegistryId>` | 是 |
| 配置 VPC 内网访问 | `tccli tcr ManageInternalEndpoint --Operation Create --VpcId <VpcId> --SubnetId <SubnetId>` | 否 |
| 开启自动内网解析 | `tccli tcr CreateInternalEndpointDns --UsePublicDomain true` | 是 |
| 获取长期访问凭证 | `tccli tcr CreateInstanceToken --RegistryId <RegistryId> --TokenType longterm --Desc "migration"` | 否（每次生成新密码） |
| 查看访问凭证列表 | `tccli tcr DescribeInstanceToken --RegistryId <RegistryId>` | 是 |
| 数据迁移（Docker 工具） | `docker run … ccr2tcr /run` | 是（重试机制） |
| 切换镜像地址 | 修改 Kubernetes YAML 中 `image` 字段 | 否（需手动） |
| 切换访问凭证 | 修改 Kubernetes YAML 中 `imagePullSec

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
```ret` | 否（需手动） |
| 验证 Docker 登录企业版 | `docker login <EnterpriseDomain>` | 是 |

## 操作步骤

### 数据迁移

#### 准备基础运行环境

1. 确认企业版实例已创建且状态为 `Running`：

```bash
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
                "RegistryType": "basic",
                "Status": "Running",
                "PublicDomain": "tcr-example.tencentcloudcr.com",
                "InternalEndpoint": "10.0.0.100",
                "RegionName": "ap-guangzhou"
            }
        ],
        "TotalCount": 1,
        "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
    }
}
```

2. 确认实例内网访问链路已建立并正常运行：

```bash
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

若 `AccessVpcSet` 为空或 `Status` 非 `Running`，需要先配置内网访问链路及自动解析：

```bash
tccli tcr ManageInternalEndpoint \
  --RegistryId <RegistryId> \
  --Operation Create \
  --VpcId <VpcId> \
  --SubnetId <SubnetId> \
  --region ap-guangzhou \
  --output json

tccli tcr CreateInternalEndpointDns \
  --InstanceId <RegistryId> \
  --VpcId <VpcId> \
  --EniLBIp "<AccessIp>" \
  --UsePublicDomain true \
  --region ap-guangzhou \
  --output json
```

3. 在迁移专用 CVM 上执行 `docker login` 验证内网可正常访问实例：

```bash
docker login tcr-example.tencentcloudcr.com
# 输入用户名和密码（即企业版长期访问凭证）
```

> 登录成功提示 `Login Succeeded`。

#### 准备迁移配置信息

**获取个人版镜像仓库访问凭证：**
- 用户名：需迁移镜像所属腾讯云账号 UIN
- 密码：初始化个人版服务时所设密码。如忘记密码，前往 [容器镜像服务控制台](https://console.cloud.tencent.com/tcr/instance) > 实例列表 > 个人版实例 > 更多 > 重置登录密码

**获取企业版镜像仓库访问凭证：**

```bash
tccli tcr CreateInstanceToken \
  --RegistryId <RegistryId> \
  --TokenType longterm \
  --Desc "migration-token" \
  --region ap-guangzhou \
  --output json
```

Output:

```json
{
    "Response": {
        "Username": "100012345678",
        "Token": "eyJhbGciOi...",
        "ExpTime": 1735660800000,
        "TokenId": "tcr-token-example",
        "Desc": "migration-token",
        "Enabled": true,
        "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
    }
}
```

> **重要**：返回的 `Token` 即为登录密码，仅在创建时返回一次，请妥善保管。

该访问凭证用户需具备该实例的全读写权限。

**获取 API 调用凭证：**

迁移过程中需调用腾讯云 API 在企业版实例内自动创建命名空间及镜像仓库。前往 [访问管理控制台](https://console.cloud.tencent.com/cam/capi) > 访问密钥 > API 密钥管理，新建或查看已有 SecretId / SecretKey。

#### 下载并执行迁移工具

下载迁移专用容器镜像：

```bash
docker pull ccr.ccs.tencentyun.com/tcrimages/image-transfer:ccr2tcr
```

查看工具使用说明：

```bash
docker run --network=host --rm ccr.ccs.tencentyun.com/tcrimages/image-transfer:ccr2tcr /run --help
```

执行全量迁移（单次 docker run，退出即停止，需采集参数后手动填入）：

```bash
docker run --network=host --rm \
  ccr.ccs.tencentyun.com/tcrimages/image-transfer:ccr2tcr \
  /run \
  --tcrName <EnterpriseInstanceName> \
  --ccrRegionName ap-guangzhou \
  --tcrRegionName ap-shanghai \
  --ccrAuth <CCR-Username>:<CCR-Password> \
  --tcrAuth <TCR-Username>:<TCR-Password> \
  --ccrSecretId <CCR-SecretId> \
  --ccrSecretKey <CCR-SecretKey> \
  --tcrSecretId <TCR-SecretId> \
  --tcrSecretKey <TCR-SecretKey> \
  --tagNum 50
```

**参数说明：**

| 参数 | 说明 |
|------|------|
| `--tcrName` | 目标企业版实例的名称（即 `RegistryName`） |
| `--ccrRegionName` / `--tcrRegionName` | 实例地域。个人版 ccR 国内默认为 `ap-guangzhou`；企业版按实际地域填写，如 `ap-shanghai` |
| `--ccrAuth` / `--tcrAuth` | 镜像仓库访问凭证，格式 `username:password`。如包含特殊字符需转义处理（如 `?` 写为 `\\?`） |
| `--ccrSecretId` / `--ccrSecretKey` / `--tcrSecretId` / `--tcrSecretKey` | 腾讯云 API 调用密钥。同账号下迁移两者相同 |
| `--tagNum` | 仅迁移每个镜像仓库内最新 N 个版本镜像 |

> **注意**：同账号下迁移，ccr 及 tcr 调用密钥（SecretId/SecretKey）相同，但需分别填入。

#### 查看及确认运行结果

迁移时间与个人版内镜像仓库数量及总大小正相关。全量迁移成功后显示：

```text
################# Finished, 0 transfer jobs failed, 0 normal urlPair generate failed, 0 jobs generate failed #################
```

若出现失败数 > 0，重新运行迁移工具重试。持续失败可通过[在线咨询](https://cloud.tencent.com/online-service?from=doc_1141)申请协助。

### 业务迁移

#### 网络环境切换

企业版支持网络访问控制，需将集群所在 VPC 接入实例以允许内网访问。推荐使用内网访问方式：

```bash
tccli tcr ManageInternalEndpoint \
  --RegistryId <RegistryId> \
  --Operation Create \
  --VpcId <VpcId> \
  --SubnetId <SubnetId> \
  --region ap-guangzhou \
  --output json
```

> 配置完成后需在 VPC 内配置 DNS 自动解析或自行管理 DNS，参见[配置内网访问控制](../../../操作指南/访问配置/访问网络控制/配置内网访问控制/tccli%20操作.md)。

不推荐开启公网访问入口，建议仅在测试时或需面向外部团队分发镜像时开放，且配置访问白名单。

#### 复用个人版镜像配置

[使用个人版域名访问企业版实例](../使用个人版域名访问企业版实例/tccli%20操作.md) 支持通过配置私有域解析，无需变更 TKE 集群、CI/CD 平台内已有镜像仓库地址及访问凭证。配置后：

- 镜像地址不变：`ccr.ccs.tencentyun.com/namespace/repo:tag`
- 访问凭证不变：仍使用个人版的用户名（UIN）+ 密码

> 长期使用建议逐步将个人版访问配置切换至企业版配置，避免后续因个人版服务变更导致访问异常。

#### 使用企业版镜像配置

**访问地址切换：**

更新工作负载 YAML 中的 `image` 字段：

| 版本 | 镜像地址格式 | 示例 |
|------|-------------|------|
| 个人版 | `ccr.ccs.tencentyun.com/namespace/repo:tag` | `ccr.ccs.tencentyun.com/team-a/nginx:latest` |
| 企业版 | `<实例名>.tencentcloudcr.com/namespace/repo:tag` | `tcr-example.tencentcloudcr.com/team-a/nginx:latest` |

企业版支持：
- 自定义域名：`xxx-company.com/namespace/repo:tag`
- 多级路径：`company-a.tencentcloudcr.com/ns/sub01/sub02/repo:tag`

```yaml
# 示例：Deployment 中切换镜像地址
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
      - name: nginx
        # 个人版地址
        # image: ccr.ccs.tencentyun.com/team-a/nginx:latest
        # 企业版地址
        image: tcr-example.tencentcloudcr.com/team-a/nginx:latest
```

**访问凭证切换：**

更新工作负载 YAML 中的 `imagePullSecret` 字段：

| 版本 | 凭证方案 | 操作 |
|------|---------|------|
| 个人版 | 默认 `qcloudregistrykey` | 新建命名空间自动下发 |
| 企业版（推荐） | TCR 专用插件自动下发免密拉取 | 删除 YAML 中 `imagePullSecret` 已有值，保持为空（仅 TKE） |
| 企业版（手动） | 长期访问凭证下发至命名空间 | 创建 `Secret` 并引用 |

手动配置企业版访问凭证：

```bash
# 获取长期访问凭证
tccli tcr CreateInstanceToken \
  --RegistryId <RegistryId> \
  --TokenType longterm \
  --Desc "pull-token" \
  --region ap-guangzhou \
  --output json

# 在 Kubernetes 集群内创建 imagePullSecret
kubectl create secret docker-registry tcr-pull-secret \
  --docker-server=tcr-example.tencentcloudcr.com \
  --docker-username=<TCR-Username> \
  --docker-password=<TCR-Token> \
  -n <Namespace>
```

```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      imagePullSecrets:
      - name: tcr-pull-secret
      containers:
      - name: nginx
        image: tcr-example.tencentcloudcr.com/team-a/nginx:latest
```

### 企业版入门最佳实践

#### 多实例规划

根据实际业务在多个地域内创建一个或多个实例，配置同步复制策略，并使用自定义域名统一管理实例访问地址。参见：
- [TCR 企业版实例同步](https://cloud.tencent.com/document/product/1141/41945)
- [TCR 企业版实例复制](https://cloud.tencent.com/document/product/1141/52095)
- [从自建 Harbor 同步镜像到 TCR 企业版](https://cloud.tencent.com/document/product/1141/44970)

#### 安全合规

**操作合规：** 为使用企业版的子账号授予最小权限，支持仓库级权限配置。使用场景建议创建专用的长期访问凭证；临时访问实例建议使用临时访问凭证（1 小时有效）。

```bash
tccli tcr CreateInstanceToken \
  --RegistryId <RegistryId> \
  --TokenType temp \
  --ExpTime 1 \
  --Desc "temp-access" \
  --region ap-guangzhou \
  --output json
```

**网络安全：** 避免开通公网访问入口，避免非授权对象访问实例或产生公网流量费用。可选为实例绑定自定义域名，并管理该域名的公网及 VPC 内解析。

#### 镜像管理

- 使用命名空间隔离业务/团队，方便权限管理与同步配置
- 支持多级路径创建多层级仓库，避免创建过多命名空间
- 支持推送镜像自动创建仓库，可在命名空间层级配置公开/私有等默认属性
- 避免在生产环境使用 `latest` tag，建议使用 CI 工具为镜像自动化命名
- **注意**：删除指定版本镜像时，相同 digest 镜像将同时被删除

#### CI/CD

企业版 CI/CD 能力基于腾讯云 DevOps 产品（需开通），支持：
- 镜像构建（代码源可选 Github、Gitlab、工蜂、码云等）
- 交付流水线（自动部署镜像至集群、弹性集群及边缘集群）

对接外部服务可使用触发器（webhook）功能，推送镜像自动触发 webhook 通知。参见[管理触发器](https://cloud.tencent.com/document/product/1141/44369)。

## 验证

### Control plane (tccli)

| 验证项 | 命令 | 期望结果 |
|--------|------|---------|
| 实例存在且可用 | `DescribeInstances` | `Status: "Running"` |
| 内网链路已建立 | `DescribeInternalEndpoints` | `Status: "Running"`, `AccessIp` 非空 |
| 内网解析已启用 | `DescribeInternalEndpointDnsStatus` | `Status: "Enabled"` |
| 访问凭证有效 | `DescribeInstanceToken` | `Enabled: true`, `ExpTime` 未过期 |

### Data plane

```bash
# 验证 Docker 登录企业版实例
docker login <EnterpriseDomain>
# 期望: Login Succeeded

# 验证可拉取已迁移镜像
docker pull <EnterpriseDomain>/<Namespace>/<Repo>:<Tag>
# 期望: 下载进度正常完成

# 验证 Kubernetes 集群可部署
kubectl apply -f <deployment.yaml>
kubectl get pods -n <Namespace>
# 期望: Pod STATUS Running
```

```text
# command executed successfully
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl` 连接超时/拒绝 | `tccli tke DescribeClusters` 返回集群 Running 但 kubectl 不可达 | CAM 策略 240463971 拒绝公网访问 APIServer 端点 | 通过 VPN/IOA 接入内网后执行 kubectl 命令 |
| 迁移工具报 `failed` 非零 | 迁移工具输出查看失败 job 数 | 镜像仓库数量多或网络波动导致部分 job 失败 | 重新运行迁移工具重试；持续失败联系[在线咨询](https://cloud.tencent.com/online-service?from=doc_1141) |
| 迁移工具报认证错误 | `docker login` 分别测试个人版/企业版 | `--ccrAuth` 密码错误或 API 密钥失效 | 确认 `--ccrAuth` 密码正确（个人版设置页可重置）；确认 API 密钥在 CAM 中为已启用 |
| `CreateInstanceToken` 报 `UnauthorizedOperation` | `tccli tcr DescribeInstanceToken --RegistryId <RegistryId>` 验证 | 子账号缺少 `tcr:CreateInstanceToken` 权限 | 联系主账号授予 `QcloudTCRFullAccess` |
| 集群 Pod 报 `ImagePullBackOff` | `kubectl describe pod <pod>` 查看 Events | YAML 中 `image` 地址或 `imagePullSecret` 错误，凭证过期 | 检查 `image` 地址与 `imagePullSecret` 是否正确；确认凭证未过期 |

## 清理

### Control plane (tccli)

```bash
# 删除迁移专用长期访问凭证
tccli tcr DeleteInstanceToken \
  --RegistryId <RegistryId> \
  --TokenId "<TokenId>" \
  --region ap-guangzhou \
  --output json

# 如不再需要内网访问链路，依次关闭解析并删除链路
# tccli tcr DeleteInternalEndpointDns --InstanceId <RegistryId> --VpcId <VpcId> --EniLBIp "<AccessIp>" --UsePublicDomain true
# tccli tcr ManageInternalEndpoint --RegistryId <RegistryId> --Operation Delete --VpcId <VpcId> --SubnetId <SubnetId>
```

### Data plane

```bash
# 删除测试部署的工作负载
kubectl delete deployment <DeploymentName> -n <Namespace>
kubectl delete secret tcr-pull-secret -n <Namespace>

# Docker 清理
docker rmi <EnterpriseDomain>/<Namespace>/<Repo>:<Tag>
```

## 下一步

- [使用个人版域名访问企业版实例](../使用个人版域名访问企业版实例/tccli%20操作.md)（page_id `82855`） — 无需变更已有镜像地址配置
- [TKE 集群使用 TCR 插件内网免密拉取容器镜像](https://cloud.tencent.com/document/product/1141/48184)
- [配置内网访问控制](../../../操作指南/访问配置/访问网络控制/配置内网访问控制/tccli%20操作.md)
- [企业版入门最佳实践](https://cloud.tencent.com/document/product/1141/39287)

## 控制台替代

登录 [容器镜像服务控制台](https://console.cloud.tencent.com/tcr) → 完成个人版→企业版迁移的前置准备 → 参考[官方迁移指南](https://cloud.tencent.com/document/product/1141/52292) 使用迁移工具进行数据迁移 → 在 TKE 集群控制台中逐工作负载修改镜像地址及访问凭证。
