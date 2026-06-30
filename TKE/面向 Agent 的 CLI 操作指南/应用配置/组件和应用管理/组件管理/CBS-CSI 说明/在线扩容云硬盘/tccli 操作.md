# 在线扩容云硬盘（tccli）

> 对照官方：[在线扩容云硬盘](https://cloud.tencent.com/document/product/457/67079) · page_id `67079`

## 概述

TKE 支持在线扩容 PV、对应的云硬盘及文件系统，无需重启 Pod 即可完成扩容。提供两种方式：重启 Pod 情况下的在线扩容（推荐，文件系统未挂载，避免扩容出错）和不重启 Pod 情况下的在线扩容（可能存在 I/O 进程导致文件系统扩容错误）。扩容前建议使用快照备份数据。

| 方式 | 说明 |
|------|------|
| 重启 Pod 情况下在线扩容 | 待扩容的云硬盘文件系统未被挂载，避免扩容出错。**推荐使用** |
| 不重启 Pod 情况下在线扩容 | 节点上挂载着待扩容的云硬盘文件系统，若存在 I/O 进程可能出现文件系统扩容错误 |

## 前置条件

- [环境准备](../../../../../环境准备.md)
- 已 [连接集群](../../../../../集群配置/集群管理/连接集群/tccli 操作.md)，kubectl 可正常访问 APIServer
- 已创建 1.16 或以上版本的 TKE 集群
- 已将 CBS-CSI 升级至最新版本
- 可选：扩容前 [使用快照备份数据](https://cloud.tencent.com/document/product/457/67080)
- 1.20 以下集群中，非 CBS-CSI 类型的 PV 不支持在线扩容
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
| 创建 StorageClass（启用扩容） | `kubectl apply -f cbs-csi-sc.yaml` | 否（同名报错） |
| 创建 PVC | `kubectl apply -f pvc.yaml` | 否（重复创建报错） |
| 扩容 PVC 容量 | `kubectl patch pvc <name> -p '{...}'` | 否（容量不能减小） |
| 查看 PVC 容量 | `kubectl get pvc <name>` | 是 |

## 操作步骤

### 步骤 1：创建允许扩容的 StorageClass（数据面）

```yaml
allowVolumeExpansion: true
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: cbs-csi
parameters:
  diskType: CLOUD_PREMIUM
provisioner: com.tencent.cloud.csi.cbs
reclaimPolicy: Delete
volumeBindingMode: Immediate
```

```bash
kubectl apply -f cbs-csi-sc.yaml
# expected: storageclass.storage.k8s.io/cbs-csi created
```

> **注意**：kubectl 数据面当前因公网端点被 CAM 策略 strategyId:240463971 拒绝而不可达。以下 kubectl 命令为参考格式，需在内网/VPN 环境执行。

### 步骤 2：创建允许扩容的 PVC（数据面）

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: nginx-pv-claim
spec:
  storageClassName: cbs-csi
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

```bash
kubectl apply -f pvc.yaml
# expected: persistentvolumeclaim/nginx-pv-claim created
```

### 步骤 3：在线扩容

#### 方式一：重启 Pod 情况下在线扩容（推荐）

**Step 1** — 确认扩容前状态：

```bash
kubectl exec <pod-name> -- df /usr/share/nginx/html
kubectl get pv <pv-name>
# expected: 显示当前容量
```

```text
NAME  STATUS  AGE
...
```

**Step 2** — 为 PV 打上非法 zone 标签使 Pod 重启后无法调度：

```bash
kubectl label pv <pv-name> failure-domain.beta.kubernetes.io/zone=nozone
# expected: persistentvolume/<pv-name> labeled
```

**Step 3** — 重启 Pod（Pod 将处于 Pending 状态）：

```bash
kubectl delete pod <pod-name>
kubectl get pod <pod-name>
kubectl describe pod <pod-name>
# expected: Pod Pending
```

```text
NAME  STATUS  AGE
...
```

**Step 4** — 修改 PVC 容量扩容至目标大小：

```bash
kubectl patch pvc <pvc-name> -p '{"spec":{"resources":{"requests":{"storage":"<NewSize>Gi"}}}}'
# expected: persistentvolumeclaim/<pvc-name> patched
```

> **注意：** 扩容后容量不允许小于当前容量；扩容后 PVC 容量必须为 10 的倍数。

**Step 5** — 去除 PV 非法标签，Pod 调度成功并扩容完成：

```bash
kubectl label pv <pv-name> failure-domain.beta.kubernetes.io/zone-
kubectl get pod <pod-name>
kubectl get pv <pv-name>
kubectl exec <pod-name> -- df /usr/share/nginx/html
# expected: Pod Running，容量已扩容
```

```text
NAME  STATUS  AGE
...
```

#### 方式二：不重启 Pod 情况下在线扩容

**Step 1** — 确认扩容前状态：

```bash
kubectl exec <pod-name> -- df /usr/share/nginx/html
kubectl get pv <pv-name>
# expected: 显示当前容量
```

```text
NAME  STATUS  AGE
...
```

**Step 2** — 直接修改 PVC 容量：

```bash
kubectl patch pvc <pvc-name> -p '{"spec":{"resources":{"requests":{"storage":"<NewSize>Gi"}}}}'
# expected: persistentvolumeclaim/<pvc-name> patched
```

> **注意：** 扩容后 PVC 容量必须为 10 的倍数，不同云硬盘类型支持的存储容量规格参考 [云硬盘类型](https://cloud.tencent.com/document/product/362/2353)。

**Step 3** — 验证扩容结果：

```bash
kubectl exec <pod-name> -- df /usr/share/nginx/html
kubectl get pv <pv-name>
# expected: 容量已扩容
```

```text
NAME  STATUS  AGE
...
```

## 验证

### 数据面（kubectl）

```bash
kubectl get pvc <pvc-name> -o jsonpath='{.status.capacity.storage}'
# expected: 扩容后容量

kubectl exec <pod-name> -- df -h <mount-path>
# expected: 文件系统容量已扩容
```

```text
NAME  STATUS  AGE
...
```

> **注意**：kubectl 数据面当前因公网端点被 CAM 策略 strategyId:240463971 拒绝而不可达。以下 kubectl 命令为参考格式，需在内网/VPN 环境执行。

## 清理

### 数据面（kubectl）

```bash
kubectl delete pvc <pvc-name>
kubectl delete sc cbs-csi
# expected: 资源已删除
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 扩容失败 | `kubectl get sc cbs-csi -o yaml` 检查 `allowVolumeExpansion` | CBS-CSI 组件版本过低或 StorageClass 未设置 `allowVolumeExpansion: true` | 检查 CBS-CSI 组件是否为最新版本；确认 StorageClass 已设置 `allowVolumeExpansion: true` |
| 文件系统扩容失败 | `kubectl describe pvc <name>` 查看 Events | 节点上存在 I/O 进程干扰 | 推荐使用方式一（重启 Pod），避免 I/O 进程干扰 |
| PVC 容量不是 10 的倍数 | `kubectl get pvc <name>` 查看容量 | CBS 要求扩容后容量为 10 的倍数 | 调整目标容量为 10 的倍数 |
| 扩容后容量小于当前容量 | `kubectl get pvc <name>` 对比容量 | 扩容不允许容量减小 | 扩容目标容量必须大于当前容量 |
| 非 CBS-CSI 类型 PV 扩容失败 | `kubectl get pv <name> -o yaml` 检查 provisioner | 1.20 以下集群非 CBS-CSI 类型的 PV 不支持在线扩容 | 升级集群至 1.20+ 或改用 CBS-CSI 类型的 PV |
| `Unable to connect to the server` | `kubectl cluster-info` 检查连通性 | 公网端点被 CAM 策略 strategyId:240463971 以 `tke:clusterExtranetEndpoint=true` 条件拒绝 | 通过 IOA/VPN 连接内网，或使用同 VPC CVM 执行 kubectl 命令 |

## 下一步

- [通过 CBS-CSI 避免云硬盘跨可用区挂载](../通过%20CBS-CSI%20避免云硬盘跨可用区挂载/tccli 操作.md)
- [创建快照和使用快照来恢复卷](../创建快照和使用快照来恢复卷/tccli 操作.md)
- [CBS-CSI 简介](../CBS-CSI 简介/tccli 操作.md)

## 控制台替代

控制台 **集群 → 存储 → StorageClass → 新建**，勾选 **启用在线扩容**；**集群 → 存储 → PersistentVolumeClaim → 新建**，指定允许扩容的 StorageClass。
