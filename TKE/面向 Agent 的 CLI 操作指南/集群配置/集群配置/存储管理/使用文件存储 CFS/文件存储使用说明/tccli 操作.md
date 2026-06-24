# 文件存储使用说明

> 对照官方：[文件存储使用说明](https://cloud.tencent.com/document/product/457/44234) · page_id `44234`

## 概述

TKE 支持通过 CFS-CSI 驱动将腾讯云文件存储 CFS 挂载为 Pod 数据卷。两种使用方式：动态创建（通过 StorageClass 自动创建 CFS 实例）和使用已有文件存储（静态创建 PV）。

真跑 L2 数据：`InstallAddon cfs v1.0.0` 成功（RequestId: `f16dfb5c-d0d5-4045-aef8-13cf6f82ce0d`），`DeleteAddon cfs` 成功（RequestId: `22b97416-fc6b-447b-828a-6a7abb162fb6`），完成 Install->Describe->Delete 闭环。

## 前置条件

- [环境准备](../../../环境准备.md)
- 已安装 CFS-CSI 组件（addon: `cfs`）
- CAM 权限：`tke:InstallAddon`、`tke:DescribeAddon`、`tke:DeleteAddon`

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 安装 CFS-CSI 组件 | `tccli tke InstallAddon --AddonName cfs` | 否（已存在则报 `UnknownParameter`） |
| 查看已安装组件 | `tccli tke DescribeAddon` | 是 |
| 卸载 CFS-CSI 组件 | `tccli tke DeleteAddon --AddonName cfs` | 是 |
| 创建 CFS 实例 | `tccli cfs CreateCfsFileSystem` | 否 |
| 创建 StorageClass（动态） | `kubectl apply -f <yaml>` | 否（名称冲突） |
| 创建 PV（静态） | `kubectl apply -f <yaml>` | 否 |

## 操作步骤

### 安装 CFS-CSI 组件

```bash
tccli tke InstallAddon --region <Region> \
    --cli-input-json '{"ClusterId":"<ClusterId>","AddonName":"cfs","AddonVersion":"1.0.0"}'
# expected: exit 0，返回 RequestId
```

安装后验证：

```bash
tccli tke DescribeAddon --region <Region> \
    --ClusterId <ClusterId> \
    | jq '.Addons[] | select(.AddonName == "cfs") | {AddonName, AddonVersion, Status}'
# expected: {"AddonName":"cfs","AddonVersion":"1.0.0","Status":"Enabled"}
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

### 动态创建文件存储

流程：创建 CFS 类型 StorageClass → 通过 StorageClass 创建 PVC（系统自动创建 CFS 实例和 PV） → 工作负载挂载 PVC。

详细步骤参见 [StorageClass 管理文件存储模板](../StorageClass 管理文件存储模板/tccli 操作.md)。

### 使用已有的文件存储

流程：通过已有 CFS 创建 PV → 创建 PVC 绑定 PV → 工作负载挂载 PVC。

详细步骤参见 [PV 和 PVC 管理文件存储](../PV 和 PVC 管理文件存储/tccli 操作.md)。

## 验证

```bash
# 确认 CFS-CSI 组件已安装
tccli tke DescribeAddon --region <Region> \
    --ClusterId <ClusterId> \
    | jq '.Addons[] | select(.AddonName == "cfs") | {AddonName, AddonVersion, Status}'
# expected: Status 为 "Enabled"
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

```bash
# 卸载 CFS-CSI 组件（需先删除所有使用 CFS 的工作负载、PVC、PV）
tccli tke DeleteAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName cfs
# expected: exit 0
```

## 排障

| 现象 | 根因 | 修复 |
|------|------|------|
| `InstallAddon cfs` 报 `UnknownParameter` | cfs 已安装，重复安装 | 使用 `DescribeAddon` 确认状态即可 |
| PV 挂载失败 | NFS v3 挂载未指定 fsid | 在 PV YAML 中添加 `fsid` 参数 |
| CFS 实例不在集群 VPC 内 | 创建 CFS 时选错网络 | 确保 CFS 与集群在同一 VPC |

## 下一步

- [StorageClass 管理文件存储模板](../StorageClass 管理文件存储模板/tccli 操作.md)
- [PV 和 PVC 管理文件存储](../PV 和 PVC 管理文件存储/tccli 操作.md)
- [使用云硬盘 CBS](../../使用云硬盘 CBS/云硬盘使用说明/tccli 操作.md)

## 控制台替代

[TKE 控制台 → 集群详情 → 组件管理](https://console.cloud.tencent.com/tke2/cluster)：新建 CFS 组件；存储 → PersistentVolume → 新建 → 选择"文件存储 CFS"。
