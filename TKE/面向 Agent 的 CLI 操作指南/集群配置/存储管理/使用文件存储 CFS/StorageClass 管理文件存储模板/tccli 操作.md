# StorageClass 管理文件存储模板

> 对照官方：[StorageClass 管理文件存储模板](https://cloud.tencent.com/document/product/457/44235) · page_id `44235` · tccli ≥3.1.107 · API 2019-07-19

## 概述

通过创建 CFS 类型的 StorageClass，实现 PVC 动态创建 CFS 文件系统并绑定到 Pod。StorageClass 中指定可用区（zone）、权限组（pgroupid）和存储类型（storagetype）等参数，PVC 创建时自动在指定可用区下新建 CFS 实例。

**与静态 PV 对比**：

| 维度 | 动态创建（StorageClass） | 静态创建（已有 CFS） |
|------|---------------------|----------------|
| CFS 实例 | PVC 创建时自动新建 | 需提前创建 |
| 管控粒度 | StorageClass 统一模板 | 每个 PV 独立配置 |
| 适合场景 | 按需自动供给、多租户 | 存量 CFS 复用、精确控制 |

## 前置条件

- [环境准备](../../../../环境准备.md)
- 已安装 CFS-CSI 组件：[文件存储使用说明](../文件存储使用说明/tccli%20操作.md)
- 集群内网端点可达（kubectl 可连接目标集群）

### 环境检查

```bash
# 1. 确认 CFS-CSI 已安装
tccli tke DescribeAddon --region <REGION> --ClusterId <CLUSTER_ID> --AddonName CFS
# expected: Phase: "Succeeded", Status: "Running"

# 2. 确认 kubectl 可达
kubectl get ns
# expected: 返回命名空间列表

# 3. 查询集群 VPC 和子网信息
tccli tke DescribeClusters --region <REGION> --ClusterIds '["<CLUSTER_ID>"]'
# expected: 记录 ClusterNetworkSettings.VpcId、可用区及子网
```

```json
{
  "ClusterName": "<ClusterName>",
  "ClusterStatus": "Running",
  "ClusterVersion": "<Version>",
  "ClusterNetworkSettings": {
    "VpcId": "<VpcId>",
    "ClusterCIDR": "<CIDR>",
    "SubnetId": "<SubnetId>"
  }
}
```

### 资源检查

```bash
# 4. 确认目标子网可用 IP 充足（如指定 subnetid）
tccli vpc DescribeSubnets --region <REGION> --Filters '[{"Name":"vpc-id","Values":["<VPC_ID>"]}]'
# expected: 返回子网列表，AvailableIpAddressCount >= 1

# 5. 确认 CFS 权限组已存在
tccli cfs DescribeCfsPGroups --region <REGION>
# expected: 返回权限组列表，记录目标权限组 PGroupId
```

> **kubectl 连通性说明**：StorageClass 和 PVC 均为 kubectl 数据面操作，需在集群端点可达的环境下执行。外网端点被 CAM 策略 `strategyId:240463971`（condition `tke:clusterExtranetEndpoint=true`）硬拒绝。当前集群仅有内网端点（`172.24.0.12`），需通过 IOA/VPN/专线或同 VPC CVM 跳板机执行 kubectl 命令。

## 控制台与 CLI 参数映射

| 控制台操作 | tccli/kubectl 命令 | 幂等 |
|-----------|-------------------|:--:|
| 安装 CFS-CSI | `tccli tke InstallAddon --AddonName CFS` | 否 |
| 创建 StorageClass | `kubectl apply -f cfs-sc.yaml` | 是 |
| 创建 PVC | `kubectl apply -f cfs-pvc.yaml` | 否 |
| 查询 StorageClass | `kubectl get sc` | 是 |

## 关键字段说明

以下说明 CFS StorageClass YAML 中的参数。必填字段（zone、pgroupid、storagetype）来自 CFS-CSI 驱动要求，可选字段可按需配置。

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `provisioner` | String | 是 | 固定值 `com.tencent.cloud.csi.cfs` | 填写其他值 → PVC 一直 Pending，Events 报 "provisioner not found" |
| `parameters.zone` | String | 是 | 文件存储所处可用区，如 `ap-guangzhou-3` | 可用区不支持 CFS → CFS 实例创建失败 |
| `parameters.pgroupid` | String | 是 | CFS 权限组 ID，格式 `pgroup-xxxxxxxx` | 权限组不存在 → CFS 创建失败 |
| `parameters.storagetype` | String | 是 | `SD`（标准型存储）或 `HP`（性能存储），默认 `SD` | 指定类型在可用区不支持 → CFS 创建失败 |
| `parameters.vpcid` | String | 否 | 文件存储所在 VPC ID，格式 `vpc-xxxxxxxx`。不填则自动使用集群 VPC | VPC 错误 → CFS 创建在错误 VPC，Pod 无法挂载 |
| `parameters.subnetid` | String | 否 | 文件存储所在子网 ID，格式 `subnet-xxxxxxxx`。不填则自动使用集群默认子网 | 子网不可用 → CFS 创建失败 |
| `parameters.vers` | String | 否 | 协议版本：`"3"`（NFSv3）或 `"4"`（NFSv4） | 不填默认使用 NFSv3 |
| `parameters.subdir-share` | String | 否 | 填写任意值（如 `"true"`）则启用共享实例模式：多个 PVC 共享同一 CFS 实例的不同子目录 | 共享实例模式下 `reclaimPolicy` 须为 `Retain` |
| `parameters.resourcetags` | String | 否 | CFS 资源标签，格式 `"a:b,c:d"`（英文逗号分隔多组键值对） | 格式错误 → CFS 创建失败 |
| `reclaimPolicy` | String | 否 | `Delete`（PVC 删则 CFS 删）或 `Retain`（保留 CFS） | `Delete` 时误删 PVC → CFS 数据丢失 |
| `volumeBindingMode` | String | 否 | `Immediate`（立即创建）或 `WaitForFirstConsumer` | `Immediate` 在 Pod 未调度时即创建 CFS，可能产生闲置费用 |

## 操作步骤

### 步骤 1：创建 StorageClass

#### 选择依据

- **provisioner**：CFS-CSI 注册的驱动名为 `com.tencent.cloud.csi.cfs`，不可更改。
- **zone**：选择与集群节点同可用区，降低跨可用区网络延迟。获取方式：`tccli vpc DescribeSubnets --region <Region> --SubnetIds '["<SubnetId>"]'` 查看 `Zone`。
- **pgroupid**：CFS 权限组控制来访客户端的读写权限。如无现有权限组，需提前创建（控制台 → 文件存储 → 权限组）。
- **storagetype**：`SD`（标准存储）适用于数据备份、文件共享、日志存储等成本敏感场景；`HP`（性能存储）适用于高性能计算、媒资渲染、机器学习等 IO 密集型工作负载。
- **reclaimPolicy**：测试环境推荐 `Delete`，生产环境推荐 `Retain`。
- **volumeBindingMode**：推荐 `Immediate`（CFS 不绑定节点，可跨可用区）。

#### 最小配置

`kerwinwjyan-cfs-sc.yaml`：

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: kerwinwjyan-cfs-sc
parameters:
  zone: <ZONE>
  pgroupid: <PGROUP_ID>
  storagetype: SD
provisioner: com.tencent.cloud.csi.cfs
reclaimPolicy: Delete
volumeBindingMode: Immediate
```

```bash
kubectl apply -f kerwinwjyan-cfs-sc.yaml
# expected: storageclass.storage.k8s.io/kerwinwjyan-cfs-sc created
```

| 占位符 | 说明 | 获取方式 |
|--------|------|---------|
| `ZONE` | CFS 实例所在可用区 | `tccli vpc DescribeSubnets --region <Region> --SubnetIds '["<SubnetId>"]'` 查看 `Zone`，如 `ap-guangzhou-3` |
| `PGROUP_ID` | CFS 权限组 ID | `tccli cfs DescribeCfsPGroups --region <Region>` 查看已有权限组，格式 `pgroup-xxxxxxxx` |

#### 增强配置

指定 VPC、子网和协议版本，并绑定云标签：

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: kerwinwjyan-cfs-sc-full
parameters:
  zone: <ZONE>
  pgroupid: <PGROUP_ID>
  storagetype: SD
  vpcid: <VPC_ID>
  subnetid: <SUBNET_ID>
  vers: "3"
  resourcetags: "billing:my-cfs-tag"
provisioner: com.tencent.cloud.csi.cfs
reclaimPolicy: Retain
volumeBindingMode: Immediate
```

```bash
kubectl apply -f kerwinwjyan-cfs-sc-full.yaml
# expected: storageclass.storage.k8s.io/kerwinwjyan-cfs-sc-full created
```

### 步骤 2：创建 PVC 自动供应 CFS

#### 选择依据

- **accessModes**：`ReadWriteMany`，CFS 支持多节点同时读写。
- **storage**：CFS 按实际使用量计费，PVC 中 `storage` 仅作容量声明，不影响 CFS 实际容量限制。

`kerwinwjyan-cfs-pvc.yaml`：

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: kerwinwjyan-cfs-pvc
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
  storageClassName: kerwinwjyan-cfs-sc
```

```bash
kubectl apply -f kerwinwjyan-cfs-pvc.yaml
# expected: persistentvolumeclaim/kerwinwjyan-cfs-pvc created
```

### 步骤 3：验证 PV/PVC 绑定

```bash
kubectl get pvc kerwinwjyan-cfs-pvc
# expected: STATUS: Bound

kubectl get pv | grep kerwinwjyan-cfs-pvc
# expected: 自动创建 PV，STATUS: Bound
```

```text
NAME                    STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS          AGE
kerwinwjyan-cfs-pvc     Bound    pvc-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx   10Gi       RWX            kerwinwjyan-cfs-sc    10s
```

## 验证

| 维度 | 命令 | 预期 |
|------|------|------|
| StorageClass 存在 | `kubectl get sc kerwinwjyan-cfs-sc` | NAME: `kerwinwjyan-cfs-sc`，PROVISIONER: `com.tencent.cloud.csi.cfs` |
| PVC 绑定 | `kubectl get pvc kerwinwjyan-cfs-pvc` | STATUS: `Bound` |
| PV 自动创建 | `kubectl get pv \| grep kerwinwjyan-cfs-pvc` | PV 存在且 STATUS: `Bound` |
| CFS 实例创建 | `tccli cfs DescribeCfsFileSystems --region <REGION>` | 出现与 PVC 对应的新 CFS 实例，LifeCycleState: `available` |

## 清理

> **警告**：`reclaimPolicy: Delete` 时，删除 PVC 会同步删除关联的 CFS 文件系统及其中所有数据。生产环境建议使用 `Retain`，避免误删数据。`Retain` 策略下删除 PVC 后，PV 变为 `Released` 状态，CFS 实例保留，需手动删除 PV 和 CFS 实例。

### 1. 清理前状态检查

```bash
kubectl get pvc kerwinwjyan-cfs-pvc
kubectl get pv | grep kerwinwjyan-cfs-pvc
# 确认是待删除的目标资源，确认 reclaimPolicy
```

```text
NAME                    STATUS   AGE
kerwinwjyan-cfs-pvc     Bound    5m
```

### 2. 删除 PVC

```bash
kubectl delete pvc kerwinwjyan-cfs-pvc
# expected: persistentvolumeclaim "kerwinwjyan-cfs-pvc" deleted
# reclaimPolicy: Delete → PV 和 CFS 实例同步删除
# reclaimPolicy: Retain → PV 保留（Released 状态），CFS 实例保留
```

### 3. 删除 StorageClass

```bash
kubectl delete sc kerwinwjyan-cfs-sc
# expected: storageclass.storage.k8s.io "kerwinwjyan-cfs-sc" deleted
```

```bash
# 如创建了增强配置的 StorageClass，一并清理
kubectl delete sc kerwinwjyan-cfs-sc-full
# expected: storageclass.storage.k8s.io "kerwinwjyan-cfs-sc-full" deleted
```

### 4. 验证已删除

```bash
kubectl get pvc kerwinwjyan-cfs-pvc 2>&1
# expected: Error from server (NotFound)

# reclaimPolicy: Delete 时确认 CFS 也已删除
tccli cfs DescribeCfsFileSystems --region <REGION>
# expected: 原 CFS 实例不再出现
```

```text
Error from server (NotFound): persistentvolumeclaims "kerwinwjyan-cfs-pvc" not found
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl apply -f cfs-sc.yaml` 报 "provisioner not found" | `kubectl get csidriver` 查看已注册 CSI 驱动 | CFS-CSI 组件未安装 | 参考 [文件存储使用说明](../文件存储使用说明/tccli%20操作.md) 安装 CFS-CSI：`tccli tke InstallAddon --ClusterId <CLUSTER_ID> --AddonName CFS` |
| kubectl 连接被拒绝（`connection refused`） | `tccli tke DescribeClusterEndpoints --region <REGION> --ClusterId <CLUSTER_ID>` 查看端点状态 | 无外网端点（CAM 硬策略 `strategyId:240463971` 拒绝 `tke:clusterExtranetEndpoint=true`）且当前环境无法访问内网端点 | 通过 IOA/VPN/专线 或同 VPC CVM 跳板机连接集群；或确认集群已有公网端点 |

### PVC 异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| PVC 一直 Pending | `kubectl describe pvc <PVC_NAME>` 查看 Events | Provisioner 无法在指定可用区/权限组创建 CFS：`zone` 不支持 CFS 或 `pgroupid` 不存在 | 1. `tccli cfs DescribeCfsPGroups --region <REGION>` 确认权限组存在；2. 确认可用区支持 CFS |
| PVC Pending，Events 报 "subnet not found" | `tccli vpc DescribeSubnets --region <REGION> --SubnetIds '["<SUBNET_ID>"]'` | `subnetid` 填错或子网已被删除 | 从 `DescribeSubnets` 输出取正确子网 ID，修改 `cfs-sc.yaml` 后重新 apply |
| PVC Bound 但 Pod 挂载失败 | `kubectl describe pod <POD_NAME>` 查看 Events | CFS IP 不可达（VPC 路由问题或安全组限制） | `tccli cfs DescribeMountTargets --region <REGION> --FileSystemId <CFS_ID>` 获取挂载点 IP，确认集群节点能访问 |

### 权限组问题

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| CFS 实例创建成功但无法挂载 | `tccli cfs DescribeCfsPGroups --region <REGION>` 查看权限组规则 | 权限组未添加来访客户端 IP 或读写权限不足 | 在控制台 [权限组](https://console.cloud.tencent.com/cfs/fs/pgroup) 中添加集群节点 IP 段的访问规则 |

## 下一步

- [PV 和 PVC 管理文件存储](../PV%20和%20PVC%20管理文件存储/tccli%20操作.md)
- [PV 和 PVC 的绑定规则](../../PV%20和%20PVC%20的绑定规则/tccli%20操作.md)
- [文件存储使用说明](../文件存储使用说明/tccli%20操作.md)
- [使用云硬盘 CBS](../../使用云硬盘%20CBS/StorageClass%20管理云硬盘模板/tccli%20操作.md)

## 控制台替代

[控制台 → 集群 → 存储 → StorageClass](https://console.cloud.tencent.com/tke2/cluster) 创建 StorageClass，选择 CFS 模板。
