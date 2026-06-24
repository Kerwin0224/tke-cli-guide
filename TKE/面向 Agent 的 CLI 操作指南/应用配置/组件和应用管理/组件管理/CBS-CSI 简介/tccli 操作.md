# CBS-CSI 简介（tccli）

> 对照官方：[CBS-CSI 简介](https://cloud.tencent.com/document/product/457/51099) · page_id `51099`

## 概述

CBS-CSI 是 TKE 的云硬盘（CBS）存储插件，通过 CSI 接口将腾讯云 CBS 挂载到 Pod，支持静态/动态 Provisioning、在线扩容、快照等功能。

### 功能特性

- **动态 Provisioning**：通过 StorageClass 自动创建 CBS 云硬盘
- **静态 Provisioning**：通过 PV/PVC 绑定已有 CBS
- **在线扩容**：在线扩展 CBS 大小（需文件系统支持）
- **快照与恢复**：通过 VolumeSnapshot 创建和使用快照
- **拓扑感知**：确保 CBS 与 Pod 在同一可用区

### StorageClass 配置

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: cbs-standard
provisioner: com.tencent.cloud.csi.cbs
parameters:
  type: CLOUD_PREMIUM
  zone: ap-guangzhou-3
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

| 参数 | 说明 | 可选值 |
|------|------|--------|
| `type` | 云硬盘类型 | `CLOUD_PREMIUM`、`CLOUD_SSD`、`CLOUD_HSSD` |
| `zone` | 可用区 | `ap-guangzhou-3` 等 |
| `reclaimPolicy` | 回收策略 | `Delete`、`Retain` |
| `volumeBindingMode` | 绑定模式 | `Immediate`、`WaitForFirstConsumer` |

## 前置条件

- [环境准备](../../../../环境准备.md)：`tccli` 已配置，`kubectl` ≥ v1.28
- 熟悉 Kubernetes 相关概念，已了解 TKE 集群架构
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
| 查看 CBS-CSI 组件详情 | `tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName cbs-csi` | 是 |
| 安装 CBS-CSI 组件 | `tccli tke InstallAddon --region <Region> --ClusterId <ClusterId> --AddonName cbs-csi` | 否 |
| 升级 CBS-CSI 组件 | `tccli tke UpdateAddon --region <Region> --ClusterId <ClusterId> --AddonName cbs-csi --AddonVersion <AddonVersion>` | 否 |
| 卸载 CBS-CSI 组件 | `tccli tke DeleteAddon --region <Region> --ClusterId <ClusterId> --AddonName cbs-csi` | 是 |
| 查看 StorageClass | `kubectl get storageclass` | 是 |
| 创建 PVC | `kubectl apply -f pvc.yaml` | 否（同名报错） |

## 操作步骤

### 步骤 1：查看 CBS-CSI 组件信息（控制面）

```bash
tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName cbs-csi \
    | jq '.Addons[0] | {AddonName, AddonVersion, Status}'
# expected: 返回 CBS-CSI 组件详细信息
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

### 步骤 2：安装 CBS-CSI 组件（控制面）

```bash
tccli tke InstallAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName cbs-csi
# expected: exit 0，组件安装请求已提交
```

### 步骤 3：查看 CBS-CSI 运行状态（数据面）

```bash
kubectl get pods -n kube-system -l app.kubernetes.io/name=cbs-csi
# expected: cbs-csi Pod Running

kubectl get storageclass
# expected: cbs StorageClass 存在
```

```text
NAME  STATUS  AGE
...
```

> **注意**：kubectl 数据面当前因公网端点被 CAM 策略 strategyId:240463971 拒绝而不可达。以下 kubectl 命令为参考格式，需在内网/VPN 环境执行。

### 步骤 4：卸载 CBS-CSI 组件（控制面）

```bash
tccli tke DeleteAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName cbs-csi
# expected: exit 0，组件卸载请求已提交
```

## 验证

### 控制面（tccli）

```bash
tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName cbs-csi \
    | jq '.Addons[0] | {AddonName, Status, AddonVersion}'
# expected: Status "Running"
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
kubectl get pods -n kube-system -l app.kubernetes.io/name=cbs-csi
# expected: cbs-csi Pod Running
```

```text
NAME  STATUS  AGE
...
```

> **注意**：kubectl 数据面当前因公网端点被 CAM 策略 strategyId:240463971 拒绝而不可达。以下 kubectl 命令为参考格式，需在内网/VPN 环境执行。

## 清理

```bash
# 卸载组件
tccli tke DeleteAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName cbs-csi
# expected: exit 0

# 验证已卸载
tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName cbs-csi \
    | jq '.Addons[] | select(.AddonName == "cbs-csi")'
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

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `InstallAddon` 返回 `FailedOperation.AddonAlreadyInstalled` | `tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId>` 检查已安装组件 | 组件已安装 | 无需重复安装；如需重装请先 `DeleteAddon` |
| 组件安装失败 | `tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName cbs-csi` 查看 Status 与 reason | 集群版本不兼容或资源不足 | 检查集群版本要求；扩容节点后重试 |
| PVC 一直 `Pending` | `kubectl get storageclass` 确认 `cbs` StorageClass 存在；`kubectl describe pvc <name>` 查看事件 | CBS-CSI 未安装或 StorageClass 不存在 | 安装 CBS-CSI 组件；确认 StorageClass 存在 |
| `Multi-Attach error` | `kubectl get pvc <name> -o yaml` 检查 `accessModes` | CBS 只能单节点读写，`accessModes` 配置错误 | 确认 `accessModes` 为 `ReadWriteOnce` |
| `Unable to connect to the server` | `kubectl cluster-info` 检查连通性 | 公网端点被 CAM 策略 strategyId:240463971 以 `tke:clusterExtranetEndpoint=true` 条件拒绝 | 通过 IOA/VPN 连接内网，或使用同 VPC CVM 执行 kubectl 命令 |

## 下一步

- [创建快照和使用快照来恢复卷](../CBS-CSI 说明/创建快照和使用快照来恢复卷/tccli 操作.md)
- [通过 CBS-CSI 避免云硬盘跨可用区挂载](../CBS-CSI 说明/通过%20CBS-CSI%20避免云硬盘跨可用区挂载/tccli 操作.md)
- [在线扩容云硬盘](../CBS-CSI 说明/在线扩容云硬盘/tccli 操作.md)
- [组件的生命周期管理](../组件的生命周期管理/tccli 操作.md)
- [扩展组件概述](../扩展组件概述/tccli 操作.md)

## 控制台替代

[TKE 控制台 - 组件管理](https://console.cloud.tencent.com/tke2/cluster) → 集群 → 组件管理。
