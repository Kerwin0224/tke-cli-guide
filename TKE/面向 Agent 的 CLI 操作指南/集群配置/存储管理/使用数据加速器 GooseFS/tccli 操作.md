# 使用数据加速器 GooseFS

> 对照官方：[使用数据加速器 GooseFS](https://cloud.tencent.com/document/product/457/116405) · page_id `116405` · tccli ≥3.1.107 · API 2022-05-19

## 概述

GooseFS 是腾讯云的数据加速器，为 TKE 集群提供高性能分布式缓存层，加速 AI 训练、大数据分析等数据密集型应用对底层存储（COS、HDFS 等）的访问。

**使用方式**：安装 GooseFS 扩展组件后，通过静态 PV（PersistentVolume）配置 GooseFS-CSI 驱动，将 GooseFS 命名空间挂载到 Pod 中。与 StorageClass 动态创建不同，静态 PV 方式直接引用已创建的 GooseFS 文件系统，适合需要精确控制 GooseFS 集群配置的场景。

**GooseFS 使用方式对比**：

| 方式 | 适用场景 | CSI 配置方式 | 管理复杂度 |
|------|---------|-------------|:--:|
| 静态 PV | 已有 GooseFS 集群，需精确指定 goosefsPath 和 master 地址 | `volumeAttributes` 中配置 `goosefsPath` 和 `javaOptions` | 中 |
| YAML 创建（GooseFSx） | 需要自动创建托管 GooseFS 集群 | `tccli goosefs CreateFileSystem --Type goosefsx` | 低（托管服务） |

## 前置条件

- [环境准备](../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 3.1.107

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置

# 3. 检查 TKE 权限
tccli tke DescribeClusters --region <Region>
# expected: exit 0，返回集群列表（可为空）

# 4. 检查 GooseFS 权限 — 需要以下 Action：
#    goosefs:CreateFileSystem, goosefs:DescribeFileSystems, goosefs:DeleteFileSystem
# 验证：执行 DescribeFileSystems 确认 GooseFS 产品可访问
tccli goosefs DescribeFileSystems --region <Region> --Offset 0 --Limit 20
# expected: exit 0，FSAttributeList 为空或含已有 GooseFS 集群

# 5. 检查 VPC 子网配额 — 确认子网存在
tccli vpc DescribeSubnets --region <Region> --Filters '[{"Name":"vpc-id","Values":["<VpcId>"]}]'
# expected: TotalCount >= 1，子网可用 IP 充足
```

```json
{
  "TotalCount": 1,
  "Clusters": [
    {
      "ClusterId": "<ClusterId>",
      "ClusterName": "<ClusterName>",
      "ClusterStatus": "Running"
    }
  ],
  "RequestId": "<RequestId>"
}
```

### 资源检查

```bash
# 6. 确认目标集群存在且状态正常
tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'
# expected: TotalCount >= 1，ClusterStatus: "Running"

# 7. 获取集群端点状态（确认控制面可达）
tccli tke DescribeClusterEndpoints --region <Region> --ClusterId <ClusterId>
# expected: 公网或内网端点至少一个 Status: "Created"
```

```json
{
  "TotalCount": 1,
  "Clusters": [
    {
      "ClusterId": "<ClusterId>",
      "ClusterName": "<ClusterName>",
      "ClusterStatus": "Running",
      "ClusterNetworkSettings": {
        "VpcId": "<VpcId>",
        "SubnetId": "<SubnetId>"
      }
    }
  ],
  "RequestId": "<RequestId>"
}
```

### 网络可达性要求

- 本页涉及 kubectl 命令（创建 PV、PVC、Pod）。需确保集群端点可达。
- **外网端点**需要 `tke:clusterExtranetEndpoint` CAM 权限（部分组织级 CAM 策略可能拒绝）。
- **内网端点**需要 IOA/VPN/专线 或同 VPC CVM 才能访问。
- 如无法获取 kubectl 访问，可跳过数据面操作步骤，仅通过 tccli 完成控制面操作。

## 控制台与 CLI 参数映射

| 控制台操作 | tccli / kubectl | 幂等 |
|-----------|-----------|:--:|
| 安装 GooseFS 扩展组件 | `tccli tke InstallAddon --AddonName goosefs` | 否 |
| 查询组件安装状态 | `tccli tke DescribeAddon --AddonName goosefs` | 是 |
| 创建数据加速器（GooseFSx） | `tccli goosefs CreateFileSystem --Type goosefsx` | 否 |
| 查询 GooseFS 文件系统 | `tccli goosefs DescribeFileSystems` | 是 |
| 创建 PV（静态配置） | `kubectl apply -f goosefs-pv.yaml` | 否 |
| 创建 PVC 绑定 PV | `kubectl apply -f goosefs-pvc.yaml` | 否 |
| 创建 Pod 挂载 GooseFS | `kubectl apply -f goosefs-pod.yaml` | 否 |

## 操作步骤

### 步骤 1：安装 GooseFS 扩展组件

#### 选择依据

- **AddonName**：固定为 `goosefs`（全小写），对应 CSI 驱动 `com.tencent.cloud.csi.goosefs`。
- **AddonVersion**：当前可用版本为 `1.0.0`。通过 `tccli tke DescribeAddonValues` 可查询最新版本。

#### 关键字段说明

| 字段名 | 类型 | 必填 | 取值与约束 | 错误后果 |
|--------|------|:--:|-----------|---------|
| `AddonName` | String | 是 | `goosefs`（全小写）。大小写敏感，`GooseFS` 或 `gooseFS` 均不合法 | `InvalidParameterValue: AddonName not found` |
| `AddonVersion` | String | 是 | 枚举值如 `1.0.0`。需先通过 `DescribeAddonValues` 确认集群可用版本 | `InvalidParameterValue.AddonVersionNotFound` 或安装失败 |
| `ClusterId` | String | 是 | TKE 集群 ID，格式 `cls-xxxxxxxx` | `ResourceNotFound.ClusterNotFound` |

```bash
# 安装 GooseFS 扩展组件
tccli tke InstallAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName goosefs \
    --AddonVersion 1.0.0
# expected: exit 0，返回 RequestId
```

> **幂等性说明**：GooseFS 组件不支持重复安装。若集群已安装 GooseFS（可通过 `DescribeAddon` 确认），再次执行 `InstallAddon` 会返回 `UnknownParameter: check addon is exist`（exit 255）。此时无需重新安装，直接跳过此步骤进入组件状态查询即可。

**预期输出**：

```json
{
    "RequestId": "<RequestId>"
}
```

#### 轮询组件安装状态

```bash
tccli tke DescribeAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName goosefs
# expected: Phase: "Succeeded"
```

**预期输出**：

```json
{
    "Addons": [
        {
            "AddonName": "goosefs",
            "AddonVersion": "1.0.0",
            "RawValues": "<Base64EncodedConfig>",
            "Phase": "Succeeded",
            "Reason": "",
            "CreateTime": "<CreateTime>"
        }
    ],
    "RequestId": "<RequestId>"
}
```

其中 `RawValues`（Base64 解码后）为默认配置：

```json
{
  "global": {
    "cluster": {
      "highAvailability": true
    }
  }
}
```

#### 验证组件就绪（kubectl）

以下 kubectl 命令需集群端点可达（见[网络可达性要求](#网络可达性要求)）。

```bash
kubectl -n kube-system get pods -l app.kubernetes.io/name=goosefs-csi
# expected: goosefs-csi 相关 Pod 均为 Running
```

```text
NAME                    READY   STATUS    RESTARTS   AGE
csi-goosefs-node-xxxxx  2/2     Running   0          1m
csi-goosefs-controller-xxxxx 2/2 Running   0         1m
```

### 步骤 2：创建数据加速器（GooseFSx）

#### 选择依据

- **Type**：当前仅支持 `goosefsx`（托管版 GooseFS），旧版 `goosefs` 类型已停止新创建。
- **Model**：可选 `C60`、`C100` 等型号，决定计算和缓存性能。C60 为基础型号。
- **Capacity**：存储容量（GB），最小 4608（4.5TB）。按量计费（postPay）。
- **VPC/Subnet**：需与 TKE 集群在同一 VPC 下，GooseFSx 通过 VPC 内网与集群通信。

> **计费警告**：GooseFSx 为按量计费（postPay）产品。创建后即使未使用也会产生存储费用。验证完成后请及时删除（见[清理](#清理)）。

#### 关键字段说明

| 字段名 | 类型 | 必填 | 取值与约束 | 错误后果 |
|--------|------|:--:|-----------|---------|
| `Type` | String | 是 | 枚举 `goosefsx`。旧值 `goosefs` 已停用 | `InvalidParameterValue: unsupported filesystem type` |
| `GooseFSxBuildElements` | JSON | 是 | JSON 字符串，含 `Model`（必填）、`Capacity`（必填，≥4608）、`MappedBucketList`（可选）。必须是合法 JSON | `InvalidParameterValue: GooseFSxBuildElements is nil` 或 JSON 解析失败 |
| `Model` | String | 是 | 枚举 `C60`、`C100` 等，影响计算/缓存性能和计费单价 | `InvalidParameterValue: unsupported model` |
| `Capacity` | Integer | 是 | ≥4608 GB。影响存储容量和计费 | `InvalidParameterValue: invalid capacity` |
| `VpcId` | String | 是 | TKE 集群所在 VPC ID | `ResourceNotFound: Vpc not found` |
| `SubnetId` | String | 是 | 与集群同可用区的子网 ID | `ResourceNotFound: Subnet not found` 或 Zone 不匹配导致创建后 FAIL |
| `Zone` | String | 是 | 可用区，需与 Subnet 所在 Zone 一致 | `InvalidParameterValue: zone mismatch` 或创建后资源供应失败 |
| `SecurityGroupId` | String | 否 | 安全组 ID，不传则使用默认 | — |
| `Tag` | Array | 否 | 标签列表，格式 `[{"Key":"k","Value":"v"}]` | — |

#### 最小创建（必填参数）

```bash
# 创建 GooseFSx 文件系统（最小配置）
tccli goosefs CreateFileSystem --region <Region> \
    --Name 'my-goosefs' \
    --Description 'test goosefs filesystem' \
    --VpcId <VpcId> \
    --SubnetId <SubnetId> \
    --Zone <Zone> \
    --Type goosefsx \
    --GooseFSxBuildElements '{"Model":"C60","Capacity":4608,"MappedBucketList":[]}'
# expected: exit 0，返回 FileSystemId
```

**预期输出**：

```json
{
    "FileSystemId": "<FileSystemId>",
    "RequestId": "<RequestId>"
}
```

#### 增强配置（绑定 COS 桶 + 其他型号）

```bash
# 创建 GooseFSx 文件系统（增强配置：绑定 COS 桶，更大容量）
tccli goosefs CreateFileSystem --region <Region> \
    --Name 'my-goosefs-enhanced' \
    --Description 'enhanced goosefs with COS bucket' \
    --VpcId <VpcId> \
    --SubnetId <SubnetId> \
    --Zone <Zone> \
    --Type goosefsx \
    --GooseFSxBuildElements '{"Model":"C100","Capacity":9216,"MappedBucketList":[{"BucketName":"my-bucket-1251707795","FileSystemPath":"/cos-data","BucketRegion":"<Region>"}]}'
# expected: exit 0，返回 FileSystemId
```

**预期输出**：

```json
{
    "FileSystemId": "<FileSystemId>",
    "RequestId": "<RequestId>"
}
```

#### 轮询文件系统状态

```bash
tccli goosefs DescribeFileSystems --region <Region> \
    --Offset 0 --Limit 20
# expected: Status 从 "CREATING" → "ACTIVE"
```

**预期输出（创建中）**：

```json
{
    "FSAttributeList": [
        {
            "Type": "goosefsx",
            "FileSystemId": "<FileSystemId>",
            "CreateTime": "<CreateTime>",
            "GooseFSxAttribute": {
                "Model": "C60",
                "Capacity": 0,
                "MappedBucketList": [],
                "ClientManagerNodeList": []
            },
            "Status": "CREATING",
            "Name": "my-goosefs",
            "Description": "test goosefs filesystem",
            "VpcId": "<VpcId>",
            "SubnetId": "<SubnetId>",
            "Zone": "<Zone>",
            "Tag": [],
            "ModifyTime": "<ModifyTime>",
            "ChargeAttribute": {
                "CurDeadline": null,
                "PayMode": "postPay",
                "AutoRenewFlag": null,
                "ResourceId": "<ResourceId>"
            }
        }
    ],
    "TotalCount": 1,
    "RequestId": "<RequestId>"
}
```

**预期输出（就绪）**：

```json
{
    "FSAttributeList": [
        {
            "Type": "goosefsx",
            "FileSystemId": "<FileSystemId>",
            "CreateTime": "<CreateTime>",
            "GooseFSxAttribute": {
                "Model": "C60",
                "Capacity": 4608,
                "MappedBucketList": [],
                "ClientManagerNodeList": [
                    {
                        "ClientManagerNodeIp": "<NodeIp>",
                        "Status": "Active"
                    }
                ]
            },
            "Status": "ACTIVE",
            "Name": "my-goosefs",
            "Description": "test goosefs filesystem",
            "VpcId": "<VpcId>",
            "SubnetId": "<SubnetId>",
            "Zone": "<Zone>",
            "Tag": [],
            "ModifyTime": "<ModifyTime>",
            "ChargeAttribute": {
                "CurDeadline": null,
                "PayMode": "postPay",
                "AutoRenewFlag": null,
                "ResourceId": "<ResourceId>"
            }
        }
    ],
    "TotalCount": 1,
    "RequestId": "<RequestId>"
}
```

创建 GooseFSx 文件系统通常需要 5-10 分钟。Status "CREATING" → "ACTIVE" 后即可使用。

| 占位符 | 说明 | 获取方式 |
|--------|------|---------|
| `<VpcId>` | 集群所在 VPC ID | `tccli tke DescribeClusters --ClusterIds '["<ClusterId>"]'` 的 `ClusterNetworkSettings.VpcId` |
| `<SubnetId>` | 与集群同可用区的子网 ID | `tccli vpc DescribeSubnets --Filters '[{"Name":"vpc-id","Values":["<VpcId>"]}]'` 筛选，或直接使用集群的 `ClusterNetworkSettings.SubnetId` |
| `<Zone>` | 可用区，如 `ap-guangzhou-3` | `tccli vpc DescribeSubnets --SubnetIds '["<SubnetId>"]'` 查看 `Zone` 字段 |
| `<FileSystemId>` | GooseFSx 文件系统 ID，格式 `x-c60-xxxxxxxx` | `CreateFileSystem` 返回，或 `DescribeFileSystems` 查询 |

### 步骤 3：创建静态 PV

#### 选择依据

静态 PV 方式直接引用已创建的 GooseFS 文件系统：
- **driver**：固定为 `com.tencent.cloud.csi.goosefs`，由步骤 1 安装的组件注册。
- **goosefsPath**：GooseFS 文件系统中已存在的命名空间路径，如 `/my-bucket-1251707795`。
- **javaOptions**：FUSE 客户端 JVM 参数，包含 master 地址、内存配置、监控开关等。

以下 kubectl 命令需集群端点可达。如端点不可达（例如外网端点被 CAM 策略拒绝，且无可达的内网端点），请记录 YAML 内容作为操作参考，待获取端点访问后执行。

`goosefs-pv.yaml`：

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: goosefs-pv
spec:
  accessModes:
  - ReadWriteMany
  capacity:
    storage: 10Gi
  csi:
    driver: com.tencent.cloud.csi.goosefs
    volumeAttributes:
      goosefsPath: <GOOSEFS_PATH>
      javaOptions: >-
        -Dgoosefs.master.embedded.journal.addresses=<MASTER_IP_1>:9202,<MASTER_IP_2>:9202,<MASTER_IP_3>:9202
        -Xms4g -Xmx8g -XX:MaxDirectMemorySize=8g
        -XX:+UseG1GC
        -Dgoosefs.fuse.web.enabled=true
        -Dgoosefs.fuse.web.port=9213
    volumeHandle: goosefs-pv
  mountOptions:
  - allow_other
  - direct_io
  persistentVolumeReclaimPolicy: Retain
  volumeMode: Filesystem
```

> **javaOptions 参数说明（来自官方文档）**：

| 参数 | 说明 | 默认值 | 必填 |
|------|------|--------|:--:|
| `-Xms4g` | JVM 最小堆内存 | 4g | 否 |
| `-Xmx8g` | JVM 最大堆内存 | 8g | 否 |
| `-XX:MaxDirectMemorySize=8g` | 堆外最大内存 | 8g | 否 |
| `-XX:+UseG1GC` | JVM GC 算法 | G1 | 否 |
| `-Dgoosefs.fuse.web.enabled` | 是否开启 FUSE Web 监控 | false | 否 |
| `-Dgoosefs.fuse.web.port` | FUSE Web 监控端口 | 9213 | 否 |

```bash
kubectl apply -f goosefs-pv.yaml
# expected: persistentvolume/goosefs-pv created
```

**预期输出**：

```text
persistentvolume/goosefs-pv created
```

| 占位符 | 说明 | 获取方式 |
|--------|------|---------|
| `<GOOSEFS_PATH>` | GooseFS 中已存在的路径 | GooseFS 控制台或 FUSE 客户端 `goosefs fs ls /` |
| `<MASTER_IP_1/2/3>` | GooseFS 集群 master 节点 IP | `tccli goosefs DescribeFileSystems` 查看 GooseFSx 详情，或 GooseFS 控制台 |

### 步骤 4：创建 PVC 绑定 PV

`goosefs-pvc.yaml`：

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: goosefs-pvc
  namespace: <NAMESPACE>
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
  storageClassName: ""
  volumeMode: Filesystem
  volumeName: goosefs-pv
```

```bash
kubectl apply -f goosefs-pvc.yaml
# expected: persistentvolumeclaim/goosefs-pvc created
```

**预期输出**：

```text
persistentvolumeclaim/goosefs-pvc created
```

| 占位符 | 说明 |
|--------|------|
| `<NAMESPACE>` | Kubernetes 命名空间，如 `default` 或 `kube-system` |

### 步骤 5：创建 Pod 挂载 GooseFS

`goosefs-pod.yaml`：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-goosefs
spec:
  containers:
  - name: pod-goosefs
    command: ["tail", "-f", "/etc/hosts"]
    image: "centos:latest"
    volumeMounts:
    - mountPath: /data
      name: goosefs
    resources:
      requests:
        memory: "128Mi"
        cpu: "0.1"
  volumes:
  - name: goosefs
    persistentVolumeClaim:
      claimName: goosefs-pvc
```

```bash
kubectl apply -f goosefs-pod.yaml
# expected: pod/pod-goosefs created
```

**预期输出**：

```text
pod/pod-goosefs created
```

#### 验证 Pod 挂载

```bash
kubectl get pod pod-goosefs
# expected: STATUS: Running

kubectl exec pod-goosefs -- ls /data
# expected: 列出 GooseFS 命名空间下的文件目录
```

## 验证

### 控制面（tccli）

| 维度 | 命令 | 预期 |
|------|------|------|
| 组件状态 | `tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName goosefs` | `Phase: "Succeeded"` |
| 集群端点状态 | `tccli tke DescribeClusterEndpoints --region <Region> --ClusterId <ClusterId>` | 公网或内网端点 `Status: "Created"` |
| GooseFSx 文件系统 | `tccli goosefs DescribeFileSystems --region <Region> --Offset 0 --Limit 20` | `Status: "ACTIVE"`, `Type: "goosefsx"` |
| Kubeconfig 获取 | `tccli tke DescribeClusterKubeconfig --region <Region> --ClusterId <ClusterId> --IsExtranet true` | 返回合法 Kubeconfig，可用于 `kubectl` 连接 |

### 数据面（kubectl）

```bash
# 验证 PV 已创建
kubectl get pv goosefs-pv
# expected: STATUS: Available 或 Bound
```

```text
NAME         CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM                  STORAGECLASS   AGE
goosefs-pv   10Gi       RWX            Retain           Bound       <NAMESPACE>/goosefs-pvc                1m
```

```bash
# 验证 PVC 已绑定
kubectl get pvc goosefs-pvc -n <NAMESPACE>
# expected: STATUS: Bound, VOLUME: goosefs-pv
```

```text
NAME          STATUS   VOLUME       CAPACITY   ACCESS MODES   STORAGECLASS   AGE
goosefs-pvc   Bound    goosefs-pv   10Gi       RWX                           1m
```

```bash
# 验证 FUSE Pod 运行
kubectl get pod pod-goosefs
# expected: STATUS: Running
```

```text
NAME           READY   STATUS    RESTARTS   AGE
pod-goosefs    2/2     Running   0          30s
```

```bash
# 验证挂载读写
kubectl exec pod-goosefs -- ls /data
# expected: 列出挂载目录内容
```

```text
file1.txt
file2.txt
subdir/
```

```bash
# 验证 GooseFS CSI 驱动已注册
kubectl get csidriver | grep goosefs
# expected: com.tencent.cloud.csi.goosefs
```

```text
NAME                            ATTACHREQUIRED   PODINFOONMOUNT   ...
com.tencent.cloud.csi.goosefs   ...               ...              ...
```

## 清理

> **警告**：清理时请严格按逆序操作。先删 Pod，再删 PVC/PV，最后删文件系统。`goosefs DeleteFileSystem` 会删除 GooseFSx 文件系统及其全部数据，不可恢复。
>
> **生命周期限制**：GooseFSx 文件系统处于 `CREATING`（创建中）状态时不可删除。API 返回 `InvalidParameterValue: filesystem in creation can't be deleted`。需等待 Status 变为 `ACTIVE` 后再执行 `DeleteFileSystem`。

### 1. 清理前状态检查

```bash
# 确认待删除资源列表
kubectl get pod pod-goosefs -o wide
kubectl get pvc goosefs-pvc -n <NAMESPACE>
kubectl get pv goosefs-pv
# 确认以上均为待清理的目标资源
```

### 数据面清理（kubectl -- 先于控制面）

#### 2. 删除测试 Pod

```bash
kubectl delete pod pod-goosefs
# expected: pod "pod-goosefs" deleted
```

#### 3. 删除 PVC

```bash
kubectl delete pvc goosefs-pvc -n <NAMESPACE>
# expected: persistentvolumeclaim "goosefs-pvc" deleted
```

#### 4. 删除 PV

```bash
kubectl delete pv goosefs-pv
# expected: persistentvolume "goosefs-pv" deleted
```

### 控制面清理（tccli）

#### 5. 删除 GooseFSx 文件系统

> **计费警告**：GooseFSx 为按量计费（postPay），未删除会持续产生费用。删除操作不可逆，文件系统中所有数据将被清除。

```bash
# 确认文件系统状态（必须 ACTIVE 才可删除）
tccli goosefs DescribeFileSystems --region <Region> \
    --Offset 0 --Limit 20
# 确认目标 FileSystemId 和 Status。Status 为 CREATING 时不可删除，需等待 ACTIVE
```

**预期输出**：

```json
{
    "FSAttributeList": [
        {
            "Type": "goosefsx",
            "FileSystemId": "<FileSystemId>",
            "Status": "ACTIVE",
            "Name": "<Name>",
            "CreateTime": "<CreateTime>"
        }
    ],
    "TotalCount": 1,
    "RequestId": "<RequestId>"
}
```

```bash
# 删除文件系统
tccli goosefs DeleteFileSystem --region <Region> \
    --FileSystemId <FileSystemId>
# expected: exit 0，返回 RequestId
```

```json
{
    "RequestId": "<RequestId>"
}
```

#### 6. 卸载 GooseFS 组件（可选）

```bash
tccli tke DeleteAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName goosefs
# expected: exit 0，返回 RequestId
```

### 7. 验证已删除

```bash
# 确认文件系统已删除
tccli goosefs DescribeFileSystems --region <Region> \
    --Offset 0 --Limit 20
# expected: FSAttributeList 中无目标 FileSystemId
```

**预期输出**：

```json
{
    "FSAttributeList": [],
    "TotalCount": 0,
    "RequestId": "<RequestId>"
}
```

```bash
# 确认组件已卸载（如执行了步骤 6）
tccli tke DescribeAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName goosefs
# expected: ResourceNotFound 或 Addons 列表为空
```

**预期输出**：

```json
{
    "Addons": [],
    "RequestId": "<RequestId>"
}
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `InstallAddon --AddonName goosefs` 返回 `UnknownParameter: check addon is exist` | `tccli tke DescribeAddon --AddonName goosefs` 检查是否已安装 | 集群已安装 GooseFS 组件（幂等冲突），无需重复安装 | 跳过安装步骤，直接进入组件状态查询 |
| `InstallAddon --AddonName goosefs` 返回参数错误 | 检查 `AddonName` 大小写 | 填了 `GooseFS`（大写 G/FS）而非 `goosefs`（全小写） | 使用 `--AddonName goosefs`（全小写） |
| `CreateFileSystem --Type goosefs` 返回 `unsupported filesystem type` | `tccli goosefs CreateFileSystem help --detail` 查看 Type 参数说明 | `goosefs` 类型已不再支持新创建 | 改用 `--Type goosefsx` 并配置 `--GooseFSxBuildElements` |
| `CreateFileSystem --Type goosefsx` 返回 `GooseFSxBuildElements is nil` | 检查是否传入了 GooseFSxBuildElements | `goosefsx` 类型要求必须传入 Model 和 Capacity | 添加 `--GooseFSxBuildElements '{"Model":"C60","Capacity":4608,"MappedBucketList":[]}'` |
| `DeleteFileSystem` 返回 `InvalidParameterValue: filesystem in creation can't be deleted` | `tccli goosefs DescribeFileSystems` 查看 Status | 文件系统处于 CREATING 状态，API 限制创建中不可删除 | 等待 Status 变为 ACTIVE 后再执行删除 |
| `CreateFileSystem` 返回成功但后续 Status 为 `FAIL` | `tccli goosefs DescribeFileSystems` 查看最终状态 | 资源供应失败，可能是可用区资源不足或 VPC/子网配置问题 | 确认 Zone 有足够资源，确认 Subnet 与 Zone 匹配。如持续失败，换 Zone 重试 |
| `kubectl apply` 返回 `connection refused` | `kubectl cluster-info` | 当前环境无法访问集群端点（外网端点被 CAM 策略 `tke:clusterExtranetEndpoint=true` 拒绝，内网端点需 IOA/VPN/专线） | 在有端点访问权限的环境执行 kubectl 命令，或使用同 VPC CVM 作为跳板机 |

### 安装成功但 Pod 异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| PV 状态为 `Available`（未被绑定） | `kubectl describe pv goosefs-pv` | PVC 命名空间与 PV 的 `volumeHandle` 不匹配 | 确认 PVC 中 `volumeName` 与 PV 名称一致 |
| PVC 一直 `Pending` | `kubectl describe pvc goosefs-pvc -n <NAMESPACE>` 查看 Events | 无匹配 PV，或 PV `storageClassName` 与 PVC 不匹配 | 确认 PV 的 `storageClassName: ""` 与 PVC 一致 |
| Pod 创建后 FUSE 侧车容器 CrashLoopBackOff | `kubectl logs pod-goosefs -c <fuse-container>` | `javaOptions` 中的 master 地址不可达，内存配置不足，或 `goosefsPath` 不存在 | 确认 master IP 可达，调整 JVM 内存参数，确认 GooseFS 命名空间路径已创建 |
| FUSE Pod 占用大量内存 | `kubectl top pod pod-goosefs` | `javaOptions` 中 `-Xms4g -Xmx8g` 按默认值分配了 4GB+ | 在 PV `javaOptions` 中调低 `-Xms` 和 `-Xmx` 值（注意: 过小可能影响缓存性能） |

## 下一步

- [使用对象存储 COS](../使用对象存储%20COS/tccli%20操作.md)
- [PV 和 PVC 的绑定规则](../PV%20和%20PVC%20的绑定规则/tccli%20操作.md)
- [使用文件存储 CFS — 文件存储使用说明](../使用文件存储%20CFS/文件存储使用说明/tccli%20操作.md)

## 控制台替代

- [TKE 控制台 → 组件管理](https://console.cloud.tencent.com/tke2/cluster) 安装 GooseFS 扩展组件
- [GooseFS 控制台](https://console.cloud.tencent.com/goosefs) 创建和管理 GooseFS 文件系统
- 控制台支持向导式创建 PV/PVC/Pod，自动填充 `javaOptions` 参数
