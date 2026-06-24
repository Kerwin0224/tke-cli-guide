# TCR 实例后端存储切换（tccli）

> 对照官方：[TCR 实例后端存储切换](https://cloud.tencent.com/document/product/1141/128966) · page_id `128966`

## 概述

TCR 实例使用 COS 桶作为容器镜像的存储后端。COS 桶分为单 AZ 存储和多 AZ 存储。在单 AZ 架构中，单个可用区因自然灾害、断电等极端情况会导致 COS 桶无法访问。TCR 提供了后端存储切换能力，允许将 TCR 实例的后端存储切换至位于其他地域复制实例所使用的 COS 桶，以提升 TCR 服务的可用性。

**适用场景：**

1. COS 单 AZ 地域单可用区中断
2. COS 服务地域性故障，地域级别的 COS 服务中断
3. 网络连通性问题导致无法访问原 COS 桶

> **建议**：在支持 COS 多 AZ 的地域（北京、广州、上海、中国香港、新加坡、上海金融），若创建 TCR 实例时未勾选多 AZ COS 桶，建议 [提交工单](https://console.cloud.tencent.com/workorder/category) 联系 COS 团队将 COS 桶原地升级到多 AZ COS 桶。

## 前置条件

- [环境准备](../../环境准备.md)
- 已完成 [创建企业版实例](../../操作指南/创建企业版实例/tccli%20操作.md)，实例 `Status` 为 `Running`
- 已为 TCR 实例创建跨地域的 [复制实例](../../操作指南/镜像分发/同实例多地域复制镜像/tccli%20操作.md)（page_id `52095`）
- 复制实例关联的 COS 桶已开启内网全球加速（在 COS 控制台完成）
- **使用限制**：
  - 仅 TCR 企业版支持此功能
  - 切换后，该实例仅支持镜像拉取操作，不支持镜像推送

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 查看实例详情（含后端存储桶） | `tccli tcr DescribeInstances` | 是 |
| 查看复制实例列表 | `tccli tcr DescribeReplicationInstances` | 是 |
| 执行后端存储切换 | `tccli tcr ModifyInstanceStorage` | 否 |
| 开启 COS 桶内网全球加速 | 跨产品控制台操作（COS），无直接 TCR CLI | — |

## 操作步骤

### 定位存储桶信息

查看主实例详情，获取当前后端存储桶：

```bash
tccli tcr DescribeInstances \
  --RegistryIds "[\"<RegistryId>\"]" \
  --region ap-guangzhou \
  --output json
```

```json
{
  "Registries": [
    {
      "RegistryId": "tcr-example",
      "RegistryName": "my-registry",
      "Status": "Running",
      "PublicDomain": "tcr-example.tencentcloudcr.com",
      "RegionName": "ap-guangzhou"
    }
  ],
  "TotalCount": 1,
  "RequestId": "eac6b301-a322-493a-8e36-83b295459397"
}
```

查看复制实例列表及其所在地域：

```bash
tccli tcr DescribeReplicationInstances \
  --RegistryId <RegistryId> \
  --region ap-guangzhou \
  --output json
```

```json
{
  "ReplicationRegistries": [
    {
      "RegistryId": "tcr-replica-16-xxxxxx",
      "RegionName": "ap-chengdu",
      "Status": "Running"
    }
  ],
  "TotalCount": 1,
  "RequestId": "f6a7b8c9-d0e1-2345-fabc-456789012345"
}
```

> COS 桶名称格式为 `[主实例ID]-[RegionCode]-[随机字符串]-[AppId]`，例如 `tcr-example-16-dykhma-125xxxx00`。可登录 [COS 控制台](https://console.cloud.tencent.com/cos/bucket) 以实例 ID 搜索找到所有关联的 COS 桶。其中包含复制实例 ID 的桶即为目标复制桶。

### 配置 COS 内网全球加速

此步骤为跨产品操作，需在 COS 控制台完成：

1. 登录 [COS 控制台](https://console.cloud.tencent.com/cos/bucket)，进入目标复制桶的详情页。
2. 在左侧选择**域名与传输管理 > 全球加速**。
3. 单击**编辑**，开启**内网全球加速**功能并保存。

> **注意**：若未开启内网全球加速，`ModifyInstanceStorage` 调用将失败。确认加速状态为已开启后再执行下一步。

### 调用 API 执行切换

调用 `ModifyInstanceStorage` API 完成存储配置切换：

```bash
tccli tcr ModifyInstanceStorage \
  --RegistryId <RegistryId> \
  --TargetRegion ap-chengdu \
  --TargetStorageName <CosBucketName> \
  --region ap-guangzhou \
  --output json
```

```json
{
  "RequestId": "a7b8c9d0-e1f2-3456-abcd-567890123456"
}
```

| 参数 | 必填 | 说明 | 示例值 |
|------|------|------|------|
| `--RegistryId` | 是 | 需切换的 TCR 主实例 ID | `<RegistryId>` |
| `--TargetRegion` | 是 | 复制实例 COS 桶所在地域 | `ap-chengdu` |
| `--TargetStorageName` | 是 | 复制实例 COS 桶的**名称**（非加速域名地址） | `<CosBucketName>` |

> **重要**：`--TargetStorageName` 参数应填写 COS 桶名称，而非其加速域名地址（如 `xxx.cos.accelerate.myqcloud.com`）。

## 验证

### Data plane (docker)

API 调用成功后，等待约 1-2 分钟后端服务滚动更新。随后使用 Docker 客户端拉取该实例内已有的容器镜像，验证切换是否生效：

```bash
docker login <tcr-instance>.tencentcloudcr.com --username=<临时登录用户名>
```

```bash
docker pull <tcr-instance>.tencentcloudcr.com/<namespace>/<image>:<tag>
```

若能成功拉取，则表明存储切换已生效。

## 清理

后端存储切换操作本身不创建新资源，无需清理。若需回切到原 COS 桶，可再次调用 `ModifyInstanceStorage`，将 `--TargetStorageName` 指定为原主实例的 COS 桶名称。

## 排障

| 现象 | 处理 |
|------|------|
| `ModifyInstanceStorage` 返回 `InvalidParameter` | 检查 `--TargetStorageName` 是否为 COS 桶名称（非加速域名）；确认桶所在地域与 `--TargetRegion` 一致 |
| `ModifyInstanceStorage` 返回 `FailedOperation` | 确认复制实例 COS 桶已开启内网全球加速；确认实例状态为 `Running` |
| 切换后 Docker 拉取失败 | 等待后端服务滚动更新完成（约 1-2 分钟）；检查镜像是否在复制桶内存在；确认网络可达 |
| 切换后无法推送镜像 | **此为预期行为**：存储切换后仅支持镜像拉取，不支持推送 |
| 复制实例 COS 桶名称格式不确定 | 登录 COS 控制台，以主实例 ID（如 `tcr-xxxxxxxx`）搜索存储桶列表，名称包含复制实例 ID 的桶即为目标桶 |

## 下一步

- [同实例多地域复制镜像](../../操作指南/镜像分发/同实例多地域复制镜像/tccli%20操作.md)（page_id `52095`）
- [创建企业版实例](../../操作指南/创建企业版实例/tccli%20操作.md)（page_id `51110`）
- [使用自定义域名及云联网实现跨地域内网访问](../使用自定义域名及云联网实现跨地域内网访问/tccli%20操作.md)（page_id `76084`）

## 控制台替代

登录 [容器镜像服务控制台](https://console.cloud.tencent.com/tcr/instance) → 选择目标实例 → 在实例概况中单击**后端存储桶**跳转至 COS 桶详情 → 返回桶列表以实例 ID 搜索定位复制桶 → 在 COS 控制台开启该桶的内网全球加速 → 前往 [API Explorer](https://console.cloud.tencent.com/api/explorer?Product=tcr&Version=2019-09-24&Action=ModifyInstanceStorage) 调用 `ModifyInstanceStorage` 完成切换 → 等待 1-2 分钟后使用 Docker 拉取镜像验证。
