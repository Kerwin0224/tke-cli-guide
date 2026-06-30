# CBS-CSI 简介（tccli）

> 对照官方：[CBS-CSI 简介](https://cloud.tencent.com/document/product/457/51099) · page_id `51099`

## 概述

CBS-CSI（Cloud Block Storage Container Storage Interface）是腾讯云云硬盘的 Kubernetes CSI 驱动，支持在 TKE 集群中动态创建和管理 CBS 持久化卷（PersistentVolume），提供高性能块存储。

## 前置条件

- [环境准备](../../../../../环境准备.md)：`tccli` 已配置，`kubectl` ≥ v1.28
- 已 [连接集群](../../../../../集群配置/集群管理/连接集群/tccli 操作.md)，kubectl 可正常访问 APIServer
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
| 查看 PVC | `kubectl get pvc` | 是 |
| 创建快照 | `kubectl apply -f snapshot.yaml` | 否 |

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

kubectl get storageclass
# expected: cbs StorageClass 存在
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
| PVC 一直 `Pending` | `kubectl get storageclass` 确认 `cbs` StorageClass 存在；`kubectl describe pvc <name>` 查看事件 | CBS-CSI 未安装或 StorageClass 不存在 | 安装 CBS-CSI 组件；确认 StorageClass 存在 |
| `Multi-Attach error` | `kubectl get pvc <name> -o yaml` 检查 `accessModes` | CBS 只能单节点读写，`accessModes` 配置错误 | 确认 `accessModes` 为 `ReadWriteOnce` |
| `Unable to connect to the server` | `kubectl cluster-info` 检查连通性 | 公网端点被 CAM 策略 strategyId:240463971 以 `tke:clusterExtranetEndpoint=true` 条件拒绝 | 通过 IOA/VPN 连接内网，或使用同 VPC CVM 执行 kubectl 命令 |

## 下一步

- [创建快照和使用快照来恢复卷](.././创建快照和使用快照来恢复卷/tccli 操作.md)
- [通过 CBS-CSI 避免云硬盘跨可用区挂载](.././通过%20CBS-CSI%20避免云硬盘跨可用区挂载/tccli 操作.md)
- [在线扩容云硬盘](.././在线扩容云硬盘/tccli 操作.md)
- [Deployment 管理](../../../../工作负载管理/Deployment 管理/tccli 操作.md)

## 控制台替代

控制台详情参见 [CBS-CSI 简介（控制台）](https://cloud.tencent.com/document/product/457/51099)。
