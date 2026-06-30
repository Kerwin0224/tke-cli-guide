# 云硬盘使用说明

> 对照官方：[云硬盘使用说明](https://cloud.tencent.com/document/product/457/44238) · page_id `44238`

## 概述

TKE 支持通过 CBS-CSI 驱动将腾讯云云硬盘 CBS 挂载为 Pod 数据卷。**关键限制：一个 CBS 云硬盘仅支持创建一个 PV，同时只能被一个集群节点挂载（ReadWriteOnce）。** 两种使用方式：动态创建和静态绑定已有云硬盘。

CBS-CSI（addon: `cbs` v1.1.15）为系统默认预装。重复安装 `InstallAddon cbs` 会报 `UnknownParameter: "check addon is exist"`。

## 前置条件

- [环境准备](../../../环境准备.md)
- CBS-CSI 组件为系统默认预装，无需手动安装
- CAM 权限：`tke:DescribeAddon`

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 查看 CBS-CSI 组件 | `tccli tke DescribeAddon` | 是 |
| 创建 StorageClass（动态） | `kubectl apply -f <yaml>` | 否（名称冲突） |
| 创建 PV（静态） | `kubectl apply -f <yaml>` | 否 |
| 创建 PVC | `kubectl apply -f <yaml>` | 否 |

## 操作步骤

### 确认 CBS-CSI 组件状态

```bash
tccli tke DescribeAddon --region <Region> \
    --ClusterId <ClusterId> \
    | jq '.Addons[] | select(.AddonName == "cbs") | {AddonName, AddonVersion, Status}'
# expected: {"AddonName":"cbs","AddonVersion":"1.1.15","Status":"Enabled"}
```

```json
{
  "Addons": "<Addons>",
  "AddonName": "<AddonName>",
  "AddonVersion": "<AddonVersion>",
  "RawValues": "<RawValues>",
  "Phase": "<Phase>",
  "Reason": "<Reason>"
}
```

### CBS 云硬盘类型

| 云盘类型 | `diskType` 枚举值 | 最小容量 | 说明 |
|---------|-------------------|:------:|------|
| 高性能云硬盘 | `CLOUD_PREMIUM` | 10GB | 性价比高 |
| SSD 云硬盘 | `CLOUD_SSD` | 20GB | 低延迟 |
| 增强型 SSD 云硬盘 | `CLOUD_HSSD` | 20GB | 支持额外性能配置 |
| 通用型 SSD 云硬盘 | `CLOUD_BSSD` | 20GB | 均衡型 |

云硬盘大小必须为 10 的倍数。

### 动态创建云硬盘

流程：创建 CBS 类型 StorageClass → 通过 StorageClass 创建 PVC（系统自动创建 CBS 实例和 PV） → 工作负载挂载 PVC。

详细步骤参见 [StorageClass 管理云硬盘模板](../StorageClass 管理云硬盘模板/tccli 操作.md)。

### 使用已有的云硬盘

流程：通过已有 CBS 云硬盘创建 PV → 创建 PVC 绑定 PV → 工作负载挂载 PVC。

详细步骤参见 [PV 和 PVC 管理云硬盘](../PV 和 PVC 管理云硬盘/tccli 操作.md)。

## 验证

```bash
# 确认 CBS-CSI 组件状态
tccli tke DescribeAddon --region <Region> \
    --ClusterId <ClusterId> \
    | jq '.Addons[] | select(.AddonName == "cbs")'
# expected: AddonVersion "1.1.15", Status "Enabled"
```

```json
{
  "Addons": "<Addons>",
  "AddonName": "<AddonName>",
  "AddonVersion": "<AddonVersion>",
  "RawValues": "<RawValues>",
  "Phase": "<Phase>",
  "Reason": "<Reason>"
}
```

## 清理

本页为概念和导航页，无资源需清理。

## 排障

| 现象 | 根因 | 修复 |
|------|------|------|
| 跨可用区挂载失败 | CBS 不支持跨可用区挂载 | 使用 nodeSelector 将 Pod 调度到 CBS 所在可用区 |
| 多 Pod 同时挂载同一 CBS | CBS 仅支持 ReadWriteOnce | 使用 RWX 存储方案或改为单副本 |
| CBS 扩容失败 | 控制台不支持 CBS 扩容 | 前往云硬盘控制台或调用 `tccli cbs ResizeDisk` |

## 下一步

- [StorageClass 管理云硬盘模板](../StorageClass 管理云硬盘模板/tccli 操作.md)
- [PV 和 PVC 管理云硬盘](../PV 和 PVC 管理云硬盘/tccli 操作.md)
- [使用文件存储 CFS](../../使用文件存储 CFS/文件存储使用说明/tccli 操作.md)

## 控制台替代

[TKE 控制台 → 集群详情 → 存储 → StorageClass](https://console.cloud.tencent.com/tke2/cluster)：默认提供 `cbs` StorageClass。
