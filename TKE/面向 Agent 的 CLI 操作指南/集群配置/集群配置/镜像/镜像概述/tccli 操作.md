# 镜像概述

> 对照官方：[镜像概述](https://cloud.tencent.com/document/product/457/68289) · page_id `68289`

## 概述

TKE 支持**公共镜像**和**自定义镜像**两种类型。公共镜像由腾讯云官方提供，经过 Kubernetes 环境兼容性适配（网络、存储、节点初始化、GPU 驱动等），提供 SLA 服务保障。自定义镜像由用户制作或导入，属非标准环境，需自行保证兼容性。

操作系统有两个级别：**集群级别**和**节点池级别**。修改操作系统只影响后续新增或重装的节点，不影响存量节点。

## 前置条件

- [环境准备](../../../../环境准备.md)
- CAM 权限：`tke:DescribeImages`

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 查看支持的公共镜像 | `tccli tke DescribeImages` | 是 |
| 创建集群选择镜像 | `tccli tke CreateCluster --ImageId <ImageId>` | 否 |
| 创建节点池选择镜像 | `tccli tke CreateClusterNodePool --ImageId <ImageId>` | 否 |
| 修改集群操作系统 | `tccli tke ModifyClusterImage` | 否（仅影响后续新增节点） |

## 操作步骤

### 查看 TKE 支持的公共镜像

```bash
tccli tke DescribeImages --region <Region>
# expected: exit 0，返回支持的公共镜像列表
```

预期输出（部分推荐镜像）：

```json
{
    "ImageInstanceSet": [
        {
            "ImageId": "img-6n21msk1",
            "OsName": "tlinux4_x86_64_public",
            "OsCustomizeType": "GENERAL",
            "ImageName": "TencentOS Server 4 for x86_64",
            "Status": "normal"
        },
        {
            "ImageId": "img-eb30mz89",
            "OsName": "tlinux3.1x86_64",
            "OsCustomizeType": "GENERAL",
            "ImageName": "TencentOS Server 3.1(TK4)",
            "Status": "normal"
        }
    ]
}
```

### 推荐用镜像

| 镜像 ID | Os Name | 控制台展示名 | OS 类型 | 内核 | 备注 |
|---------|---------|-------------|---------|------|------|
| `img-6n21msk1` | tlinux4_x86_64_public | TencentOS Server 4 for x86_64 | Tencent OS Server | 6.6 | **推荐** |
| `img-7rqxtnh9` | tlinux3.3x86_64 | TencentOS Server 3.3(TK4) | Tencent OS Server | 5.4 | 全量发布 |
| `img-eb30mz89` | tlinux3.1x86_64 | TencentOS Server 3.1(TK4) | Tencent OS Server | 5.4.119 | 全量发布 |
| `img-487zeit5` | ubuntu22.04x86_64 | Ubuntu Server 22.04 LTS 64位 | Ubuntu | 公版内核 | 全量发布 |
| `img-22trbn9x` | ubuntu20.04x86_64 | Ubuntu Server 20.04.1 LTS 64位 | Ubuntu | 公版内核 | 全量发布 |

### 查看特定镜像详情

```bash
tccli tke DescribeImages --region <Region> \
    --Filters '[{"Name":"image-id","Values":["img-6n21msk1"]}]'
# expected: 返回该镜像详细信息
```

```json
{
  "ImageInfoList": "<ImageInfoList>",
  "Digest": "<Digest>",
  "Size": "<Size>",
  "ImageVersion": "<ImageVersion>",
  "UpdateTime": "<UpdateTime>",
  "Kind": "<Kind>"
}
```

## 验证

```bash
tccli tke DescribeImages --region <Region> | jq '.TotalCount'
# expected: 返回总镜像数 > 0
```

```json
{
  "ImageInfoList": "<ImageInfoList>",
  "Digest": "<Digest>",
  "Size": "<Size>",
  "ImageVersion": "<ImageVersion>",
  "UpdateTime": "<UpdateTime>",
  "Kind": "<Kind>"
}
```

## 清理

本页为概念说明页，无资源需清理。

## 排障

| 现象 | 根因 | 修复 |
|------|------|------|
| 控制台看不到某公共镜像 | 该镜像在 CVM 侧未全量开放 | 提交工单申请白名单 |
| 某网络特性下镜像不可选 | Dataplane V2、注册节点等特性对镜像有特定要求 | 查看对应特性的镜像兼容说明 |
| 自定义镜像不可见 | 未使用 TKE 支持的基础镜像制作 | 使用 TKE 公共镜像作为基础镜像重新制作 |

## 下一步

- [自定义镜像说明](../自定义镜像说明/tccli 操作.md)
- [TencentOS Rainbow 镜像介绍](../TencentOS Rainbow/镜像介绍/tccli 操作.md)

## 控制台替代

[TKE 控制台 → 创建集群/新建节点](https://console.cloud.tencent.com/tke2/cluster)：在"操作系统"下拉列表中选择支持的公共镜像或自定义镜像。
