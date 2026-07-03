---
doc_type: How-to
subtype: 6A
fused: true
---
# 管理集群插件

> 安装、更新、卸载 TKE 集群插件。插件是封装好的 Helm Chart，扩展集群功能。异步操作。

## 概述

插件生命周期三步：安装 → 更新 → 卸载。用户通常在一个会话内完成。

| 操作 | 接口 | 作用 |
|:-----|:-----|:-----|
| 安装 | `InstallAddon` | 部署插件到集群 |
| 更新 | `UpdateAddon` | 升级版本或改配置 |
| 查询 | `DescribeAddon` | 看插件状态 |
| 卸载 | `DeleteAddon` | 移除插件 |

操作是**异步**的：接口返回即提交，插件就绪需轮询 `DescribeAddon` 直到 `Phase=Succeeded`。

## 准备工作

### 环境检查

```bash
tccli --version
# expected: tccli 版本号

tccli tke DescribeClusterStatus --region ap-guangzhou --ClusterIds '["<CLUSTER_ID>"]' \
  --filter "ClusterStatusSet[0].ClusterState"
# expected: "Running"
```

### 资源检查

```bash
# 确认插件未安装（避免重复）
tccli tke DescribeAddon --region ap-guangzhou --ClusterId "<CLUSTER_ID>" --AddonName "<ADDON_NAME>" \
  --filter "Addons[].{name:AddonName,phase:Phase}"
# expected: 空数组（未安装）或 Phase=Succeeded（已装）
```

## 关键字段

> 来源：`tccli tke InstallAddon --generate-cli-skeleton` + `UpdateAddon`。

### InstallAddon

| 字段 | 类型 | 必填 | 约束 | 填错时的错误 |
|:------|------|:--------:|------------|---------------|
| ClusterId | string | 是 | `cls-xxxxxxxx` | `ResourceNotFound` |
| AddonName | string | 是 | 插件名，如 `cbs-csi` | `InvalidParameterValue` |
| AddonVersion | string | 是 | 插件版本 | `InvalidParameterValue.AddonVersion` |
| RawValues | string | 否 | base64 编码的配置 JSON | `InvalidParameterValue` |
| DryRun | boolean | 否 | 仅校验不执行 | — |

### UpdateAddon

> 比 Install 多 `UpdateStrategy`（更新策略）。

| 字段 | 类型 | 必填 | 作用 |
|:------|------|:--------:|:-----|
| UpdateStrategy | string | 否 | 更新策略（如 `rolling`） |

> `RawValues` 是 base64 编码的 JSON 配置。用 `echo '<json>' | base64` 生成。查询时返回的 `RawValues` 可 `echo '<base64>' | base64 -d` 解码看配置。

### 查询可用插件 (GetTkeAppChartList)

安装前查可用插件 chart 列表，按集群架构过滤：

| 字段 | 类型 | 必填 | 约束 | 填错时的错误 |
|:------|------|:--------:|------------|---------------|
| ChartName | string | 否 | 插件名关键词 | — |
| Arch | string | 否 | 架构枚举：`amd64` / `arm64` / `arm32`。须与节点架构匹配，否则装上后 Pod 因镜像架构不符起不来 | `InvalidParameterValue` |

```bash
tccli tke GetTkeAppChartList --region ap-guangzhou --Arch amd64
# expected: ChartList[] 含 ChartName/ChartVersion/Arch，按架构过滤
```

> `Arch` 三值代表三种 CPU 架构。ARM 集群（如鲲鹏）必须传 `arm64`/`arm32`，x86 集群传 `amd64`。装错架构的插件 Pod 会报 `ImagePullBackOff` 或 `CrashLoopBackOff`。

## 操作步骤

### 步骤 1：决策 — 插件版本选择

#### 为什么选最新兼容版本

- **最新版本 vs 指定版本**: 最新版含功能与安全修复；指定历史版本兼容旧集群
- **默认推荐**: 用 `DescribeAddonValues` 查兼容的最新版本
- **能降级吗?**: 能，`UpdateAddon` 指定较低版本，但可能有数据迁移风险

### 步骤 2：安装 — 最小化

```bash
tccli tke InstallAddon --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" --AddonName "<ADDON_NAME>" --AddonVersion "<VERSION>"
# expected: exit 0, 返回 RequestId
```

| 占位符 | 含义 | 约束 | 如何获取 |
|:------------|:-----|:-----|:---------|
| `<CLUSTER_ID>` | 集群 ID | `cls-xxxxxxxx` | `tccli tke DescribeClusters` |
| `<ADDON_NAME>` | 插件名 | 官方插件列表 | `tccli tke DescribeAddonValues` |

> 安装前用 `DescribeAddonValues` 查插件在该集群的可用版本与默认配置（参数以 `--generate-cli-skeleton` 为准）：

```bash
# 查询插件可用版本与配置值（ClusterId + AddonName 定位）
tccli tke DescribeAddonValues --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" --AddonName "<ADDON_NAME>"
# expected: 返回 Values+DefaultValues; 插件模板渲染异常报 ResourceUnavailable（如 cfs 插件 nil pointer）
```
| `<VERSION>` | 插件版本 | 兼容集群版本 | `tccli tke DescribeAddonValues` |

### 步骤 3：安装 — 增强：自定义配置

```bash
# 生成 base64 配置
VALUES=$(echo '{"resources":{"limits":{"cpu":"500m"}}}' | base64)

tccli tke InstallAddon --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" --AddonName "<ADDON_NAME>" --AddonVersion "<VERSION>" \
  --RawValues "$VALUES"
# expected: exit 0
```

### 步骤 4：更新 — 升级版本

```bash
tccli tke UpdateAddon --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" --AddonName "<ADDON_NAME>" --AddonVersion "<NEW_VERSION>" \
  --UpdateStrategy rolling
# expected: exit 0
```

### 步骤 5：验证

```bash
tccli tke DescribeAddon --region ap-guangzhou --ClusterId "<CLUSTER_ID>" --AddonName "<ADDON_NAME>" \
  --filter "Addons[].{name:AddonName,ver:AddonVersion,phase:Phase,reason:Reason}"
# expected: Phase="Succeeded", Reason 为空
```

| 维度 | 命令 | 预期 |
|:-----|:-----|:-----|
| 插件状态 | `DescribeAddon` → `Phase` | `Succeeded` |
| 版本一致 | `DescribeAddon` → `AddonVersion` | 等于安装/更新的版本 |
| 运行状态 | `kubectl get pods -n kube-system -l app=<ADDON_NAME>` | Pod Running |
| 无异常 | `DescribeAddon` → `Reason` | 空 |

> `Phase` 枚举：`Pending`/`Succeeded`/`Failed`。`Failed` 时查 `Reason` 字段定位原因。

## 清理

> **副作用警告**：卸载插件会移除其管理的资源。如 `cbs-csi` 卸载后已有 PVC 可能无法挂载。

```bash
# 1. 卸载
tccli tke DeleteAddon --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" --AddonName "<ADDON_NAME>"
# expected: exit 0

# 2. 验证已卸载
tccli tke DescribeAddon --region ap-guangzhou --ClusterId "<CLUSTER_ID>" --AddonName "<ADDON_NAME>"
# expected: Addons 为空数组
```

## 故障恢复

### 命令返回错误 (exit ≠ 0)

| 现象 | 诊断 | 根因 | 修复 |
|:--------|:----------|:------------|:-----|
| `InvalidParameterValue.AddonVersion` | `DescribeAddonValues` 查可用版本 | 版本不存在或不兼容集群 | 用兼容版本 |
| `ResourceNotFound` | `DescribeClusters` 核对 ID | ClusterId 错 | 确认集群 ID |
| `ResourceInUse` | `DescribeAddon` 看是否已装 | 插件已安装，重复 Install | 先 `DeleteAddon` 再装，或用 `UpdateAddon` |
| `FailedOperation` | `DescribeClusterStatus` 看状态 | 集群非 Running | 等集群 Running |

### 命令成功但状态不对 (exit = 0)

| 现象 | 诊断 | 根因 | 修复 |
|:--------|:----------|:------------|:-----|
| `Phase=Failed` | `DescribeAddon` → `Reason` | 配置错或镜像拉取失败 | 查 Reason，修正 RawValues 或镜像源 |
| 长时间 `Phase=Pending` | `kubectl get pods -n kube-system` | Pod 未就绪（资源不足/调度失败） | 查 Pod 事件，补资源或修污点 |
| 更新后版本未变 | `DescribeAddon` → `AddonVersion` | 更新未完成或版本号同 | 等异步完成；确认新版本号不同 |
| 插件 Running 但功能异常 | `kubectl logs -n kube-system -l app=<ADDON>` | 配置不兼容 | 解码 RawValues 核对配置 |

## 镜像缓存

> 镜像缓存（ImageCache）预置一组镜像到 CVM 快照，加速 Pod 拉起（避免每次拉远程镜像）。属插件域的进阶功能。

```bash
# 查询镜像缓存列表 (支持按 ID/名/过滤)
tccli tke DescribeImageCaches --region <REGION> --Limit 10
# expected: exit 0, ImageCaches[] + TotalCount (无缓存则空)
```
```json
{"TotalCount": 0, "ImageCaches": [], "RequestId": "..."}
```

```bash
# 创建镜像缓存 (需 VPC/子网/安全组, 创建 CVM 制作快照)
tccli tke CreateImageCache --region <REGION> \
  --Images '["nginx:1.25","redis:7"]' \
  --ImageCacheName "<CACHE_NAME>" \
  --VpcId "<VPC_ID>" --SubnetId "<SUBNET_ID>" --SecurityGroupIds '["<SG_ID>"]'
# expected: exit 0, 返回 ImageCacheId

# 匹配最合适的镜像缓存 (按待拉镜像列表匹配)
tccli tke GetMostSuitableImageCache --region <REGION> --Images '["nginx:1.25"]' --Snapshotter overlayfs
# expected: exit 0, 返回匹配的 ImageCacheId

# 更新镜像缓存 (改凭证/镜像列表)
tccli tke UpdateImageCache --region <REGION> --ImageCacheId "<CACHE_ID>" --ImageCacheName "<NEW_NAME>"
# expected: exit 0

# 删除镜像缓存 (批量)
tccli tke DeleteImageCaches --region <REGION> --ImageCacheIds '["<CACHE_ID>"]'
# expected: exit 0
```

> `CreateImageCache` 创建 CVM 实例制作快照，需 VpcId/SubnetId/SecurityGroupIds（有 CVM 计费成本）。`GetMostSuitableImageCache` 按镜像列表匹配已有缓存，`Snapshotter` 如 `overlayfs`。`UpdateImageCache` 可配 `ImageRegistryCredentials[]`（私有镜像仓库凭证）。

## 下一步

- [应用发布](../releases/manage.md) — 插件本质是 Helm Release
- [创建集群](../clusters/create.md) — 建集群时选装插件
- [故障排查](../troubleshooting.md) — 插件异常诊断

## 控制台替代方案

[容器服务控制台 - 插件管理](https://console.cloud.tencent.com/tke2/addon)
