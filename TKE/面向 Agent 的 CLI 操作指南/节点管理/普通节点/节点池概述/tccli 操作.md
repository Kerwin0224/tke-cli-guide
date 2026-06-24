# 节点池概述

> 对照官方：[节点池概述](https://cloud.tencent.com/document/product/457/43719) · page_id `43719`

## 概述

节点池（NodePool）是 TKE 集群中一组具有相同配置的节点的集合。通过节点池，您可以对节点进行分组管理、统一扩缩容、批量配置。节点池内所有节点共享相同的机型、操作系统、容器运行时、Label/Taint 等配置。

以本集群 `cls-xxxxxxxx`（`ap-guangzhou`，托管集群，Kubernetes 1.30.0，containerd 1.6.9）为例，节点池 `np-lmvsqkuu`（`kerwinwjyan-rewrite-s3-np`）即是一个普通节点池。

### 节点池类型

`Type` 参数决定节点池类型，可选值如下：

| Type 值 | 含义 | 说明 |
|---------|------|------|
| `""`（空字符串） | **普通节点池** | 标准节点池，节点基于 CVM 实例，支持弹性伸缩 |
| `"Native"` | **原生节点池** | 基于原生节点（Native Node），提供更轻量的节点管理能力 |
| `"Super"` | **超级节点池** | Serverless 节点池，节点由系统托管，无需用户管理底层 CVM |

本文聚焦于**普通节点池**（`Type: ""`）。

### 节点池 vs 游离节点

| 维度 | 节点池内节点 | 游离节点 |
|------|-------------|---------|
| 管理方式 | 通过节点池统一管理 | 独立管理，不在任何节点池内 |
| 扩缩容 | 支持弹性伸缩（Auto Scaling） | 不支持弹性伸缩 |
| 批量操作 | 一键调整所有节点配置 | 需逐节点操作 |
| 创建方式 | `CreateClusterNodePool` 或控制台创建 | `AddExistedInstances` 将已有 CVM 加入集群 |
| 互转 | 不支持节点池节点转为游离节点 | 可通过节点池 ID 参数将游离节点加入节点池 |

> **提示：** 推荐使用节点池管理节点，便于统一运维和弹性伸缩。游离节点适用于一次性加入的临时节点或迁移场景。

## 前置条件

- [环境准备](../../../环境准备.md)：`tccli` 已配置，具备 TKE 读写权限
- 已存在运行中的 TKE 集群

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0

# 2. 检查 TKE 操作权限
tccli tke DescribeClusterStatus --region <Region> --ClusterIds '["<ClusterId>"]'
# expected: exit 0，返回集群状态
```

```json
{
  "ClusterStatusSet": "<ClusterStatusSet>",
  "ClusterId": "<ClusterId>",
  "ClusterState": "<ClusterState>",
  "ClusterInstanceState": "<ClusterInstanceState>",
  "ClusterBMonitor": "<ClusterBMonitor>",
  "ClusterInitNodeNum": "<ClusterInitNodeNum>"
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 查看节点池列表 | `tccli tke DescribeClusterNodePools` | 是 |
| 查看节点池详情 | `tccli tke DescribeClusterNodePools --ClusterId <ClusterId>` | 是 |
| 查看 OS 镜像列表 | `tccli tke DescribeOSImages` | 是 |

## 操作步骤

### 查看集群节点池

```bash
tccli tke DescribeClusterNodePools \
    --ClusterId <ClusterId> \
    --region <Region>
```

```json
{
  "NodePoolSet": [],
  "NodePoolId": "<NodePoolId>",
  "Name": "<Name>",
  "ClusterInstanceId": "<ClusterInstanceId>"

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
```,
  "LifeState": "<LifeState>",
  "LaunchConfigurationId": "<LaunchConfigurationId>",
  "AutoscalingGroupId": "<AutoscalingGroupId>",
  "Labels": []
}
```

以 `cls-xxxxxxxx` 为例：

```bash
tccli tke DescribeClusterNodePools \
    --ClusterId cls-xxxxxxxx \
    --region ap-guangzhou
```

参考输出：

```json
{
    "TotalCount": 1,
    "NodePoolSet": [
        {
            "NodePoolId": "np-lmvsqkuu",
            "Name": "kerwinwjyan-rewrite-s3-np",
            "LifeState": "normal",
            "MaxNodesNum": 1,
            "MinNodesNum": 1,
            "DesiredNodesNum": 1,
            "NodePoolOs": "tlinux3.1x86_64",
            "OsCustomizeType": "GENERAL",
            "RuntimeConfig": {
                "RuntimeType": "containerd",
                "RuntimeVersion": "1.6.9"
            },
            "NodeCountSummary": {
                "AutoscalingAdded": {
                    "Normal": 1,
                    "Total": 1
                }
            },
            "AutoscalingGroupStatus": "disabled",
            "DeletionProtection": false
        }
    ],
    "RequestId": "00000000-0000-0000-0000-000000000000"
}
```

### 查看支持的 OS 镜像

```bash
tccli tke DescribeOSImages --region <Region>
```

常用 OS 镜像：`tlinux3.1x86_64`、`tlinux4_x86_64_public`、`ubuntu22.04x86_64`、`ubuntu20.04x86_64`、`debian11.11x86_64`、`debian12.8x86_64`、`centos7.9.0_x86_64`、`TencentOS Server 3.1 (TK4)`、`TencentOS Server 2.4 (TK4)`

### 关键字段说明

| 字段 | 类型 | 说明 |
|------|------|------|
| `NodePoolId` | String | 节点池 ID，全局唯一，格式 `np-xxxxxxxx` |
| `Name` | String | 节点池名称，同一集群内不可重名 |
| `Type` | String | 节点池类型：`""`（普通）/ `"Native"`（原生）/ `"Super"`（超级） |
| `LifeState` | String | 生命周期状态：`normal`（正常）/ `creating`（创建中）/ `deleting`（删除中） |
| `MaxNodesNum` | Integer | 最大节点数，弹性伸缩上限 |
| `MinNodesNum` | Integer | 最小节点数，弹性伸缩下限 |
| `DesiredNodesNum` | Integer | 期望节点数，当前目标规模 |
| `NodePoolOs` | String | 节点操作系统镜像 |
| `RuntimeConfig` | Object | 容器运行时配置（`RuntimeType` / `RuntimeVersion`） |
| `NodeCountSummary` | Object | 节点计数汇总（按计费类型分组） |
| `AutoscalingGroupStatus` | String | 弹性伸缩组状态：`enabled` / `disabled` |
| `DeletionProtection` | Boolean | 是否开启删除保护 |

## 验证

```bash
# 验证节点池列表
tccli tke DescribeClusterNodePools \
    --ClusterId <ClusterId> \
    --region <Region>
```

| 维度 | 检查内容 | 预期 |
|------|---------|------|
| 列表非空 | `TotalCount` | >= 0（可为空） |
| 节点池状态 | `NodePoolSet[].LifeState` | `normal` |
| 节点计数 | `NodePoolSet[].NodeCountSummary` | 与预期节点数一致 |

## 清理

<!-- 概述页无需清理操作 -->

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DescribeClusterNodePools` 返回 `ResourceNotFound` | 确认 ClusterId 格式和所属地域 | 集群不存在或 ClusterId 错误 | `tccli tke DescribeClusters --region <Region>` 确认集群列表 |
| `TotalCount` 为 0 | 确认集群是否已创建节点池 | 集群无节点池 | 参考 [创建节点池](../创建节点池/tccli%20操作.md) |
| `LifeState` 非 `normal` | 等待，状态转换通常秒级 | 节点池仍在创建/删除中 | 等待 1-2 分钟后重新查询 |

## 下一步

- [创建节点池](../创建节点池/tccli%20操作.md) — 通过 CLI 创建普通节点池
- [查看节点池](../查看节点池/tccli%20操作.md) — 查看节点池详情
- [调整节点池](../调整节点池/tccli%20操作.md) — 修改节点池配置
- [新增游离普通节点](../新增游离普通节点/tccli%20操作.md) — 将已有 CVM 加入集群

## 控制台替代

[容器服务控制台 → 集群 → 节点管理 → 节点池](https://console.cloud.tencent.com/tke2/nodepool) — 查看和管理节点池。
