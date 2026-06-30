# StorageClass 管理文件存储模板

> 对照官方：[StorageClass 管理文件存储模板](https://cloud.tencent.com/document/product/457/44235) · page_id `44235`

## 概述

通过创建 CFS 类型的 StorageClass，实现 PVC 动态创建 CFS 文件系统并绑定到 Pod。StorageClass 中指定 VPC、子网和 CFS 协议版本，PVC 创建时自动在该 VPC/子网下新建 CFS 实例。

**与静态 PV 对比**：

| 维度 | 动态创建（StorageClass） | 静态创建（已有 CFS） |
|------|---------------------|----------------|
| CFS 实例 | PVC 创建时自动新建 | 需提前创建 |
| 管控粒度 | StorageClass 统一模板 | 每个 PV 独立配置 |
| 适合场景 | 按需自动供给、多租户 | 存量 CFS 复用、精确控制 |

## 前置条件

- [环境准备](../../../../环境准备.md)
- 已安装 CFS-CSI 组件：[文件存储使用说明](../文件存储使用说明/tccli%20操作.md)
- kubectl 已连接目标集群

### 环境检查

```bash
# 1. 确认 CFS-CSI 已安装
tccli tke DescribeAddon --region <Region> --ClusterId CLUSTER_ID --AddonName CFS
# expected: Phase: "Succeeded", Status: "Running"

# 2. 确认 kubectl 可达
kubectl get ns
# expected: 返回命名空间列表

# 3. 查询集群 VPC 和子网（CFS 创建时使用）
tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
# expected: 记录 ClusterNetworkSettings.VpcId，以及 VPC 下可用子网
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

### 资源检查

```bash
# 4. 确认目标子网可用 IP 充足
tccli vpc DescribeSubnets --region <Region> --Filters '[{"Name":"vpc-id","Values":["VPC_ID"]}]'
# expected: 返回子网列表，AvailableIpAddressCount ≥ 1
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli/kubectl 命令 | 幂等 |
|-----------|-------------------|:--:|
| 安装 CFS-CSI | `InstallAddon --AddonName CFS` | 否 |
| 创建 StorageClass | `kubectl apply -f cfs-sc.yaml` | 否 |
| 创建 PVC | `kubectl apply -f cfs-pvc.yaml` | 否 |

## 关键字段说明

以下说明 CFS StorageClass YAML 中的关键参数。

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `provisioner` | String | 是 | 固定值 `com.tencent.cloud.csi.cfs` | 填写其他值 → PVC 一直 Pending，Events 报 "provisioner not found" |
| `parameters.vpcid` | String | 是 | 集群所在 VPC ID，格式 `vpc-xxxxxxxx` | VPC 错误 → CFS 创建在错误的 VPC，Pod 无法挂载 |
| `parameters.subnetid` | String | 是 | VPC 内子网 ID，格式 `subnet-xxxxxxxx` | 子网不可用 → CFS 创建失败 |
| `parameters.vers` | String | 否 | CFS 协议版本：`"3.0"`（NFSv3）或 `"4.0"`（NFSv4） | 填 `"3"`（不带 .0）→ CFS 创建使用默认值 NFSv3（但不推荐依赖默认值） |
| `parameters.resourcetags` | String | 否 | CFS 资源标签，格式 `"key1=val1&key2=val2"` | 标签格式错误 → CFS 创建失败 |
| `reclaimPolicy` | String | 否 | `Delete`（PVC 删则 CFS 删）或 `Retain`（保留 CFS） | `Delete` 时误删 PVC → CFS 数据丢失 |
| `volumeBindingMode` | String | 否 | `Immediate`（立即创建）或 `WaitForFirstConsumer` | `Immediate` 在 Pod 未调度时即创建 CFS，可能产生闲置费用 |

## 操作步骤

### 步骤 1：创建 StorageClass

#### 选择依据

- **provisioner**：CSF-CSI 注册的驱动名为 `com.tencent.cloud.csi.cfs`，不可更改。
- **reclaimPolicy**：推荐 `Delete`（测试环境）或 `Retain`（生产环境）。`Retain` 时 PVC 删除后 CFS 实例保留，避免误删数据。
- **volumeBindingMode**：推荐 `Immediate`（CFS 不绑定节点，可跨可用区）。
- **vers**：推荐 `"3.0"`（NFSv3），兼容性最好。

#### 最小配置

`cfs-sc.yaml`：

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: cfs-standard
parameters:
  vpcid: VPC_ID
  subnetid: SUBNET_ID
  vers: "3.0"
provisioner: com.tencent.cloud.csi.cfs
reclaimPolicy: Delete
volumeBindingMode: Immediate
```

```bash
kubectl apply -f cfs-sc.yaml
# expected: storageclass.storage.k8s.io/cfs-standard created
```

| 占位符 | 说明 | 获取方式 |
|--------|------|---------|
| `VPC_ID` | 集群所在 VPC ID | `tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'` 查看 `ClusterNetworkSettings.VpcId` |
| `SUBNET_ID` | VPC 内可用子网 ID | `tccli vpc DescribeSubnets --region <Region>` 筛选同 VPC 的子网 |

### 步骤 2：创建 PVC 自动供应 CFS

#### 选择依据

- **accessModes**：`ReadWriteMany`，CFS 的核心优势之一。
- **storage**：CFS 按实际使用量计费，PVC 中 `storage` 仅作容量声明，不影响 CFS 实际容量限制。

`cfs-pvc.yaml`：

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cfs-pvc-dynamic
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
  storageClassName: cfs-standard
```

```bash
kubectl apply -f cfs-pvc.yaml
# expected: persistentvolumeclaim/cfs-pvc-dynamic created
```

### 步骤 3：验证 PV/PVC 绑定

```bash
kubectl get pvc cfs-pvc-dynamic
# expected: STATUS: Bound

kubectl get pv | grep cfs-pvc-dynamic
# expected: 自动创建 PV，STATUS: Bound
```

```text
NAME  STATUS  AGE
...
```

## 验证

| 维度 | 命令 | 预期 |
|------|------|------|
| StorageClass 存在 | `kubectl get sc cfs-standard` | NAME: `cfs-standard`，PROVISIONER: `com.tencent.cloud.csi.cfs` |
| PVC 绑定 | `kubectl get pvc cfs-pvc-dynamic` | STATUS: `Bound` |
| PV 自动创建 | `kubectl get pv \| grep cfs-pvc-dynamic` | PV 存在且 STATUS: `Bound` |
| CFS 实例创建 | `tccli cfs DescribeCfsFileSystems --region <Region>` | 出现与 PVC 对应的新 CFS 实例，LifeCycleState: `available` |

## 清理

> **警告**：`reclaimPolicy: Delete` 时，删除 PVC 会同步删除关联的 CFS 文件系统及其中所有数据。**生产环境建议使用 `Retain`**，避免误删数据。

### 1. 清理前状态检查

```bash
kubectl get pvc cfs-pvc-dynamic
kubectl get pv | grep cfs-pvc-dynamic
# 确认是待删除的目标资源，确认 reclaimPolicy
```

```text
NAME  STATUS  AGE
...
```

### 2. 删除 PVC

```bash
kubectl delete pvc cfs-pvc-dynamic
# expected: persistentvolumeclaim "cfs-pvc-dynamic" deleted
# reclaimPolicy: Delete → PV 和 CFS 实例同步删除
# reclaimPolicy: Retain → PV 保留（Released 状态），CFS 实例保留
```

### 3. 删除 StorageClass（可选）

```bash
kubectl delete sc cfs-standard
# expected: storageclass.storage.k8s.io "cfs-standard" deleted
```

### 4. 验证已删除

```bash
kubectl get pvc cfs-pvc-dynamic 2>&1
# expected: Error from server (NotFound)

# reclaimPolicy: Delete 时确认 CFS 也已删除
tccli cfs DescribeCfsFileSystems --region <Region>
# expected: 原 CFS 实例不再出现
```

```text
NAME  STATUS  AGE
...
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl apply -f cfs-sc.yaml` 报 "provisioner not found" | `kubectl get csidriver` 查看已注册 CSI 驱动 | CFS-CSI 组件未安装 | 参考 [文件存储使用说明](../文件存储使用说明/tccli%20操作.md) 安装 CFS-CSI |

### PVC 异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| PVC 一直 Pending | `kubectl describe pvc cfs-pvc-dynamic` 查看 Events | Provisioner 无法在指定 VPC/子网创建 CFS：`vpcid`/`subnetid` 不存在或不可用 | `tccli vpc DescribeSubnets --region <Region>` 确认子网存在且 AvailableIpCount > 0 |
| PVC Pending，Events 报 "subnet not found" | `tccli vpc DescribeSubnets --region <Region> --SubnetIds '["SUBNET_ID"]'` | 子网 ID 填错或子网已被删除 | 从 `DescribeSubnets` 输出取正确子网 ID，修改 `cfs-sc.yaml` 后重新 apply |
| PVC Bound 但 Pod 挂载失败 | `kubectl describe pod POD_NAME` 查看 Events | CFS IP 不可达（VPC 路由问题或安全组限制） | 确认集群节点能访问 CFS 挂载点 IP：`tccli cfs DescribeMountTargets --region <Region> --FileSystemId CFS_ID` 获取 IP |

## 下一步

- [PV 和 PVC 管理文件存储](../PV和PVC管理文件存储/tccli%20操作.md)
- [PV 和 PVC 的绑定规则](../../PV和PVC的绑定规则/tccli%20操作.md)
- [使用云硬盘 CBS](../../使用云硬盘CBS/StorageClass管理云硬盘模板/tccli%20操作.md)

## 控制台替代

[控制台 → 集群 → 存储 → StorageClass](https://console.cloud.tencent.com/tke2/cluster) 创建 StorageClass，选 CFS 模板。
