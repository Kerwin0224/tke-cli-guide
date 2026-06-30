# qGPU 概述

> 对照官方：[qGPU 概述](https://cloud.tencent.com/document/product/457/61448) · page_id `61448`

## 概述

qGPU 是 TKE 的 GPU 共享技术，将一张物理 GPU 虚拟化为多张小 GPU（vGPU），支持算力隔离和显存隔离，允许多个 Pod 共享同一张物理 GPU。

| 维度 | 标准 GPU（nvidia.com/gpu） | qGPU 共享 |
|------|--------------------------|----------|
| 资源粒度 | 整卡分配（1 张、2 张...） | 百分比算力 + GiB 显存 |
| 多 Pod 共享 | 不支持 | 支持 |
| 算力隔离 | 无需（整卡独占） | `qgpu-core`：1-100 单卡，>100 多卡 |
| 显存隔离 | 无需 | `qgpu-memory`：GiB 显存 |
| 在线/离线混部 | 不支持 | 支持（`app-class` annotation） |
| 多卡互联 | 无需（直接申请多张） | `qgpu-core > 100` 自动跨卡 |
| 集群开关 | 无额外开关 | `CreateCluster` → `QGPUShareEnable: true` |
| 组件 | `nvidia-gpu` 即可 | 需额外 `qgpu` 组件 |

**选型建议**：
- 独占 GPU 推理/训练且 GPU 型号满足需求 → 标准 GPU（`nvidia.com/gpu`）
- 多业务共享 GPU、碎片化推理、开发测试环境 → qGPU
- 在线业务需要高优保障、离线任务利用空闲 GPU → qGPU 离在线混部

## 前置条件

- [环境准备](../../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置，region 为目标地域

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:DescribeClusters, tke:DescribeAddon
# 验证：执行 DescribeClusters 确认权限
tccli tke DescribeClusters --region <Region>
# expected: exit 0，返回集群列表（可为空）

# 4. 检查 kubectl 版本（数据面验证需要）
kubectl version --client
# expected: Client Version: v1.28 或更高
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

### 资源检查

```bash
# 5. 确认目标集群存在并检查 QGPU 开关
tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
# expected: TotalCount >= 1，ClusterStatus 为 Running，ClusterAdvancedSettings.QGPUShareEnable 字段可见（true 或 false）

# 6. 获取 kubeconfig 并验证连通性（数据面验证需要）
tccli tke DescribeClusterKubeconfig --region <Region> \
    --ClusterId CLUSTER_ID | jq -r '.Kubeconfig' > ~/.kube/config
# expected: exit 0，kubeconfig 写入 ~/.kube/config

kubectl get ns
# expected: 返回 default 等系统命名空间，确认 kubectl 可达
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

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `REGION` | 集群所在地域 | 如 `ap-guangzhou` | `tccli configure list` 确认当前 region |
| `CLUSTER_ID` | 集群 ID | 格式 `cls-xxxxxxxx` | `tccli tke DescribeClusters --region <Region>` |
| `GPU_NODE` | GPU 节点名 | 节点须有 GPU，且 qgpu 组件已就绪 | `kubectl get nodes -l accelerator=nvidia` |

### 版本与规格要求

- K8s 版本：qGPU 要求 K8s >= 1.14.x，推荐 >= 1.30。确认可用版本：`tccli tke DescribeVersions --region <Region>`
- 节点要求：原生节点，GPU 架构为 V100 / T4 / A100 / A10，驱动版本 450 / 470 / 515 / 525 / 535 / 550 / 570 系列
- 运行时：`containerd`（托管集群强制）
- 本文验证环境：`MANAGED_CLUSTER`，无 GPU 节点。数据面命令中涉及 `GPU_NODE`、`nvidia-smi` 的步骤在无 GPU 节点时仅作概念说明，标注预期输出供有 GPU 节点的环境参考

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看集群 QGPU 开关 | `DescribeClusters` | 是 |
| 查看 qgpu 组件状态 | `DescribeAddon --AddonName qgpu` | 是 |
| 创建集群时开启 QGPU | `CreateCluster` → `QGPUShareEnable=true` | 否 |
| 安装 qgpu 组件 | `InstallAddon --AddonName qgpu` | 否 |

## 关键字段说明

`DescribeClusters` 返回结果中与 qGPU 相关的核心字段：

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `QGPUShareEnable` | Boolean | 否（创建时） | `true`（启用 qGPU）或 `false`（禁用，默认）。创建时在 `ClusterAdvancedSettings` 中设置 | 设为 `false` 或不传 → 集群不支持 qGPU，后续无法使用 vGPU |
| `qgpu-core` | 资源 | 否（Pod 级） | 1-100：单卡百分比算力；>100：多卡（如 200 = 2 张 GPU 各 100%） | 填 0 或超过节点总 GPU 算力 → Pod 无法调度 |
| `qgpu-memory` | 资源 | 否（Pod 级） | GiB 整数。不能超过单张 GPU 显存总量 | 填值超过物理 GPU 显存 → Pod 无法调度 |
| `qgpu-core-greedy` | 资源 | 否（离线 Pod） | 1-100：贪婪算力百分比，离线 Pod 专用 | 用于非离线 Pod → 调度行为不符合预期 |

## 操作步骤

### 步骤 1：确认集群 QGPU 开关状态

查询集群是否已开启 QGPU：

```bash
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterAdvancedSettings.QGPUShareEnable 可见
```

**预期输出**（已启用 QGPU 的集群）：

```json
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "cls-example",
            "ClusterName": "CLUSTER_NAME",
            "ClusterType": "MANAGED_CLUSTER",
            "ClusterStatus": "Running",
            "ClusterAdvancedSettings": {
                "QGPUShareEnable": true
            }
        }
    ],
    "RequestId": "..."
}
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `CLUSTER_ID` | 集群 ID | 格式 `cls-xxxxxxxx` | `tccli tke DescribeClusters --region <Region>` |
| `REGION` | 地域 | 如 `ap-guangzhou` | `tccli tke DescribeRegions` |

如返回的 `QGPUShareEnable` 为 `false` 或不存在，说明集群创建时未启用 QGPU。QGPU 开关只能在创建集群时设置，无法事后修改。需重新创建集群并在 `ClusterAdvancedSettings` 中设置 `QGPUShareEnable: true`。

### 步骤 2：确认 qgpu 组件状态

```bash
tccli tke DescribeAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName qgpu
# expected: Phase 为 Succeeded 或 Installing
```

**预期输出**（已安装的 qGPU 组件）：

```json
{
    "Addons": [
        {
            "AddonName": "qgpu",
            "AddonVersion": "1.0.0",
            "Phase": "Succeeded",
            "Reason": ""
        }
    ],
    "RequestId": "..."
}
```

| Phase | 说明 |
|-------|------|
| `Succeeded` | 组件已安装并就绪 |
| `Installing` | 组件正在安装中 |
| `Failed` | 安装失败，参见 [排障](#排障) |

### 步骤 3：查看节点 qGPU 资源（数据面，需 GPU 节点）

qgpu 组件就绪后，GPU 节点会注册 `tke.cloud.tencent.com/qgpu-core` 和 `tke.cloud.tencent.com/qgpu-memory` 扩展资源。**此步骤需要 GPU 节点**，无 GPU 节点的环境仅作概念了解。

```bash
# 列出带 GPU 标签的节点
kubectl get nodes -l accelerator=nvidia
# expected: 返回 GPU 节点列表（无 GPU 节点则返回 No resources found）
```

```text
NAME  STATUS  AGE
...
```

```bash
# 查看节点的 qGPU 扩展资源
kubectl describe node GPU_NODE
# expected: Capacity/Allocatable 中包含 tke.cloud.tencent.com/qgpu-core 和 tke.cloud.tencent.com/qgpu-memory
```

**预期输出**（GPU 节点资源摘要）：

```text
Capacity:
  cpu:                             64
  memory:                          256Gi
  nvidia.com/gpu:                  2
  tke.cloud.tencent.com/qgpu-core: 200
  tke.cloud.tencent.com/qgpu-memory: 40
Allocatable:
  tke.cloud.tencent.com/qgpu-core: 200
  tke.cloud.tencent.com/qgpu-memory: 40
```

> `qgpu-core: 200` 表示该节点 2 张 GPU 各 100 算力，总计 200 可分配算力。
> `qgpu-memory: 40` 表示该节点 2 张 GPU 各 20GiB 显存，总计 40GiB 可分配显存。

### 步骤 4：qGPU 能力总览

qGPU 提供以下使用模式：

| 模式 | Annotation/资源 | 说明 | 详见 |
|------|---------------|------|------|
| **基础 qGPU** | `qgpu-core` + `qgpu-memory` | 单 Pod 使用 vGPU 算力和显存 | [使用 qGPU](../使用%20qGPU/tccli%20操作.md) |
| **多卡互联** | `qgpu-core > 100` | 单 Pod 跨多张物理 GPU | [qGPU 多卡互联](../qGPU%20多卡互联/tccli%20操作.md) |
| **离在线混部** | `app-class: offline/online` | 在线高优 + 离线低优同节点共用 GPU | [使用 qGPU 离在线混部](../qGPU%20离在线混部/使用%20qGPU%20离在线混部/tccli%20操作.md) |

## 验证

### 控制面（tccli）

```bash
# 1. 确认 QGPU 集群级开关已启用
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterAdvancedSettings.QGPUShareEnable = true
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

```bash
# 2. 确认 qgpu 组件已安装
tccli tke DescribeAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName qgpu
# expected: Phase = "Succeeded"
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

### 数据面（需 GPU 节点）

```bash
# 3. 确认节点有 qGPU 扩展资源
kubectl describe node GPU_NODE
# expected: Allocatable 含 tke.cloud.tencent.com/qgpu-core 和 qgpu-memory，值 > 0
```

```text
NAME  STATUS  AGE
...
```

| 验证维度 | 命令 | 预期 |
|---------|------|------|
| 集群开关 | `DescribeClusters` | `QGPUShareEnable = true` |
| 组件状态 | `DescribeAddon --AddonName qgpu` | `Phase = Succeeded` |
| 节点算力 | `kubectl describe node` | `qgpu-core > 0` |
| 节点显存 | `kubectl describe node` | `qgpu-memory > 0` |

> 本文验证环境为 `MANAGED_CLUSTER` 且无 GPU 节点：控制面（tccli）验证可完整执行，数据面（kubectl）的 `kubectl describe node GPU_NODE` 步骤需有 GPU 节点的集群才能返回预期内容。无 GPU 节点时 `kubectl get nodes -l accelerator=nvidia` 返回 `No resources found`，属预期行为。

## 清理

本页为概念概述页，无资源需清理。如需卸载 qGPU 组件，参见 [使用 qGPU](../使用%20qGPU/tccli%20操作.md) 的清理章节。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DescribeAddon --AddonName qgpu` 返回 `ResourceNotFound` | `tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'` 检查集群是否存在 | 集群 ID 错误或 qgpu 组件未安装 | 确认集群 ID 正确；如未安装 qgpu，参见 [使用 qGPU](../使用%20qGPU/tccli%20操作.md) 安装 |
| `DescribeClusters` 返回 `InvalidParameter.ClusterId` | 检查 `--ClusterIds` 中的集群 ID 格式 | 集群 ID 格式错误或集群不属于当前账号/地域 | 用 `tccli tke DescribeClusters --region <Region>` 列出全部集群确认 ID；检查 `tccli configure list` 确认 `region` 与集群所在区域一致 |
| `DescribeClusters` 返回 `UnauthorizedOperation` | `tccli configure list` 检查凭据 | 子账号缺少 `tke:DescribeClusters` 权限（此为环境限制，非命令错误） | 联系主账号授予 `QcloudTKEFullAccess` 或最小权限 `tke:DescribeClusters` |
| `kubectl describe node` 无 `qgpu-core` 资源 | `kubectl get nodes GPU_NODE -o yaml | grep -A5 "capacity"` 检查资源 | 节点非 GPU 节点，或 qgpu 组件未安装/未就绪 | 确认节点含 GPU（`kubectl get nodes -l accelerator=nvidia`）；确认 qgpu 组件 Phase=Succeeded |
| `kubectl get nodes -l accelerator=nvidia` 返回 `No resources found` | `kubectl get nodes` 查看全部节点 | 集群无 GPU 节点（本文环境即如此，属预期行为） | 本文为概念页，无需 GPU 节点。如需实操验证，参考 [使用 qGPU](../使用%20qGPU/tccli%20操作.md) 添加 GPU 原生节点 |

### QGPU 未启用

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 集群创建后无法使用 qGPU | `tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]' | jq '.Clusters[0].ClusterAdvancedSettings.QGPUShareEnable'` | 创建集群时未设置 `QGPUShareEnable: true` | QGPU 只能在创建时启用，无法事后修改。需创建新集群并在 `ClusterAdvancedSettings` 中设置 `QGPUShareEnable: true` |
| qgpu 组件已安装但 Pod 无法调度 vGPU | `kubectl describe node GPU_NODE | grep qgpu` 检查节点扩展资源 | 节点 GPU 驱动未安装或 nvidia-gpu 组件未安装 | 先安装 `nvidia-gpu` 组件，再安装 `qgpu` 组件 |

## 下一步

- [使用 qGPU](https://cloud.tencent.com/document/product/457/65734) — 安装 qgpu 组件并部署使用 vGPU 的 Pod
- [qGPU 多卡互联](https://cloud.tencent.com/document/product/457/127667) — Pod 跨多张物理 GPU 使用 vGPU
- [使用 qGPU 离在线混部](https://cloud.tencent.com/document/product/457/81034) — 在线高优 + 离线低优同节点混部
- [GPU 故障检测与自愈](https://cloud.tencent.com/document/product/457/127503) — NPDPlus 检测 GPU 故障并自动恢复
- [GPU 监控指标获取](https://cloud.tencent.com/document/product/457/90912) — 通过 DCGM 获取 GPU 监控指标

## 控制台替代

[TKE 控制台 → 集群 → 组件管理](https://console.cloud.tencent.com/tke2/cluster)：可在集群详情中查看 qGPU 组件的安装状态。
