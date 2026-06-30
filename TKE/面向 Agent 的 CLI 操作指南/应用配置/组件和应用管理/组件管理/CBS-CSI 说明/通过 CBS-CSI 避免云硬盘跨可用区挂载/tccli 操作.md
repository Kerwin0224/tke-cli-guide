# 通过 CBS-CSI 避免云硬盘跨可用区挂载（tccli）

> 对照官方：[通过 CBS-CSI 避免云硬盘跨可用区挂载](https://cloud.tencent.com/document/product/457/67078) · page_id `67078`

## 概述

云硬盘不支持跨可用区挂载到节点。在跨可用区的集群环境中，通过 CBS-CSI 拓扑感知特性来避免跨可用区挂载问题。将 StorageClass 的 `volumeBindingMode` 设为 `WaitForFirstConsumer`，使 PV 在与 Pod 相同可用区创建，避免云硬盘和 Node 在不同可用区而无法挂载。

### 实现原理

拓扑感知调度流程：

1. PV controller 观察 PVC，检查 StorageClass 的 `VolumeBindingMode` 是否为 `WaitForFirstConsumer`，如是则不立即处理 PVC 创建事件
2. Scheduler 调度 Pod 后，将 nodeName 以 annotation 方式加入 PVC：`volume.kubernetes.io/selected-node: 10.0.0.72`
3. PV controller 获取 PVC 更新事件，根据 nodeName 获取 Node 对象传给 external-provisioner
4. external-provisioner 根据 Node 的 label 获取可用区（`failure-domain.beta.kubernetes.io/zone`）后在对应可用区创建 PV

## 前置条件

- [环境准备](../../../../../环境准备.md)
- 已 [连接集群](../../../../../集群配置/集群管理/连接集群/tccli 操作.md)，kubectl 可正常访问 APIServer
- 已安装 1.14 或以上版本的 TKE 集群
- 已将 CBS-CSI 或 In-Tree 组件更新为最新版本
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
| 创建拓扑感知 StorageClass | `kubectl apply -f cbs-topo-sc.yaml` | 否（同名报错） |
| 查看 StorageClass | `kubectl get sc` | 是 |
| 删除 StorageClass | `kubectl delete sc <name>` | 是 |

## 操作步骤

### 步骤 1：创建拓扑感知 StorageClass（数据面）

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: cbs-topo
parameters:
  type: cbs
provisioner: com.tencent.cloud.csi.cbs
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

```bash
kubectl apply -f cbs-topo-sc.yaml
# expected: storageclass.storage.k8s.io/cbs-topo created
```

> **说明：** CBS-CSI 和 In-Tree 组件均支持该操作。

> **注意**：kubectl 数据面当前因公网端点被 CAM 策略 strategyId:240463971 拒绝而不可达。以下 kubectl 命令为参考格式，需在内网/VPN 环境执行。

### 步骤 2：查看 StorageClass（数据面）

```bash
kubectl get sc cbs-topo -o yaml
# expected: volumeBindingMode: WaitForFirstConsumer

kubectl get sc
# expected: cbs-topo 存在
```

```text
NAME  STATUS  AGE
...
```

## 验证

### 数据面（kubectl）

```bash
kubectl get sc cbs-topo -o jsonpath='{.volumeBindingMode}'
# expected: WaitForFirstConsumer
```

```text
NAME  STATUS  AGE
...
```

> **注意**：kubectl 数据面当前因公网端点被 CAM 策略 strategyId:240463971 拒绝而不可达。以下 kubectl 命令为参考格式，需在内网/VPN 环境执行。

## 清理

### 数据面（kubectl）

```bash
kubectl delete sc cbs-topo
# expected: storageclass.storage.k8s.io "cbs-topo" deleted
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| PV 创建在错误可用区 | `kubectl get sc cbs-topo -o yaml` 检查 `volumeBindingMode` | StorageClass 未设置 `WaitForFirstConsumer` | 确认 StorageClass `volumeBindingMode` 为 `WaitForFirstConsumer` |
| Pod 调度后 PVC 未绑定 | `kubectl describe pvc <name>` 查看 Events | CBS-CSI 组件版本过低 | 检查 CBS-CSI 组件版本是否为最新 |
| 跨可用区挂载失败 | `kubectl get nodes -L failure-domain.beta.kubernetes.io/zone` 查看节点可用区 | CBS 不支持跨可用区挂载，Pod 和 PV 在不同 AZ | 确保 Pod 和 PV 在同一 AZ |
| `Unable to connect to the server` | `kubectl cluster-info` 检查连通性 | 公网端点被 CAM 策略 strategyId:240463971 以 `tke:clusterExtranetEndpoint=true` 条件拒绝 | 通过 IOA/VPN 连接内网，或使用同 VPC CVM 执行 kubectl 命令 |

## 下一步

- [在线扩容云硬盘](../在线扩容云硬盘/tccli 操作.md)
- [创建快照和使用快照来恢复卷](../创建快照和使用快照来恢复卷/tccli 操作.md)
- [CBS-CSI 简介](../CBS-CSI 简介/tccli 操作.md)

## 控制台替代

控制台 **集群 → 存储 → StorageClass → 新建**，勾选 **WaitForFirstConsumer** 模式。
