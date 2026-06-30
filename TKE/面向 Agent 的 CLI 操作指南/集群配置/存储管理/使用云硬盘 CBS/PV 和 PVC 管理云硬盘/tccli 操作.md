# PV 和 PVC 管理云硬盘

> 对照官方：[PV 和 PVC 管理云硬盘](https://cloud.tencent.com/document/product/457/44240) · page_id `44240` · tccli ≥3.1.107 · API 2017-03-12

## 概述

通过 PersistentVolume（PV）和 PersistentVolumeClaim（PVC）管理 CBS 云硬盘。支持两种创建方式：

| 方式 | 说明 | 适用场景 |
|------|------|---------|
| **动态创建**（推荐） | 创建 PVC 时由 CBS-CSI 自动创建 PV 和云硬盘，无需预先创建 CBS 云盘 | 日常使用，自动管理云盘生命周期 |
| **静态创建** | 先手动创建 CBS 云硬盘，再创建 PV 绑定云盘，最后 PVC 绑定 PV | 已有存量云盘需在集群内使用的场景 |

**关键限制**：

- CBS 仅支持 **ReadWriteOnce**，一个云硬盘同时只能挂载到一个节点的一个 Pod。
- 云硬盘**不支持跨可用区挂载**，Pod 所在节点必须与云硬盘在同一可用区。
- 云硬盘大小必须是 10 的倍数。高性能云硬盘最小 10GB，SSD 和增强型 SSD 最小 20GB。

## 前置条件

- [环境准备](../../../../环境准备.md)
- 已安装 CBS-CSI 组件（集群创建后通常已默认安装）
- kubectl 已连接目标集群

### 环境检查

```bash
# 1. 确认 CBS-CSI 组件已安装并运行正常
tccli tke DescribeAddon --region <REGION> --ClusterId <CLUSTER_ID> --AddonName CBS
# expected: Phase: "Succeeded"

# 2. 检查默认 StorageClass
kubectl get sc cbs
# expected: 显示 cbs StorageClass，PROVISIONER: com.tencent.cloud.csi.cbs

# 3. 确认 kubectl 可连接集群
kubectl get ns
# expected: 返回包含 default、kube-system 等命名空间列表
```

## 控制台与 CLI 参数映射

| 控制台操作 | kubectl 命令 | 幂等 |
|-----------|-------------|:--:|
| 创建 PVC（动态） | `kubectl apply -f pvc.yaml` | 是（同名覆盖） |
| 创建 PV（静态） | `kubectl apply -f pv.yaml` | 是（同名覆盖） |
| 查看 PV 列表 | `kubectl get pv` | 是 |
| 查看 PVC 列表 | `kubectl get pvc` | 是 |
| 查看 PV 详情 | `kubectl describe pv <PV_NAME>` | 是 |
| 删除 PVC | `kubectl delete pvc <PVC_NAME>` | 是 |
| 查询云硬盘 | `tccli cbs DescribeDisks --region <REGION>` | 是 |

## 关键字段说明

### PVC YAML 关键字段（动态创建）

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `spec.storageClassName` | String | 否 | `cbs`（默认 CBS StorageClass），省略则使用默认 StorageClass | 指定不存在的 StorageClass → PVC 一直 Pending |
| `spec.accessModes` | []String | 是 | CBS 仅支持 `ReadWriteOnce` | 填其他值 → 创建失败 |
| `spec.resources.requests.storage` | String | 是 | 如 `10Gi`。大小必须为 10 的倍数，最小 10Gi | 填 11Gi 等非 10 倍数值 → 创建失败 |

### PV YAML 关键字段（静态创建）

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `spec.csi.driver` | String | 是 | 固定值 `com.tencent.cloud.csi.cbs` | 填错 → PV 无法被 CBS-CSI 识别 |
| `spec.csi.fsType` | String | 否 | `ext4`（默认）或 `xfs` | 类型与云盘现有文件系统不匹配 → 挂载失败 |
| `spec.csi.volumeHandle` | String | 是 | CBS 云硬盘 ID，格式 `disk-xxxxxxxx` | ID 错误或不存在 → 挂载失败 |
| `spec.persistentVolumeReclaimPolicy` | String | 否 | `Retain`（保留云盘）或 `Delete`（PVC 删则云盘删） | `Delete` 误删 PVC → 云盘数据丢失 |
| `spec.storageClassName` | String | 是 | 静态 PV 设为 `""`（空字符串） | 设非空值 → 可能被 StorageClass 匹配导致意外行为 |

## 操作步骤

### 方式一：动态创建 PVC（推荐）

创建 PVC 时，CBS-CSI 自动创建 CBS 云硬盘和对应的 PV，无需预先准备云硬盘。

#### 选择依据

- **storageClassName**：使用集群默认的 `cbs` StorageClass（或省略此字段自动使用默认）。
- **storage 大小**：最小 10Gi，大小必须是 10 的倍数。测试场景选最小规格以控制费用。
- **accessModes**：CBS 仅支持 `ReadWriteOnce`。

#### 第 1 步：创建 PVC

`pvc-dynamic.yaml`：

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: <PVC_NAME>
spec:
  storageClassName: cbs
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

```bash
kubectl apply -f pvc-dynamic.yaml
# expected: persistentvolumeclaim/<PVC_NAME> created
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `PVC_NAME` | PVC 名称 | 小写字母、数字、连字符 | 自定义，如 `my-cbs-pvc` |

#### 第 2 步：等待 PVC 绑定

```bash
kubectl get pvc <PVC_NAME>
# expected: STATUS: Bound, CAPACITY: 10Gi
```

**预期输出**：

```text
NAME         STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
my-cbs-pvc   Bound    pvc-example-xxxx                           10Gi       RWO            cbs            15s
```

PVC 状态变为 `Bound` 表示云硬盘已创建并绑定成功。同时 CBS-CSI 自动创建了对应的 PV。

#### 第 3 步：查看自动创建的 PV

```bash
kubectl get pv | grep <PVC_NAME>
# expected: 显示自动创建的 PV，RECLAIM POLICY: Delete，STATUS: Bound
```

**预期输出**：

```text
pvc-example-xxxx   10Gi  RWO  Delete  Bound  default/my-cbs-pvc  cbs  15s
```

#### 第 4 步：查看 PV 详情（获取云硬盘 ID）

```bash
kubectl describe pv <PV_NAME>
# expected: VolumeHandle 字段显示 CBS 云硬盘 ID（disk-xxxxxxxx）
```

**预期输出**（关键字段）：

```text
Source:
    Type:              CSI
    Driver:            com.tencent.cloud.csi.cbs
    FSType:            ext4
    VolumeHandle:      disk-example
```

#### 第 5 步：通过 tccli 验证云硬盘

```bash
tccli cbs DescribeDisks --region <REGION> --DiskIds '["<DISK_ID>"]'
# expected: DiskState: "UNATTACHED"，DiskSize: 10
```

**预期输出**：

```json
{
    "DiskSet": [{
        "DiskId": "disk-example",
        "DiskName": "cls-example/pvc-example-xxxx",
        "DiskSize": 10,
        "DiskState": "UNATTACHED",
        "DiskType": "CLOUD_PREMIUM"
    }]
}
```

云硬盘处于 `UNATTACHED` 状态——尚未挂载到任何节点。被 Pod 使用时状态会变为 `ATTACHED`。

### 方式二：静态创建 PV（已有云硬盘）

适用于已有存量 CBS 云硬盘需要在集群内使用的场景。

#### 选择依据

- **persistentVolumeReclaimPolicy**：`Retain`（推荐），删除 PVC 时保留云盘及数据。`Delete` 会同步删除云盘。
- **storageClassName**：`""`（空字符串），避免与动态 StorageClass 冲突。
- **csi.fsType**：`ext4`，Linux 标准文件系统，兼容性最好。

#### 第 1 步：查询可用云硬盘

```bash
tccli cbs DescribeDisks --region <REGION>
# expected: 返回 DiskSet，找出 DiskState 为 "UNATTACHED" 的目标云盘
```

**预期输出**：

```json
{
    "DiskSet": [{
        "DiskId": "disk-example01",
        "DiskSize": 20,
        "DiskType": "CLOUD_PREMIUM",
        "DiskState": "UNATTACHED",
        "Placement": {"Zone": "ap-guangzhou-3"}
    }]
}
```

#### 第 2 步：创建 PV

`pv-static.yaml`：

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: <PV_NAME>
spec:
  accessModes:
    - ReadWriteOnce
  capacity:
    storage: 20Gi
  csi:
    driver: com.tencent.cloud.csi.cbs
    fsType: ext4
    volumeHandle: <DISK_ID>
  persistentVolumeReclaimPolicy: Retain
  storageClassName: ""
```

```bash
kubectl apply -f pv-static.yaml
# expected: persistentvolume/<PV_NAME> created
```

| 占位符 | 说明 | 获取方式 |
|--------|------|---------|
| `PV_NAME` | PV 名称 | 自定义，如 `cbs-pv-static` |
| `DISK_ID` | CBS 云硬盘 ID，格式 `disk-xxxxxxxx` | `tccli cbs DescribeDisks --region <REGION>` 输出中 `DiskId` 字段 |

#### 第 3 步：创建 PVC 绑定

`pvc-static.yaml`：

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: <PVC_NAME>
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
  storageClassName: ""
  volumeName: <PV_NAME>
```

```bash
kubectl apply -f pvc-static.yaml
# expected: persistentvolumeclaim/<PVC_NAME> created
```

### 使用 PVC 挂载到 Deployment

PVC 绑定后，可创建 Deployment 加载 PVC 作为数据卷。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: <DEPLOY_NAME>
spec:
  replicas: 1
  selector:
    matchLabels:
      app: <APP_LABEL>
  template:
    metadata:
      labels:
        app: <APP_LABEL>
    spec:
      containers:
        - image: nginx:alpine
          name: nginx
          volumeMounts:
            - mountPath: "/data"
              name: data-vol
      volumes:
        - name: data-vol
          persistentVolumeClaim:
            claimName: <PVC_NAME>
```

```bash
kubectl apply -f deploy-with-pvc.yaml
# expected: deployment.apps/<DEPLOY_NAME> created

kubectl get pods -l app=<APP_LABEL>
# expected: READY: 1/1, STATUS: Running
```

| 占位符 | 说明 |
|--------|------|
| `DEPLOY_NAME` | Deployment 名称 |
| `APP_LABEL` | 应用标签，用于选择器匹配 |
| `PVC_NAME` | 已创建的 PVC 名称 |

## 验证

### 数据面（kubectl）

```bash
# 验证 PV 状态
kubectl get pv <PV_NAME>
# expected: STATUS: Bound

# 验证 PVC 状态
kubectl get pvc <PVC_NAME>
# expected: STATUS: Bound

# 验证 PVC 详情
kubectl describe pvc <PVC_NAME>
# expected: 显示 Type: CSI, Driver: com.tencent.cloud.csi.cbs
```

### 控制面（tccli）

| 维度 | 命令 | 预期 |
|------|------|------|
| 云盘状态 | `tccli cbs DescribeDisks --region <REGION> --DiskIds '["<DISK_ID>"]'` | `DiskState: "ATTACHED"`（如已挂载 Pod），或 `"UNATTACHED"`（PVC 绑定但未挂载 Pod） |
| 云盘大小 | 同上 | `DiskSize` 与 PVC 声明的 `storage` 一致 |

## 清理

> **警告**：动态创建的 PVC（StorageClass 未显式指定 `reclaimPolicy`）默认使用 `Delete` 策略——删除 PVC 会**同步删除 CBS 云硬盘及所有数据**，不可恢复。
>
> 静态创建且 `persistentVolumeReclaimPolicy: Retain` 时，删除 PVC/PV 后云硬盘**保留且持续计费**，需手动退还。

### 1. 清理前状态检查

```bash
# 检查 PVC 状态和关联的 PV
kubectl get pvc <PVC_NAME>
kubectl get pv | grep <PVC_NAME>
# 确认 reclaimPolicy 和绑定状态
```

### 2. 删除使用 PVC 的 Deployment（如有）

```bash
kubectl delete deploy <DEPLOY_NAME>
# expected: deployment.apps "<DEPLOY_NAME>" deleted
```

### 3. 删除 PVC

```bash
kubectl delete pvc <PVC_NAME>
# expected: persistentvolumeclaim "<PVC_NAME>" deleted
```

动态创建的 PVC 删除后，CBS-CSI 会自动删除关联的 PV 和 CBS 云硬盘。

### 4. 删除 PV（静态创建时，Retain 策略）

```bash
kubectl delete pv <PV_NAME>
# expected: persistentvolume "<PV_NAME>" deleted
```

### 5. 验证清理结果

```bash
# 验证 PVC 已删除
kubectl get pvc <PVC_NAME> 2>&1
# expected: Error from server (NotFound)

# 验证 PV 已删除（动态创建时）
kubectl get pv <PV_NAME> 2>&1
# expected: Error from server (NotFound)

# 验证云硬盘已删除（动态创建 Delete 策略）
tccli cbs DescribeDisks --region <REGION> --DiskIds '["<DISK_ID>"]' 2>&1
# expected: TotalCount: 0（云硬盘已自动退还）
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| PVC 创建后一直 `Pending` | `kubectl describe pvc <PVC_NAME>` 查看 Events | CBS-CSI 组件未安装或未运行 | `tccli tke DescribeAddon --region <REGION> --ClusterId <CLUSTER_ID> --AddonName CBS` 检查并安装 CBS-CSI 组件 |
| `kubectl apply -f pv.yaml` 报驱动未找到 | `kubectl get csidriver \| grep cbs` | CBS CSI Driver 未注册 | 确认 CBS-CSI 组件正常运行：`kubectl get pods -n kube-system \| grep csi-cbs` |
| PVC 大小报错 | 检查 PVC YAML 中的 `storage` 值 | 大小不是 10 的倍数或低于最小限制 | 改为 10Gi、20Gi 等 10 倍数值。高性能云硬盘最小 10Gi，SSD 最小 20Gi |

### PVC 绑定后 Pod 异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Pod 挂载失败（`Multi-Attach` 错误） | `kubectl describe pod <POD_NAME>` 查看 Events | 同一 PVC 被多个 Pod 挂载（不同节点） | CBS 仅支持 ReadWriteOnce。确保只有一个 Pod 使用该 PVC |
| Pod 挂载失败（可用区不匹配） | PVC 详情中 `topology.com.tencent.cloud.csi.cbs/zone` 与 Pod 所在节点 zone 对比 | 云盘可用区与 Pod 节点可用区不一致 | 删除 Pod 使其重新调度到与云盘同可用区的节点，或使用 nodeSelector 指定 zone |
| Pod `Pending` 调度失败 | `kubectl describe pod <POD_NAME>` 查看 Events | 调度器提示 `volume node affinity conflict`：节点可用区与云盘不匹配 | 确认集群节点分布，云盘创建在哪个可用区则 Pod 需调度到同可用区 |

## 下一步

- [StorageClass 管理云硬盘模板](../StorageClass%20管理云硬盘模板/tccli%20操作.md)
- [PV 和 PVC 的绑定规则](../../PV%20和%20PVC%20的绑定规则/tccli%20操作.md)
- [使用文件存储 CFS](../../使用文件存储%20CFS/PV%20和%20PVC%20管理文件存储/tccli%20操作.md)

## 控制台替代

[控制台 → 集群 → 存储 → PersistentVolumeClaim](https://console.cloud.tencent.com/tke2/cluster) 新建 PVC，系统自动创建对应 PV 和 CBS 云硬盘。
