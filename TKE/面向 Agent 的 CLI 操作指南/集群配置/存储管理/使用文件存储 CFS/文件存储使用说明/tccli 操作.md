# 文件存储使用说明

> 对照官方：[文件存储使用说明](https://cloud.tencent.com/document/product/457/44234) · page_id `44234` · tccli ≥3.1.107 · API 2019-07-19

## 概述

腾讯云容器服务 TKE 支持通过创建 PV/PVC 并为工作负载挂载数据卷的方式使用腾讯云文件存储 CFS。CFS 提供标准 NFS 协议、可扩展的共享文件系统，支持 ReadWriteMany 访问模式，适合多 Pod 共享数据的场景。

两种使用方式对应不同的 CLI 操作路径：

| 方式 | 说明 | 控制面（tccli） | 数据面（kubectl） | 适用场景 |
|------|------|:---:|:---:|---------|
| 动态创建 | 通过 StorageClass 模板自动创建 CFS 实例并绑定 PV/PVC | 安装 CFS-CSI 组件 | 创建 StorageClass → 创建 PVC | 新建 CFS、按需自动供给 |
| 使用已有 CFS | 为已存在的 CFS 实例创建静态 PV，再通过 PVC 绑定 | 查询 CFS 实例 | 创建 PV → 创建 PVC | 存量 CFS 迁移、手动管控 |

**前提要求**：集群需安装 CFS-CSI 扩展组件，且集群 VPC 与 CFS 实例在**同一地域、同一 VPC**。

## 前置条件

- [环境准备](../../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    cfs:DescribeCfsServiceStatus, cfs:DescribeCfsFileSystems
#    cfs:DescribeMountTargets, cfs:DescribeCfsPGroups
#    cfs:DescribeAvailableZoneInfo
#    tke:InstallAddon, tke:DescribeAddon, tke:DeleteAddon
# 验证：执行 DescribeClusters 确认 TKE 权限
tccli tke DescribeClusters --region <Region>
# expected: exit 0，返回集群列表（可为空）
```

```json
{
    "Clusters": [
        {
            "ClusterId": "cls-example",
            "ClusterName": "example-cluster",
            "ClusterStatus": "Running",
            "ClusterVersion": "1.32.2"
        }
    ],
    "TotalCount": 1
}
```

### 资源检查

```bash
# 4. 确认目标集群存在且状态正常
tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'
# expected: TotalCount >= 1，ClusterStatus: "Running"

# 5. 确认 CFS 服务已开通
tccli cfs DescribeCfsServiceStatus --region <Region>
# expected: "CfsServiceStatus": "created"
```

```json
{
    "CfsServiceStatus": "created",
    "RequestId": "5a3e0786-a6d1-4c68-ab71-1df518f70bd1"
}
```

```bash
# 6. 确认集群 VPC（CFS 需与集群在同一 VPC）
tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'
# expected: 记录 ClusterNetworkSettings.VpcId，后续选择 CFS 实例时需与其 VPC 一致
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli/kubectl 命令 | 幂等 |
|-----------|-------------------|:--:|
| 检查 CFS 服务开通状态 | `tccli cfs DescribeCfsServiceStatus` | 是 |
| 查看已有 CFS 文件系统 | `tccli cfs DescribeCfsFileSystems` | 是 |
| 查看 CFS 挂载目标 | `tccli cfs DescribeMountTargets` | 是 |
| 查看 CFS 权限组 | `tccli cfs DescribeCfsPGroups` | 是 |
| 查看 CFS 可用区及存储类型 | `tccli cfs DescribeAvailableZoneInfo` | 是 |
| 查看文件系统客户端列表 | `tccli cfs DescribeCfsFileSystemClients` | 是 |
| 安装 CFS-CSI 组件 | `tccli tke InstallAddon --AddonName CFS` | 是 |
| 查询组件安装状态 | `tccli tke DescribeAddon --AddonName CFS` | 是 |
| 动态创建 StorageClass | `kubectl apply -f sc.yaml` | 是 |
| 动态创建 PVC | `kubectl apply -f pvc.yaml` | 否 |
| 创建 PV（已有 CFS） | `kubectl apply -f pv.yaml` | 否 |
| 创建 PVC（绑定已有 PV） | `kubectl apply -f pvc.yaml` | 否 |

## 操作步骤

### 步骤 1：确认 CFS 服务状态

在使用 CFS 之前，需确认账号已在目标地域开通 CFS 服务。若未开通，需先执行开通操作。

```bash
tccli cfs DescribeCfsServiceStatus --region <Region>
# expected: "CfsServiceStatus": "created"
```

输出示例：

```json
{
    "CfsServiceStatus": "created",
    "RequestId": "5a3e0786-a6d1-4c68-ab71-1df518f70bd1"
}
```

`CfsServiceStatus` 取值说明：

| 状态值 | 含义 | 后续操作 |
|--------|------|---------|
| `created` | 已开通 CFS 服务 | 继续后续步骤 |
| `not_created` | 未开通 | 执行 `tccli cfs SignUpCfsService --region <Region>` 开通 |

### 步骤 2：安装 CFS-CSI 扩展组件

TKE 通过 CFS-CSI 扩展组件在集群中提供文件存储的 PV/PVC 支持。安装前先检查组件状态。

#### 2.1 检查组件是否已安装

```bash
tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName CFS
# expected: 若已安装，返回组件信息（含 AddonVersion）；若返回 "ResourceNotFound" 则表示未安装
```

若组件未安装时的预期输出（`ResourceNotFound`）：

```json
{
    "Error": {
        "Code": "ResourceNotFound",
        "Message": "get addon failed"
    },
    "RequestId": "8dd24043-78e9-49a9-8686-54c2af674f0d"
}
```

> **版本发现限制**：`tccli tke DescribeAddonValues` 仅在组件**已安装**后返回 chart 信息（组件未安装时返回 `ResourceUnavailable: not found chart`）。以下为两种场景的版本获取方式：
>
> **场景 A：组件已安装**（DescribeAddon 返回组件信息）：
> ```bash
> tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName CFS
> # 从输出中的 AddonVersion 字段获取当前安装的版本号
> ```
>
> **场景 B：组件未安装**（DescribeAddon 返回 ResourceNotFound）：
> 1. 登录 [TKE 控制台](https://console.cloud.tencent.com/tke2) → 目标集群 → 组件管理 → 新建
> 2. 找到 CFS-CSI 组件，记录**可用版本列表**中最新版本的版本号
> 3. 返回 CLI，使用上述版本号执行 InstallAddon
>
> 也可在已安装 CFS-CSI 的同集群版本的其他集群上执行 `DescribeAddon` 获取版本号作为参考。

#### 2.2 安装组件

| 字段名 | 类型 | 必填 | 取值与约束 | 错误后果 |
|--------|------|:--:|---------|---------|
| `ClusterId` | String | 是 | 目标集群 ID，格式 `cls-xxxxxxxx` | 填入不存在的集群 → `ResourceNotFound` |
| `AddonName` | String | 是 | `CFS`（固定值） | 填入其他值 → `UnknownParameter` |
| `AddonVersion` | String | 是 | CFS-CSI 组件版本号，从 TKE 控制台组件管理页面获取（见 2.1 版本发现限制） | 版本不存在 → `UnknownParameter: addon version not found` |
| `RawValues` | String | 否 | 组件自定义参数（JSON 字符串），默认留空使用默认配置 | — |
| `DryRun` | Boolean | 否 | 是否仅验证不实际安装，默认 `false` | — |

```bash
tccli tke InstallAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName CFS \
    --AddonVersion <AddonVersion>
# expected: exit 0，返回 RequestId
```

输出示例：

```json
{
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

> **影响范围**：安装 CFS-CSI 组件后，将在集群 `kube-system` 命名空间下创建 `cfs-csi-driver` 相关的 DaemonSet/Deployment 及 RBAC 资源（ServiceAccount、ClusterRole、ClusterRoleBinding）。集群中**所有节点**均会运行 `cfs-csi-nodeplugin` DaemonSet Pod。
>
> **不可逆**：组件安装操作不可逆。可通过 `tccli tke DeleteAddon` 卸载，但卸载会清理所有相关的 PVC/PV 挂载资源，正在使用 CFS 卷的 Pod 将无法正常工作。执行卸载前务必确认无生产负载依赖 CFS 存储。

| 占位符 | 说明 | 获取方式 |
|--------|------|---------|
| `<AddonVersion>` | CFS-CSI 组件版本号 | 从 TKE 控制台 → 集群 → 组件管理 → 新建 → 查看 CFS 可用版本获取。或在已有 CFS-CSI 的同集群版本上通过 `DescribeAddon` 获取 |

#### 2.3 轮询组件安装状态

组件安装是异步操作，执行后需等待组件就绪：

```bash
tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName CFS
# expected: Phase: "Succeeded"，Status: "Running"
```

输出示例（已安装状态）：

```json
{
    "Addon": {
        "AddonName": "CFS",
        "AddonVersion": "1.2.0",
        "Phase": "Succeeded",
        "Status": "Running"
    },
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

### 步骤 3：查询可用 CFS 文件存储实例

查询目标地域已有的 CFS 文件存储实例，了解可用资源（存储类型、容量、可用区等）。

```bash
tccli cfs DescribeCfsFileSystems --region <Region>
# expected: "FileSystems" 数组，每个实例含 FileSystemId、FsName、LifeCycleState 等字段
```

输出示例（截取关键字段）：

```json
{
    "FileSystems": [
        {
            "FileSystemId": "cfs-example",
            "FsName": "my-cfs-instance",
            "LifeCycleState": "available",
            "StorageType": "SD",
            "Protocol": "NFS",
            "SizeByte": 1048576,
            "Zone": "ap-guangzhou-6",
            "PGroup": {
                "PGroupId": "pgroupbasic",
                "Name": "默认权限组"
            },
            "CreationTime": "2026-01-01 00:00:00",
            "Version": "v3.1"
        }
    ],
    "TotalCount": 1
}
```

关键字段说明：

| 字段 | 类型 | 说明 |
|------|------|------|
| `FileSystemId` | String | 文件存储实例 ID，格式 `cfs-xxxxxxxx`。后续创建 PV 时需要 |
| `FsName` | String | 文件存储名称 |
| `LifeCycleState` | String | 生命周期状态：`creating` / `available` / `deleting`。只有 `available` 状态可供挂载 |
| `StorageType` | String | 存储类型：`SD`（标准型）/ `HP`（性能型）/ `TB`（吞吐型） |
| `Protocol` | String | 协议类型，固定 `NFS` |
| `Zone` | String | 可用区（如 `ap-guangzhou-6`） |
| `PGroup` | Object | 关联的权限组。`PGroupId` 为 `pgroupbasic` 表示使用默认权限组 |

### 步骤 4：查询 CFS 挂载目标

查询指定文件存储实例的挂载目标信息，获取挂载点 IP 地址（用于后续创建 PV）。

```bash
tccli cfs DescribeMountTargets --region <Region> --FileSystemId <FileSystemId>
# expected: 返回挂载目标列表，含 IpAddress 字段
```

输出示例：

```json
{
    "MountTargets": [
        {
            "FileSystemId": "cfs-example",
            "MountTargetId": "cfs-example",
            "IpAddress": "10.0.32.35",
            "FSID": "7anyvaro",
            "LifeCycleState": "available",
            "NetworkInterface": "VPC",
            "VpcId": "vpc-example",
            "VpcName": "example-vpc",
            "SubnetId": "subnet-example",
            "SubnetName": "example-subnet",
            "CcnID": "-",
            "CidrBlock": "-"
        }
    ],
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

### 步骤 5：查询 CFS 可用区及存储类型

查询目标地域支持的可用区和存储类型，为创建 CFS 实例或选择现有 CFS 实例时提供参考。

```bash
tccli cfs DescribeAvailableZoneInfo --region <Region>
# expected: 返回 RegionZones 列表，每个可用区含 Types 子列表（支持的存储类型及售卖状态）
```

### 步骤 6：查询 CFS 权限组

查询 CFS 文件存储的权限组信息，了解访问控制规则。

```bash
tccli cfs DescribeCfsPGroups --region <Region>
# expected: 返回 PGroupList，含每个权限组的 PGroupId、Name、BindCfsNum
```

输出示例：

```json
{
    "PGroupList": [
        {
            "PGroupId": "pgroupbasic",
            "Name": "默认权限组",
            "DescInfo": "",
            "CDate": "2024-01-01 00:00:00",
            "BindCfsNum": 30
        },
        {
            "PGroupId": "pgroup-example",
            "Name": "my-pgroup",
            "DescInfo": "自定义权限组",
            "CDate": "2026-05-18 17:25:51",
            "BindCfsNum": 6
        }
    ],
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

### 步骤 7：后续操作路径

完成以上准备步骤后，根据需求选择对应路径：

- **动态创建文件存储**：见 [StorageClass 管理文件存储模板](../StorageClass%20管理文件存储模板/tccli%20操作.md)
- **使用已有文件存储**：见 [PV 和 PVC 管理文件存储](../PV%20和%20PVC%20管理文件存储/tccli%20操作.md)

## 验证

### 控制面（tccli）

| 维度 | 命令 | 预期 |
|------|------|------|
| CFS 服务 | `tccli cfs DescribeCfsServiceStatus --region <Region>` | `CfsServiceStatus: "created"` |
| CFS-CSI 组件 | `tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName CFS` | `Status: "Running"` |
| CFS 实例 | `tccli cfs DescribeCfsFileSystems --region <Region>` | 可列出文件系统列表（可为空） |
| 挂载目标 | `tccli cfs DescribeMountTargets --region <Region> --FileSystemId <FileSystemId>` | `LifeCycleState: "available"` |
| 集群 VPC | `tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'` | 确认 VpcId 与 CFS 实例 VpcId 一致 |

### 数据面（kubectl）

```bash
# 验证 CFS-CSI 驱动 Pod 运行状态
kubectl get pods -n kube-system | grep cfs
# expected: cfs-csi 相关 Pod 均为 Running

# 验证 StorageClass
kubectl get sc
# expected: 列表中包含 CFS 相关的 StorageClass

# 验证 PV/PVC 绑定状态
kubectl get pv,pvc -A
# expected: STATUS 为 Available 或 Bound
```

## 清理

> **警告**：删除 PVC 时，若绑定的 PV 的 `persistentVolumeReclaimPolicy` 为 `Delete`，则底层 CFS 实例和数据将被级联删除，该操作不可逆。执行前务必确认数据已备份。

清理顺序：PVC → PV →（如需）CFS 实例 → CFS-CSI 组件

```bash
# 1. 删除使用 CFS 的 PVC（如有）
kubectl delete pvc <PvcName> -n <Namespace>

# 2. 删除静态 PV（如有）
kubectl delete pv <PvName>

# 3. 删除 CFS 实例（仅当不再需要，且 PV ReclaimPolicy 非 Delete 时才需手动执行）
tccli cfs DeleteCfsFileSystem --region <Region> --FileSystemId <FileSystemId>

# 4. 验证已删除
tccli cfs DescribeCfsFileSystems --region <Region> \
  --FileSystemIds '["<FileSystemId>"]'
# expected: ResourceNotFound 或列表为空

# 5. 卸载 CFS-CSI 组件（可选，仅当不再需要文件存储功能时）
# 卸载前检查：确认无 PVC 使用 CFS StorageClass
kubectl get pvc -A | grep cfs

tccli tke DeleteAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName CFS
# expected: exit 0
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `ResourceNotFound: get addon failed` | 执行 `tccli tke DescribeAddon --AddonName CFS` | CFS-CSI 组件未安装 | 执行 `tccli tke InstallAddon --AddonName CFS --AddonVersion <Version>` 安装组件 |
| `CfsServiceStatus: not_created` | 执行 `tccli cfs DescribeCfsServiceStatus` | CFS 服务未开通 | 执行 `tccli cfs SignUpCfsService --region <Region>` 开通服务 |
| `ResourceNotFound` 查询 CFS 实例时 | 执行 `tccli cfs DescribeCfsFileSystems --FileSystemIds '["<id>"]'` | 指定的 CFS 实例不存在或已删除 | 使用 `tccli cfs DescribeCfsFileSystems --region <Region>` 列出可用实例确认 ID |
| `UnauthorizedOperation` 执行 cfs 命令 | 检查 CAM 策略 | `cfs:Describe*` 权限缺失 | 联系 CAM 管理员授予 `cfs:DescribeCfsFileSystems` 等读权限 |
| PVC 长时间 Pending | `kubectl describe pvc <PvcName> -n <Namespace>` 查看 Events | StorageClass 配置错误或 CFS-CSI 组件异常 | 检查 provisioner 是否为 `com.tencent.cloud.csi.cfs`；检查 CSI driver Pod：`kubectl get pods -n kube-system \| grep cfs` |
| kubectl 命令返回 `Unable to connect to the server` | `kubectl cluster-info` 不通 | 集群无公网端点，或当前网络环境无法访问集群 VPC | 通过 IOA/VPN 或同 VPC CVM/跳板机执行 kubectl 操作 |

### 安装成功但使用时异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Pod 无法挂载 CFS 卷 | `kubectl describe pod <PodName> -n <Namespace>` 查看 Events | NFS 挂载失败，可能网络不通或权限组配置不当 | 确认 CFS 挂载点 IP 可达（在节点上 `ping <CFS_IP>`）；检查 CFS 权限组规则：`tccli cfs DescribeCfsRules --region <Region> --PGroupId <PGroupId> --FileSystemId <FileSystemId>` |
| CFS 写入权限拒绝 | 在 Pod 中执行 `touch /mnt/cfs/test` | 权限组规则限制或 NFS 导出配置为只读 | 检查权限组规则的 `RWPermission` 是否为 `rw`：`tccli cfs DescribeCfsRules --region <Region> --PGroupId <PGroupId>` |
| CFS 实例与集群不在同一 VPC | `tccli cfs DescribeMountTargets --region <Region> --FileSystemId <FileSystemId>` 查 VpcId，与 `tccli tke DescribeClusters` 的 VpcId 对比 | 跨 VPC 的 CFS 实例无法直接挂载到集群节点 | 在集群所在 VPC 中创建新的 CFS 实例，或通过云联网 CCN 打通 VPC |

## 下一步

- [StorageClass 管理文件存储模板](../StorageClass%20管理文件存储模板/tccli%20操作.md) — 通过 tccli/kubectl 创建和管理 CFS StorageClass
- [PV 和 PVC 管理文件存储](../PV%20和%20PVC%20管理文件存储/tccli%20操作.md) — 通过 kubectl 创建 PV/PVC 使用已有 CFS
- [PV 和 PVC 的绑定规则](../../PV%20和%20PVC%20的绑定规则/tccli%20操作.md) — 了解 PV 与 PVC 的绑定机制和排障
- [腾讯云文件存储 CFS 产品文档](https://cloud.tencent.com/document/product/582) — CFS 产品完整文档

## 控制台替代

[CFS 控制台](https://console.cloud.tencent.com/cfs) 管理文件系统；[TKE 控制台](https://console.cloud.tencent.com/tke2) → 集群 → 组件管理 → 安装 CFS-CSI 组件。
