# StorageClass 管理文件存储模板

> 对照官方：[StorageClass 管理文件存储模板](https://cloud.tencent.com/document/product/457/44235) · page_id `44235`

## 概述

通过创建 CFS 类型 StorageClass，配合 PVC 动态创建文件存储实例。支持两种创建模式：**创建新实例**（每个 PVC 创建一个 CFS 实例）和**共享实例**（多个 PVC 共享同一 CFS 实例的不同子目录）。

## 前置条件

- [环境准备](../../../环境准备.md)
- 已安装 CFS-CSI 组件（addon: `cfs`）
- 已在私有网络下创建子网
- 已创建 CFS 权限组并添加权限组规则
- 已获取文件系统 FSID（NFS v3 挂载需要）
- kubectl 可达集群以 apply YAML

> **kubectl 可达性说明**：当前共享集群 kubectl 不可达。以下 StorageClass/PVC 创建步骤 YAML 完整，但 `kubectl apply` 无法真跑验证。

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 创建 StorageClass | `kubectl apply -f <yaml>` | 否（名称冲突） |
| 创建 PVC | `kubectl apply -f <yaml>` | 否（名称冲突） |
| 创建 Deployment 挂载 PVC | `kubectl apply -f <yaml>` | 是 |

## 操作步骤

### 创建 StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: cfs-sc
parameters:
  pgroupid: pgroup-example
  storagetype: SD
  vpcid: vpc-example
  subnetid: subnet-example
  vers: "3"
  resourcetags: ""
  zone: <REGION>-<ZONE>
provisioner: com.tencent.cloud.csi.cfs
reclaimPolicy: Delete
volumeBindingMode: Immediate
```

Parameters 参数说明：

| 参数 | 必填 | 描述 |
|------|:--:|------|
| `zone` | 是 | 文件存储所在地域可用区 |
| `pgroupid` | 是 | 文件存储所归属的权限组 ID |
| `storagetype` | 是 | `SD`（标准型存储）或 `HP`（性能存储） |
| `vpcid` | 是 | 文件存储所在私有网络 ID |
| `subnetid` | 是 | 文件存储所在子网 ID |
| `vers` | 是 | 协议版本：`"3"`（推荐，NFS v3）或 `"4"`（NFS v4） |
| `subdir-share` | 否 | 填写则启用共享实例模式。provisioner 改为 `com.tencent.cloud.csi.tcfs.<sc-name>` |
| `resourcetags` | 否 | 云标签，多个标签用英文逗号分隔，形如 `"a:b,c:d"` |

```bash
kubectl apply -f cfs-storageclass.yaml
# expected: storageclass.storage.k8s.io/cfs-sc created
```

### 使用指定 StorageClass 创建 PVC

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cfs-pvc
  namespace: default
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
  storageClassName: cfs-sc
  volumeMode: Filesystem
```

CFS PVC 参数说明：

| 参数 | 描述 |
|------|------|
| `spec.accessModes` | 仅支持 `ReadWriteMany`（多机读写） |
| `spec.resources.requests.storage` | 无实际意义，CFS 自动扩展，默认 10Gi |
| `storageClassName` | 指定已创建的 CFS StorageClass |

```bash
kubectl apply -f cfs-pvc.yaml
# expected: persistentvolumeclaim/cfs-pvc created
```

## 验证

```bash
kubectl get sc cfs-sc
kubectl get pvc cfs-pvc -n default
# expected: StorageClass 存在，PVC Status 为 Bound
```

```text
NAME  STATUS  AGE
...
```

## 清理

```bash
kubectl delete pvc cfs-pvc -n default
kubectl delete sc cfs-sc
# 注意：若 reclaimPolicy 为 Delete，删除 PVC 会同步删除 CFS 实例及其数据
```

## 排障

| 现象 | 根因 | 修复 |
|------|------|------|
| PVC 一直 Pending | StorageClass parameters 中权限组、子网等参数错误 | 核对 `pgroupid`、`vpcid`、`subnetid` |
| PVC 创建失败，共享实例模式 | 回收策略需为 Retain | 设置 `reclaimPolicy: Retain` |

## 下一步

- [PV 和 PVC 管理文件存储](../PV 和 PVC 管理文件存储/tccli 操作.md)
- [文件存储使用说明](../文件存储使用说明/tccli 操作.md)

## 控制台替代

[TKE 控制台 → 集群详情 → 存储 → StorageClass](https://console.cloud.tencent.com/tke2/cluster)：新建 → 选择"文件存储 CFS" Provisioner。
