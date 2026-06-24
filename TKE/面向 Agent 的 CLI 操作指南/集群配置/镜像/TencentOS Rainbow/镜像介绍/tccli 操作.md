# 镜像介绍

> 对照官方：[镜像介绍](https://cloud.tencent.com/document/product/457/131166) · page_id `131166`

## 概述

**TencentOS Rainbow** 是腾讯云面向云原生场景的轻量级容器操作系统，作为 TKE **官方公共镜像**提供。相对传统 Linux 发行版，聚焦节点镜像构建、升级与配置，集成 Kernel、发行版与容器运行时，为 Kubernetes 集群提供节点底座。

Rainbow 通过 `DescribeOSImages` API 查询，`OsName` 为 `rainbow1.30.0x86_64`。创建集群时在 `ClusterBasicSettings.ClusterOs` 中指定该值即可选用。

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
#    tke:DescribeOSImages, tke:DescribeClusters, tke:CreateCluster
# 验证：执行 DescribeOSImages 确认权限
tccli tke DescribeOSImages --region <REGION>
# expected: exit 0，返回 OSImageSeriesSet（可为空）

# 验证 DescribeClusters 权限（如需查询已有 Rainbow 集群）
tccli tke DescribeClusters --region <REGION>
# expected: exit 0，返回集群列表（可为空）
```

```json
{
  "OSImageSeriesSet": "<OSImageSeriesSet>",
  "SeriesName": "<SeriesName>",
  "Alias": "<Alias>",
  "OsName": "<OsName>",
  "OsCustomizeType": "<OsCustomizeType>",
  "Status": "<Status>"
}
```

### 资源检查

```bash
# 4. 确认 Rainbow 镜像在目标地域在线
tccli tke DescribeOSImages --region <REGION> \
    | jq '[.OSImageSeriesSet[] | select(.OsName == "rainbow1.30.0x86_64" and .Status == "online")] | length'
# expected: 1（表示 Rainbow 镜像在线可用）

# 5. 确认 K8s 1.30 版本可用（Rainbow 仅支持 1.30）
tccli tke DescribeVersions --region <REGION> \
    | jq '.VersionInstanceSet[] | select(.Version | startswith("1.30"))'
# expected: 返回 1.30.x 版本列表
```

```json
{
  "OSImageSeriesSet": "<OSImageSeriesSet>",
  "SeriesName": "<SeriesName>",
  "Alias": "<Alias>",
  "OsName": "<OsName>",
  "OsCustomizeType": "<OsCustomizeType>",
  "Status": "<Status>"
}
```

## 关键字段说明

Rainbow 镜像在 `DescribeOSImages` 返回结果中的核心字段：

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `OsName` | String | 是（创建时） | `rainbow1.30.0x86_64`。创建集群时传入此值到 `ClusterOs` | 填错 → 创建使用非 Rainbow 镜像的集群 |
| `Status` | String | 仅输出 | `online`（可用）/ `offline`（已下线）。在线才可选 | — |
| `ImageId` | String | 仅输出 | Rainbow 对应的 CVM 侧 ImageId，格式 `img-xxxxxxxx` | — |
| `Arch` | String | 仅输出 | `x86_64`（当前仅支持 x86_64 架构） | — |
| `Alias` | String | 仅输出 | `RainbowOS Server(TK4)`，控制台展示名为「RainbowOS Server」 | — |

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看镜像列表 | DescribeImages | 是 |
## 操作步骤

### 步骤 1：查询 Rainbow 镜像详情

```bash
tccli tke DescribeOSImages --region <REGION> \
    | jq '.OSImageSeriesSet[] | select(.OsName == "rainbow1.30.0x86_64")'
# expected: exit 0，返回 Rainbow 镜像完整信息
```

**预期输出**：

```json
{
    "OsName": "rainbow1.30.0x86_64",
    "SeriesName": "rainbow1.30.0x86_64",
    "ImageId": "img-ah6pq81v",
    "Status": "online",
    "Alias": "RainbowOS Server(TK4)",
    "Arch": "x86_64"
}
```

### 步骤 2：了解 Rainbow 核心架构

TencentOS Rainbow 由四大模块构成：

**1. 基础 OS（不可变发行版）**
- 根文件系统只读：系统组件默认只读，无法通过 `dnf/yum` 随意安装传统软件包，防止碎片化与配置漂移
- 配置及用户数据可写：`/etc`、`/var` 等通过 overlay 挂载到数据分区
- 极简软件包：仅保留 K8s 与 containerd 必需组件及升级/运维工具

**2. 升级管理（OS Update Operator）**
- 内核态与用户态分离管理，可单独或共同更新
- A/B 双分区：双 Boot + 双 Root，支持快速更新与回滚
- 通过 **NodeOS** 等 CR 触发原子化升级、分批灰度、重试与回退（集群内 `kubectl` 操作，非 `tccli tke` 控制面 API）

**3. 配置管理（OS Config Operator）**
- GitOps / **ConfOS** 声明式下发 sysctl、containerd、kubelet 等配置
- **fanotify** 监控关键配置，异常篡改通过 Kubernetes Event 上报

**4. 运维管理（Admin Container）**
- 按需特权容器，共享宿主机 PID/网络命名空间
- 内置 sshd 与 tcpdump、perf、strace、bpftrace 等，不污染基础 OS

### 步骤 3：与传统 TencentOS Server 对比

| 对比维度 | TencentOS Server（如 3.1 / 4） | TencentOS Rainbow |
|----------|-------------------------------|-------------------|
| 系统架构与定位 | 通用 Linux，传统业务与虚拟化 | 不可变基础设施，专用于容器负载 |
| 软件包管理 | 完整 RPM 仓库，`yum`/`dnf` 自由安装 | 剔除传统包管理器与非必须服务（默认无 `sshd`） |
| 内核参数与调优 | 通用场景，参数较多 | 云原生定制内核，高并发网络与容器隔离调优 |
| 配置管理 | SSH 手工改 `/etc/sysctl.conf` 等 | K8s 声明式 API（**ConfOS** CRD），禁止手工热改 |
| 系统升级 | 逐包 `yum update`，回滚难 | A/B 分区原子整包替换，异常可一键回滚 |

### 步骤 4：查询使用 Rainbow 的集群

```bash
# 筛选当前地域使用 Rainbow OS 的集群
tccli tke DescribeClusters --region <REGION> \
    | jq '.Clusters[] | select(.ClusterOs == "rainbow1.30.0x86_64") | {ClusterId, ClusterOs, ClusterVersion, ClusterStatus}'
# expected: exit 0，返回使用 Rainbow OS 的集群列表（可为空）
```

**预期输出**（有 Rainbow 集群时）：

```json
{
    "ClusterId": "cls-example",
    "ClusterOs": "rainbow1.30.0x86_64",
    "ClusterVersion": "1.30.0",
    "ClusterStatus": "Running"
}
```

### 步骤 5：创建使用 Rainbow 的集群

创建集群时在 JSON 中指定 `ClusterOs` 为 `rainbow1.30.0x86_64`。**Rainbow 仅支持 K8s 1.30 版本**，`ClusterVersion` 须为 `1.30.x`。

`cluster-rainbow.json`：

```json
{
    "ClusterType": "MANAGED_CLUSTER",
    "ClusterBasicSettings": {
        "ClusterName": "<CLUSTER_NAME>",
        "ClusterVersion": "1.30.0",
        "ClusterOs": "rainbow1.30.0x86_64",
        "VpcId": "<VPC_ID>",
        "SubnetId": "<SUBNET_ID>"
    },
    "ClusterCIDRSettings": {
        "ClusterCIDR": "10.1.0.0/16",
        "ServiceCIDR": "10.2.0.0/16",
        "MaxNodePodNum": 64,
        "MaxClusterServiceNum": 4096
    },
    "ClusterAdvancedSettings": {
        "ContainerRuntime": "containerd",
        "DeletionProtection": true
    }
}
```

```bash
tccli tke CreateCluster --region <REGION> --cli-input-json file://cluster-rainbow.json
# expected: exit 0，返回 ClusterId
```

创建为异步操作，轮询直到状态为 `Running`。详细创建步骤见 [创建集群](../../../集群管理/创建集群/tccli%20操作.md)。

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `CLUSTER_NAME` | 集群名称 | 长度 1-60 字符，以字母开头 | 自定义 |
| `VPC_ID` | VPC 实例 ID | 格式 `vpc-xxxxxxxx` | `tccli vpc DescribeVpcs --region <REGION>` |
| `SUBNET_ID` | 子网 ID | 格式 `subnet-xxxxxxxx` | `tccli vpc DescribeSubnets --region <REGION>` |
| `REGION` | 地域 | 如 `ap-guangzhou` | `tccli configure list` 查看当前地域 |

### 产品优势

| 维度 | 说明 |
|------|------|
| **极致轻量** | 相较 TencentOS Server 4（约 4.5GB），Rainbow OS 组件约 **393MB**。内存与缓存占用相对 CentOS/Ubuntu/TS2/TS3 等显著降低 |
| **节点启动快** | 相对通用镜像在 S5.MEDIUM2 上总耗时提升约 **66%–77%** |
| **安全加固** | 只读 rootfs、Config Monitor 防篡改 |
| **Kubernetes 原生管理** | 通过 `kubectl`/K8s API 做 OS 升级、回滚与内核参数调优，无需逐台 SSH |

## 验证

```bash
# 1. 确认 Rainbow 镜像在线
tccli tke DescribeOSImages --region <REGION> \
    | jq '[.OSImageSeriesSet[] | select(.OsName == "rainbow1.30.0x86_64")] | {OsName: .[0].OsName, Status: .[0].Status, ImageId: .[0].ImageId}'
# expected: Status 为 "online"，ImageId 非空

# 2. 确认 K8s 1.30 版本可用
tccli tke DescribeVersions --region <REGION> \
    | jq '.VersionInstanceSet[] | select(.Version | startswith("1.30"))'
# expected: 返回 1.30.x 版本详情

# 3. 确认 Rainbow 集群（如有）状态正常
tccli tke DescribeClusters --region <REGION> \
    --ClusterIds '["<CLUSTER_ID>"]' \
    | jq '.Clusters[0] | {ClusterOs, ClusterVersion, ClusterStatus}'
# expected: ClusterOs 为 "rainbow1.30.0x86_64"，ClusterVersion 为 1.30.x，ClusterStatus 为 Running

# 4. 节点连通性（Rainbow 集群创建完成后）
tccli tke DescribeClusterKubeconfig --region <REGION> \
    --ClusterId <CLUSTER_ID> \
    | jq -r '.Kubeconfig' > ~/.kube/config
kubectl get nodes
# expected: 返回节点列表，确认 kubectl 可达
```

```json
{
  "OSImageSeriesSet": "<OSImageSeriesSet>",
  "SeriesName": "<SeriesName>",
  "Alias": "<Alias>",
  "OsName": "<OsName>",
  "OsCustomizeType": "<OsCustomizeType>",
  "Status": "<Status>"
}
```

## 清理

本页为说明与只读查询。若创建了 Rainbow 测试集群，需删除以避免持续产生集群管理费和 CVM 费用。

> **警告**：`DeleteCluster` 配合 `InstanceDeleteMode: "terminate"` 会**释放/删除**集群关联的所有 CVM 实例。生产环境执行前务必确认集群名称和 ID。

### 1. 清理前状态检查

```bash
tccli tke DescribeClusters --region <REGION> \
    --ClusterIds '["<CLUSTER_ID>"]'
# 确认是待删除的目标 Rainbow 测试集群，记录 ClusterId、NodeCount
```

```json
{
  "TotalCount": 0,
  "Clusters": [],
  "ClusterId": "<ClusterId>",
  "ClusterName": "<ClusterName>",
  "ClusterDescription": "<ClusterDescription>",
  "ClusterVersion": "<ClusterVersion>",
  "ClusterOs": "<ClusterOs>",
  "ClusterType": "<ClusterType>"
}
```

### 2. 关闭删除保护（如启用）

```bash
tccli tke ModifyClusterAttribute --region <REGION> \
    --ClusterId <CLUSTER_ID> \
    --DeletionProtection false
# expected: exit 0
```

### 3. 删除集群

```bash
tccli tke DeleteCluster --region <REGION> \
    --ClusterId <CLUSTER_ID> \
    --InstanceDeleteMode terminate
# 注意：--InstanceDeleteMode terminate 会级联删除所有关联 CVM 节点
```

### 4. 验证已删除

```bash
tccli tke DescribeClusters --region <REGION> \
    --ClusterIds '["<CLUSTER_ID>"]'
# expected: ResourceNotFound 或空列表
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

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DescribeOSImages` 中未找到 `rainbow1.30.0x86_64` | `tccli tke DescribeOSImages --region <REGION> \| jq '[.OSImageSeriesSet[] \| select(.OsName \| startswith("rainbow"))]'` 查询所有 Rainbow 系列镜像 | Rainbow 镜像在当前地域未上线（此为环境限制，非命令错误） | 换地域重试（如 `ap-guangzhou`）；确认 Rainbow 镜像在该地域开放。保留 RequestId 以备 [提交工单](https://console.cloud.tencent.com/workorder) |
| `DescribeOSImages` 返回 `UnauthorizedOperation` | `tccli configure list` 检查当前凭据 | 子账号缺少 `tke:DescribeOSImages` 权限（此为环境限制，非命令错误） | 联系主账号授予 `QcloudTKEFullAccess` 或最小权限 `tke:DescribeOSImages` |
| 创建集群时返回 `InvalidParameter.ClusterVersion` | `tccli tke DescribeVersions --region <REGION> \| jq '.VersionInstanceSet[].Version'` 确认可用版本 | Rainbow 仅支持 K8s 1.30，创建时传了其他版本 | 将 `ClusterVersion` 改为 `1.30.x`，保持与 Rainbow 限制一致 |
| `CreateCluster` 返回 `InvalidParameter.ClusterOs` | 检查 `ClusterOs` 值是否为 `rainbow1.30.0x86_64` | `ClusterOs` 值拼写错误或该镜像在当前地域不可用 | 确认 OsName 精确为 `rainbow1.30.0x86_64`；先用 `DescribeOSImages` 确认 Status 为 `online` |

### 使用 Rainbow 后的运维问题

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 需要 SSH 登录 Rainbow 节点调试 | Rainbow 默认不安装 `sshd`，无法直接 SSH 登录 | Rainbow 采用不可变基础设施设计，禁止在只读 rootfs 上安装软件 | 使用 Admin Container（特权容器共享宿主机 PID/网络命名空间，内置 sshd 与调试工具），而非在宿主机安装 sshd |
| 需要修改内核参数 | Rainbow 的 `fanotify` 监控关键配置，手工修改会在 K8s Event 中产生告警 | `sysctl` 等内核参数通过 ConfOS CRD 声明式管理，禁止手工热改 | 使用 `kubectl apply -f` 通过 ConfOS CR 声明式下发配置，非 `sysctl -w` |
| OS 升级需求 | Rainbow 支持 A/B 分区原子升级，通过 NodeOS CR 触发 | `yum update` 逐包升级不适用（基础 OS 为只读 rootfs） | 在集群内通过 `kubectl` 操作 NodeOS CR 触发 A/B 分区升级与回滚，非 `tccli tke` 控制面 API |
| 节点初始化慢或失败 | 检查集群是否开启了注册节点、Dataplane v2 等网络特性 | 部分网络特性下 Rainbow 镜像选择受限 | 确认当前集群网络模式下 Rainbow 是否可用；如不可用，改用 TencentOS Server 系列 |
| 想安装传统 RPM 包 | 执行 `yum install` 或 `dnf install` 会失败（rootfs 只读） | Rainbow 剔除了传统包管理器，基础 OS 为只读 | 如确需特定软件，考虑使用 Admin Container 按需运行，而非在基础 OS 上安装 |

## 下一步

- [镜像概述](../../镜像概述/tccli%20操作.md) — TKE 支持的完整公共镜像列表
- [自定义镜像说明](../../自定义镜像说明/tccli%20操作.md) — 基于 TKE 基础镜像制作自定义镜像
- [创建集群](../../../集群管理/创建集群/tccli%20操作.md) — 完整的集群创建流程（含参数选择决策）
- [更改集群操作系统](../../../集群管理/更改集群操作系统/tccli%20操作.md) — 修改存量集群的 OS
- [GPU 故障检测与自愈](https://cloud.tencent.com/document/product/457/131167) — 官方下一篇，Rainbow 在 GPU 场景下的运维能力

## 控制台替代

[TKE 创建集群](https://console.cloud.tencent.com/tke2/cluster/create) 镜像列表中的 **RainbowOS Server(TK4)** 对应 `OsName` `rainbow1.30.0x86_64`。
