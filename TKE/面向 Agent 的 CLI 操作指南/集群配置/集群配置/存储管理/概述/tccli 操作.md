# 存储管理概述

> 对照官方：[存储概述](https://cloud.tencent.com/document/product/457/46962) · page_id `46962`

## 概述

TKE 集群支持多种存储类型，通过 PV（PersistentVolume）和 PVC（PersistentVolumeClaim）机制为工作负载挂载持久化数据卷。

| 存储类型 | 说明 | CSI 驱动 | Addon 名 | 访问模式 |
|---------|------|---------|----------|:--------:|
| 云硬盘 CBS | 数据块级持久存储，高可用高可靠高性能，适合数据库、文件系统等 | `com.tencent.cloud.csi.cbs` | `cbs` | ReadWriteOnce |
| 文件存储 CFS | NFS 协议共享文件系统，弹性容量和性能扩展，适合大数据分析、媒体处理 | `com.tencent.cloud.csi.cfs` | `cfs` | ReadWriteMany |
| 对象存储 COS | 海量文件分布式存储，适合静态数据存储 | `com.tencent.cloud.csi.cosfs` | `cos` | ReadWriteMany |
| 数据加速器 GooseFS | 高性能数据加速，适合大文件读写场景 | `com.tencent.cloud.csi.goosefs` | `goosefs` | ReadWriteMany |
| CFSTURBO-CSI | Turbo 系列高性能并行文件存储 | - | `cfs-turbo` | ReadWriteMany |
| 其他 | 临时路径（emptyDir）、主机路径（hostPath）、NFS、ConfigMap、Secret 等本地存储 | - | - | 取决于类型 |

建议使用云存储服务，本地存储数据在节点异常时无法恢复。

## 前置条件

- [环境准备](../../../../环境准备.md)
- CAM 权限：`tke:DescribeAddon`、`tke:InstallAddon`、`tke:DeleteAddon`

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 查看已安装存储组件 | `tccli tke DescribeAddon --ClusterId <ClusterId>` | 是 |
| 安装存储组件 | `tccli tke InstallAddon --ClusterId <ClusterId> --AddonName <AddonName>` | 否（已存在则报 `UnknownParameter`） |
| 卸载存储组件 | `tccli tke DeleteAddon --ClusterId <ClusterId> --AddonName <AddonName>` | 是 |
| 查看 PV | `kubectl get pv` | 是 |
| 查看 PVC | `kubectl get pvc` | 是 |
| 查看 StorageClass | `kubectl get sc` | 是 |

## 操作步骤

### 查看已安装存储组件

```bash
tccli tke DescribeAddon --region <Region> \
    --ClusterId <ClusterId>
# expected: exit 0，返回各 addon 及其状态
```

预期输出中与存储相关的 addon 字段：

```json
{
    "Addons": [
        {
            "AddonName": "cbs",
            "AddonVersion": "1.1.15",
            "Status": "Enabled"
        },
        {
            "AddonName": "cfs",
            "AddonVersion": "1.0.0",
            "Status": "Enabled"
        }
    ]
}
```

### 安装存储 CSI 组件

各存储类型对应的 addon 名称：

| 存储类型 | AddonName |
|---------|----------|
| CBS | `cbs` |
| CFS | `cfs` |
| COS | `cos` |
| CFSTURBO | `cfs-turbo` |
| GooseFS | `goosefs` |

```bash
# 安装 CFS-CSI 组件
tccli tke InstallAddon --region <Region> \
    --cli-input-json '{"ClusterId":"<ClusterId>","AddonName":"cfs","AddonVersion":"1.0.0"}'
# expected: exit 0，返回 RequestId

# cbs 为系统默认预装 addon，重复安装会报错:
# UnknownParameter: "check addon is exist"
```

## 验证

```bash
# 确认存储组件已安装
tccli tke DescribeAddon --region <Region> \
    --ClusterId <ClusterId> \
    | jq '.Addons[] | select(.AddonName == "cbs" or .AddonName == "cfs" or .AddonName == "cos") | {AddonName, AddonVersion, Status}'
# expected: 各组件 Status 为 "Enabled"
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

本页为概述页，无资源需清理。

## 排障

| 现象 | 根因 | 修复 |
|------|------|------|
| `InstallAddon cbs` 报 `UnknownParameter: "check addon is exist"` | cbs 为系统默认 addon，已预装 | 直接使用 cbs，无需安装 |
| `DescribeAddon` 中不显示某组件 | 组件未安装 | 执行 `InstallAddon` 安装 |
| Pod 挂载 PVC 失败 | 对应 CSI 驱动未安装 | 确认 `DescribeAddon` 中该 addon Status 为 `Enabled` |

## 下一步

- [使用云硬盘 CBS](../使用云硬盘 CBS/云硬盘使用说明/tccli 操作.md)
- [使用文件存储 CFS](../使用文件存储 CFS/文件存储使用说明/tccli 操作.md)
- [使用对象存储 COS](../使用对象存储 COS/tccli 操作.md)
- [使用数据加速器 GooseFS](../使用数据加速器 GooseFS/tccli 操作.md)
- [其他存储卷使用说明](../其他存储卷使用说明/tccli 操作.md)
- [PV 和 PVC 的绑定规则](../PV 和 PVC 的绑定规则/tccli 操作.md)

## 控制台替代

[TKE 控制台 → 集群详情 → 组件管理](https://console.cloud.tencent.com/tke2/cluster)：查看已安装存储组件；存储 → StorageClass / PV / PVC 管理存储资源。
