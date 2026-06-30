# 云硬盘使用说明

> 对照官方：[云硬盘使用说明](https://cloud.tencent.com/document/product/457/44238) · page_id `44238` · tccli ≥3.1.107 · API 2017-03-12

## 概述

腾讯云容器服务 TKE 支持通过 PV/PVC 为工作负载挂载腾讯云云硬盘 CBS。CBS 提供块级存储，适合数据库等需要低延迟、频繁随机读写的 IO 密集型场景。

两种使用方式：

| 方式 | 说明 | 适用场景 | 详细文档 |
|------|------|---------|---------|
| 动态创建 | 通过 StorageClass 模板自动创建云硬盘并绑定 PV/PVC | 新建云盘、按需供给 | [StorageClass 管理云硬盘模板](../StorageClass%20管理云硬盘模板/tccli%20操作.md) |
| 使用已有云硬盘 | 通过已有 CBS 云硬盘创建静态 PV，再绑定 PVC | 存量云盘复用、数据迁移 | [PV 和 PVC 管理云硬盘](../PV%20和%20PVC%20管理云硬盘/tccli%20操作.md) |

**关键限制**：

- CBS 云硬盘仅支持 **ReadWriteOnce** 访问模式（一个云硬盘同时只能被一个节点上的一个 Pod 挂载）。
- 云硬盘**不支持跨可用区挂载**，Pod 所在节点的可用区必须与云硬盘所在可用区一致。
- 一个云硬盘仅支持创建一个 PV。

## 前置条件

- [环境准备](../../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version 1.0.0 或更高

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置

# 3. 验证 TKE 权限
tccli tke DescribeClusters --region <REGION>
# expected: exit 0，返回集群列表（可为空）
```

```json
{
  "TotalCount": 1,
  "Clusters": [
    {
      "ClusterId": "cls-example",
      "ClusterName": "example-cluster",
      "ClusterStatus": "Running",
      "ClusterVersion": "1.32.2"
    }
  ],
  "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

**所需 CAM Action**：`tke:InstallAddon`、`tke:DescribeAddon`、`tke:DescribeAddonValues`、`cbs:DescribeDisks`。

### 资源检查

```bash
# 4. 确认目标集群存在且状态正常
tccli tke DescribeClusters --region <REGION> --ClusterIds '["<CLUSTER_ID>"]'
# expected: TotalCount >= 1，ClusterStatus: "Running"

# 5. 确认 kubectl 可达（需要集群端点网络可达）
kubectl cluster-info
# expected: Kubernetes control plane is running at https://...
```

```json
{
  "TotalCount": 1,
  "Clusters": [
    {
      "ClusterId": "cls-example",
      "ClusterName": "example-cluster",
      "ClusterStatus": "Running",
      "ClusterVersion": "1.32.2",
      "ClusterType": "MANAGED_CLUSTER"
    }
  ],
  "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

> **注意**：kubectl 需要集群端点网络可达。如果集群仅有内网端点（无公网端点），需通过 IOA/VPN/专线 或同 VPC 内 CVM（跳板机）才能连接。确认方式见下方验证节。

## 控制台与 CLI 参数映射

| 控制台操作 | tccli/kubectl 命令 | 幂等 |
|-----------|-------------------|:--:|
| 查看 CBS-CSI 组件状态 | `tccli tke DescribeAddon --AddonName cbs` | 是 |
| 查看可用组件版本 | `tccli tke DescribeAddonValues --AddonName cbs` | 是 |
| 安装 CBS-CSI 组件 | `tccli tke InstallAddon --AddonName cbs` | 否 |
| 查看已有 CBS 云硬盘 | `tccli cbs DescribeDisks` | 是 |
| 查看 StorageClass | `kubectl get sc` | 是 |
| 查看 PV/PVC | `kubectl get pv,pvc -A` | 是 |
| 动态创建 StorageClass / PVC | 见 [StorageClass 管理云硬盘模板](../StorageClass%20管理云硬盘模板/tccli%20操作.md) | — |
| 静态 PV / PVC / Workload | 见 [PV 和 PVC 管理云硬盘](../PV%20和%20PVC%20管理云硬盘/tccli%20操作.md) | — |
| 扩容云硬盘 | `kubectl patch pvc <PVC_NAME> -p '{"spec":{"resources":{"requests":{"storage":"<NEW_SIZE>Gi"}}}}'` | 否 |

## 操作步骤

### 步骤 1：确认 CBS-CSI 组件已安装

TKE 托管集群默认预装 CBS-CSI 组件。先确认组件状态。

```bash
tccli tke DescribeAddon --region <REGION> --ClusterId <CLUSTER_ID> --AddonName cbs
# expected: Phase: "Succeeded"
```

```json
{
  "Addons": [
    {
      "AddonName": "cbs",
      "AddonVersion": "1.1.15",
      "RawValues": "Z2xvYmFsOgogIGNsdXN0ZX...",
      "Phase": "Succeeded",
      "Reason": "",
      "CreateTime": "2026-06-23T05:26:45Z"
    }
  ],
  "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

| 字段 | 含义 | 正常值 |
|------|------|--------|
| `AddonName` | 组件名称（注意是小写 `cbs`） | `cbs` |
| `AddonVersion` | 当前安装的版本 | 如 `1.1.15` |
| `Phase` | 安装阶段 | `Succeeded` 表示安装成功 |

### 步骤 2：查看 CBS-CSI 组件详细配置

如需了解组件的资源配置、映像版本等，可查询组件默认值。

```bash
tccli tke DescribeAddonValues --region <REGION> --ClusterId <CLUSTER_ID> --AddonName cbs
# expected: 返回 JSON 配置
```

```json
{
  "Values": "{\"global\":{\"cluster\":{\"id\":\"cls-example\",...}},\"controller\":{\"replicas\":1,...}}"
}
```

输出的 `Values` 是序列化的 JSON 字符串，含以下关键配置组：

| 配置组 | 说明 |
|--------|------|
| `global.cluster` | 集群信息（id、类型、K8s 版本） |
| `global.image` | 各组件容器镜像版本 |
| `controller.*` | 控制器副本数、资源限制、CSI 参数 |
| `nodeplugin.cbscsi` | 节点插件资源配置 |

### 步骤 3：查看已有 CBS 云硬盘

使用 `tccli cbs DescribeDisks` 查看账号下所有云硬盘。

```bash
# 查看前 5 个云硬盘
tccli cbs DescribeDisks --region <REGION> --Limit 5
# expected: 返回 DiskSet 列表
```

```json
{
  "TotalCount": 11240,
  "DiskSet": [
    {
      "DiskId": "disk-example",
      "DiskName": "example-disk",
      "DiskType": "CLOUD_BSSD",
      "DiskState": "ATTACHED",
      "DiskSize": 50,
      "DiskUsage": "SYSTEM_DISK",
      "DiskChargeType": "POSTPAID_BY_HOUR",
      "Placement": {
        "Zone": "ap-guangzhou-6"
      },
      "Shareable": false
    }
  ],
  "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

| 字段 | 含义 | 说明 |
|------|------|------|
| `DiskId` | 云硬盘 ID | 静态 PV 绑定时需用此 ID |
| `DiskType` | 云硬盘类型 | `CLOUD_BSSD`（通用型 SSD）、`CLOUD_PREMIUM`（高性能）、`CLOUD_SSD`（SSD） |
| `DiskState` | 状态 | `UNATTACHED`（待挂载）可用于创建 PV；`ATTACHED` 需先卸载 |
| `DiskSize` | 容量（GB） | |
| `Placement.Zone` | 可用区 | Pod 所在节点必须与此可用区一致 |
| `Shareable` | 是否共享 | CBS 云硬盘为 `false`（非共享） |

### 后续操作

完成以上检查后，根据需求选择对应操作：

- **动态创建云硬盘**：见 [StorageClass 管理云硬盘模板](../StorageClass%20管理云硬盘模板/tccli%20操作.md)
- **使用已有云硬盘**：见 [PV 和 PVC 管理云硬盘](../PV%20和%20PVC%20管理云硬盘/tccli%20操作.md)

## 验证

### 控制面（tccli）

| 维度 | 命令 | 预期 |
|------|------|------|
| CBS-CSI 组件状态 | `tccli tke DescribeAddon --region <REGION> --ClusterId <CLUSTER_ID> --AddonName cbs` | `Phase: "Succeeded"` |
| CBS 云硬盘存量 | `tccli cbs DescribeDisks --region <REGION> --Limit 5` | `TotalCount >= 0`，`DiskSet` 含云硬盘列表 |
| 集群端点可达 | `tccli tke DescribeClusterEndpoints --region <REGION> --ClusterId <CLUSTER_ID>` | 确认端点类型和地址 |

### 数据面（kubectl）

> **前置**：以下命令需要 kubectl 可连接到集群 API Server。如果集群仅有内网端点（`ClusterExternalEndpoint` 为空），需通过 IOA/VPN/专线或同 VPC CVM 跳板机连接。先确认端点状态：

```bash
# 确认端点类型
tccli tke DescribeClusterEndpoints --region <REGION> --ClusterId <CLUSTER_ID>
# 关注字段：ClusterExternalEndpoint（公网）、ClusterIntranetEndpoint（内网）
```

```json
{
  "ClusterExternalEndpoint": "",
  "ClusterIntranetEndpoint": "172.24.0.12",
  "ClusterDomain": "cls-example.ccs.tencent-cloud.com"
}
```

若 `ClusterExternalEndpoint` 为空，需通过内网端点访问：

```bash
# 获取内网端点 kubeconfig
tccli tke DescribeClusterKubeconfig --region <REGION> --ClusterId <CLUSTER_ID> --IsExtranet false
```

kubectl 可达后，执行以下验证：

```bash
# 1. 查看 CBS-CSI 组件 Pod 状态
kubectl get pods -n kube-system -l app=cbs-csi-controller
kubectl get pods -n kube-system -l app=cbs-csi-node
# expected: 所有 Pod STATUS 为 Running，READY 为 1/1

# 2. 查看 StorageClass（默认含 cbs）
kubectl get sc
# expected: 列表中含 cbs StorageClass（TKE 默认提供）
```

```text
NAME              PROVISIONER                       RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
cbs (default)     com.tencent.cloud.csi.cbs         Delete          Immediate              true                   1h
cbs-snapshot      com.tencent.cloud.csi.cbs         Delete          Immediate              true                   1h
```

```bash
# 3. 查看现有 PV 和 PVC
kubectl get pv,pvc -A
# expected: 列出所有 PV 和 PVC（可为空）
```

## 清理

本页为概念说明页，无写入操作，无需清理资源。如需卸载 CBS-CSI 组件（请谨慎——卸载后所有 CBS PVC 将无法正常工作）：

```bash
# ⚠️ 卸载前必须确认无 PVC 使用 CBS StorageClass
kubectl get pvc -A -o wide
# 检查 STORAGECLASS 列是否含 cbs

# 卸载组件
tccli tke DeleteAddon --region <REGION> \
    --ClusterId <CLUSTER_ID> \
    --AddonName cbs
# expected: exit 0，返回 RequestId
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DescribeAddon --AddonName CBS` 返回 `ResourceNotFound` | 检查组件名是否大写 | AddonName 应为小写 `cbs`，大写 `CBS` 不匹配 | 使用 `--AddonName cbs` |
| `DescribeAddonValues --AddonName CBS` 返回 `ResourceUnavailable` | 同上，大小写问题 | 组件名大小写敏感 | 使用 `--AddonName cbs` |
| `cbs DescribeDisks` 返回 `UnauthorizedOperation` | 检查 CAM 策略 | `cbs:DescribeDisks` 权限缺失 | 联系 CAM 管理员授予 `cbs:DescribeDisks` 权限 |
| kubectl 返回 `dial tcp: lookup ... no such host` | `kubectl cluster-info` 或 `traceroute` 到集群域名 | 无公网端点或 DNS 不可解析 | 1. 检查 `ClusterExternalEndpoint` 是否为空 2. 若仅内网端点，使用 IOA/VPN/专线或同 VPC CVM 连接 |

### 挂载失败

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Pod 挂载云硬盘失败（`Multi-Attach` 错误） | `kubectl describe pod <POD_NAME>` 查看 Events | 同一云硬盘被挂载到两个不同节点的 Pod | CBS 仅支持 ReadWriteOnce，不要在多个 Deployment 的 Pod 中引用同一 PVC |
| Pod 挂载失败，Events 报 "zone mismatch" | `tccli cbs DescribeDisks --region <REGION> --DiskIds '["<DISK_ID>"]'` 查看云盘可用区 | Pod 所在节点可用区与云硬盘可用区不一致 | 使用 `WaitForFirstConsumer` 的 StorageClass，或确保 Pod 调度到与云盘同可用区的节点 |
| PVC Pending（动态创建） | `kubectl describe pvc <PVC_NAME>` 查看 Events | CBS-CSI 未安装或 StorageClass 不存在 | 确认 CBS-CSI `Phase: "Succeeded"` 且 StorageClass 已创建 |

## 下一步

- [StorageClass 管理云硬盘模板](../StorageClass%20管理云硬盘模板/tccli%20操作.md)
- [PV 和 PVC 管理云硬盘](../PV%20和%20PVC%20管理云硬盘/tccli%20操作.md)
- [PV 和 PVC 的绑定规则](../../PV%20和%20PVC%20的绑定规则/tccli%20操作.md)

## 控制台替代

- [控制台 → 集群 → 组件管理](https://console.cloud.tencent.com/tke2/cluster) 安装/查看 CBS-CSI 组件
- [CBS 控制台](https://console.cloud.tencent.com/cvm/cbs) 管理云硬盘
- [控制台 → 集群 → 存储 → StorageClass](https://console.cloud.tencent.com/tke2/cluster) 管理 StorageClass
- [控制台 → 集群 → 存储 → PV/PVC](https://console.cloud.tencent.com/tke2/cluster) 管理 PV 和 PVC
