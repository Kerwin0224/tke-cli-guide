---
doc_type: How-to
subtype: 6A
fused: true
---
# 管理集群插件

> 控制台: [容器服务控制台 - 插件管理](https://console.cloud.tencent.com/tke2/addon)
> 安装、更新、卸载 TKE 集群插件。插件是封装好的 Helm Chart，扩展集群功能。异步操作。
>
> 官方文档：[组件与应用概述](https://cloud.tencent.com/document/product/457/81234)
>
> 配额：集群插件受集群配额限制（单地域集群数默认 20），组件版本兼容性见 [VPC-CNI 组件变更记录](https://cloud.tencent.com/document/product/457/64920)。[配额说明](https://cloud.tencent.com/document/product/457/9087)

## 触发条件

- `DescribeAddon` → `Phase=Failed` 或长时间 `Pending`，插件安装/更新后状态异常
- `GetTkeAppChartList` 含目标插件但集群未安装，需 `InstallAddon` 部署
- `kubectl get pods -n kube-system -l app=<ADDON_NAME>` 显示 Pod `ImagePullBackOff` 或 `CrashLoopBackOff` — 看 [故障恢复]段


## 概述

插件生命周期三步：安装 → 更新 → 卸载。用户多在一个会话内完成。

| 操作 | 接口 | 作用 |
|:-----|:-----|:-----|
| 安装 | `InstallAddon` | 部署插件到集群 |
| 更新 | `UpdateAddon` | 升级版本或改配置 |
| 查询 | `DescribeAddon` | 查看插件状态 |
| 卸载 | `DeleteAddon` | 移除插件 |

操作是**异步**的：接口返回即提交，插件就绪需轮询 `DescribeAddon` 直到 `Phase=Succeeded`。

### 组件分桶（选型前）

| 类型 | 范围 | 服务边界 | 升级 |
|:-----|:-----|:---------|:-----|
| **系统组件** | CBS-CSI、IPAMD、监控、日志等默认核心组件 | TKE 优先保障稳定性；异常可能导致集群故障 | 部分特殊功能版本**不会**后台自动更新，须自行按 [组件版本维护说明](https://cloud.tencent.com/document/product/457/71800) 升级 |
| **增强组件**（扩展组件） | 用户按需安装的扩展功能包 | TKE 保障稳定性与安全/兼容修复；过旧版本可能失效 | 同上；安装前用 `DescribeAddonValues` 核集群可用版本 |
| **应用市场** | Helm Chart / 第三方应用 | **仅保障**在支持的集群类型与 K8s 版本上**可安装**；运行期问题、自定义配置导致的异常、不支持的版本组合由用户负责；无调试 SLA | 控制台/应用更新路径；非本文 `InstallAddon` 主路径 |

> 本文 `InstallAddon`/`UpdateAddon`/`DeleteAddon` 覆盖**增强组件**主路径。系统组件默认安装、通常不可当普通插件删除。应用市场走 Helm/应用市场文档，勿与 Addon Action 混用。

## 准备工作

### 环境检查

```bash
tccli --version
# expected: tccli 版本号

tccli tke DescribeClusterStatus --region ap-guangzhou --filter "ClusterStatusSet[?ClusterId=='<CLUSTER_ID>'] | [0].ClusterState"
# expected: "Running"
```

### 资源检查

```bash
# 确认插件未安装（避免重复）
tccli tke DescribeAddon --region ap-guangzhou --ClusterId "<CLUSTER_ID>" --AddonName "<ADDON_NAME>" \
  --filter "Addons[].{name:AddonName,phase:Phase}"
# expected: 列表字段为 Addons[]（非 AddonSet）；空数组（未安装）或 Phase=Succeeded（已装）。不传 AddonName 返回集群全部插件
```

## 关键字段

### InstallAddon

| 字段 | 类型 | 必填 | 约束 | 填错时的错误 |
|:------|------|:--------:|------------|---------------|
| ClusterId | string | 是 | `cls-xxxxxxxx` | `ResourceNotFound` |
| AddonName | string | 是 | 插件名：`cbs` / `eniipamd` / `tcr` / `tke-log-agent` / `cluster-autoscaler` 等；以 `DescribeAddonValues` 返回为准（部分集群 CBS 为 `cbs-csi`） | `InvalidParameterValue` |
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

### 步骤 2：安装插件

`InstallAddon` 必传 `ClusterId`/`AddonName`/`AddonVersion`。按场景**二选一**：A 默认配置（用插件 DefaultValues）或 B 自定义配置（传 `RawValues`）。

> ⚠️ **A 与 B 是二选一变体，不是先做 A 再做 B**——两者各调一次 `InstallAddon` 会装**两次同名插件**（第二次报已存在）。插件装好后改配置用 `UpdateAddon`，**禁用第二次 `InstallAddon` 改配置**。

#### 选项 A：默认配置

```bash
tccli tke InstallAddon --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" --AddonName "<ADDON_NAME>" --AddonVersion "<VERSION>"
# expected: exit 0, 返回 RequestId
```

| 占位符 | 含义 | 约束 | 如何获取 |
|:------------|:-----|:-----|:---------|
| `<CLUSTER_ID>` | 集群 ID | `cls-xxxxxxxx` | `tccli tke DescribeClusters` |
| `<ADDON_NAME>` | 插件名 | 见关键字段 `AddonName`；先 `GetTkeAppChartList` / `DescribeAddonValues` | `tccli tke DescribeAddonValues` |
| `<VERSION>` | 插件版本 | 兼容集群版本 | `tccli tke DescribeAddonValues` |

> 安装前用 `DescribeAddonValues` 查插件在该集群的可用版本与默认配置（参数以 `tccli tke DescribeAddonValues help --detail` 为准）：

```bash
# 查询插件可用版本与配置值（ClusterId + AddonName 定位）
tccli tke DescribeAddonValues --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" --AddonName "<ADDON_NAME>"
# expected: 返回 Values+DefaultValues; 插件模板渲染异常报 ResourceUnavailable（如 cfs 插件 nil pointer）
```

#### 选项 B：自定义配置

> **与 A 二选一，非在 A 之后执行**。用 `RawValues` 传 base64 编码的自定义 Values 覆盖默认配置。

```bash
# 生成 base64 配置
VALUES=$(echo '{"resources":{"limits":{"cpu":"500m"}}}' | base64)

tccli tke InstallAddon --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" --AddonName "<ADDON_NAME>" --AddonVersion "<VERSION>" \
  --RawValues "$VALUES"
# expected: exit 0
```

### 步骤 3：更新 — 升级版本

```bash
tccli tke UpdateAddon --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" --AddonName "<ADDON_NAME>" --AddonVersion "<NEW_VERSION>" \
  --UpdateStrategy rolling
# expected: exit 0
```

### 步骤 4：验证

```bash
tccli tke DescribeAddon --region ap-guangzhou --ClusterId "<CLUSTER_ID>" --AddonName "<ADDON_NAME>" \
  --filter "Addons[].{name:AddonName,ver:AddonVersion,phase:Phase,reason:Reason}"
# expected: Addons[].Phase="Succeeded", Reason 为空（列表键 Addons，项含 AddonName/AddonVersion/Phase/Reason/RawValues/CreateTime）
```

| 维度 | 命令 | 预期 |
|:-----|:-----|:-----|
| 插件状态 | `DescribeAddon` → `Addons[].Phase` | `Succeeded` |
| 版本一致 | `DescribeAddon` → `Addons[].AddonVersion` | 等于安装/更新的版本 |
| 运行状态 | `kubectl get pods -n kube-system -l app=<ADDON_NAME>` | Pod Running |
| 无异常 | `DescribeAddon` → `Addons[].Reason` | 空 |

> `Phase` 枚举：`Pending`/`Succeeded`/`Failed`。`Failed` 时查 `Reason` 字段定位原因。

## 清理

> **副作用警告**：卸载插件会移除其管理的资源。如 `cbs-csi` 卸载后已有 PVC 可能无法挂载。
>
> ⚠️ **高危操作**：误删核心组件（如 DNS 插件、网络插件 CBS-CSI/IPAMD）会导致集群 DNS 解析失败、存储挂载异常或 Pod 网络不通，可能引发生产故障。[常见高危操作](https://cloud.tencent.com/document/product/457/39539)

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
| `FailedOperation` | `DescribeClusterStatus` 查看状态 | 集群非 Running | 等集群 Running |

### 命令成功但状态不对 (exit = 0)

| 现象 | 诊断 | 根因 | 修复 |
|:--------|:----------|:------------|:-----|
| `Phase=Failed` | `DescribeAddon` → `Reason` | 配置错或镜像拉取失败 | 查 Reason，修正 RawValues 或镜像源 |
| 长时间 `Phase=Pending` | `kubectl get pods -n kube-system` | Pod 未就绪（资源不足/调度失败） | 查 Pod 事件，补资源或修污点 |
| 更新后版本未变 | `DescribeAddon` → `AddonVersion` | 更新未完成或版本号同 | 等异步完成；确认新版本号不同 |
| 插件 Running 但功能异常 | `kubectl logs -n kube-system -l app=<ADDON>` | 配置不兼容 | 解码 RawValues 核对配置 |

## 镜像缓存

> 镜像缓存（ImageCache）预置一组镜像到 CVM 快照，加速 Pod 启动（避免每次拉远程镜像）。属插件域的进阶功能。

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

## 收尾确认

> kubectl（K8s 原生命令，非 tccli；TCCLI 管 TKE 抽象层不提供 K8s 资源操作能力）
<!-- tccli管Addon生命周期，kubectl查Pod部署状态，非tccli边界 -->
```bash
# 插件 Phase=Succeeded（Verify 查 Phase/版本/Reason，此处端到端核 Pod 真运行 + 衔接前置）
tccli tke DescribeAddon --region ap-guangzhou --ClusterId "<CLUSTER_ID>" --AddonName "<ADDON_NAME>" \
  --filter "{name:AddonName,phase:Phase,version:Version}"
# expected: phase=Succeeded

# 业务可用性端到端：插件 Pod 真运行且 Ready（Verify 查 Phase=Succeeded 但未核 Pod Ready 数）
kubectl get pods -n kube-system -l app=<ADDON_NAME> -o wide \
  --no-headers | awk '{print $2, $3}'
# expected: READY 列全为 1/1（或 N/N），STATUS=Running

# 衔接下一步前置：插件就绪是使用其功能的前置（如 cbs-csi 就绪才能创建 PVC）
kubectl get pods -n kube-system -l app=<ADDON_NAME> --no-headers | wc -l
# expected: ≥1（插件 Pod 已调度运行）→ 插件管理闭环完成
```

> 插件 Phase=Succeeded + Pod 全 Ready + Pod 数 ≥1 = 端到端闭环。Verify 段查 TKE 侧 Phase 与版本，此处用 kubectl 核 Pod 真运行（业务可用性），确认插件功能可被集群使用（如 cbs-csi 就绪才能创建 PVC，是进下一阶段的前置）。

---

## 下一步

- [应用发布](../releases/manage.md) — 插件本质是 Helm Release
- [创建集群](../clusters/create.md) — 建集群时选装插件
- [故障排查](../troubleshooting.md) — 插件异常诊断
