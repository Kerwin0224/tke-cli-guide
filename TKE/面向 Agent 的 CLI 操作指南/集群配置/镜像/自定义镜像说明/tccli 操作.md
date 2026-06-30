# 自定义镜像说明

> 对照官方：[自定义镜像说明](https://cloud.tencent.com/document/product/457/39563) · page_id `39563`

## 概述

自定义镜像是用户基于 TKE **基础镜像**（即 [镜像概述](../镜像概述/tccli%20操作.md) 中的公共镜像）在 CVM 侧制作的系统镜像。制作完成后可在 TKE 创建集群或节点池时选用。自定义镜像未经平台兼容性适配，需用户自行保证在 Kubernetes 下可用，TKE 原则上不提供 SLA。

**核心 API**：使用 `ModifyClusterImage` 修改集群的默认操作系统镜像，影响后续新增或重装节点使用的 OS。存量节点 OS 不受影响。

## 前置条件

- [环境准备](../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置，region 为目标地域

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:ModifyClusterImage, tke:DescribeOSImages, tke:DescribeClusters
# 验证：执行 DescribeClusters 确认权限
tccli tke DescribeClusters --region <REGION>
# expected: exit 0，返回集群列表（可为空）

# 验证 DescribeOSImages 权限
tccli tke DescribeOSImages --region <REGION>
# expected: exit 0，返回 OSImageSeriesSet（可为空）
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

### 资源检查

```bash
# 4. 确认目标集群存在且状态正常
tccli tke DescribeClusters --region <REGION> --ClusterIds '["<CLUSTER_ID>"]'
# expected: TotalCount >= 1，ClusterStatus 为 Running

# 5. 确认基础镜像在 TKE 目录中在线
tccli tke DescribeOSImages --region <REGION> \
    | jq '[.OSImageSeriesSet[] | select(.OsName == "tlinux3.1x86_64" and .Status == "online")] | length'
# expected: 1（表示该基础镜像在线可用）

# 6. 确认自定义镜像已制作完成且与集群同地域
#    在 CVM 控制台或通过 DescribeImages 确认 ImageId 已存在
tccli cvm DescribeImages --region <REGION> \
    --Filters '[{"Name":"image-type","Values":["PRIVATE_IMAGE"]}]'
# expected: 返回自定义镜像列表，含目标 ImageId
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

## 关键字段说明

`ModifyClusterImage` 的主要参数：

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `ClusterId` | String | 是 | 目标集群 ID，格式 `cls-xxxxxxxx`。`tccli tke DescribeClusters --region <REGION>` 获取 | 集群不存在或不属于当前账号 → `InvalidParameter.ClusterId` |
| `ImageId` | String | 是 | CVM 侧镜像 ID，格式 `img-xxxxxxxx`。必须是基于 TKE 公共镜像制作的自定义镜像。公共镜像传 OsName（通过 CreateCluster），自定义镜像传 ImageId | 镜像不存在或地域不匹配 → `InvalidParameter.ImageId` |
| `OsName` | String | 否 | 公共镜像的 OsName（如 `tlinux4_x86_64_public`），与 ImageId 二选一 | 与 ImageId 同时传入 → 参数冲突 |
| `ImageType` | String | 否 | 镜像类别：`PUBLIC_IMAGE`（公共镜像）/ `PRIVATE_IMAGE`（自定义镜像）。若传 ImageId 则此项可省略 | 类型与镜像不匹配 → `InvalidParameter.ImageType` |

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看镜像列表 | DescribeImages | 是 |
| 查看自定义镜像 | DescribeOSImages | 是 |
## 操作步骤

### 步骤 1：制作自定义镜像（CVM 控制台）

**使用须知：**

- 仅支持**同类型**操作系统制作（如 CentOS 基础镜像制作 CentOS 类自定义镜像）
- **必须**使用 TKE 提供的基础镜像制作，否则 TKE 控制台不会显示
- 制作前**请勿**随意修改 `/etc/fstab`
- 制作后请及时清理 `/var/lib/cloud` 目录

```bash
# 清理 cloud-init 残留数据（在制作镜像的 CVM 上执行）
curl --proto '=https' --tlsv1.2 -sSf https://mirrors.tencent.com/install/tke/clean-node.sh | bash
# expected: exit 0，清理完成
```

**制作步骤（在 CVM 控制台完成）：**
1. 登录 [云服务器控制台](https://console.cloud.tencent.com/cvm/instance/create)，使用 TKE 基础镜像创建 CVM 实例
2. 登录 CVM 实例，执行自定义配置（如安装特定软件包、写入测试文件）
3. 在 CVM 控制台对该实例执行「创建自定义镜像」完成制作
4. 记录生成的 ImageId（格式 `img-xxxxxxxx`）

### 步骤 2：确认自定义镜像可在 TKE 使用

制作完成后，确认自定义镜像基于 TKE 基础镜像：

```bash
# 查询自定义镜像详情
tccli cvm DescribeImages --region <REGION> \
    --ImageIds '["<IMAGE_ID>"]'
# expected: exit 0，返回镜像详情
```

**预期输出**：

```json
{
    "ImageSet": [
        {
            "ImageId": "img-example",
            "ImageName": "CUSTOM_IMAGE_NAME",
            "ImageType": "PRIVATE_IMAGE",
            "ImageState": "NORMAL",
            "Platform": "TencentOS",
            "Architecture": "x86_64",
            "CreatedTime": "..."
        }
    ],
    "RequestId": "..."
}
```

### 步骤 3：修改集群默认镜像

#### 选择依据

- **公共镜像 vs 自定义镜像**：若仅需调整 OS 版本（如从 TencentOS 3.1 升级到 4），直接使用 `ModifyClusterImage` 传入新的公共镜像 `OsName`。若需使用包含定制软件/配置的镜像，传入自定义镜像 `ImageId`。
- **影响范围**：修改集群镜像**仅影响后续新增节点或重装节点**。存量运行中的节点不受影响，如需统一需逐台重装或通过节点池滚动更新。
- **运行时兼容性**：镜像修改不改变集群已有的容器运行时配置。自定义镜像若预装了容器运行时，可能与集群配置冲突 — 建议自制镜时不预装运行时。

#### 使用公共镜像修改

```bash
tccli tke ModifyClusterImage --region <REGION> \
    --ClusterId <CLUSTER_ID> \
    --ImageId <PUBLIC_IMAGE_ID>
# expected: exit 0
```

**预期输出**：

```json
{
    "RequestId": "..."
}
```

#### 使用自定义镜像修改

```bash
tccli tke ModifyClusterImage --region <REGION> \
    --ClusterId <CLUSTER_ID> \
    --ImageId <CUSTOM_IMAGE_ID> \
    --ImageType PRIVATE_IMAGE
# expected: exit 0
```

**预期输出**：

```json
{
    "RequestId": "..."
}
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `CLUSTER_ID` | 目标集群 ID | 格式 `cls-xxxxxxxx` | `tccli tke DescribeClusters --region <REGION>` |
| `PUBLIC_IMAGE_ID` | TKE 公共镜像的 ImageId | 格式 `img-xxxxxxxx` | `tccli tke DescribeOSImages --region <REGION>` 中的 `ImageId` |
| `CUSTOM_IMAGE_ID` | 自定义镜像的 ImageId | 格式 `img-xxxxxxxx`，须为 PRIVATE_IMAGE | CVM 控制台制作完成后生成 |
| `REGION` | 地域 | 须与集群和镜像同地域 | `tccli configure list` 查看当前地域 |

### 步骤 4：验证镜像已生效

```bash
tccli tke DescribeClusters --region <REGION> \
    --ClusterIds '["<CLUSTER_ID>"]' \
    | jq '.Clusters[0] | {ClusterId, ClusterOs, OsCustomizeType, ImageId}'
# expected: ClusterOs 或 ImageId 与修改参数一致
```

**预期输出**（使用自定义镜像后）：

```json
{
    "ClusterId": "cls-example",
    "ClusterOs": "img-example",
    "OsCustomizeType": "CUSTOMIZE",
    "ImageId": "img-example"
}
```

## 验证

```bash
# 1. 确认镜像修改已生效
tccli tke DescribeClusters --region <REGION> \
    --ClusterIds '["<CLUSTER_ID>"]' \
    | jq '.Clusters[0].OsCustomizeType'
# expected: "CUSTOMIZE"（自定义镜像）或 "GENERAL"（公共镜像）

# 2. 确认 ImageId 与传入值一致
tccli tke DescribeClusters --region <REGION> \
    --ClusterIds '["<CLUSTER_ID>"]' \
    | jq '.Clusters[0].ImageId'
# expected: 返回修改时传入的 ImageId

# 3. 新增节点验证（可选，如需确认镜像实际可用）
#    通过节点池扩容一台新节点，登录后验证自定义内容
tccli tke DescribeClusterInstances --region <REGION> \
    --ClusterId <CLUSTER_ID> \
    | jq '.InstanceSet[-1:][] | .InstanceId'
# expected: 返回最新节点的 InstanceId，可用于登录验证
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

## 清理

> **警告**：`ModifyClusterImage` 是幂等操作，修改本身不产生额外资源费用。但若为验证镜像而创建了测试节点，需清理对应 CVM 实例，否则将持续产生 CVM 费用。

### 1. 清理前状态检查

```bash
# 确认集群当前状态和节点信息
tccli tke DescribeClusters --region <REGION> \
    --ClusterIds '["<CLUSTER_ID>"]' \
    | jq '.Clusters[0] | {ClusterId, ClusterNodeNum, ClusterOs}'
# expected: 确认是目标集群，记录节点数
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

### 2. 删除测试节点（如适用）

```bash
# 若通过节点池扩容了测试节点，缩容节点池
tccli tke ModifyClusterNodePool --region <REGION> \
    --ClusterId <CLUSTER_ID> \
    --NodePoolId <NODE_POOL_ID> \
    --DesiredNodesNum <DESIRED_NODES_NUM>
# expected: exit 0
```

### 3. 删除自定义镜像（CVM 控制台，可选）

在 [CVM 控制台 → 镜像](https://console.cloud.tencent.com/cvm/image) 中删除不再使用的自定义镜像。

> **注意**：自定义镜像被集群引用时，不建议直接删除。先确认集群已不再使用该镜像（`DescribeClusters` 查看 `ImageId`）。

### 4. 验证已清理

```bash
# 确认集群镜像已恢复为目标状态
tccli tke DescribeClusters --region <REGION> \
    --ClusterIds '["<CLUSTER_ID>"]' \
    | jq '.Clusters[0] | {ClusterOs, ImageId}'
# expected: ImageId 与预期值一致
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

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `ModifyClusterImage` 返回 `InvalidParameter.ClusterId` | `tccli tke DescribeClusters --region <REGION> --ClusterIds '["<CLUSTER_ID>"]'` 确认集群存在 | 集群 ID 格式错误或集群不属于当前账号/地域 | 用 `tccli tke DescribeClusters --region <REGION>` 列出全部集群确认正确的 ClusterId |
| `ModifyClusterImage` 返回 `InvalidParameter.ImageId` | `tccli cvm DescribeImages --region <REGION> --ImageIds '["<IMAGE_ID>"]'` 确认镜像存在 | 镜像 ID 不存在、已删除、或地域与集群不一致 | 确认镜像与集群在同一地域；确认 ImageId 格式为 `img-xxxxxxxx`；公共镜像 ImageId 用 `tccli tke DescribeOSImages --region <REGION>` 获取 |
| `ModifyClusterImage` 返回 `InvalidParameter.ImageType` | 检查传入的 `--ImageType` 与实际镜像类型 | 传入的 `ImageType` 与镜像实际类型不匹配（如 PRIVATE_IMAGE 镜像传了 PUBLIC_IMAGE） | 用 `tccli cvm DescribeImages --region <REGION> --ImageIds '["<IMAGE_ID>"]'` 查看 `ImageType`，传入对应值 |
| `ModifyClusterImage` 返回 `UnauthorizedOperation` | `tccli configure list` 检查当前凭据 | 子账号缺少 `tke:ModifyClusterImage` 权限（此为环境限制，非命令错误） | 联系主账号授予 `QcloudTKEFullAccess` 或最小权限 `tke:ModifyClusterImage` |

### 修改成功但节点未使用新镜像

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `ModifyClusterImage` 成功，但新增节点仍使用旧镜像 | `tccli tke DescribeClusters --region <REGION> --ClusterIds '["<CLUSTER_ID>"]' \| jq '.Clusters[0].ClusterOs'` 确认集群级镜像已更新 | 节点池级别的 `OsName`/`ImageId` 覆盖了集群级别设置 | 检查并更新节点池镜像配置：`tccli tke DescribeClusterNodePools --region <REGION> --ClusterId <CLUSTER_ID> \| jq '.NodePoolSet[] \| {Name, OsName, ImageId}'`；若不一致则修改节点池 |
| 自定义镜像在集群创建向导中不可选 | `tccli tke DescribeOSImages --region <REGION> \| jq '[.OSImageSeriesSet[] \| .OsName]'` 对比基础镜像列表 | 制作自定义镜像时未使用 TKE 基础镜像，或基础镜像已下线 | 使用 TKE 公共镜像重新制作自定义镜像。确认基础镜像 `Status` 为 `online` |
| 使用自定义镜像的节点初始化失败 | 登录 CVM 控制台查看节点系统日志 | 自定义镜像预装了容器运行时、修改了 `/etc/fstab`、或 `/var/lib/cloud` 残留导致 `cloud-init` 失败 | 清理 `/var/lib/cloud` 目录后重新制作镜像；切勿预装运行时；切勿 `chattr +i /etc/resolv.conf` |

## 下一步

- [镜像概述](../镜像概述/tccli%20操作.md) — TKE 支持的公共镜像完整列表与查询方式
- [创建集群](../../集群管理/创建集群/tccli%20操作.md) — 创建集群时选择自定义镜像
- [更改集群操作系统](../../集群管理/更改集群操作系统/tccli%20操作.md) — 修改存量集群的 OS
- [TencentOS Rainbow 镜像介绍](../TencentOS%20Rainbow/镜像介绍/tccli%20操作.md) — 云原生轻量级容器操作系统
- [CVM 自定义镜像](https://cloud.tencent.com/document/product/213/4942) — CVM 侧制作自定义镜像官方文档

## 控制台替代

[CVM 控制台](https://console.cloud.tencent.com/cvm) 制作镜像；[TKE 创建集群](https://console.cloud.tencent.com/tke2/cluster/create) 选择「自定义镜像」提供方。
