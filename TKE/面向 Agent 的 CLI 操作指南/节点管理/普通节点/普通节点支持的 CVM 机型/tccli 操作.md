# 普通节点支持的 CVM 机型

> 对照官方：[普通节点支持的 CVM 机型](https://cloud.tencent.com/document/product/457/84660) · page_id `84660`

## 概述

普通节点池的节点基于 CVM 实例。TKE 支持多种 CVM 机型作为节点，不同可用区支持的机型列表不同。本文介绍如何通过 CLI 查询指定地域和可用区的可用 CVM 机型，为创建节点池（`CreateClusterNodePool`）选择 `InstanceTypes` 参数提供依据。

核心查询方式：通过 `cvm:DescribeZoneInstanceConfigInfos` 按可用区+机型族过滤，`Status=SELL` 表示可购买。

查询方式有两种：

| 方式 | CLI | 适用场景 |
|------|-----|---------|
| **CVM 通用查询** | `tccli cvm DescribeZoneInstanceConfigInfos` | 查询某可用区所有可售机型，适用于所有 CVM 场景 |
| **TKE 专用查询** | `tccli tke DescribeInstanceConfigInfos` | 查询 TKE 兼容的机型列表，专用且更精确 |

## 前置条件

- [环境准备](../../../环境准备.md)：`tccli` 已配置，具备 CVM 只读权限
- 已确定目标地域和可用区

### 环境检查

```bash
# 确认 CVM 只读权限
tccli cvm DescribeZones --region <Region>
# expected: exit 0，返回到可用区列表
```

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 查看可用区机型列表 | `tccli cvm DescribeZoneInstanceConfigInfos --region <Region> --Zone <Zone>` | 是 |
| 查看 TKE 兼容机型 | `tccli tke DescribeInstanceConfigInfos --region <Region>` | 是 |
| 按机型族过滤 | `tccli cvm DescribeZoneInstanceConfigInfos --region <Region> --Filters '[{"Name":"zone","Values":["<Zone>"]},{"Name":"instance-family","Values":["S5"]}]'` | 是 |
| 查询指定机型详情 | `tccli cvm DescribeInstanceTypeConfigs --region <Region>` | 是 |

## 操作步骤

### 方案一：CVM 通用机型查询

通过 CVM 产品的 `DescribeZoneInstanceConfigInfos` API 查询指定可用区所有可售实例配置。

```bash
tccli cvm DescribeZoneInstanceConfigInfos \
    --region <Region> \
    --Zone <Zone>
```

以本集群所在环境为例：集群 `cls-xxxxxxxx`（`ap-guangzhou`）：

```bash
tccli cvm DescribeZoneInstanceConfigInfos \
    --region ap-guangzhou \
    --Zone ap-guangzhou-3
```

参考输出（节选）：

```json
{
    "InstanceTypeQuotaSet": [
        {
            "Zone": "ap-guangzhou-3",
            "InstanceType": "S5.MEDIUM4",
            "InstanceFamily": "S5",
            "InstanceChargeType": "POSTPAID_BY_HOUR",
            "Cpu": 2,
            "Memory": 4,
            "Status": "SELL",
            "NetworkCard": 25,
            "Externals": {},
            "RemoteDiskAttr": {}
        },
        {
            "Zone": "ap-guangzhou-3",
            "InstanceType": "S5.LARGE8",
            "InstanceFamily": "S5",
            "InstanceChargeType": "POSTPAID_BY_HOUR",
            "Cpu": 4,
            "Memory": 8,
            "Status": "SELL",
            "NetworkCard": 25,
            "Externals": {},
            "RemoteDiskAttr": {}
        }
    ],
    "RequestId": "00000000-0000-0000-0000-000000000000"
}
```

**过滤条件说明：**

| 过滤参数 | 用途 | 示例 |
|---------|------|------|
| `--Zone` | 指定可用区 | `ap-guangzhou-3` |
| `--Filters '[{"Name":"instance-family","Values":["S5"]}]'` | 按机型族过滤 | 仅返回 `S5` 系列 |
| `--Filters '[{"Name":"zone","Values":["<Zone>"]}]'` | 在 Filters 中指定可用区（与 `--Zone` 等效） | 同上 |
| `--Filters '[{"Name":"instance-charge-type","Values":["POSTPAID_BY_HOUR"]}]'` | 按计费模式过滤 | 仅返回按量计费机型 |

只筛选 `Status: "SELL"`（可售）的机型即可用于创建节点池。

### 方案二：TKE 专用机型查询

通过 TKE 产品的 `DescribeInstanceConfigInfos` API 查询 TKE 集群兼容的机型列表，返回结果已过滤掉不支持作为 TKE 节点的机型。

```bash
tccli tke DescribeInstanceConfigInfos \
    --region <Region>
```

参考输出（节选）：

```json
{
    "InstanceConfigInfoSet": [
        {
            "InstanceFamily": "S5",
            "InstanceType": "S5.MEDIUM4",
            "CPU": 2,
            "Memory": 4,
            "GPU": 0,
            "Status": "SELL",
            "Zones": ["ap-guangzhou-3", "ap-guangzhou-4"]
        }
    ],
    "RequestId": "00000000-0000-0000-0000-000000000000"
}
```

### 常用机型系列速查

| 系列 | 特点 | 适用场景 | 机型示例 |
|------|------|---------|---------|
| **S5** | 标准型，均衡计算/内存/网络 | 通用 Web 应用、中小型数据库 | `S5.MEDIUM4`（2C4G）、`S5.LARGE8`（4C8G） |
| **S6** | 新一代标准型 | 高性能通用计算 | `S6.MEDIUM4`（2C4G）、`S6.LARGE8`（4C8G） |
| **M5** | 内存型，大内存配比 | 内存数据库、大数据分析 | `M5.MEDIUM16`（2C16G）、`M5.LARGE32`（4C32G） |
| **C5** | 计算型，高 CPU 配比 | 高性能计算、批量处理 | `C5.MEDIUM8`（2C8G）、`C5.LARGE16`（4C16G） |
| **IT5** | 高 IO 型，高性能本地 NVMe SSD | 高性能数据库、NoSQL | `IT5.4XLARGE64`（16C64G） |
| **GN10Xp** | GPU 型，搭载 NVIDIA Tesla GPU | AI 训练/推理、图形渲染 | `GN10Xp.2XLARGE40`（10C40G + 1xV100） |

> **提示：** `CreateClusterNodePool` 的 `InstanceTypes` 参数支持传入多种机型，弹性伸缩时将按优先级尝试创建。建议至少传入 2 种不同机型以提高节点创建成功率。

### 选择机型的考量因素

| 因素 | 说明 | 查询方法 |
|------|------|---------|
| **可用区 availability** | 不同可用区可用机型不同 | `DescribeZoneInstanceConfigInfos --Zone <Zone>` |
| **售卖状态** | `Status: "SELL"` 才可购买 | 检查返回结果中的 `Status` 字段 |
| **网络性能** | `NetworkCard` 值越大网络带宽越高 | 查看 `NetworkCard` 字段 |
| **计费模式** | 按量计费/包年包月 | 按 `InstanceChargeType` 过滤 |
| **操作系统兼容性** | TKE 支持的 OS 可能与机型有限制 | 参考 `DescribeOSImages` 和 `DescribeInstanceConfigInfos` |

## 验证

```bash
# 验证指定可用区的可用机型
tccli cvm DescribeZoneInstanceConfigInfos \
    --region <Region> \
    --Zone <Zone> \
    --Filters '[{"Name":"status","Values":["SELL"]},{"Name":"instance-charge-type","Values":["POSTPAID_BY_HOUR"]}]'
```

| 维度 | 检查内容 | 预期 |
|------|---------|------|
| 有可售机型 | `InstanceTypeQuotaSet` 非空 | >= 1 个机型 |
| 状态为 SELL | `Status` | `SELL` |
| 机型格式 | `InstanceType` | 格式如 `S5.MEDIUM4` |
| 资源充足 | CPU/Memory 满足业务需求 | 按实际需求判断 |

## 清理

<!-- 查询操作无需清理 -->

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DescribeZoneInstanceConfigInfos` 返回空列表 | 确认 Zone 名称是否正确（如 `ap-guangzhou-3`） | 指定可用区当前无可售机型 | 更换可用区或地域 |
| 返回 `InvalidParameterValue` — `Zone` 格式错误 | Zone 格式为 `{region}-{id}` | Zone 参数格式错误 | 使用正确格式如 `ap-guangzhou-3` |
| 指定机型创建节点池失败 | `tccli cvm DescribeZoneInstanceConfigInfos` 检查机型 `Status` | 机型已售罄或在该子网/可用区不可用 | 更换机型或加多备选 InstanceTypes |
| `DescribeInstanceConfigInfos` 返回机型少 | 此为 TKE 过滤后的机型列表 | 部分 CVM 机型不适合作为 TKE 节点 | 使用 TKE 返回的机型；若需更多选择，使用 CVM 的 `DescribeZoneInstanceConfigInfos` |
| 创建节点池时报 `InvalidParameterValue` — `InstanceType` | 检查 `InstanceTypes` 参数中机型格式 | 机型名称拼写错误或大小写不规范 | 确认机型名称与 `DescribeZoneInstanceConfigInfos` 返回完全一致 |

## 下一步

- [创建节点池](../创建节点池/tccli%20操作.md) — 使用查询到的机型创建节点池
- [节点池概述](../节点池概述/tccli%20操作.md) — 了解节点池类型和概念
- [查看节点池](../查看节点池/tccli%20操作.md) — 查看节点池详情和节点状态

## 控制台替代

[容器服务控制台 → 集群 → 节点管理 → 节点池 → 新建 → 选择机型](https://console.cloud.tencent.com/tke2/nodepool/create) — 在控制台创建节点池时可视化选择 CVM 机型。
