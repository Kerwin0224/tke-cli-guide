# 在 TKE Serverless 上运行深度学习（tccli）

> 对照官方：[在 TKE Serverless 上运行深度学习](https://cloud.tencent.com/document/product/457/60221) · page_id `60221`

## 概述

TKE Serverless（弹性容器实例）支持 GPU 实例，可直接运行深度学习训练任务。通过 `CreateEKSContainerInstances` 指定 `GpuType` 和 `GpuCount` 创建 GPU 容器实例。

## 前置条件

- [环境准备](../../../../环境准备.md)
- GPU 资源规格已确认（`DescribeEKSContainerInstanceRegions`）

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 创建 GPU 实例 | `tccli tke CreateEKSContainerInstances --cli-input-json` 含 `GpuType/GpuCount` | 否 |
| 查询 GPU 规格 | `tccli tke DescribeEKSContainerInstanceRegions` | 是 |
| 删除 | `tccli tke DeleteEKSContainerInstances` | 否 |

## 操作步骤

### 创建 GPU 深度学习实例

```json
{
    "EksCiName": "dl-gpu-training",
    "VpcId": "vpc-example",
    "SubnetId": "subnet-example",
    "SecurityGroupIds": ["sg-example"],
    "Cpu": 8,
    "Memory": 32,
    "GpuType": "V100",
    "GpuCount": 1,
    "Containers": [{
        "Name": "trainer",
        "Image": "tensorflow/tensorflow:latest-gpu",
        "Cpu": 8,
        "Memory": 32,
        "GpuLimit": 1,
        "Commands": ["python", "train.py"]
    }]
}
```

```bash
tccli tke CreateEKSContainerInstances --region ap-guangzhou --cli-input-json file://gpu-instance.json
```

## 验证

```bash
tccli tke DescribeEKSContainerInstances --region ap-guangzhou --cli-input-json "{\"EksCiIds\":[\"eksci-example\"]}"
```

```json
{
  "TotalCount": 0,
  "EksCis": [],
  "AutoCreatedEipId": "<AutoCreatedEipId>",
  "CamRoleName": "<CamRoleName>",
  "Containers": [],
  "Image": "<Image>",
  "Name": "<Name>",
  "Args": []
}
```

## 清理

```bash
tccli tke DeleteEKSContainerInstances --region ap-guangzhou --cli-input-json "{\"EksCiIds\":[\"eksci-example\"]}"
```

## 下一步

- [构建深度学习容器镜像](../构建深度学习容器镜像/tccli%20操作.md)
- [快速创建一个容器实例](../../../容器实例管理/快速创建一个容器实例/tccli%20操作.md)

## 控制台替代

控制台：弹性容器 → 新建 → 选择 GPU 规格。
