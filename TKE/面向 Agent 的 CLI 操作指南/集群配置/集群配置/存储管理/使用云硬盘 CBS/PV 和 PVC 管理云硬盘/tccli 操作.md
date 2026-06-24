# PV 和 PVC 管理云硬盘

> 对照官方：[PV 和 PVC 管理云硬盘](https://cloud.tencent.com/document/product/457/44240) · page_id `44240`

## 概述

通过静态创建 PV 的方式使用已有 CBS 云硬盘。适用于已有存量云硬盘并在集群内使用的场景。**注意：云硬盘不支持跨可用区挂载，Pod 迁移至其他可用区将导致挂载失败。**

## 前置条件

- [环境准备](../../../环境准备.md)
- CBS-CSI 组件（addon: `cbs`）为系统默认预装
- 已在云硬盘控制台创建 CBS 云硬盘（与集群同可用区）
- kubectl 可达集群以 apply YAML

> **kubectl 可达性说明**：当前共享集群 kubectl 不可达。以下 YAML 完整，但 `kubectl apply` 无法真跑验证。

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 查询 CBS 云硬盘 | `tccli cbs DescribeDisks` | 是 |
| 创建 PV（静态） | `kubectl apply -f <yaml>` | 否（名称冲突） |
| 创建 PVC | `kubectl apply -f <yaml>` | 否（名称冲突） |
| 创建 Deployment 挂载 PVC | `kubectl apply -f <yaml>` | 是 |

## 操作步骤

### 查询已有 CBS 云硬盘

```bash
tccli cbs DescribeDisks --region <Region> \
    --Filters '[{"Name":"disk-usage","Values":["DATA_DISK"]}]'
# expected: 返回云硬盘列表，含 DiskId、DiskSize、DiskState
```

### 创建 PV（静态）

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: cbs-pv
spec:
  accessModes:
    - ReadWriteOnce
  capacity:
    storage: 10Gi
  csi:
    driver: com.tencent.cloud.csi.cbs
    fsType: ext4
    readOnly: false
    volumeHandle: disk-example
  storageClassName: cbs
  persistentVolumeReclaimPolicy: Retain
  volumeMode: Filesystem
```

```bash
kubectl apply -f cbs-pv.yaml
# expected: persistentvolume/cbs-pv created
```

### 创建 PVC

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: cbs-pvc
  namespace: default
spec:
  storageClassName: cbs
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

```bash
kubectl apply -f cbs-pvc.yaml
# expected: persistentvolumeclaim/cbs-pvc created
```

### 创建 Deployment 使用 PVC

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-cbs
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-cbs
  template:
    metadata:
      labels:
        app: nginx-cbs
    spec:
      containers:
      - image: nginx
        name: nginx
        volumeMounts:
        - mountPath: /opt
          name: pvc-test
      volumes:
      - name: pvc-test
        persistentVolumeClaim:
          claimName: cbs-pvc
```

CBS 数据卷只能挂载到一台 Node 主机上。

```bash
kubectl apply -f nginx-cbs-deployment.yaml
# expected: deployment.apps/nginx-cbs created
```

## 验证

```bash
kubectl get pv cbs-pv
kubectl get pvc cbs-pvc -n default
kubectl get deploy nginx-cbs
# expected: PV Status Bound，PVC Status Bound
```

```text
NAME  STATUS  AGE
...
```

## 清理

```bash
kubectl delete deploy nginx-cbs
kubectl delete pvc cbs-pvc -n default
kubectl delete pv cbs-pv
# CBS 云硬盘不会随 PV 删除而删除（回收策略为 Retain），需手动删除
tccli cbs TerminateDisks --region <Region> --DiskIds '["disk-example"]'
```

## 排障

| 现象 | 根因 | 修复 |
|------|------|------|
| PV 创建失败 | `volumeHandle` 中的 disk ID 不存在或属于其他可用区 | 确认 disk ID 正确，可用区与集群一致 |
| PVC 一直 Pending | PV 与 PVC 的 StorageClass 不匹配 | 确保 PVC 的 `storageClassName` 与 PV 一致 |
| Released 状态 PV 无法绑定 | PV 中存在残留 `claimRef` 字段 | `kubectl edit pv cbs-pv` 删除 `spec.claimRef` |

## 下一步

- [StorageClass 管理云硬盘模板](../StorageClass 管理云硬盘模板/tccli 操作.md)
- [云硬盘使用说明](../云硬盘使用说明/tccli 操作.md)

## 控制台替代

[TKE 控制台 → 集群详情 → 存储 → PersistentVolume](https://console.cloud.tencent.com/tke2/cluster)：新建 → 来源设置"静态创建" → 选择"云硬盘 CBS" → 选择已有云硬盘。
