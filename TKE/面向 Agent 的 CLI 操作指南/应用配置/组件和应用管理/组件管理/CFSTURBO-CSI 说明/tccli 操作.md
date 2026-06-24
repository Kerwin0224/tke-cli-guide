# CFSTURBO-CSI 说明（tccli）

> 对照官方：[CFSTURBO-CSI 说明](https://cloud.tencent.com/document/product/457/96272) · page_id `96272`

## 概述

kubernetes-csi-tencentcloud CFSTURBO 插件实现 CSI 接口，使容器集群内可使用腾讯云 Turbo 文件存储（CFS Turbo）。CFS Turbo 适合大规模吞吐型和混合负载型业务，采用私有协议挂载方式，单客户端即可达到存储集群性能级别。

### 部署的 Kubernetes 对象

| 对象名称 | 类型 | 默认资源 | 命名空间 |
|---------|------|---------|---------|
| cfsturbo-csi-controller | — | — | kube-system |
| cfsturbo-csi-nodeplugin | DaemonSet | — | kube-system |

## 前置条件

- [环境准备](../../../../环境准备.md)
- 已 [连接集群](../../../../集群配置/集群管理/连接集群/tccli 操作.md)，kubectl 可正常访问 APIServer
- Kubernetes 版本 **>= 1.14**
- 已创建 CFS Turbo 实例（在 [CFS 控制台](https://console.cloud.tencent.com/cfs) 创建），且与集群在同 VPC 内
- 集群需启用特权容器（组件需挂载宿主机 `/var/lib/kubelet` 目录）
- 集群状态 `Running`

```bash
# 检查集群状态
tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'
# expected: ClusterStatus "Running"
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

- 需要 CAM 权限：`tke:DescribeClusters`、`tke:DescribeAddon`、`tke:InstallAddon`、`tke:UpdateAddon`、`tke:DeleteAddon`

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 查看集群信息 | `tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'` | 是 |
| 查看 CFSTURBO-CSI 组件详情 | `tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName cfsturbo` | 是 |
| 安装 CFSTURBO-CSI 组件 | `tccli tke InstallAddon --region <Region> --ClusterId <ClusterId> --AddonName cfsturbo` | 否 |
| 升级 CFSTURBO-CSI 组件 | `tccli tke UpdateAddon --region <Region> --ClusterId <ClusterId> --AddonName cfsturbo --AddonVersion <AddonVersion>` | 否 |
| 卸载 CFSTURBO-CSI 组件 | `tccli tke DeleteAddon --region <Region> --ClusterId <ClusterId> --AddonName cfsturbo` | 是 |
| 创建 CFS Turbo StorageClass | `kubectl apply -f cfsturbo-storage-class.yaml` | 否（同名报错） |
| 创建 CFS Turbo PVC | `kubectl apply -f cfsturbo-pvc.yaml` | 否（同名报错） |
| 创建 CFS Turbo PV（静态） | `kubectl apply -f cfsturbo-pv.yaml` | 否（同名报错） |

## 操作步骤

### 步骤 1：安装 CFSTURBO-CSI 组件（控制面）

```bash
tccli tke InstallAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName cfsturbo
# expected: exit 0，组件安装请求已提交
```

### 步骤 2：创建 CFS Turbo 存储资源（数据面）

#### 方式一：指定 StorageClass（动态创建 PV）

创建 CFS Turbo StorageClass：

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: cfsturbo-sc
provisioner: com.tencent.cloud.csi.cfsturbo
parameters:
  host: <CFSTurboIP>
  fsid: <FileSystemId>
  protocol: lustre
  rootdir: /
reclaimPolicy: Retain
```

| 参数 | 说明 |
|------|------|
| `provisioner` | 固定值 `com.tencent.cloud.csi.cfsturbo`，选择「文件存储 CFS Turbo」 |
| `host` | CFS Turbo 实例 IP 地址 |
| `fsid` | CFS Turbo 文件系统 ID |
| `protocol` | 固定值 `lustre`（CFS Turbo 私有协议） |
| `rootdir` | CFS Turbo 根目录 |
| `reclaimPolicy` | 回收策略：建议设置为 `Retain`（保留），防止误删数据。`Delete` 会在 PVC 删除时同时删除 PV 和存储实例 |

```bash
kubectl apply -f cfsturbo-storage-class.yaml
# expected: storageclass.storage.k8s.io/cfsturbo-sc created
```

创建 PVC：

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cfsturbo-pvc
  namespace: default
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: cfsturbo-sc
  resources:
    requests:
      storage: 20Gi
```

> **说明：** CFS Turbo 仅支持多机读写（ReadWriteMany）模式。指定 StorageClass 后，若没有匹配的 PV 存在，系统将自动动态创建 PV。

```bash
kubectl apply -f cfsturbo-pvc.yaml
# expected: persistentvolumeclaim/cfsturbo-pvc created
```

#### 方式二：不指定 StorageClass（静态创建 PV）

创建 PV：

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: cfsturbo-pv
spec:
  accessModes:
    - ReadWriteMany
  capacity:
    storage: 20Gi
  csi:
    driver: com.tencent.cloud.csi.cfsturbo
    volumeHandle: <FileSystemId>
    volumeAttributes:
      host: <CFSTurboIP>
      fsid: <FileSystemId>
      rootdir: /
      protocol: lustre
  persistentVolumeReclaimPolicy: Retain
  storageClassName: ""
```

> **说明：** `storageClassName: ""` 表示不指定 StorageClass。此模式下 PV 完全静态管理，不会触发动态创建。`volumeHandle` 使用 CFS Turbo 文件系统 ID。

```bash
kubectl apply -f cfsturbo-pv.yaml
# expected: persistentvolume/cfsturbo-pv created
```

创建 PVC 并绑定 PV：

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cfsturbo-pvc-static
  namespace: default
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: ""
  volumeName: cfsturbo-pv
  resources:
    requests:
      storage: 20Gi
```

> **说明：** 不指定 StorageClass 时必须指定 PersistentVolume（通过 `volumeName`），系统不允许在此模式下同时不指定 PersistentVolume。

```bash
kubectl apply -f cfsturbo-pvc-static.yaml
# expected: persistentvolumeclaim/cfsturbo-pvc-static created
```

### 步骤 3：创建工作负载挂载 PVC（数据面）

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cfsturbo-demo
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cfsturbo-demo
  template:
    metadata:
      labels:
        app: cfsturbo-demo
    spec:
      containers:
        - name: app
          image: nginx:latest
          volumeMounts:
            - name: cfs-data
              mountPath: /data
      volumes:
        - name: cfs-data
          persistentVolumeClaim:
            claimName: <cfsturbo-pvc 或 cfsturbo-pvc-static>
```

```bash
kubectl apply -f cfsturbo-deployment.yaml
# expected: deployment.apps/cfsturbo-demo created
```

> **注意**：kubectl 数据面当前因公网端点被 CAM 策略 strategyId:240463971 拒绝而不可达。以下 kubectl 命令为参考格式，需在内网/VPN 环境执行。

### 组件 RBAC 权限（控制面已自动创建）

组件安装时自动创建 ClusterRole `cfsturbo-csi-controller-role`：

| 功能 | 涉及对象 | 操作 |
|------|---------|------|
| 动态创建/删除 CFS Turbo 子目录型 PV | persistentvolumeclaims / persistentvolumes / storageclasses | get, list, watch, create, delete, update |
| 节点信息获取 | nodes | get, list |
| 事件通知 | events | get, list, watch, create, update, patch |

## 验证

### 控制面（tccli）

```bash
tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName cfsturbo \
    | jq '.Addons[0] | {AddonName, Status, AddonVersion}'
# expected: Status "Running"，AddonName "cfsturbo"
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

### 数据面（kubectl）

```bash
kubectl get sc cfsturbo-sc
# expected: provisioner 显示 com.tencent.cloud.csi.cfsturbo

kubectl get pv
kubectl get pvc -n default
# expected: PV 状态 Bound（动态创建）或 Available（静态创建），PVC 状态 Bound

kubectl exec -n default deploy/cfsturbo-demo -- df -h /data
# expected: 显示 CFS Turbo 文件系统挂载点及容量信息
```

```text
NAME  STATUS  AGE
...
```

> **注意**：kubectl 数据面当前因公网端点被 CAM 策略 strategyId:240463971 拒绝而不可达。以下 kubectl 命令为参考格式，需在内网/VPN 环境执行。

## 清理

### 数据面（kubectl）

按创建逆序删除：

```bash
kubectl delete deploy cfsturbo-demo -n default
kubectl delete pvc cfsturbo-pvc -n default
kubectl delete pv cfsturbo-pv
kubectl delete sc cfsturbo-sc
# expected: 资源已删除
```

### 控制面（tccli）

```bash
tccli tke DeleteAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName cfsturbo
# expected: exit 0，组件卸载请求已提交

# 验证已卸载
tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName cfsturbo \
    | jq '.Addons[] | select(.AddonName == "cfsturbo")'
# expected: 空结果
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

> **计费提醒：** 删除 PV 或 StorageClass 不会自动删除 CFS Turbo 实例。如果不再需要 CFS Turbo 实例，请前往 [CFS 控制台](https://console.cloud.tencent.com/cfs) 手动删除以避免持续计费。

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| PVC 一直处于 `Pending` | `kubectl describe pvc <name>` 查看 Events | StorageClass 中的 CFS Turbo IP 或文件系统 ID 错误 | 检查 CFS Turbo 实例是否在集群 VPC 内可达，验证 `host` 和 `fsid` 参数 |
| 工作负载 Pod 启动报 `MountVolume.SetUp failed` | `kubectl describe pod <name>` 查看 Events | 节点与 CFS Turbo 实例网络不通 | 确认集群 VPC 与 CFS Turbo 实例 VPC 一致或有对等连接/云联网互通 |
| 组件 DaemonSet Pod 异常 | `kubectl describe pod -n kube-system -l app=cfsturbo-csi-node` 查看 Events | 特权容器未启用 | 检查集群是否允许特权容器 |
| PV 状态 `Released` 而不是被删除 | `kubectl get pv <name>` 查看状态 | reclaimPolicy 为 `Retain` | 手动清理 PVC 引用的 PV 资源：`kubectl delete pv <pv-name>` |
| `Unable to connect to the server` | `kubectl cluster-info` 检查连通性 | 公网端点被 CAM 策略 strategyId:240463971 以 `tke:clusterExtranetEndpoint=true` 条件拒绝 | 通过 IOA/VPN 连接内网，或使用同 VPC CVM 执行 kubectl 命令 |

## 下一步

- [CFS 系统限制](https://cloud.tencent.com/document/product/582/9135) — 了解 CFS/CIFS Turbo 文件系统的使用限制
- [CFS-CSI 说明](../CFS-CSI 说明/tccli 操作.md) — 通用文件存储 CFS 组件（NFS 协议）
- [COS-CSI 说明](../COS-CSI 说明/tccli 操作.md) — 对象存储组件
- [组件的生命周期管理](../组件的生命周期管理/tccli 操作.md) — 组件安装、升级与卸载

## 控制台替代

通过 [TKE 控制台安装 CFSTURBO-CSI 组件](https://cloud.tencent.com/document/product/457/96272) 操作。
