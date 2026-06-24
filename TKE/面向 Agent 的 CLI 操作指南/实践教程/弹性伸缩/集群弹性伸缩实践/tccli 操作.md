# 集群弹性伸缩实践（tccli）

> 对照官方：[集群弹性伸缩实践](https://cloud.tencent.com/document/product/457/36173) · page_id `36173`

## 概述

TKE 集群弹性伸缩通过 Cluster Autoscaler 自动调整节点数量。`ModifyClusterNodePool` 配置伸缩范围，`ModifyNodePoolDesiredCapacityAboutAsg` 手动触发扩容。

## 前置条件

- [环境准备](../../../环境准备.md)
- 已有节点池

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 启用自动伸缩 | `tccli tke ModifyClusterNodePool --EnableAutoscale true --MinSize 1 --MaxSize 10` | 否 |
| 手动调整大小 | `tccli tke ModifyNodePoolDesiredCapacityAboutAsg --DesiredCapacity 3` | 否 |
| 查看伸缩记录 | `tccli tke DescribeClusterNodePools` | 是 |

## 操作步骤

### 1. 启用节点池自动伸缩

```bash
tccli tke ModifyClusterNodePool --region ap-guangzhou --cli-input-json file://autoscale.json
```

```json
{
    "ClusterId": "<ClusterId>",
    "NodePoolId": "np-example",
    "EnableAutoscale": true,
    "MinSize": 1,
    "MaxSize": 10
}
```

### 2. 查看节点池状态

```bash
tccli tke DescribeClusterNodePools --region ap-guangzhou --ClusterId <ClusterId>
```

```json
{
  "NodePoolSet": [],
  "NodePoolId": "<NodePoolId>",
  "Name": "<Name>",
  "ClusterInstanceId": "<ClusterInstanceId>",
  "LifeState": "<LifeState>",
  "LaunchConfigurationId": "<LaunchConfigurationId>",
  "AutoscalingGroupId": "<AutoscalingGroupId>",
  "Labels": []
}
```

```output

```json
{
  "NodePoolSet": [],
  "NodePoolId": "<NodePoolId>",
  "Name": "<Name>",
  "ClusterInstanceId": "<ClusterInstanceId>",
  "LifeState": "<LifeState>",
  "LaunchConfigurationId": "<LaunchConfigurationId>",
  "AutoscalingGroupId": "<AutoscalingGroupId>",
  "Labels": []
}
```
{"NodePoolSet": [{"NodePoolId": "np-example", "AutoscalingGroupPara": "DesiredCapacity:3,MinSize:1,MaxSize:10"}]}
```

## 验证

```bash
kubectl get nodes
tccli tke DescribeClusterNodePools --region ap-guangzhou --ClusterId <ClusterId> --filter "NodePoolSet[].AutoscalingGroupPara"
```

```text
NAME  STATUS  AGE
...
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl` 命令报错/无响应（本环境实测） | `tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]' --filter "Clusters[0].ClusterStatus"` | CAM 拒绝公网端点（strategyId:240463971），数据面不可达 | 接入 VPN/IOA 内网后执行 kubectl；控制面 tccli 不受影响 |
| 节点池不扩容 | `tccli tke DescribeClusterAsGroupOption --region ap-guangzhou --ClusterId <ClusterId>` | Cluster Autoscaler 未启用或 `MaxSize` 已达上限 | 启用 autoscaler；调高节点池 `MaxSize` |
| `ModifyClusterNodePool` 报权限错误 | 同上诊断命令 | 当前子用户缺少 `tke:ModifyClusterNodePool` CAM 权限 | 授予对应 CAM 策略 |

## 清理

关闭自动伸缩以停止计费：

```bash
tccli tke ModifyClusterNodePool --region ap-guangzhou --cli-input-json file://disable-autoscale.json
```

## 下一步

- [HPA 弹性伸缩](../在%20TKE%20上利用%20HPA%20实现业务的弹性伸缩/tccli%20操作.md)
- [placeholder 秒级伸缩](../使用%20tke-autoscaling-placeholder%20实现秒级弹性伸缩/tccli%20操作.md)

## 控制台替代

控制台：节点池 → 弹性伸缩 → 开启。
