# 组件的生命周期管理（tccli）

> 对照官方：[组件的生命周期管理](https://cloud.tencent.com/document/product/457/49442) · page_id `49442`

## 概述

管理 TKE 扩展组件的完整生命周期：安装、查看、升级、卸载。所有操作通过 `tccli tke` 的 Addon 相关接口完成。

## 前置条件

- [环境准备](../../../../环境准备.md)：`tccli` 已配置
- 已创建集群且状态 `Running`

```bash
# 检查集群状态
tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'
# expected: ClusterStatus "Running"
```

```json
{
  "TotalCount": "<TotalCount>",
  "Clusters": "<Clusters>",
  "ClusterId": "<ClusterId>",
  "ClusterName": "<ClusterName>",
  "ClusterDescription": "<ClusterDescription>",
  "ClusterVersion": "<ClusterVersion>"
}
```

- 需要 CAM 权限：`tke:DescribeClusters`、`tke:DescribeAddon`、`tke:InstallAddon`、`tke:UpdateAddon`、`tke:DeleteAddon`

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 查看集群信息 | `tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'` | 是 |
| 查看组件列表 | `tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId>` | 是 |
| 查看指定组件详情 | `tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName <AddonName>` | 是 |
| 安装组件 | `tccli tke InstallAddon --region <Region> --ClusterId <ClusterId> --AddonName <AddonName>` | 否 |
| 升级组件 | `tccli tke UpdateAddon --region <Region> --ClusterId <ClusterId> --AddonName <AddonName> --AddonVersion <AddonVersion>` | 否 |
| 卸载组件 | `tccli tke DeleteAddon --region <Region> --ClusterId <ClusterId> --AddonName <AddonName>` | 是 |

## 操作步骤

### 步骤 1：查看已安装组件

```bash
tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId>
# expected: 返回 Addons 列表
```

预期输出（字段示意，使用虚构 ID）：

```json
{
    "Addons": [
        {
            "AddonName": "CBS-CSI",
            "AddonVersion": "1.0.0",
            "Status": "Running",
            "CreatedTime": "2026-06-16T10:00:00Z"
        },
        {
            "AddonName": "CoreDNS",
            "AddonVersion": "1.9.3",
            "Status": "Running",
            "CreatedTime": "2026-06-16T10:00:00Z"
        }
    ]
}
```

### 步骤 2：安装组件

```bash
tccli tke InstallAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName <AddonName> \
    --AddonVersion <AddonVersion>
# expected: exit 0，组件安装请求已提交
```

| 参数 | 说明 | 必填 |
|------|------|:--:|
| `--ClusterId` | 集群 ID | 是 |
| `--AddonName` | 组件名称（如 `CBS-CSI`、`NginxIngress`） | 是 |
| `--AddonVersion` | 组件版本号 | 否（默认最新） |
| `--RawValues` | 组件自定义参数（JSON 字符串） | 否 |

### 步骤 3：升级组件

```bash
tccli tke UpdateAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName <AddonName> \
    --AddonVersion <NewVersion>
# expected: exit 0，升级进行中
```

### 步骤 4：卸载组件

```bash
tccli tke DeleteAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName <AddonName>
# expected: exit 0，组件卸载请求已提交
```

## 验证

### 控制面（tccli）

```bash
# 检查集群状态
tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'
# expected: ClusterStatus "Running"

# 检查组件安装状态
tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> \
    | jq '.Addons[] | select(.AddonName == "<AddonName>") | {AddonName, Status, AddonVersion}'
# expected: Status "Running"
```

```json
{
  "TotalCount": "<TotalCount>",
  "Clusters": "<Clusters>",
  "ClusterId": "<ClusterId>",
  "ClusterName": "<ClusterName>",
  "ClusterDescription": "<ClusterDescription>",
  "ClusterVersion": "<ClusterVersion>"
}
```

## 清理

移除测试组件：

```bash
tccli tke DeleteAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName <AddonName>
# expected: exit 0

# 验证已卸载
tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> \
    | jq '.Addons[] | select(.AddonName == "<AddonName>")'
# expected: 空结果
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

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `InstallAddon` 返回 `FailedOperation.AddonAlreadyInstalled` | `tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId>` 检查已安装组件 | 组件已安装 | 无需重复安装；如需重装请先 `DeleteAddon` |
| `InstallAddon` 失败 | `tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName <AddonName>` 查看可用版本 | 集群版本不兼容或组件版本不存在 | 检查集群 Kubernetes 版本；确认 `--AddonVersion` 可用 |
| 组件状态 `Failed` | `tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName <AddonName>` 查看 Status 与 reason | 安装过程中出错 | 根据 reason 排查；检查集群资源是否充足 |
| `UpdateAddon` 失败 | `tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'` 确认集群状态 | 版本不兼容或集群状态异常 | 确认集群 Running；检查目标版本兼容性 |
| `Unable to connect to the server` | `kubectl cluster-info` 检查连通性 | 公网端点被 CAM 策略 strategyId:240463971 以 `tke:clusterExtranetEndpoint=true` 条件拒绝 | 通过 IOA/VPN 连接内网，或使用同 VPC CVM 执行 kubectl 命令 |

## 下一步

- [扩展组件概述](../扩展组件概述/tccli 操作.md) — 了解所有可用增强组件
- [组件高可用](../组件高可用/tccli 操作.md) — 跨可用区高可用配置
- 各组件操作页面

## 控制台替代

[TKE 控制台 - 组件管理](https://console.cloud.tencent.com/tke2/cluster) → 集群 → 组件管理 → 安装/升级/删除。
