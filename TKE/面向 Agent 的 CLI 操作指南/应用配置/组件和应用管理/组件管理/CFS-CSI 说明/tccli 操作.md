# CFS-CSI 说明（tccli）

> 对照官方：[CFS-CSI 说明](https://cloud.tencent.com/document/product/457/49258) · page_id `49258`

## 概述

CFS-CSI 是 TKE 的文件存储（CFS）插件，通过 CSI 接口将腾讯云 CFS 挂载到 Pod，支持 NFS v3/v4 协议，适合共享存储场景。

### 功能特性

- 基于 NFS 协议，支持多 Pod 共享读写
- 支持 CFS 通用型和 Turbo 型
- 动态 Provisioning 和静态 Provisioning
- 支持跨可用区访问

### StorageClass 配置

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: cfs-standard
provisioner: com.tencent.cloud.csi.cfs
parameters:
  vpcid: vpc-example
  subnetid: subnet-example
  vers: "4.0"
reclaimPolicy: Delete
```

## 前置条件

- [环境准备](../../../../环境准备.md)
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
| 查看 CFS-CSI 组件详情 | `tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName cfs-csi` | 是 |
| 安装 CFS-CSI 组件 | `tccli tke InstallAddon --region <Region> --ClusterId <ClusterId> --AddonName cfs-csi` | 否 |
| 升级 CFS-CSI 组件 | `tccli tke UpdateAddon --region <Region> --ClusterId <ClusterId> --AddonName cfs-csi --AddonVersion <AddonVersion>` | 否 |
| 卸载 CFS-CSI 组件 | `tccli tke DeleteAddon --region <Region> --ClusterId <ClusterId> --AddonName cfs-csi` | 是 |
| 创建 CFS StorageClass | `kubectl apply -f cfs-sc.yaml` | 否（同名报错） |
| 查看 StorageClass | `kubectl get sc` | 是 |

## 操作步骤

### 步骤 1：查看 CFS-CSI 组件信息（控制面）

```bash
tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName cfs-csi \
    | jq '.Addons[0] | {AddonName, AddonVersion, Status}'
# expected: 返回 CFS-CSI 组件详细信息
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

### 步骤 2：安装 CFS-CSI 组件（控制面）

```bash
tccli tke InstallAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName cfs-csi
# expected: exit 0，组件安装请求已提交
```

### 步骤 3：查看 CFS-CSI 运行状态（数据面）

```bash
kubectl get pods -n kube-system -l app.kubernetes.io/name=cfs-csi
# expected: cfs-csi Pod Running
```

```text
NAME  STATUS  AGE
...
```

> **注意**：kubectl 数据面当前因公网端点被 CAM 策略 strategyId:240463971 拒绝而不可达。以下 kubectl 命令为参考格式，需在内网/VPN 环境执行。

### 步骤 4：卸载 CFS-CSI 组件（控制面）

```bash
tccli tke DeleteAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName cfs-csi
# expected: exit 0，组件卸载请求已提交
```

## 验证

### 控制面（tccli）

```bash
tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName cfs-csi \
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
kubectl get pods -n kube-system -l app.kubernetes.io/name=cfs-csi
# expected: cfs-csi Pod Running
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
    --AddonName cfs-csi
# expected: exit 0

# 验证已卸载
tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName cfs-csi \
    | jq '.Addons[] | select(.AddonName == "cfs-csi")'
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
| 组件安装失败 | `tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName cfs-csi` 查看 Status 与 reason | 集群版本不兼容或资源不足 | 检查集群版本要求；扩容节点后重试 |
| CFS PV 挂载失败 | `kubectl describe pod <name>` 查看 Events | CFS 文件系统网络不通或权限不足 | 确认 CFS 实例与集群在同一 VPC；检查访问权限 |
| `Unable to connect to the server` | `kubectl cluster-info` 检查连通性 | 公网端点被 CAM 策略 strategyId:240463971 以 `tke:clusterExtranetEndpoint=true` 条件拒绝 | 通过 IOA/VPN 连接内网，或使用同 VPC CVM 执行 kubectl 命令 |

## 下一步

- [组件的生命周期管理](../组件的生命周期管理/tccli 操作.md)
- [扩展组件概述](../扩展组件概述/tccli 操作.md)
- [CFSTURBO-CSI 说明](../CFSTURBO-CSI 说明/tccli 操作.md)
- [CBS-CSI 简介](../CBS-CSI 简介/tccli 操作.md)

## 控制台替代

[TKE 控制台 - 组件管理](https://console.cloud.tencent.com/tke2/cluster) → 集群 → 组件管理。
