# 镜像和内核参数说明（tccli）

> 对照官方：[镜像和内核参数说明](https://cloud.tencent.com/document/product/457/124612) · page_id `124612`  

## 概述

原生节点池使用 **TencentOS Server** 等镜像；支持通过 Management 参数配置 **Kubelet、Kernel、hosts、nameservers**（Management 专页已排除 scope，此处保留官方能力说明）。查询可选镜像与节点池 `NodePoolOs` / `RuntimeConfig`。

## 前置条件

- [环境准备](../../../../../环境准备.md) · [tccli 专页（TKE）](../../../../../tccli 专页（TKE）.md)

## 控制台与 CLI 参数映射

| 控制台 / 官方 | tccli / 数据面 |
|---------------|----------------|
| OS 镜像系列 | `DescribeOSImages` |
| 节点池 OS/运行时 | `DescribeClusterNodePoolDetail` |
| containerd 版本 | `DescribeSupportedRuntime` |

## 操作步骤

```bash
tccli tke DescribeOSImages --region ap-guangzhou --output json \
  --filter "OSImageSeriesSet[?contains(SeriesName, `Tencent`) || contains(SeriesName, `tlinux`)].{SeriesName:SeriesName,OsName:OsName,Status:Status} | [0:5]"
tccli tke DescribeClusterNodePoolDetail --ClusterId <ClusterId> --NodePoolId <NodePoolId> --region ap-guangzhou --output json \
  --filter "NodePool.{NodePoolOs:NodePoolOs,RuntimeConfig:RuntimeConfig,OsCustomizeType:OsCustomizeType}"
tccli tke DescribeSupportedRuntime --K8sVersion 1.34.1 --region ap-guangzhou --output json \
  --filter "RuntimeSet[?RuntimeType=='containerd'] | [0:2]"
```

```json
{
  "OSImageSeriesSet": [],
  "SeriesName": "<SeriesName>",
  "Alias": "<Alias>",
  "OsName": "<OsName>",
  "OsCustomizeType": "<OsCustomizeType>",
  "Status": "<Status>",
  "ImageId": "<ImageId>",
  "TotalCount": 0
}
```

内核与 kubelet 细项见官方 Management 参数表（控制台节点池「运维参数」）。

## 验证

三条查询均成功。

## 清理

本页以只读查询或声明式配置为主，无额外 API 清理步骤。

## 排障

## 下一步

- [修改原生节点](../../../../../集群配置/节点管理/原生节点/修改原生节点/tccli 操作.md)

## 控制台替代

容器服务控制台 → **集群** → 目标集群 → **节点管理**，按[官方文档](https://cloud.tencent.com/document/product/457/124612)对应菜单操作。
