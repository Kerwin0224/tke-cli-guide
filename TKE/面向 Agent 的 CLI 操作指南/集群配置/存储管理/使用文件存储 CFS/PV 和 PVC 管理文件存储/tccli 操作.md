# PV 和 PVC 管理文件存储

> 对照官方：[PV 和 PVC 管理文件存储](https://cloud.tencent.com/document/product/457/44236) · page_id `44236` · tccli ≥3.1.107 · API 2019-07-19

## 概述

通过已有 CFS 文件系统创建静态 PV，再通过 PVC 绑定后挂载到 Pod。适用于存量 CFS 实例复用场景。

**与动态创建对比**：

| 维度 | 静态创建（本页） | 动态创建（StorageClass） |
|------|-------------|---------------------|
| CFS 来源 | 提前手动创建 CFS | PVC 创建时自动新建 |
| 文件系统 ID / IP | 手动填入 PV YAML | 自动分配 |
| 控制粒度 | 精确控制每个 PV 参数 | 统一模板 |
| 适合场景 | 存量迁移、手动管控 | 按需供给、新建 |

## 前置条件

- [环境准备](../../../../环境准备.md)
- 已安装 CFS-CSI 组件：[文件存储使用说明](../文件存储使用说明/tccli%20操作.md)
- 已有 CFS 文件系统：在 [CFS 控制台](https://console.cloud.tencent.com/cfs) 或通过 `tccli cfs CreateCfsFileSystem` 创建
- kubectl 已连接目标集群

### 环境检查

```bash
# 1. 检查 tccli 版本和凭据
tccli --version
# expected: tccli version >= 1.0.0

tccli configure list
# expected: secretId、secretKey、region 均已配置

# 2. 确认集群可访问
tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'
# expected: ClusterStatus: "Running"

# 3. 确认 CFS-CSI 已安装
tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName CFS
# expected: Phase: "Succeeded", Status: "Running"

# 4. 确认 CFS 文件系统存在且可用
tccli cfs DescribeCfsFileSystems --region <Region>
# expected: 返回 CFS 实例列表，记录 FileSystemId 和 IpAddress
```

**预期输出 — CFS-CSI 组件状态**：

```json
{
    "AddonName": "CFS",
    "AddonVersion": "v1.2.0",
    "Phase": "Succeeded",
    "Reason": "",
    "CreateTime": "2026-06-23T10:00:00+08:00",
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

**预期输出 — 集群状态**：

```json
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "cls-example",
            "ClusterName": "cluster-name",
            "ClusterVersion": "1.32.2",
            "ClusterType": "MANAGED_CLUSTER",
            "ClusterStatus": "Running",
            "ClusterNetworkSettings": {
                "VpcId": "vpc-example",
                "ClusterCIDR": "10.0.0.0/16",
                "ServiceCIDR": "10.1.0.0/20"
            }
        }
    ]
}
```

**预期输出 — CFS 文件系统列表**：

```json
{
    "FileSystems": [
        {
            "FileSystemId": "cfs-example01",
            "FsName": "my-cfs",
            "LifeCycleState": "available",
            "Protocol": "NFS",
            "IpAddress": "10.0.1.5",
            "VpcId": "vpc-example",
            "Zone": "ap-guangzhou-6",
            "StorageType": "HP",
            "SizeLimit": 1000000,
            "PGroup": {
                "PGroupId": "pgroupbasic",
                "Name": "默认权限组"
            }
        }
    ]
}
```

> **kubectl 连通性说明**：PV/PVC 创建需在集群端点可达的环境下执行。外网端点受 CAM 策略 `strategyId:240463971` 条件硬拒绝（`tke:clusterExtranetEndpoint=true → effect:deny`），自建安全组无法绕过。如需从本地执行 kubectl 命令，需通过 IOA/VPN/专线 或同 VPC 内的 CVM 跳板机连接集群内网端点。tccli 控制面命令（`InstallAddon`、`DescribeAddon`、`cfs DescribeCfsFileSystems` 等）不受影响。

## 控制台与 CLI 参数映射

| 控制台操作 | tccli/kubectl 命令 | 幂等 |
|-----------|-------------------|:--:|
| 查询 CFS 实例 | `cfs DescribeCfsFileSystems` | 是 |
| 查询挂载点 | `cfs DescribeMountTargets` | 是 |
| 安装 CFS-CSI 组件 | `tke InstallAddon --AddonName CFS` | 是（重复安装无影响） |
| 创建 PV | `kubectl apply -f cfs-pv.yaml` | 是（同名覆盖） |
| 创建 PVC | `kubectl apply -f cfs-pvc.yaml` | 是（同名覆盖） |
| 查看 PV/PVC | `kubectl get pv,pvc` | 是 |
| 删除 PVC | `kubectl delete pvc <Name>` | 是 |
| 删除 PV | `kubectl delete pv <Name>` | 是 |

## 关键字段说明

以下说明 CFS 静态 PV YAML 中的关键参数。

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `csi.driver` | String | 是 | 固定值 `com.tencent.cloud.csi.cfs` | 填错 → PV 创建成功但无法被 CSI 插件识别 |
| `csi.volumeAttributes.host` | String | 是 | CFS 挂载点 IP 地址，如 `10.0.1.5`。通过 `tccli cfs DescribeCfsFileSystems --region <Region>` 获取 `IpAddress` 字段 | IP 错误 → Pod 挂载超时 |
| `csi.volumeAttributes.vers` | String | 否 | `"3.0"`（NFSv3）或 `"4.0"`（NFSv4），需与 CFS 实例协议一致。默认 `"3.0"` | 版本不匹配 → 挂载失败 |
| `csi.volumeHandle` | String | 是 | CFS 文件系统 ID，格式 `cfs-xxxxxxxx`。通过 `tccli cfs DescribeCfsFileSystems` 获取 `FileSystemId` 字段 | ID 错误 → Pod 挂载失败，`kubectl describe pv` 显示驱动报错 |
| `spec.storageClassName` | String | 是 | 静态 PV 必须设为 `""`（空字符串），表示不属于任何 StorageClass | 设为某个 SC 名 → PVC 可能被其他动态卷抢占，导致绑定到错误的 PV |
| `spec.capacity.storage` | String | 是 | PV 声明的容量，如 `100Gi`。仅用于 PVC 的容量匹配，不限制 CFS 实际可用空间 | 容量小于 PVC 请求 → PVC 无法绑定此 PV |
| `spec.persistentVolumeReclaimPolicy` | String | 是 | `Retain`（推荐，删除 PV 时保留 CFS 数据）或 `Delete`（删除 PV 时同步删除 CFS 实例） | 设为 `Delete` → 删除 PV 会**不可恢复地**删除 CFS 文件系统 |

## 操作步骤

### 步骤 1：查询 CFS 实例信息

#### 选择依据

需要从 CFS 实例列表中获取两个关键字段：
- `FileSystemId`（`cfs-xxxxxxxx` 格式）— 填入 PV 的 `volumeHandle`
- `IpAddress`（挂载点 IP）— 填入 PV 的 `volumeAttributes.host`

使用 `DescribeCfsFileSystems` 列出当前地域所有 CFS 实例，按 `FsName` 或 `Tags` 筛选目标实例。

```bash
tccli cfs DescribeCfsFileSystems --region <Region>
# expected: exit 0，返回 FileSystems 列表
```

**预期输出**：

```json
{
    "FileSystems": [
        {
            "FileSystemId": "cfs-example01",
            "FsName": "my-cfs",
            "LifeCycleState": "available",
            "Protocol": "NFS",
            "IpAddress": "10.0.1.5",
            "VpcId": "vpc-example",
            "Zone": "ap-guangzhou-6",
            "StorageType": "HP",
            "SizeLimit": 1000000
        }
    ],
    "TotalCount": 1,
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

> **关键字段说明**：`LifeCycleState` 必须为 `available`，其他状态（如 `creating`、`deleting`）下无法挂载。`Protocol` 确认 NFS 版本兼容性：若为 `NFS`（默认），PV 中 `vers` 可填 `"3.0"` 或 `"4.0"`。

如需查询 CFS 的挂载点详细信息：

```bash
tccli cfs DescribeMountTargets --region <Region> --FileSystemId <FileSystemId>
# expected: exit 0，返回挂载目标列表
```

**预期输出**：

```json
{
    "MountTargets": [
        {
            "FileSystemId": "cfs-example01",
            "SubnetId": "subnet-example",
            "IpAddress": "10.0.1.5",
            "Status": "available"
        }
    ],
    "NumberOfMountTargets": 1,
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

### 步骤 2：创建 PV

#### 选择依据

- **accessModes**：`ReadWriteMany`，CFS 文件存储的核心优势——支持多 Pod 同时读写同一文件系统。
- **persistentVolumeReclaimPolicy**：`Retain`（推荐），删除 PV 时保留 CFS 实例和全部数据。设为 `Delete` 会级联删除 CFS 文件系统，数据不可恢复。
- **storageClassName**：`""`（空字符串），标识此 PV 为静态 PV，不被任何 StorageClass 的动态供给抢占。这是静态绑定的核心要求。
- **capacity.storage**：声明性容量，仅用于 PVC 请求匹配。实际可用空间取决于 CFS 实例大小，不受此值硬限制。

#### 最小配置

`cfs-pv-minimal.yaml`：

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: cfs-pv-static
spec:
  accessModes:
  - ReadWriteMany
  capacity:
    storage: 100Gi
  csi:
    driver: com.tencent.cloud.csi.cfs
    volumeAttributes:
      host: <CFS_IP_ADDRESS>
      vers: "3.0"
    volumeHandle: <CFS_FILE_SYSTEM_ID>
  persistentVolumeReclaimPolicy: Retain
  storageClassName: ""
```

```bash
kubectl apply -f cfs-pv-minimal.yaml
# expected: persistentvolume/cfs-pv-static created
```

**预期输出**：

```text
persistentvolume/cfs-pv-static created
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<CFS_IP_ADDRESS>` | CFS 文件系统挂载点 IP | 必须是目标 CFS 的 NFS 挂载点 IP | `tccli cfs DescribeCfsFileSystems --region <Region>` 输出中的 `IpAddress` 字段 |
| `<CFS_FILE_SYSTEM_ID>` | CFS 文件系统 ID | 格式 `cfs-xxxxxxxx`，必须已存在且状态为 `available` | 同上，`FileSystemId` 字段 |

#### 增强配置（加标签和挂载选项）

`cfs-pv-enhanced.yaml`：

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: cfs-pv-static
  labels:
    app: cfs-storage
    env: production
spec:
  accessModes:
  - ReadWriteMany
  capacity:
    storage: 200Gi
  csi:
    driver: com.tencent.cloud.csi.cfs
    volumeAttributes:
      host: <CFS_IP_ADDRESS>
      vers: "4.0"
    volumeHandle: <CFS_FILE_SYSTEM_ID>
  mountOptions:
  - nfsvers=4.0
  - rsize=1048576
  - wsize=1048576
  - hard
  - timeo=600
  - retrans=2
  persistentVolumeReclaimPolicy: Retain
  storageClassName: ""
```

### 步骤 3：创建 PVC 绑定

`cfs-pvc.yaml`：

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cfs-pvc-static
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 100Gi
  storageClassName: ""
  volumeName: cfs-pv-static
```

```bash
kubectl apply -f cfs-pvc.yaml
# expected: persistentvolumeclaim/cfs-pvc-static created
```

**预期输出**：

```text
persistentvolumeclaim/cfs-pvc-static created
```

> **绑定机制**：`storageClassName: ""` + `volumeName: cfs-pv-static` 确保 PVC 直接绑定到指定 PV，不经过动态供给。如果 PV 不存在或 `storageClassName` 不匹配，PVC 将保持 `Pending` 状态。

### 步骤 4：验证绑定状态

```bash
# 检查 PV 状态
kubectl get pv cfs-pv-static
# expected: STATUS: Bound, CLAIM: default/cfs-pvc-static

# 检查 PVC 状态
kubectl get pvc cfs-pvc-static
# expected: STATUS: Bound, VOLUME: cfs-pv-static

# 查看 PV 详细信息
kubectl describe pv cfs-pv-static
# expected: Phase: Bound, 显示 NFS 服务器地址和路径
```

**预期输出 — kubectl get pv**：

```text
NAME            CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                      STORAGECLASS   REASON   AGE
cfs-pv-static   100Gi      RWX            Retain           Bound    default/cfs-pvc-static                           5m
```

**预期输出 — kubectl get pvc**：

```text
NAME             STATUS   VOLUME          CAPACITY   ACCESS MODES   STORAGECLASS   AGE
cfs-pvc-static   Bound    cfs-pv-static   100Gi      RWX                           5m
```

**预期输出 — kubectl describe pv**：

```text
Name:            cfs-pv-static
Status:          Bound
Claim:           default/cfs-pvc-static
Reclaim Policy:  Retain
Access Modes:    RWX
Capacity:        100Gi
Source:
    Type:              CSI (a Container Storage Interface (CSI) volume source)
    Driver:            com.tencent.cloud.csi.cfs
    VolumeHandle:      cfs-example01
    ...
```

> **注意**：PVC 创建后如果无 Pod 消费，状态为 `Bound` 是正常的。`Bound` 仅表示 PV 和 PVC 匹配成功，不要求已有 Pod 挂载。

## 验证

| 维度 | 命令 | 预期 |
|------|------|------|
| PV 状态 | `kubectl get pv cfs-pv-static` | STATUS: `Bound` |
| PVC 状态 | `kubectl get pvc cfs-pvc-static` | STATUS: `Bound`，VOLUME 指向 `cfs-pv-static` |
| PV 详情 | `kubectl describe pv cfs-pv-static` | Phase: `Bound`，驱动为 `com.tencent.cloud.csi.cfs` |
| CFS 实例状态 | `tccli cfs DescribeCfsFileSystems --region <Region>` | LifeCycleState: `available`，未被删除 |

**验证驱动可用性**：

```bash
kubectl get csidriver | grep cfs
# expected: com.tencent.cloud.csi.cfs 出现且未显示异常
```

**预期输出**：

```text
com.tencent.cloud.csi.cfs    cfs.csi.tencent.com    true
```

## 清理

> **警告**：清理顺序必须是 **先删 PVC，再删 PV**。如果先删 PV 而 PVC 仍存在，PVC 将进入 `Lost` 状态。
>
> `persistentVolumeReclaimPolicy: Retain` 时，删除 PV 后 **CFS 实例和数据保留**。如不再需要 CFS 实例，需在 [CFS 控制台](https://console.cloud.tencent.com/cfs) 手动删除，否则持续产生 CFS 存储费用。
>
> `persistentVolumeReclaimPolicy: Delete` 时，删除 PVC 会**同步删除 CFS 文件系统及所有数据**，不可恢复。生产环境务必确认后再操作。

### 1. 清理前状态检查

```bash
# 确认 PV 和 PVC 的当前状态
kubectl get pv cfs-pv-static
kubectl get pvc cfs-pvc-static
# 确认 reclaimPolicy（Retain 或 Delete）
kubectl get pv cfs-pv-static -o jsonpath='{.spec.persistentVolumeReclaimPolicy}'
```

**预期输出 — PV 和 PVC 状态**：

```text
NAME            CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                      STORAGECLASS   REASON   AGE
cfs-pv-static   100Gi      RWX            Retain           Bound    default/cfs-pvc-static                           5m
```

```text
NAME             STATUS   VOLUME          CAPACITY   ACCESS MODES   STORAGECLASS   AGE
cfs-pvc-static   Bound    cfs-pv-static   100Gi      RWX                           5m
```

**预期输出 — reclaimPolicy**：

```text
Retain
```

### 2. 删除 PVC

```bash
kubectl delete pvc cfs-pvc-static
# expected: persistentvolumeclaim "cfs-pvc-static" deleted
```

### 3. 清理 PV

```bash
# PVC 删除后，PV 状态变为 Released
kubectl get pv cfs-pv-static
# expected: STATUS: Released
```

**预期输出**：

```text
NAME            CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS     CLAIM                      STORAGECLASS   REASON   AGE
cfs-pv-static   100Gi      RWX            Retain           Released   default/cfs-pvc-static                           10m
```

```bash
# 删除 PV
kubectl delete pv cfs-pv-static
# expected: persistentvolume "cfs-pv-static" deleted
```

### 4. 手动清理 CFS 实例（Retain 策略时）

```bash
# 确认 CFS 实例仍存在
tccli cfs DescribeCfsFileSystems --region <Region>
# expected: FileSystems 列表中仍包含目标 CFS 实例
```

**预期输出**：

```json
{
    "FileSystems": [
        {
            "FileSystemId": "cfs-example01",
            "FsName": "my-cfs",
            "LifeCycleState": "available",
            "Protocol": "NFS",
            "IpAddress": "10.0.1.5",
            "VpcId": "vpc-example",
            "Zone": "ap-guangzhou-6",
            "StorageType": "HP"
        }
    ],
    "TotalCount": 1,
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

> **注意**：CFS 实例删除不可逆，且会清除所有数据。请确认数据已备份或无保留价值后再操作。

### 5. 验证已删除

```bash
kubectl get pv cfs-pv-static 2>&1
# expected: Error from server (NotFound): persistentvolumes "cfs-pv-static" not found

kubectl get pvc cfs-pvc-static 2>&1
# expected: Error from server (NotFound): persistentvolumeclaims "cfs-pvc-static" not found
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl apply -f cfs-pv.yaml` 报 `driver name com.tencent.cloud.csi.cfs not found in the list of registered CSI drivers` | `kubectl get csidriver \| grep cfs` 检查驱动是否注册 | CFS-CSI 组件未安装或未就绪 | 安装 CFS-CSI 组件：`tccli tke InstallAddon --region <Region> --ClusterId <ClusterId> --AddonName CFS --AddonVersion <Version>` |
| PV 创建成功但 PVC 保持 `Pending` 不绑定 | `kubectl describe pvc cfs-pvc-static` 查看 Events 字段 | PVC 的 `storageClassName`、`volumeName`、`accessModes` 或 `storage` 请求与 PV 不匹配 | 确认 PVC 的 `storageClassName: ""` 与 PV 一致，`volumeName` 指向正确的 PV 名称，`accessModes` 和容量匹配 |
| `kubectl apply` 报 `connection refused` 或 `Unable to connect to the server` | `kubectl cluster-info` 测试连通性 | kubectl 无法连接集群端点。本环境外网端点受 CAM 策略 `strategyId:240463971` 硬拒绝（`tke:clusterExtranetEndpoint=true → effect:deny`） | 通过 IOA/VPN/专线 连接集群内网端点，或从同 VPC 的 CVM 跳板机执行 kubectl 命令。此为 CAM 策略限制，非命令错误 |
| `cfs DescribeCfsFileSystems` 返回空列表 | `tccli cfs DescribeCfsFileSystems --region <Region>` 确认地域正确 | 当前地域无 CFS 实例，或使用了错误的地域参数 | 确认 CFS 实例所在地域，使用 `--region` 指定正确地域。如需创建 CFS：`tccli cfs CreateCfsFileSystem` |

### PVC 绑定后 Pod 异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Pod 挂载 CFS 超时或持续 `ContainerCreating` | `kubectl describe pod <PodName>` 查看 Events 中的挂载错误 | CFS IP 不可达——可能 IP 填错、CFS 实例不在同一 VPC、或安全组/网络策略阻止 NFS 流量 | 确认 `volumeAttributes.host` 与 `DescribeCfsFileSystems` 输出的 `IpAddress` 完全一致，且 CFS 实例与集群在同一 VPC |
| Pod 挂载成功但读写报 `Permission denied` | `kubectl exec <PodName> -- ls /mnt/cfs` | NFS 挂载权限问题——Pod 的 UID/GID 与 CFS 文件权限不匹配 | 在 Pod 的 `securityContext` 中设置 `fsGroup`：`securityContext: {fsGroup: <GID>}`，或调整 CFS 权限组的访问规则 |
| PVC `Bound` 但 CFS 实例与期望不符 | `kubectl describe pv cfs-pv-static` 查看 `VolumeHandle`，与 `tccli cfs DescribeCfsFileSystems --region <Region>` 对比 | `volumeHandle` 填错导致绑定到错误的 CFS 实例 | 删除 PV/PVC，修正 `cfs-pv.yaml` 中的 `volumeHandle` 后重建 |
| PV 删除后状态卡在 `Terminating` | `kubectl describe pv cfs-pv-static` 查看 finalizer | PV 被 finalizer 保护，通常是因为仍有 PVC 绑定 | 先删除关联 PVC：`kubectl delete pvc cfs-pvc-static`，等待 PVC 完全删除后 PV 自动清理 |

## 下一步

- [StorageClass 管理文件存储模板](../StorageClass%20管理文件存储模板/tccli%20操作.md) — 了解如何通过 StorageClass 动态创建 CFS
- [文件存储使用说明](../文件存储使用说明/tccli%20操作.md) — CFS-CSI 组件安装与环境准备
- [PV 和 PVC 的绑定规则](../../PV%20和%20PVC%20的绑定规则/tccli%20操作.md) — 深入理解 PV/PVC 绑定机制
- [使用云硬盘 CBS](../../使用云硬盘%20CBS/PV%20和%20PVC%20管理云硬盘/tccli%20操作.md) — 对比 CBS 的 PV/PVC 管理

## 控制台替代

[控制台 → 集群 → 存储 → PersistentVolume](https://console.cloud.tencent.com/tke2/cluster) 创建 PV，选择"使用已有 CFS"。
