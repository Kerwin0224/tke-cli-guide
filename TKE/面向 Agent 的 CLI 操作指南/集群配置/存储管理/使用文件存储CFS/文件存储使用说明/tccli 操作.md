# 文件存储使用说明

> 对照官方：[文件存储使用说明](https://cloud.tencent.com/document/product/457/44234) · page_id `44234`

## 概述

腾讯云容器服务 TKE 支持通过 PV/PVC 为工作负载挂载腾讯云文件存储 CFS。CFS 支持 ReadWriteMany 访问模式，适合多 Pod 共享数据的场景。

两种使用方式：

| 方式 | 说明 | 适用场景 | 详细文档 |
|------|------|---------|---------|
| 动态创建 | 通过 StorageClass 模板自动创建 CFS 文件系统并绑定 PV/PVC | 新建 CFS、按需自动供给 | [StorageClass 管理文件存储模板](../StorageClass管理文件存储模板/tccli%20操作.md) |
| 使用已有 CFS | 通过已有 CFS 实例创建静态 PV，再绑 PVC | 存量 CFS 迁移、手动管控 | [PV 和 PVC 管理文件存储](../PV和PVC管理文件存储/tccli%20操作.md) |

**前置要求**：集群需安装 CFS-CSI 扩展组件，且集群 VPC 与 CFS 实例在**同一地域、同一 VPC**。

## 前置条件

- [环境准备](../../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:InstallAddon, tke:DescribeAddon, tke:DescribeAddonValues
#    cfs:DescribeCfsFileSystems, cfs:DescribeMountTargets, cfs:CreateCfsFileSystem
# 验证：执行 DescribeClusters 确认 TKE 权限
tccli tke DescribeClusters --region <Region>
# expected: exit 0，返回集群列表（可为空）
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

### 资源检查

```bash
# 4. 确认目标集群存在且状态正常
tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
# expected: TotalCount >= 1，ClusterStatus: "Running"

# 5. 查询集群 VPC（CFS 需同 VPC）
tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
# expected: 记录 Output 中的 ClusterNetworkSettings.VpcId
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

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 安装 CFS-CSI 组件 | `InstallAddon --AddonName CFS` | 否 |
| 查询组件安装状态 | `DescribeAddon --AddonName CFS` | 是 |
| 查看已有 CFS 文件系统 | `cfs DescribeCfsFileSystems` | 是 |
| 查看挂载点信息 | `cfs DescribeMountTargets` | 是 |
| 创建 CFS 文件系统 | `cfs CreateCfsFileSystem` | 否 |
| 动态 StorageClass / PVC | 见 [StorageClass 管理文件存储模板](../StorageClass管理文件存储模板/tccli%20操作.md) | — |
| 静态 PV / PVC / Workload | 见 [PV 和 PVC 管理文件存储](../PV和PVC管理文件存储/tccli%20操作.md) | — |

## 操作步骤

### 步骤 1：安装 CFS-CSI 组件

#### 选择依据

- **AddonName**：固定为 `CFS`，对应 CFS-CSI 驱动。
- **AddonVersion**：通过 `DescribeAddonValues` 查询当前集群可用的最新版本。

```bash
# 查询可用版本
tccli tke DescribeAddonValues --region <Region> --ClusterId CLUSTER_ID --AddonName CFS
# expected: 返回可用版本列表
```

```json
{
  "Values": "<Values>",
  "DefaultValues": "<DefaultValues>",
  "RequestId": "<RequestId>"
}
```

#### 安装命令

```bash
tccli tke InstallAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName CFS \
    --AddonVersion ADDON_VERSION
# expected: exit 0，返回 RequestId
```

**预期输出**：

```json
{
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

| 占位符 | 说明 | 获取方式 |
|--------|------|---------|
| `ADDON_VERSION` | CFS-CSI 组件版本 | `tccli tke DescribeAddonValues --region <Region> --ClusterId CLUSTER_ID --AddonName CFS` |

### 步骤 2：轮询组件安装状态

```bash
tccli tke DescribeAddon --region <Region> --ClusterId CLUSTER_ID --AddonName CFS
# expected: Phase: "Succeeded", Status: "Running"
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

### 步骤 3：确认集群 VPC（CFS 需同 VPC）

```bash
tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
# expected: 记录输出中的 VpcId，后续创建或选择 CFS 时使用
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

### 后续操作

- **动态创建**：见 [StorageClass 管理文件存储模板](../StorageClass管理文件存储模板/tccli%20操作.md)
- **使用已有 CFS**：见 [PV 和 PVC 管理文件存储](../PV和PVC管理文件存储/tccli%20操作.md)

## 验证

### 控制面（tccli）

| 维度 | 命令 | 预期 |
|------|------|------|
| 组件状态 | `tccli tke DescribeAddon --region <Region> --ClusterId CLUSTER_ID --AddonName CFS` | `Phase: "Succeeded"`, `Status: "Running"` |
| CFS 存量 | `tccli cfs DescribeCfsFileSystems --region <Region>` | 可列出文件系统列表（可为空） |
| 集群 VPC | `tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'` | 确认 VpcId 与 CFS 所在 VPC 一致 |

### 数据面（kubectl）

```bash
kubectl get pods -n kube-system | grep cfs
# expected: cfs-csi 相关 Pod 均为 Running
```

```text
NAME  STATUS  AGE
...
```

## 清理

本页仅安装 CFS-CSI 组件。如需卸载：

```bash
# 卸载前检查：确认无 PVC 使用 CFS StorageClass
kubectl get pvc -A | grep cfs
# 若有结果，先删除对应 PVC 和 Pod

tccli tke DeleteAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName CFS
# expected: exit 0，返回 RequestId
```

```text
NAME  STATUS  AGE
...
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `InstallAddon` 返回 "addon version not found" | `tccli tke DescribeAddonValues --region <Region> --ClusterId CLUSTER_ID --AddonName CFS` 查看可用版本 | 指定的 `AddonVersion` 不存在 | 从 `DescribeAddonValues` 输出取正确版本号 |
| `InstallAddon` 返回 `InvalidParameter.AddonName` | 检查 `AddonName` 字段 | 填写了小写 `cfs` 或其他非标准名称 | 使用 `--AddonName CFS`（大写） |
| `cfs DescribeCfsFileSystems` 返回 `UnauthorizedOperation` | 检查 CAM 策略 | `cfs:DescribeCfsFileSystems` 权限缺失（此为环境限制） | 联系 CAM 管理员授予 `cfs:DescribeCfsFileSystems` 权限 |
| 组件安装后 Pod 挂载失败 | `kubectl describe pod POD_NAME` 查看 Events | CFS 实例与集群不在同一 VPC | 确认 CFS 的 VpcId 与集群 VpcId 一致：`tccli cfs DescribeCfsFileSystems --region <Region> --FileSystemId CFS_ID` |

### 安装成功但 PVC 异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| PVC 一直 Pending | `kubectl describe pvc PVC_NAME` 查看 Events | 无匹配 PV 或 StorageClass 参数错误 | 参考 [PV 和 PVC 的绑定规则](../../PV和PVC的绑定规则/tccli%20操作.md) |
| PVC Bound 但 Pod 挂载超时 | `kubectl describe pod POD_NAME` 查看 Events，检查 CFS 挂载点可访问性 | 集群节点到 CFS 挂载点 IP 网络不通 | 确认集群 VPC 内有到 CFS IP 的路由：在集群节点上 `ping CFS_IP` |

## 下一步

- [StorageClass 管理文件存储模板](../StorageClass管理文件存储模板/tccli%20操作.md)
- [PV 和 PVC 管理文件存储](../PV和PVC管理文件存储/tccli%20操作.md)
- [PV 和 PVC 的绑定规则](../../PV和PVC的绑定规则/tccli%20操作.md)

## 控制台替代

[控制台 → 集群 → 组件管理](https://console.cloud.tencent.com/tke2/cluster) 安装 CFS-CSI 组件；[CFS 控制台](https://console.cloud.tencent.com/cfs) 管理文件系统。
