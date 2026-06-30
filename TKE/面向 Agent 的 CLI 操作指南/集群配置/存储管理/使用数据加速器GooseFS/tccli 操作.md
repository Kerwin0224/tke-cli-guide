# 使用数据加速器 GooseFS

> 对照官方：[使用数据加速器 GooseFS](https://cloud.tencent.com/document/product/457/116405) · page_id `116405`

## 概述

GooseFS 是腾讯云的数据加速器，通过 GooseFS-CSI 组件为 TKE 集群提供高性能分布式缓存层，加速 AI 训练、大数据分析等数据密集型应用的数据读取。

**使用方式**：安装 GooseFS-CSI 组件后，通过 StorageClass 动态创建 PVC，Pod 挂载后即可通过 GooseFS 分布式缓存访问底层存储（COS、HDFS 等）。

## 前置条件

- [环境准备](../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:InstallAddon, tke:DescribeAddon, tke:DescribeAddonValues
#    goosefs:DescribeFileSystems, goosefs:DescribeNamespaces
# 验证：执行 DescribeClusters 确认 TKE 权限
tccli tke DescribeClusters --region <Region>
# expected: exit 0，返回集群列表（可为空）
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
# 4. 确认目标集群存在且状态正常
tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
# expected: TotalCount >= 1，ClusterStatus: "Running"

# 5. 确认已开通 GooseFS 服务（如已开通则在 Describe 中可见）
tccli goosefs DescribeFileSystems --region <Region>
# expected: exit 0，返回 GooseFS 集群列表（可为空）
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

### 版本与规格选择

- GooseFS-CSI 组件版本：通过 `tccli tke DescribeAddonValues --region <Region> --ClusterId CLUSTER_ID --AddonName GooseFS` 查询可用版本。

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 安装 GooseFS-CSI 组件 | `InstallAddon --AddonName GooseFS` | 否 |
| 查询组件安装状态 | `DescribeAddon --AddonName GooseFS` | 是 |
| 创建 StorageClass | `kubectl apply -f goosefs-sc.yaml` | 否 |
| 创建 PVC | `kubectl apply -f goosefs-pvc.yaml` | 否 |

## 关键字段说明

以下说明 `InstallAddon` 中与 GooseFS-CSI 相关的关键参数。

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `AddonName` | String | 是 | 固定值 `GooseFS` | 填小写或其他名称 → `InvalidParameter.AddonName` |
| `AddonVersion` | String | 是 | 组件版本号，`DescribeAddonValues` 查询 | 版本不存在 → "addon version not found" |
| `ClusterId` | String | 是 | 目标集群 ID，格式 `cls-xxxxxxxx` | 集群不存在 → `ResourceNotFound.ClusterNotFound` |

## 操作步骤

### 步骤 1：安装 GooseFS-CSI 组件

#### 选择依据

- **AddonName**：固定为 `GooseFS`，对应 CSI 驱动 `com.tencent.cloud.csi.goosefs`。
- **AddonVersion**：通过 `DescribeAddonValues` 查询当前集群可用版本。

```bash
# 查询可用版本
tccli tke DescribeAddonValues --region <Region> --ClusterId CLUSTER_ID --AddonName GooseFS
# expected: 返回可用版本列表
```

```json
{
  "Values": "<Values>",
  "DefaultValues": "<DefaultValues>",
  "RequestId": "<RequestId>"
}
```

#### 安装命令

```bash
tccli tke InstallAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName GooseFS \
    --AddonVersion ADDON_VERSION
# expected: exit 0，返回 RequestId
```

**预期输出**：

```json
{
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

| 占位符 | 说明 | 获取方式 |
|--------|------|---------|
| `ADDON_VERSION` | GooseFS-CSI 组件版本 | `tccli tke DescribeAddonValues --region <Region> --ClusterId CLUSTER_ID --AddonName GooseFS` |

### 步骤 2：轮询组件安装状态

```bash
tccli tke DescribeAddon --region <Region> --ClusterId CLUSTER_ID --AddonName GooseFS
# expected: Phase: "Succeeded", Status: "Running"
```

**预期输出**：

```json
{
    "Addon": {
        "AddonName": "GooseFS",
        "AddonVersion": "1.0.0",
        "Phase": "Succeeded",
        "Status": "Running"
    }
}
```

### 步骤 3：创建 StorageClass

#### 选择依据

- **provisioner**：`com.tencent.cloud.csi.goosefs`，GooseFS-CSI 注册的 CSI 驱动名。
- **reclaimPolicy**：`Delete`，PVC 删除时对应 GooseFS 命名空间缓存也清理。
- **goosefsNamespace**：GooseFS 中用于隔离不同租户缓存空间的命名空间标识。

`goosefs-sc.yaml`：

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: goosefs
parameters:
  goosefsNamespace: GOOSEFS_NAMESPACE
  secretName: GOOSEFS_SECRET
provisioner: com.tencent.cloud.csi.goosefs
reclaimPolicy: Delete
```

```bash
kubectl apply -f goosefs-sc.yaml
# expected: storageclass.storage.k8s.io/goosefs created
```

| 占位符 | 说明 | 获取方式 |
|--------|------|---------|
| `GOOSEFS_NAMESPACE` | GooseFS 命名空间名称 | GooseFS 控制台或 `goosefs DescribeNamespaces` 查询 |
| `GOOSEFS_SECRET` | 访问 GooseFS 的 Kubernetes Secret 名称 | 需预先在集群中创建 |

### 步骤 4：创建 PVC

`goosefs-pvc.yaml`：

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: goosefs-pvc
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
  storageClassName: goosefs
```

```bash
kubectl apply -f goosefs-pvc.yaml
# expected: persistentvolumeclaim/goosefs-pvc created
```

## 验证

### 控制面（tccli）

| 维度 | 命令 | 预期 |
|------|------|------|
| 组件状态 | `tccli tke DescribeAddon --region <Region> --ClusterId CLUSTER_ID --AddonName GooseFS` | `Phase: "Succeeded"`, `Status: "Running"` |
| GooseFS 服务 | `tccli goosefs DescribeFileSystems --region <Region>` | 返回 GooseFS 集群列表 |

### 数据面（kubectl）

```bash
kubectl get sc goosefs
# expected: NAME: goosefs, PROVISIONER: com.tencent.cloud.csi.goosefs

kubectl get pvc goosefs-pvc
# expected: STATUS: Bound

kubectl get pods -n kube-system | grep goosefs
# expected: goosefs-csi 相关 Pod 均为 Running
```

```text
NAME  STATUS  AGE
...
```

## 清理

> **警告**：`reclaimPolicy: Delete` 时，删除 PVC 会同步清理 GooseFS 中对应命名空间的缓存数据。如保留缓存数据，请将 StorageClass 的 `reclaimPolicy` 改为 `Retain`。

### 1. 清理前状态检查

```bash
kubectl get pvc goosefs-pvc
kubectl get pv | grep goosefs
# 确认是待删除的目标资源
```

```text
NAME  STATUS  AGE
...
```

### 2. 删除 PVC 和关联 PV

```bash
kubectl delete pvc goosefs-pvc
# expected: persistentvolumeclaim "goosefs-pvc" deleted
# reclaimPolicy: Delete 时，关联 PV 同步删除
```

### 3. 删除 StorageClass（可选）

```bash
kubectl delete sc goosefs
# expected: storageclass.storage.k8s.io "goosefs" deleted
```

### 4. 验证已删除

```bash
kubectl get pvc goosefs-pvc 2>&1
# expected: Error from server (NotFound): ... "goosefs-pvc" not found

kubectl get sc goosefs 2>&1
# expected: Error from server (NotFound): ... "goosefs" not found
```

```text
NAME  STATUS  AGE
...
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `InstallAddon` 返回 "addon version not found" | `tccli tke DescribeAddonValues --region <Region> --ClusterId CLUSTER_ID --AddonName GooseFS` 查看可用版本 | 指定的 `AddonVersion` 不存在 | 从 `DescribeAddonValues` 输出取正确版本号 |
| `InstallAddon` 返回 `InvalidParameter.AddonName` | 检查 `AddonName` 字段 | 填写了非 `GooseFS` 的值 | 使用 `--AddonName GooseFS`（首字母大写驼峰格式） |
| `kubectl apply -f goosefs-sc.yaml` 报 provisioner 未找到 | `kubectl get csidriver` 查看已注册驱动 | GooseFS-CSI 组件未安装或未就绪 | 回到步骤 1-2 安装并确认组件 `Phase: "Succeeded"` |

### 安装成功但 PVC 异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| PVC 一直 Pending | `kubectl describe pvc goosefs-pvc` 查看 Events | Provisioner 无法创建卷，可能是 goosefsNamespace 或 secretName 错误 | 检查 StorageClass 中 `goosefsNamespace` 和 `secretName` 参数是否与 GooseFS 服务端配置一致 |
| PVC Bound 但 Pod 挂载后无法读写 | `kubectl describe pod POD_NAME` 查看 Events | Secret 凭证错误或 GooseFS 命名空间权限不足 | 确认 Secret 中的凭证有对应 GooseFS 命名空间的读写权限 |

## 下一步

- [使用对象存储 COS](../使用对象存储COS/tccli%20操作.md)
- [PV 和 PVC 的绑定规则](../PV和PVC的绑定规则/tccli%20操作.md)
- [使用云硬盘 CBS](../使用云硬盘CBS/PV和PVC管理云硬盘/tccli%20操作.md)

## 控制台替代

[控制台 → 集群 → 组件管理](https://console.cloud.tencent.com/tke2/cluster) 安装 GooseFS-CSI 组件；[GooseFS 控制台](https://console.cloud.tencent.com/goosefs) 管理 GooseFS 集群。
