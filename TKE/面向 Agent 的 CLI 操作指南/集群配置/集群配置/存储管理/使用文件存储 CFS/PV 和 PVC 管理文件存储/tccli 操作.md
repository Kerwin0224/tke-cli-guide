# PV 和 PVC 管理文件存储

> 对照官方：[PV 和 PVC 管理文件存储](https://cloud.tencent.com/document/product/457/44236) · page_id `44236`

## 概述

通过静态创建 PV 的方式使用已有 CFS 文件存储。适用于已有存量文件存储并在集群内使用的场景。

## 前置条件

- [环境准备](../../../环境准备.md)
- 已安装 CFS-CSI 组件（addon: `cfs`）
- 已在文件存储控制台创建 CFS 实例（与集群同 VPC）
- 已获取文件系统挂载点 IP、子目录路径和 FSID
- kubectl 可达集群以 apply YAML

> **kubectl 可达性说明**：当前共享集群 kubectl 不可达。以下 PV/PVC 创建步骤 YAML 完整，但 `kubectl apply` 无法真跑验证。

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 创建 CFS 实例 | `tccli cfs CreateCfsFileSystem` | 否 |
| 查询 CFS 实例 | `tccli cfs DescribeCfsFileSystems` | 是 |
| 创建 PV（静态） | `kubectl apply -f <yaml>` | 否（名称冲突） |
| 创建 PVC | `kubectl apply -f <yaml>` | 否（名称冲突） |
| 创建 Deployment 挂载 PVC | `kubectl apply -f <yaml>` | 是 |

## 操作步骤

### 创建 CFS 文件系统（tccli）

```bash
tccli cfs CreateCfsFileSystem --region <Region> \
    --Zone <ZONE> \
    --NetInterface vpc \
    --VpcId vpc-example \
    --SubnetId subnet-example \
    --PGroupId pgroup-example \
    --FsName cfs-test \
    --StorageType SD \
    --Protocol NFS
# expected: exit 0，返回 FileSystemId 和 RequestId
```

### 获取挂载信息

```bash
tccli cfs DescribeCfsFileSystems --region <Region> \
    --FileSystemId cfs-example
# expected: 返回文件系统详情，含 IpAddress（挂载点 IP）和 FsId（FSID）
```

### 创建 PV（静态）

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: cfs-pv
spec:
  accessModes:
  - ReadWriteMany
  capacity:
    storage: 10Gi
  csi:
    driver: com.tencent.cloud.csi.cfs
    volumeAttributes:
      fsid: <fsid>
      host: <mount-ip>
      path: /<sub-path>
      vers: "3"
    volumeHandle: cfs-pv
  persistentVolumeReclaimPolicy: Retain
  storageClassName: cfs
  volumeMode: Filesystem
```

PV 参数说明：

| 参数 | 必填 | 描述 |
|------|:--:|------|
| `fsid` | NFS v3 必填 | 文件系统 fsid，可在挂载点信息中查看。`vers: "4"` 无需此参数 |
| `host` | 是 | 文件系统挂载点 IP 地址 |
| `path` | 是 | 文件系统子目录，挂载后无法访问该子目录的上层目录 |
| `vers` | 是 | `"3"`（推荐）或 `"4"` |

```bash
kubectl apply -f cfs-pv.yaml
# expected: persistentvolume/cfs-pv created
```

### 创建 PVC 绑定 PV

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
  storageClassName: cfs
  volumeMode: Filesystem
  volumeName: cfs-pv
```

```bash
kubectl apply -f cfs-pvc.yaml
# expected: persistentvolumeclaim/cfs-pvc created
```

### 创建 Deployment 使用 PVC

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-cfs
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-cfs
  template:
    metadata:
      labels:
        app: nginx-cfs
    spec:
      containers:
      - image: nginx
        name: nginx
        volumeMounts:
        - mountPath: /usr/share/nginx/html
          name: cfs-vol
      volumes:
      - name: cfs-vol
        persistentVolumeClaim:
          claimName: cfs-pvc
```

```bash
kubectl apply -f nginx-cfs-deployment.yaml
# expected: deployment.apps/nginx-cfs created
```

## 验证

```bash
kubectl get pv cfs-pv
kubectl get pvc cfs-pvc -n default
kubectl get deploy nginx-cfs
# expected: PV Status Bound，PVC Status Bound
```

```text
NAME  STATUS  AGE
...
```

## 清理

```bash
kubectl delete deploy nginx-cfs
kubectl delete pvc cfs-pvc -n default
kubectl delete pv cfs-pv
# CFS 实例不会随 PV 删除而删除（回收策略为 Retain），需手动删除
tccli cfs DeleteCfsFileSystem --region <Region> --FileSystemId cfs-example
```

## 排障

| 现象 | 根因 | 修复 |
|------|------|------|
| PV 挂载失败，报 fsid 错误 | NFS v3 未指定 fsid | 在 `volumeAttributes` 中添加 `fsid: <正确的fsid>` |
| Released 状态 PV 无法绑定 | PV YAML 中存在残留 `claimRef` 字段 | `kubectl edit pv cfs-pv` 删除 `spec.claimRef` |

## 下一步

- [StorageClass 管理文件存储模板](../StorageClass 管理文件存储模板/tccli 操作.md)
- [文件存储使用说明](../文件存储使用说明/tccli 操作.md)

## 控制台替代

[TKE 控制台 → 集群详情 → 存储 → PersistentVolume](https://console.cloud.tencent.com/tke2/cluster)：新建 → 来源设置"静态创建" → 选择"文件存储 CFS" Provisioner。
