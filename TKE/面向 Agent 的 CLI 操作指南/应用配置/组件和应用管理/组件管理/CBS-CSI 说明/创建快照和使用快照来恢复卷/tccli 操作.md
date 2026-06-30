# 创建快照和使用快照来恢复卷（tccli）

> 对照官方：[创建快照和使用快照来恢复卷](https://cloud.tencent.com/document/product/457/67080) · page_id `67080`

## 概述

通过 CBS-CSI 插件可为 PVC 数据盘创建快照备份数据，或从备份快照恢复数据到新 PVC。快照功能基于 Kubernetes VolumeSnapshot 机制和腾讯云 CBS 快照能力实现。

## 前置条件

- [环境准备](../../../../../环境准备.md)
- 已 [连接集群](../../../../../集群配置/集群管理/连接集群/tccli 操作.md)，kubectl 可正常访问 APIServer
- 已创建 1.18 或以上版本的 TKE 集群
- 已安装最新版的 CBS-CSI 组件
- 集群状态 `Running`

```bash
# 检查集群状态
tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'
# expected: ClusterStatus "Running"
```

- 需要 CAM 权限：`tke:DescribeClusters`、`tke:DescribeAddon`、`tke:InstallAddon`、`tke:UpdateAddon`、`tke:DeleteAddon`

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 查看集群信息 | `tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'` | 是 |
| 查看 CBS-CSI 组件详情 | `tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName cbs-csi` | 是 |
| 创建 VolumeSnapshotClass | `kubectl apply -f volumesnapshotclass.yaml` | 否（同名报错） |
| 创建快照（VolumeSnapshot） | `kubectl apply -f volumesnapshot.yaml` | 否（重复创建报错） |
| 从快照恢复 PVC | `kubectl apply -f restore-pvc.yaml` | 否（重复创建报错） |
| 查看快照列表 | `kubectl get volumesnapshot` | 是 |
| 查看 VolumeSnapshotClass | `kubectl get volumesnapshotclass` | 是 |
| 查看快照内容 | `kubectl get volumesnapshotcontent` | 是 |

## 操作步骤

### 步骤 1：创建 VolumeSnapshotClass（数据面）

```yaml
apiVersion: snapshot.storage.k8s.io/v1beta1
kind: VolumeSnapshotClass
metadata:
  name: cbs-snapclass
driver: com.tencent.cloud.csi.cbs
deletionPolicy: Delete
```

```bash
kubectl apply -f VolumeSnapshotClass.yaml
# expected: volumesnapshotclass.snapshot.storage.k8s.io/cbs-snapclass created

kubectl get volumesnapshotclass
# expected: cbs-snapclass 存在
```

> **注意**：kubectl 数据面当前因公网端点被 CAM 策略 strategyId:240463971 拒绝而不可达。以下 kubectl 命令为参考格式，需在内网/VPN 环境执行。

### 步骤 2：创建 PVC 快照 VolumeSnapshot（数据面）

```yaml
apiVersion: snapshot.storage.k8s.io/v1beta1
kind: VolumeSnapshot
metadata:
  name: new-snapshot-demo
spec:
  volumeSnapshotClassName: cbs-snapclass
  source:
    persistentVolumeClaimName: <pvc-name>
```

> **说明：** 将 `<pvc-name>` 替换为实际需要创建快照的 PVC 名称。

```bash
kubectl apply -f VolumeSnapshot.yaml
# expected: volumesnapshot.snapshot.storage.k8s.io/new-snapshot-demo created
```

验证 VolumeSnapshot 和 VolumeSnapshotContent 是否创建成功（`READYTOUSE` 为 `true`）：

```bash
kubectl get volumesnapshot
kubectl get volumesnapshotcontent
# expected: READYTOUSE true
```

```text
NAME  STATUS  AGE
...
```

### 步骤 3：获取快照 ID（数据面）

```bash
kubectl get volumesnapshotcontent <snapcontent-name> -o yaml
# expected: status.snapshotHandle 字段（如 snap-e406fc9m）
```

查看 `status.snapshotHandle` 字段，可在 [云服务控制台 → 快照列表](https://console.cloud.tencent.com/cvm/snapshot) 确认快照是否存在。

### 步骤 4：从快照恢复数据到新 PVC（数据面）

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: restore-test
spec:
  storageClassName: <storageclass-name>
  dataSource:
    name: new-snapshot-demo
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

```bash
kubectl apply -f restore-pvc.yaml
# expected: persistentvolumeclaim/restore-test created
```

验证恢复的 PVC：

```bash
kubectl get pvc restore-test
kubectl get pv <bound-pv-name> -o yaml
# expected: PVC 状态 Bound
```

```text
NAME  STATUS  AGE
...
```

> **说明：** 如果 StorageClass 使用了拓扑感知（`volumeBindingMode: WaitForFirstConsumer`），则需先部署挂载 PVC 的 Pod 才会触发创建 PV。

## 验证

### 数据面（kubectl）

```bash
kubectl get volumesnapshot new-snapshot-demo -o jsonpath='{.status.readyToUse}'
# expected: true

kubectl get pvc restore-test -o jsonpath='{.status.phase}'
# expected: Bound

kubectl get volumesnapshotcontent
# expected: 快照内容存在
```

```text
NAME  STATUS  AGE
...
```

> **注意**：kubectl 数据面当前因公网端点被 CAM 策略 strategyId:240463971 拒绝而不可达。以下 kubectl 命令为参考格式，需在内网/VPN 环境执行。

## 清理

### 数据面（kubectl）

```bash
kubectl delete pvc restore-test
kubectl delete volumesnapshot new-snapshot-demo
kubectl delete volumesnapshotclass cbs-snapclass
# expected: 资源已删除
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `READYTOUSE` 为 `false` | `kubectl describe volumesnapshot new-snapshot-demo` 查看事件 | CBS-CSI 组件版本过低或快照创建中 | 检查 CBS-CSI 组件版本；等待 CBS 快照创建完成 |
| 恢复的 PVC 未绑定 PV | `kubectl describe pvc restore-test` 查看 Events | StorageClass 不支持动态创建或启用 `WaitForFirstConsumer` | 检查 StorageClass 是否支持动态创建；若启用 `WaitForFirstConsumer`，需先创建使用 PVC 的 Pod |
| `snapshotHandle` 为空 | `kubectl get volumesnapshotcontent <name> -o yaml` 查看状态 | 快照创建未完成 | 等待 CBS 快照创建完成 |
| `Unable to connect to the server` | `kubectl cluster-info` 检查连通性 | 公网端点被 CAM 策略 strategyId:240463971 以 `tke:clusterExtranetEndpoint=true` 条件拒绝 | 通过 IOA/VPN 连接内网，或使用同 VPC CVM 执行 kubectl 命令 |

## 下一步

- [通过 CBS-CSI 避免云硬盘跨可用区挂载](../通过%20CBS-CSI%20避免云硬盘跨可用区挂载/tccli 操作.md)
- [在线扩容云硬盘](../在线扩容云硬盘/tccli 操作.md)
- [CBS-CSI 简介](../CBS-CSI 简介/tccli 操作.md)

## 控制台替代

控制台 **集群 → 存储 → VolumeSnapshot → 新建快照**；**集群 → 存储 → PersistentVolumeClaim → 新建**，在数据源中选择已有快照恢复。
