# GooseFS-CSI 说明（tccli）

> 对照官方：https://cloud.tencent.com/document/product/457/116406

## 概述

kubernetes-csi-tencentcloud GooseFS 插件实现 CSI 接口，使容器集群可使用腾讯云数据加速器（GooseFS）。GooseFS 是多协议、高性能、大吞吐的数据加速服务，提供统一命名空间和访问协议，以对象存储为数据底座，加速大数据分析、机器学习、AI 数据访问，支持冷热数据自动分层。

部署对象：

| 对象名称 | 类型 | 默认资源 | 命名空间 |
| --- | --- | --- | --- |
| goosefs-csi-controller | Deployment | 0.02C / 220MB | kube-system |
| goosefs-csi-nodeplugin | DaemonSet | 0.06C / 270MB | kube-system |
| csi-goosefs-controller | ServiceAccount / ClusterRole / ClusterRoleBinding | — | kube-system |
| csi-goosefs-node | ServiceAccount / ClusterRole / ClusterRoleBinding | — | kube-system |
| goosefs-csi-fuse-config | ConfigMap | — | kube-system |
| com.tencent.cloud.csi.goosefs | CSIDriver | — | — |

## 前置条件

环境准备与集群连通性请参见 [环境准备](../../../../环境准备.md) 与 [连接集群](../../../../集群配置/集群管理/连接集群/tccli 操作.md)。

先确认集群状态正常：

```bash
tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'
```

```json
{
  "TotalCount": 0,
  "Clusters": [],
  "ClusterId": "<ClusterId>",
  "ClusterName": "<ClusterName>",
  "ClusterDescription": "<ClusterDescription>",
  "ClusterVersion": "<ClusterVersion>",
  "ClusterOs": "<ClusterOs>",
  "ClusterType": "<ClusterType>"
}
```

确认返回的集群 `Status.Phase` 为 `Running` 后再继续。

- Kubernetes 版本 >= 1.14。
- 集群所在地域须在 GooseFS 支持的可用区。
- 已在数据加速器控制台创建 GooseFS 实例，且与集群处于同一 VPC。
- 集群需启用特权容器（组件需挂载宿主机 `/var/lib/kubelet` 目录）。

CAM 权限：调用 TKE 组件接口需具备相应权限，建议为操作账号授予 `QcloudTKEFullAccess` 或包含以下操作的策略：

| 操作 | 说明 |
| --- | --- |
| tke:DescribeClusters | 查询集群 |
| tke:DescribeAddon | 查询组件 |
| tke:InstallAddon | 安装组件 |
| tke:UpdateAddon | 更新组件 |
| tke:DeleteAddon | 删除组件 |

## 控制台与 CLI 参数映射

| 操作 | 控制台路径 | tccli 命令 |
| --- | --- | --- |
| 查询集群 | 集群管理 | `tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'` |
| 查询组件 | 集群 > 组件管理 | `tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName goosefs` |
| 安装组件 | 集群 > 组件管理 > 新建 | `tccli tke InstallAddon --region <Region> --ClusterId <ClusterId> --AddonName goosefs --AddonVersion <AddonVersion>` |
| 更新组件 | 集群 > 组件管理 > 更新 | `tccli tke UpdateAddon --region <Region> --ClusterId <ClusterId> --AddonName goosefs --AddonVersion <AddonVersion>` |
| 删除组件 | 集群 > 组件管理 > 删除 | `tccli tke DeleteAddon --region <Region> --ClusterId <ClusterId> --AddonName goosefs` |

## 操作步骤

### 1. 安装 GooseFS-CSI 组件（控制面）

```bash
tccli tke InstallAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName goosefs \
    --AddonVersion <AddonVersion>
```

### 2. 准备 GooseFS 实例

在数据加速器控制台创建或确认已有的 GooseFS 命名空间，获取以下信息：

- GooseFS Master 地址（`<GoosefsMasterHost>`，格式如 `goosefs-xxxxxxxx-master0.goosefs.svc.cluster.local` 或 IP 地址）
- GooseFS 路径（`<GoosefsPath>`，如 `/ns/my-namespace`）

### 3. 创建 GooseFS StorageClass（数据面）

> **注意**：kubectl 数据面当前因公网端点被 CAM 策略 strategyId:240463971 拒绝而不可达。以下 kubectl 命令为参考格式，需在内网/VPN 环境执行。

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: goosefs-sc
provisioner: com.tencent.cloud.csi.goosefs
parameters:
  goosefsPath: <GoosefsPath>
  goosefsMaster: <GoosefsMasterHost>
reclaimPolicy: Retain
```

StorageClass 参数说明：

| 参数 | 说明 |
| --- | --- |
| `provisioner` | 固定值 `com.tencent.cloud.csi.goosefs` |
| `goosefsPath` | GooseFS 中的命名空间路径，如 `/ns/my-namespace` |
| `goosefsMaster` | GooseFS Master 节点地址 |
| `reclaimPolicy` | 建议 `Retain`（保留）以保护数据 |

```bash
kubectl apply -f goosefs-storage-class.yaml
```

### 4. 创建 PV（数据面）

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
    volumeHandle: <GoosefsPath>
    volumeAttributes:
      goosefsMaster: <GoosefsMasterHost>
      goosefsPath: <GoosefsPath>
  persistentVolumeReclaimPolicy: Retain
  storageClassName: goosefs-sc
```

```bash
kubectl apply -f goosefs-pv.yaml
```

### 5. 创建 PVC（数据面）

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: goosefs-pvc
  namespace: default
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: goosefs-sc
  resources:
    requests:
      storage: 10Gi
```

> 说明：GooseFS 支持 ReadWriteMany 多节点读写。`storage` 字段为占位值，实际容量由 GooseFS 命名空间配置决定。

```bash
kubectl apply -f goosefs-pvc.yaml
```

### 6. 创建工作负载挂载 PVC（数据面）

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: goosefs-demo
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: goosefs-demo
  template:
    metadata:
      labels:
        app: goosefs-demo
    spec:
      containers:
        - name: app
          image: nginx:latest
          volumeMounts:
            - name: goosefs-data
              mountPath: /data
      volumes:
        - name: goosefs-data
          persistentVolumeClaim:
            claimName: goosefs-pvc
```

```bash
kubectl apply -f goosefs-deployment.yaml
```

### 组件 RBAC 权限（控制面已自动创建）

组件安装时自动创建两个 ClusterRole：

**csi-goosefs-controller（控制器权限）：**

| API Group | 资源 | 操作 |
| --- | --- | --- |
| `""` (core) | persistentvolumes, persistentvolumeclaims, nodes, events | get, list, watch, create, delete, update, patch |
| `storage.k8s.io` | storageclasses, csinodes | get, list, watch |
| `snapshot.storage.k8s.io` | volumesnapshots, volumesnapshotcontents | get, list |
| `coordination.k8s.io` | leases | get, list, watch, create, update, patch, delete（多副本选主）

```json
{
  "Addons": [],
  "AddonName": "<AddonName>",
  "AddonVersion": "<AddonVersion>",
  "RawValues": "<RawValues>",
  "Phase": "<Phase>",
  "Reason": "<Reason>",
  "CreateTime": "<CreateTime>",
  "RequestId": "<RequestId>"
}
``` |

**csi-goosefs-node（节点权限）：**

| API Group | 资源 | 操作 |
| --- | --- | --- |
| `""` (core) | pods | create, delete（启动/清理 FUSE 伴生 Pod） |

## 验证

### 控制面（tccli）

```bash
tccli tke DescribeAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName goosefs
```

```json
{
  "RequestId": "..."
}
```

预期输出：组件状态为 `Succeeded`，AddonName 为 `goosefs`。

### 数据面（kubectl）

> **注意**：kubectl 数据面当前因公网端点被 CAM 策略 strategyId:240463971 拒绝而不可达。以下 kubectl 命令为参考格式，需在内网/VPN 环境执行。

验证组件 Deployment 和 DaemonSet：

```bash
kubectl get deploy -n kube-system goosefs-csi-controller
kubectl get ds -n kube-system goosefs-csi-nodeplugin
```

```text
NAME  STATUS  AGE
...
```

预期输出：Deployment `READY` = `1/1`，DaemonSet `DESIRED` = `READY`。

查看 CSIDriver 是否注册：

```bash
kubectl get csidriver com.tencent.cloud.csi.goosefs
```

```text
NAME  STATUS  AGE
...
```

预期输出：NAME 为 `com.tencent.cloud.csi.goosefs`。

验证 PV/PVC 绑定：

```bash
kubectl get pv goosefs-pv
kubectl get pvc goosefs-pvc -n default
```

```text
NAME  STATUS  AGE
...
```

预期输出：PV 状态为 `Bound`，PVC 状态为 `Bound`。

验证工作负载挂载（FUSE 挂载点）：

```bash
kubectl exec -n default deploy/goosefs-demo -- df -h /data
```

预期输出：显示 GooseFS FUSE 挂载点。

跨 Pod 读写验证：

```bash
kubectl exec -n default deploy/goosefs-demo -- sh -c "echo hello > /data/test.txt"
kubectl exec -n default deploy/goosefs-demo -- sh -c "cat /data/test.txt"
```

预期输出：不同 Pod 副本均可读写同一文件。

## 清理

### 数据面（kubectl）

按创建逆序删除。

> **注意**：kubectl 数据面当前因公网端点被 CAM 策略 strategyId:240463971 拒绝而不可达。以下 kubectl 命令为参考格式，需在内网/VPN 环境执行。

```bash
kubectl delete deploy goosefs-demo -n default
kubectl delete pvc goosefs-pvc -n default
kubectl delete pv goosefs-pv
kubectl delete sc goosefs-sc
```

### 控制面（tccli）

```bash
tccli tke DeleteAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName goosefs
```

> 计费提醒：删除 PV/PVC 不会自动删除 GooseFS 实例及底层 COS 数据。请前往数据加速器控制台手动管理 GooseFS 实例。

## 排障

| 现象 | 诊断 | 根因 | 修复 |
| --- | --- | --- | --- |
| DaemonSet Pod 异常 | `kubectl describe pod -n kube-system <goosefs-csi-nodeplugin-pod>` 查看事件 | 特权容器未启用或 `/var/lib/kubelet` 挂载失败 | 检查集群特权容器策略，启用后重新调度 Pod |
| PVC 状态 Pending | 检查 StorageClass 参数与 GooseFS 连通性 | `goosefsMaster` 地址不可达或 `goosefsPath` 不存在 | 验证 GooseFS Master 地址正确且同 VPC 网络可达，确认命名空间已创建 |
| Pod 挂载失败 Fuse mount failed | 查看 Pod 事件与日志 | FUSE 内核模块未加载或 GooseFS FUSE 配置错误 | `lsmod \| grep fuse` 检查模块，验证 `goosefs-csi-fuse-config` ConfigMap |
| Pod 读写性能差 | 对比 GooseFS COS 桶与集群地域 | GooseFS 命名空间底层 COS 桶与集群不在同一地域 | 确保 GooseFS 关联的 COS 桶与集群同地域以减少跨地域延迟 |
| kubectl Unable to connect to server | 检查 CAM 策略与公网端点 | 公网端点被 CAM 策略 strategyId:240463971 拒绝（tke:clusterExtranetEndpoint=true） | 在内网/VPN 环境执行，或调整 CAM 策略放行公网端点 |

## 下一步

- 使用数据加速器 GooseFS — 详细的 GooseFS 工作负载挂载指南
- GooseFS 产品概述 — GooseFS 功能与架构
- COS-CSI 说明 — 底层对象存储组件
- 组件的生命周期管理 — 组件安装、升级与卸载

## 控制台替代

如需通过控制台操作，可在 **容器服务控制台 > 集群 > 组件管理** 中找到 goosefs 组件，执行安装、更新、删除；并在 **集群 > 存储** 中创建 StorageClass、PV、PVC 及挂载工作负载。对应的 tccli 命令见上文 [控制台与 CLI 参数映射](#控制台与-cli-参数映射)。
