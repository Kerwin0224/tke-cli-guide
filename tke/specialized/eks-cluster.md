---
doc_type: How-to
subtype: 6A
fused: true
---
# EKS 弹性容器实例

> 存量 EKS 集群运维 + 控制台「容器实例」CPU/GPU（`*EKSContainerInstance*`）。
> 控制台: [容器实例 · CPU 实例](https://console.cloud.tencent.com/tke2/eksci) · [容器实例 · GPU 实例](https://console.cloud.tencent.com/tke2/eksci-gpu)
>
> 官方文档：[超级节点资源规格](https://cloud.tencent.com/document/product/457/39808) · [边缘集群迁移至标准集群](https://cloud.tencent.com/document/product/457/110447)
>
> 配额：弹性集群数与容器实例规格受 [资源规格](https://cloud.tencent.com/document/product/457/39808) 限制，集群数默认 20。[配额说明](https://cloud.tencent.com/document/product/457/9087)

> ⚠️ **「Serverless」不是单一产品**，勿把下列三条当成同一条 Action 路径：
>
> | 路径 | 控制台 | 建算力 Action | 与 `CreateCluster` |
> |:-----|:-------|:--------------|:-------------------|
> | **新建免 CVM 算力（推荐）** | 标准集群 → 虚拟节点 / 超级节点 | ① `CreateCluster` 建标准集群 → ② `CreateClusterVirtualNode(Pool)` 或 `CreateNodePool Type=Super` | `CreateCluster` **只建控制面**；免 CVM 容量在第 ② 步，见 [虚拟节点](../nodes/virtual-nodes.md) |
> | **存量 EKS 集群** | 新建入口已关闭 | `CreateEKSCluster`（勿再用于新建） | **无关** `CreateCluster` |
> | **容器实例 CPU/GPU** | 容器实例 → CPU / GPU | `CreateEKSContainerInstances`（无 `ClusterId`） | **无关** 任何集群创建 |
>
> 新建免 CVM 算力：**禁止**再用本文 `CreateEKSCluster`。两步走 [创建标准集群](../clusters/create.md) → [虚拟节点/超级节点](../nodes/virtual-nodes.md)。存量 EKS 集群与存量容器实例可继续按本文运维。
>
> **本文创建流（容器实例）**：控制台左侧 **容器实例** 下分 **CPU 实例** / **GPU 实例** 两页（非标准集群 4 步）。两页同一 Action 族 `*EKSContainerInstance*`，用规格字段区分 CPU/GPU。勿套用 [tke/index 托管四步全景](../index.md#控制台创建流全景)。

> ⚠️ **产品调整（功能迁移 + 存量可用）**：EKS 集群（独立 Serverless 控制面）**新建入口已关闭**，不再承接新建需求；新建免 CVM 算力改走 [标准集群](../clusters/create.md)（`CreateCluster`）+ [虚拟节点/超级节点](../nodes/virtual-nodes.md)。**存量 EKS 集群与存量容器实例可继续按本文运维**（功能迁移升级至标准集群，非强制下线）。
>
> ⚠️ **高危操作**：产品已调整，禁止再用 `CreateEKSCluster` 新建 EKS 集群；建议存量 EKS 集群尽早迁移至标准集群 + 虚拟节点方案。[常见高危操作](https://cloud.tencent.com/document/product/457/39539)

## 触发条件

- **新建免 CVM 算力（要 K8s 编排）** → 不用本文：走 [标准集群](../clusters/create.md)（`CreateCluster`）+ [虚拟节点](../nodes/virtual-nodes.md)；**不是**一次 `CreateCluster` 就等于 Serverless
- 控制台「容器实例 → CPU 实例 / GPU 实例」→ 本文 [创建容器实例](#创建容器实例-部署-pod)（`CreateEKSContainerInstances`）；与节点池「CPU 节点 / GPU 节点」、与虚拟节点 **都不是**同一功能
- 存量：`DescribeEKSClusters` 已有 EKS 集群，需查凭证/更新/事件 — 用本文「EKS 集群」段
- 存量容器实例出现 `ImagePullBackOff`/`OutOfCpu`/`CreatePodSandboxFailed` — 用 `DescribeEKSContainerInstanceEvent`

## 准备工作

- 已安装 TCCLI 并配置凭证 (见 [配置凭证](../../getting-started/credentials.md))
- 已确认地域支持容器实例：`DescribeEKSContainerInstanceRegions`（勿仅用 `DescribeRegions`）
- 规格在支持表内：见 [资源规格](https://cloud.tencent.com/document/product/457/39808)；GPU 还须账号对该 `GpuType` 有售卖/配额
- EKS / 容器实例相关资源配额充足

## 概述

「EKS」在文档与控制台里常指两类对象，须分开：

| 对象 | 含义 | 主 Action |
|:-----|:-----|:----------|
| **EKS 集群**（存量） | 独立 Serverless K8s 控制面；新建入口已关闭 | `*EKSCluster*` |
| **EKS 容器实例** | 无集群归属的单次/常驻容器；控制台「容器实例」CPU/GPU | `*EKSContainerInstance*` |

二者都按 vCPU/内存（及 GPU）用量计费，**都不是** `CreateCluster`。标准集群上的免 CVM 容量走 [虚拟节点](../nodes/virtual-nodes.md)。

| 特性 | EKS 容器实例 | 标准集群 + 虚拟节点 | 标准集群 + CVM 节点 |
|------|:---:|:---:|:---:|
| 是否要先有集群 | 否（无 `ClusterId`） | 是（先 `CreateCluster`） | 是 |
| 节点形态 | 无 K8s Node 对象 | 虚拟/超级节点（eklet） | CVM 节点 |
| 计费 | 按实例规格用量 | Pod 按用量 | CVM 按实例 |
| 适合场景 | 无编排需求的单次任务 | 要 K8s 编排的免 CVM 负载 | 常驻、需 OS 级管控 |

### 控制台「CPU 实例 / GPU 实例」↔ tccli

| 控制台 | URL 路径 | tccli | 规格区分 |
|:-------|:---------|:------|:---------|
| 容器实例 → **CPU 实例** | `/tke2/eksci` | `CreateEKSContainerInstances`（不传 `GpuType`/`GpuCount`） | 顶层必填 `--Cpu` / `--Memory`；可选 `--CpuType`（`intel` / `amd` / `amd,intel`） |
| 容器实例 → **GPU 实例** | `/tke2/eksci-gpu` | **同一** `CreateEKSContainerInstances` | 另传 `--GpuType` + `--GpuCount`；容器内可设 `GpuLimit` |
| 列表 / 重启 / 更新 / 删除 | 两页共用 | `Describe*` / `Restart*` / `Update*` / `Delete*EKSContainerInstance(s)` | 出参含 `GpuType`/`GpuCount`；CPU 实例多为空/`0` |

> **不是**集群节点池里的「CPU 节点 / GPU 节点」（那是 `CreateNodePool` 的 `InstanceTypes` + `DescribeGPUInfo`/`GPUArgs`，见 [节点池创建](../nodes/nodepool-create.md) / [节点实例运维](../nodes/instance-ops.md#查询-gpu-驱动版本-2022-05-01)）。

> **控制台维度**：控制台「容器实例 → CPU 实例 / GPU 实例」是两页，但对应 tccli 同一 `CreateEKSContainerInstances`，变量只在是否传 `GpuType`/`GpuCount`（见下字段分叉决策表）。

> **CPU / GPU 字段分叉决策表（同一 `CreateEKSContainerInstances`，靠规格字段分叉）**：控制台「CPU 实例」与「GPU 实例」是两页，但 `tccli` 是**同一个 Action**，变量只在是否传 `GpuType`/`GpuCount`。下表为执行决策依据（字段以 `tccli tke CreateEKSContainerInstances help --detail` 与 API 契约为准；`Cpu`/`Memory`/`VpcId`/`SubnetId`/`SecurityGroupIds`/`EksCiName`/`Containers` 为顶层必填，不随 CPU/GPU 变化）。

| 决策点 | CPU 实例（控制台 `/tke2/eksci`） | GPU 实例（控制台 `/tke2/eksci-gpu`） | 字段契约 |
|:---|:---|:---|:---|
| 是否传 `GpuType` | **不传** | **传**（`1/4*V100`/`1/2*V100`/`V100`/`1/4*T4`/`1/2*T4`/`T4`） | `GpuType` 顶层 optional；缺省空字符串 |
| 是否传 `GpuCount` | **不传**（出参 `GpuCount=0`） | **传**（GPU 卡数，须与 `GpuType` 规格表匹配） | `GpuCount` 顶层 optional int；缺省 0 |
| `Cpu` / `Memory` | 必传（与 CPU 规格表匹配） | 必传（须与所选 `GpuType` 规格表匹配，≥ GPU 配套的最小 CPU/内存） | 顶层 required=True；单位 核 / GiB |
| `CpuType` | 可选（`intel`/`amd`/`amd,intel` 优先级） | 可选（GPU 场景通常省略，由 `GpuType` 定卡型） | 顶层 optional；不填则不强制 CPU 型号 |
| `Containers[].GpuLimit` | 不传（CPU 实例无 GPU 概念） | 可选（该容器可用 GPU 上限，整数；须 ≤ `GpuCount`） | `Container.GpuLimit` 容器内 optional int |
| 出参 `GpuType`/`GpuCount` | 空 / `0` | 与创建一致 | `DescribeEKSContainerInstances` 出参 |

> **分叉约束**：CPU 与 GPU 不是两个 Action、不是「先建 CPU 再升级 GPU」——同一 Action 调两次建**两个**实例；改规格走 `UpdateEKSContainerInstance`（`Containers[]` 覆盖式整体替换，须带上 `GpuLimit` 才能保留 GPU 配额）。GPU 库存不足时 Create 仍可能 exit 0 返回 `EksCiIds`，随后事件 `CreatePodSandboxFailed`（Message 含 `gpu instance types is empty` / `not found t4 gpu info`），须 `Describe`+`DescribeEKSContainerInstanceEvent` 确认 Running，勿凭 exit 码判可用。

## 关键操作

### 识别旧 EKS 集群创建脚本（禁止调用新建）

控制台新建 Serverless/EKS 集群入口已关闭。旧脚本中的 `CreateEKSCluster` 仅用于识别待迁移资产，不得执行；历史字段包括 `ClusterName`、`VpcId` 和 `SubnetIds`。新建免 CVM 编排需求改用 [创建标准集群](../clusters/create.md) + [虚拟节点](../nodes/virtual-nodes.md)。

### 查询 EKS 集群

```bash
# 列出所有 EKS 集群
# Filters 当前仅支持单一 ClusterName 条件
tccli tke DescribeEKSClusters --region ap-guangzhou \
  --Filters '[{"Name":"ClusterName","Values":["<CLUSTER_NAME>"]}]'
# expected: 顶层 TotalCount/Clusters/RequestId (tccli 剥离 Response 包装层, 无 Response 键); Clusters[] 元素含 ClusterId/ClusterName/VpcId/SubnetIds/K8SVersion/Status/ClusterDesc/CreatedTime

# 获取凭证（须传 EKS 集群 ID；标准托管集群 ID → ResourceNotFound: `… is not exist`）
tccli tke DescribeEKSClusterCredential --region ap-guangzhou --ClusterId "<CLUSTER_ID>"
# expected: 返回 kubeconfig；非 EKS ID → ResourceNotFound

# 查询支持的地域
tccli tke DescribeEKSContainerInstanceRegions --region ap-guangzhou
# expected: Region 列表
```

### 创建容器实例 (部署 Pod) {#创建容器实例-部署-pod}

> 对应控制台 **容器实例** 下的 CPU / GPU 两页。`CreateEKSContainerInstances` **无 `ClusterId`**：顶层是 `VpcId`/`SubnetId`/`SecurityGroupIds`/`EksCiName`/`Containers`/`Cpu`/`Memory`/`RestartPolicy` 等（EKS CI 是独立 Serverless 容器，不归属某集群）。完整入参以 `tccli tke CreateEKSContainerInstances help --detail` 为准。
>
> **选项 A（CPU）与选项 B（GPU）二选一**：各调一次 `CreateEKSContainerInstances` 会建**两个**实例；不是先建 CPU 再「升级」为 GPU。改规格用 `UpdateEKSContainerInstance`（`Containers[]` 覆盖式整体替换）。

#### 选项 A：CPU 实例（控制台「CPU 实例」）

顶层 `--Cpu` / `--Memory` 为 API 必填（单位：核 / GiB）。不传 `GpuType`/`GpuCount`。拉公网镜像时加 `--AutoCreateEip true`（可与 `--AutoCreateEipAttribute` 同传带宽/计费；与 `ExistedEipIds` 互斥）。

```bash
tccli tke CreateEKSContainerInstances \
  --region ap-guangzhou \
  --VpcId "<VPC_ID>" \
  --SubnetId "<SUBNET_ID>" \
  --SecurityGroupIds '["<SECURITY_GROUP_ID>"]' \
  --EksCiName "<EKSCI_NAME>" \
  --Cpu 0.5 \
  --Memory 1.0 \
  --CpuType "intel" \
  --RestartPolicy Always \
  --AutoCreateEip true \
  --AutoCreateEipAttribute '{"InternetMaxBandwidthOut":1,"InternetChargeType":"TRAFFIC_POSTPAID_BY_HOUR"}' \
  --Containers '[
    {
      "Name": "nginx",
      "Image": "nginx:1.25-alpine",
      "Cpu": 0.5,
      "Memory": 1
    }
  ]'
# expected: { "EksCiIds": ["eksci-xxxxxxxx"], "RequestId": "..." }（注意是数组 EksCiIds，不是单数字段 EksCiId）
```

| 字段 | 位置 | 说明 |
|:-----|:-----|:-----|
| `Cpu` / `Memory` | 顶层（必填） | 实例总规格；须落在 [资源规格](https://cloud.tencent.com/document/product/457/39808) 表内 |
| `CpuType` | 顶层（可选） | `intel` / `amd`；可写优先级如 `amd,intel`（优先 amd，不足则 intel） |
| `Containers[].Cpu` / `Memory` | 容器（可选） | 单容器上限，不可超过实例总核数/总内存 |
| `AutoCreateEip` / `AutoCreateEipAttribute` | 顶层（可选） | 公网拉镜像或对外服务时用；无公网且镜像不可达 → 事件 `ErrImagePull` / `ImagePullBackOff` |

创建返回后轮询 `DescribeEKSContainerInstances`：`Status` 经 `Pending` → `Running`；`Pending` 期间顶层 `Cpu`/`Memory` 可能仍为 `0`，就绪后回填为创建值。CPU 实例出参多为 `GpuType=""`、`GpuCount=0`，`Containers[].GpuLimit` 为空。

#### 选项 B：GPU 实例（控制台「GPU 实例」）

与选项 A **同一 Action**；另传顶层 `GpuType` + `GpuCount`，容器侧可设 `GpuLimit`。`Cpu`/`Memory` 仍必填，且须与所选 `GpuType` 的规格表匹配。

```bash
tccli tke CreateEKSContainerInstances \
  --region ap-guangzhou \
  --VpcId "<VPC_ID>" \
  --SubnetId "<SUBNET_ID>" \
  --SecurityGroupIds '["<SECURITY_GROUP_ID>"]' \
  --EksCiName "<EKSCI_NAME>-gpu" \
  --Cpu 2.0 \
  --Memory 8.0 \
  --GpuType "1/4*T4" \
  --GpuCount 1 \
  --RestartPolicy Always \
  --Containers '[
    {
      "Name": "train",
      "Image": "<GPU_IMAGE>",
      "Cpu": 2,
      "Memory": 8,
      "GpuLimit": 1
    }
  ]'
# expected: 参数合法时 exit 0，体为 { "EksCiIds": ["eksci-xxxxxxxx"], "RequestId": "..." }
# 注意：Create 成功只表示受理；库存/机型不可用时 Status 可长期 Pending，事件 Reason=CreatePodSandboxFailed
# （常见 Message：`gpu instance types is empty` / `not found t4 gpu info`）。须再 Describe + DescribeEKSContainerInstanceEvent 确认 Running，失败则 Delete。
```

| 字段 | 位置 | 说明 |
|:-----|:-----|:-----|
| `GpuType` | 顶层 | help 列出的型号：`1/4*V100` / `1/2*V100` / `V100` / `1/4*T4` / `1/2*T4` / `T4`（以 `help --detail` 与资源规格页最新表为准） |
| `GpuCount` | 顶层 | GPU 卡数；须与 `GpuType` 及规格表匹配 |
| `Containers[].GpuLimit` | 容器 | 该容器可用的 GPU 上限（整数） |

> 镜像须含对应 CUDA/驱动用户态；卡型与规格半常量见 [资源规格](https://cloud.tencent.com/document/product/457/39808)。标准集群内超级节点上的 GPU Pod 用 Annotation `gpu-type`，见 [虚拟节点](../nodes/virtual-nodes.md)，**不是**本 Action 的 `GpuType`。
>
> 入参非法时 Create 可直接 exit≠0（`InvalidParameter*` 等，以 `Error.Code` 为准）。入参合法但地域无对应 GPU 库存时，Create 仍可能 exit 0 并返回 `EksCiIds`，随后事件报 `CreatePodSandboxFailed`——与控制台创建后列表长期 Pending / 失败一致，用事件 Message 排查，不要仅凭 Create 的 exit 码判断可用。

#### 容器实例的进阶组合

`CreateEKSContainerInstances` 顶层另有独立任务型组合参数，按场景选配（与 CPU/GPU 选项正交，可叠加）：

| 参数 | 触发条件 | 说明 |
|------|---------|------|
| `EksCiVolume` | 容器需要持久存储 | CBS 云盘或 NFS 挂载 |
| `InitContainers` | 主容器启动前执行初始化 | 结构同 `Containers`（可含 `GpuLimit`） |
| `AutoCreateEip` / `AutoCreateEipAttribute` | 容器需要公网 EIP（拉公网镜像等） | `AutoCreateEip=true` 自动建绑 EIP；带宽/计费用 Attribute；与 `ExistedEipIds` 互斥 |
| `DnsConfig` | 自定义 DNS | nameserver/search/options |

##### 持久存储（EksCiVolume）

```bash
--EksCiVolume '{
  "CbsVolumes":[{"Name":"data","DiskType":"CLOUD_PREMIUM","DiskSize":50}],
  "NfsVolumes":[{"Path":"/mnt/nfs","Server":"<NFS_SERVER>","ReadOnly":false}]
}'
```

> CBS 云盘随实例生命周期创建销毁；NFS 是外部共享存储。

## 跨字段约束

| `ExistedEipIds` | `AutoCreateEip` | `AutoCreateEipAttribute` | `Replicas` | 关系 |
|:----------------|:----------------|:-------------------------|:-----------|:-----|
| 传已有 EIP 列表 | `false` 或不传 | 不传 | 必须等于 EIP 数量 | 绑定已有 EIP |
| 不传 | `true` | 可选，用于带宽与计费参数 | 任意合法副本数 | 自动创建 EIP |
| 不传 | `false` 或不传 | 不传 | 任意合法副本数 | 不绑定 EIP |

`ExistedEipIds` 与自动创建 EIP 的两项参数互斥；不能一边绑定已有 EIP，一边要求自动创建。

##### 自动 EIP（AutoCreateEip / AutoCreateEipAttribute）

```bash
--AutoCreateEip true \
--AutoCreateEipAttribute '{"InternetMaxBandwidthOut":1,"InternetChargeType":"TRAFFIC_POSTPAID_BY_HOUR"}'
```

> 容器需要公网访问（拉公网镜像、对外服务）时配。`AutoCreateEip=true` 与 `ExistedEipIds` 互斥。EIP 计费独立。

##### 初始化容器（InitContainers）与 DNS（DnsConfig）

```bash
--InitContainers '[{"Name":"init","Image":"busybox:latest","Command":["sh","-c","echo init"]}]'
--DnsConfig '{"DnsServers":[{"Name":"183.60.83.19"}]}'
```

> `InitContainers` 结构与 `Containers` 相同。`Containers`/`InitContainers` 内的 `LivenessProbe`/`ReadinessProbe`/`SecurityContext`/`Capabilities`/`VolumeMounts`/`Resources` 等是 **K8s 标准字段**，传参格式见 [Kubernetes 官方文档](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.30/)；TCCLI 文档不展开这些字段，只讲 EKS 独有层（EksCiVolume/AutoCreateEipAttribute 等）。

#### 更新容器实例

```bash
tccli tke UpdateEKSContainerInstance --region ap-guangzhou \
  --EksCiId "<EKSCI_ID>" \
  --Containers '[{"Name":"nginx","Image":"nginx:1.25","Cpu":1,"Memory":2}]'
# expected: exit 0
```

> `UpdateEKSContainerInstance` 的 `EksCiVolume`/`InitContainers` 同 `Create`，可覆盖更新；`Containers` 是覆盖式整体替换（非增量），调用前先 `DescribeEKSContainerInstances` 取当前定义再改。

### 管理容器实例

```bash
# 查询实例（出参 EksCis[]；CPU：GpuType=""、GpuCount=0；GPU：GpuType/GpuCount 与创建一致）
tccli tke DescribeEKSContainerInstances \
  --region ap-guangzhou \
  --Filters '[{"Name":"status","Values":["Running"]}]' \
  --filter "EksCis[].{id:EksCiId,name:EksCiName,status:Status,cpu:Cpu,mem:Memory,cpuType:CpuType,gpuType:GpuType,gpuCount:GpuCount}"
# expected: Status=Running 的实例列表；cpu/mem 与创建一致
```

`Filters[].Name` 支持 `eks-ci-name`、`status`、`private-ip`、`eip-address`、`vpc-id`；其中 `status` 的合法值为 `Pending`、`Running`、`Succeeded`、`Failed`。

```bash
# 查询实例事件（Create 成功后 Status 非 Running 时必查）
tccli tke DescribeEKSContainerInstanceEvent \
  --region ap-guangzhou \
  --EksCiId "<EKSCI_ID>"
# expected: 事件列表；ImagePullBackOff / CreatePodSandboxFailed 等 Reason + Message 用于排查

# 重启实例
tccli tke RestartEKSContainerInstances --region ap-guangzhou \
  --EksCiIds '["<EKSCI_ID>"]'
# expected: exit 0

# 查询日志
tccli tke DescribeEksContainerInstanceLog \
  --region ap-guangzhou \
  --EksCiId "<EKSCI_ID>" \
  --LimitBytes 1048576
# expected: 返回不超过 1048576 字节的容器日志文本（实例未 Running 时可能为空）
```

`LimitBytes` 限制本次返回的日志总大小，单位为字节；需要缩小响应或控制终端输出量时显式设置。还可用 `SinceSeconds` 只查询最近一段时间的日志。

```bash
# 更新容器实例（EksCiId 定位，Containers[] 覆盖式更新镜像/资源，RestartPolicy 重启策略）
tccli tke UpdateEKSContainerInstance --region ap-guangzhou \
  --EksCiId "<EKSCI_ID>" \
  --RestartPolicy "Always" \
  --Containers '[{"Name":"nginx","Image":"nginx:1.25","Cpu":0.5,"Memory":1.0}]'
# expected: CAM 拦截 AuthFailure.UnauthorizedOperation；授权后返回 EksCiId
```

> ⚠️ `UpdateEKSContainerInstance` 需 CAM 授权；授权后返回 `EksCiId`。`Containers[]` 是覆盖式整体替换（非增量），调用前先 `DescribeEKSContainerInstances` 取当前容器定义再改，避免遗漏。`RestartPolicy` 取值 `Always`/`OnFailure`/`Never`。与 `RestartEKSContainerInstances`（重启，不改定义）区别。GPU 实例更新时若需保留卡配额，在 `Containers[]` 中继续传 `GpuLimit`。

### EKS 日志采集配置

> EKS 容器实例的日志采集配置（投递到 CLS）。

```bash
# 创建 EKS 日志配置 (LogConfig 采集规则 + LogsetId CLS 日志集)
tccli tke CreateEksLogConfig --ClusterId "<CLUSTER_ID>" --region <REGION> \
  --LogConfig "<LOG_CONFIG_JSON>" --LogsetId "<LOGSET_ID>"
# expected: exit 0
```

> `CreateEksLogConfig` 的 `LogConfig` 是采集规则 JSON，`LogsetId` 是 CLS 日志集 ID（见 [CLS 服务](https://cloud.tencent.com/document/product/614)）。EKS 日志区别于普通集群日志（见 [日志采集](../observability/logging.md)）。

## 更新 EKS 集群与事件持久化

> 修改 EKS 集群属性（名称/描述/子网/CLB/DNS）与开启事件持久化（事件投递到 CLS）。参数见各 Action 的 `help --detail`。
>
> ⚠️ `UpdateEKSCluster` 返回 `UnauthorizedOperation.CamNoAuth`、`EnableEksEventPersistence` 返回 `FailedOperation.CamNoAuth`——均需 CAM 授权；授权后 `UpdateEKSCluster` exit 0、`EnableEksEventPersistence` exit 0。

```bash
# 更新 EKS 集群属性（ClusterId 定位，ClusterName/ClusterDesc/SubnetIds 等覆盖式更新）
tccli tke UpdateEKSCluster --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" \
  --ClusterName "<NEW_NAME>" --ClusterDesc "<NEW_DESC>"
# expected: CAM 拦截 UnauthorizedOperation.CamNoAuth；授权后 exit 0
```

```bash
# 开启事件持久化（ClusterId + CLS LogsetId/TopicId/TopicRegion）
tccli tke EnableEksEventPersistence --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" \
  --LogsetId "<CLS_LOGSET_ID>" --TopicId "<CLS_TOPIC_ID>" --TopicRegion "<REGION>"
# expected: CAM 拦截 FailedOperation.CamNoAuth；授权后 exit 0，集群事件投递到 CLS
```

| 占位符 | 含义 | 如何获取 |
|:-------|:-----|:---------|
| `<CLS_LOGSET_ID>` | CLS 日志集 ID | `tccli cls DescribeLogsets` |
| `<CLS_TOPIC_ID>` | CLS 日志主题 ID | `tccli cls DescribeTopics` |
| `<TOPIC_REGION>` | CLS 主题所在地域 | 与集群地域一致或 CLS 实例地域 |

> `UpdateEKSCluster` 覆盖式更新集群属性（`SubnetIds[]`/`PublicLB`/`InternalLB`/`ServiceSubnetId`/`DnsServers[]` 等均可改）。`EnableEksEventPersistence` 需先在 CLS 创建日志集与主题，再把三个 ID 传入，集群事件（Pod 创建/失败等）即投递到 CLS 供检索。

## 验证存量 EKS 集群健康状态

```bash
# 核对存量 EKS 集群状态（须用 DescribeEKSClusters，非 DescribeClusters）
tccli tke DescribeEKSClusters --region ap-guangzhou --ClusterIds '["<CLUSTER_ID>"]' \
  --filter "Clusters[0].{id:ClusterId,state:Status,name:ClusterName}"
# expected: state=Running，id/name 与目标存量集群一致

# 核对存量集群凭证仍可获取
tccli tke DescribeEKSClusterCredential --region ap-guangzhou --ClusterId "<CLUSTER_ID>" \
  --filter "Kubeconfig"
# expected: 非空 kubeconfig 文本（含 apiVersion/clusters）
```

`Status=Running` 且 kubeconfig 非空表示存量 EKS 集群控制面可继续运维，不表示可以创建新的 EKS 集群。

## API 参考

本页覆盖 EKS / 容器实例相关 **15** 个 Action:

| 分类 | API | 说明 |
|------|-----|------|
| 集群（存量） | `CreateEKSCluster`（新建入口已关）/ `DeleteEKSCluster` / `UpdateEKSCluster` / `DescribeEKSClusters` | 存量 EKS 集群 CRUD；新建免 CVM 编排勿用 |
| 凭证 | `DescribeEKSClusterCredential` | 获取 kubeconfig |
| 容器实例 | `CreateEKSContainerInstances`（CPU：`Cpu`/`Memory`[/`CpuType`]；GPU：另加 `GpuType`/`GpuCount`，容器 `GpuLimit`）/ `DeleteEKSContainerInstances` / `DescribeEKSContainerInstances` / `RestartEKSContainerInstances` / `UpdateEKSContainerInstance` | 对应控制台 CPU 实例 / GPU 实例 |
| 地域 | `DescribeEKSContainerInstanceRegions` | 支持的地域 |
| 事件/日志 | `DescribeEKSContainerInstanceEvent` / `DescribeEksContainerInstanceLog` | 事件和日志 |
| 日志采集 | `CreateEksLogConfig` | EKS 容器实例日志投递 CLS |
| 持久化 | `EnableEksEventPersistence` | 开启事件持久化 |

## 清理与退役（二选一）

先按目标对象选择一个分支。容器实例没有 `ClusterId`；删除容器实例不会退役 EKS 集群，退役 EKS 集群时也不得把容器实例 ID 混入操作。

### 分支 A：清理容器实例

仅在确认 `<EKSCI_ID>` 对应的容器实例不再承载任务、日志和所需数据已保全后执行：

```bash
tccli tke DeleteEKSContainerInstances --region ap-guangzhou \
  --EksCiIds '["<EKSCI_ID>"]'
# expected: exit 0
```

执行本分支后停止；不要继续执行分支 B。

### 分支 B：退役存量 EKS 集群

仅用于完成迁移后的存量 EKS 集群。执行前确认工作负载和流量已迁至标准集群 + 虚拟节点，配置、密钥、日志及持久数据已备份或迁移，目标集群已通过业务验证，并保留回退窗口。再次核对 `<CLUSTER_ID>` 后进行二次确认：

```bash
read -r -p "输入 DELETE 确认退役存量 EKS 集群 <CLUSTER_ID>: " CONFIRM
[ "$CONFIRM" = "DELETE" ] || { printf '%s\n' "已取消删除"; exit 1; }

tccli tke DeleteEKSCluster --region ap-guangzhou --ClusterId "<CLUSTER_ID>"
# expected: exit 0

tccli tke DescribeEKSClusters --region ap-guangzhou --ClusterIds '["<CLUSTER_ID>"]'
# expected: Clusters 为空或目标 ClusterId 不存在
```

任一迁移或业务保全条件未满足时，不执行分支 B。

---

## 下一步

- [边缘集群](edge-cluster.md) — 边缘计算场景
- [虚拟节点 (超级节点)](../nodes/virtual-nodes.md) — 标准集群内免 CVM 容量（新建推荐：先 `CreateCluster` 再虚拟节点）
- [专用工作负载概览](index.md) — EKS 集群 / 容器实例 / 边缘选型
- [标准集群概览](../clusters/index.md) — 对比标准集群

## Action 字段契约

| 字段 | 所属 Action | 必填 | 说明 |
|:---|:---|:---:|:---|
| `EksCiIds` | `DeleteEKSContainerInstances` | 是 | 待删除容器实例 ID 列表 |
