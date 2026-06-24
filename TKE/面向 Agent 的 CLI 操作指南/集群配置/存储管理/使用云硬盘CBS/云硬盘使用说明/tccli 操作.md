# 云硬盘使用说明

> 对照官方：[云硬盘使用说明](https://cloud.tencent.com/document/product/457/44238) · page_id `44238`

## 概述

腾讯云容器服务 TKE 支持通过 PV/PVC 为工作负载挂载腾讯云云硬盘 CBS。CBS 提供块级存储，适合数据库等需要低延迟、频繁随机读写的场景。

两种使用方式：

| 方式 | 说明 | 适用场景 | 详细文档 |
|------|------|---------|---------|
| 动态创建 | 通过 StorageClass 模板自动创建云硬盘并绑定 PV/PVC | 新建云盘、按需供给 | [StorageClass 管理云硬盘模板](../StorageClass管理云硬盘模板/tccli%20操作.md) |
| 使用已有云硬盘 | 通过已有 CBS 云硬盘创建静态 PV，再绑定 PVC | 存量云盘复用、数据迁移 | [PV 和 PVC 管理云硬盘](../PV和PVC管理云硬盘/tccli%20操作.md) |

**关键限制**：

- CBS 云硬盘仅支持 **ReadWriteOnce** 访问模式，一个云硬盘同时只能被一个节点上的一个 Pod 挂载。
- 云硬盘**不支持跨可用区挂载**，Pod 所在节点的可用区必须与云硬盘所在可用区一致。

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
#    cbs:DescribeDisks, cbs:CreateDisks, cbs:AttachDisks, cbs:DetachDisks
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

# 5. 确认 kubectl 可达
kubectl get ns
# expected: 返回命名空间列表
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
| 安装 CBS-CSI 组件 | `InstallAddon --AddonName CBS` | 否 |
| 查询组件安装状态 | `DescribeAddon --AddonName CBS` | 是 |
| 查看已有 CBS 云硬盘 | `cbs DescribeDisks` | 是 |
| 动态 StorageClass / PVC | 见 [StorageClass 管理云硬盘模板](../StorageClass管理云硬盘模板/tccli%20操作.md) | — |
| 静态 PV / PVC / Workload | 见 [PV 和 PVC 管理云硬盘](../PV和PVC管理云硬盘/tccli%20操作.md) | — |
| 扩容云硬盘 | `kubectl patch pvc`（修改 storage 字段） | 否 |

## 操作步骤

### 步骤 1：安装 CBS-CSI 组件

#### 选择依据

- **AddonName**：固定为 `CBS`，对应 CBS-CSI 驱动（`com.tencent.cloud.csi.cbs`）。CBS-CSI 是 TKE 托管集群默认预装的组件之一，大多数集群已安装。
- **AddonVersion**：通过 `DescribeAddonValues` 查询当前集群可用的最新版本。

```bash
# 查询可用版本
tccli tke DescribeAddonValues --region <Region> --ClusterId CLUSTER_ID --AddonName CBS
# expected: 返回可用版本列表
```

```json
{
  "Values": "<Values>",
  "DefaultValues": "<DefaultValues>",
  "RequestId": "<RequestId>"
}
```

#### 安装命令（如未预装）

```bash
tccli tke InstallAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName CBS \
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
| `ADDON_VERSION` | CBS-CSI 组件版本 | `tccli tke DescribeAddonValues --region <Region> --ClusterId CLUSTER_ID --AddonName CBS` |

### 步骤 2：轮询组件安装状态

```bash
tccli tke DescribeAddon --region <Region> --ClusterId CLUSTER_ID --AddonName CBS
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

### 后续操作

- **动态创建**：见 [StorageClass 管理云硬盘模板](../StorageClass管理云硬盘模板/tccli%20操作.md)
- **使用已有云硬盘**：见 [PV 和 PVC 管理云硬盘](../PV和PVC管理云硬盘/tccli%20操作.md)

## 验证

### 控制面（tccli）

| 维度 | 命令 | 预期 |
|------|------|------|
| 组件状态 | `tccli tke DescribeAddon --region <Region> --ClusterId CLUSTER_ID --AddonName CBS` | `Phase: "Succeeded"`, `Status: "Running"` |
| CBS 存量 | `tccli cbs DescribeDisks --region <Region>` | 可列出云硬盘列表 |

### 数据面（kubectl）

```bash
kubectl get pods -n kube-system | grep cbs
# expected: cbs-csi 相关 Pod 均为 Running

kubectl get sc
# expected: 列表中含默认 cbs StorageClass
```

```text
NAME  STATUS  AGE
...
```

## 清理

本页仅安装 CBS-CSI 组件。如需卸载：

```bash
# 卸载前检查：确认无 PVC 使用 CBS StorageClass
kubectl get pvc -A | grep cbs
# 若有结果，先删除对应 PVC 和 Pod

tccli tke DeleteAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName CBS
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
| `InstallAddon` 返回 "addon version not found" | `tccli tke DescribeAddonValues --region <Region> --ClusterId CLUSTER_ID --AddonName CBS` 查看可用版本 | 指定的 `AddonVersion` 不存在 | 从 `DescribeAddonValues` 输出取正确版本号 |
| `InstallAddon` 返回 `InvalidParameter.AddonName` | 检查 `AddonName` 字段 | 填写了小写 `cbs` 或其他非标准名称 | 使用 `--AddonName CBS`（大写） |
| `cbs DescribeDisks` 返回 `UnauthorizedOperation` | 检查 CAM 策略 | `cbs:DescribeDisks` 权限缺失（此为环境限制） | 联系 CAM 管理员授予 `cbs:DescribeDisks` 权限 |

### 挂载失败

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Pod 挂载云硬盘失败（`Multi-Attach` 错误） | `kubectl describe pod POD_NAME` 查看 Events | 同一云硬盘被挂载到两个不同节点的 Pod | CBS 仅支持 ReadWriteOnce，不要在多个 Deployment 中引用同一 PVC |
| Pod 挂载失败，Events 报 "zone mismatch" | `tccli cbs DescribeDisks --region <Region> --DiskIds '["DISK_ID"]'` 查看云盘可用区 | Pod 所在节点可用区与云硬盘可用区不一致 | 使用 `WaitForFirstConsumer` 的 StorageClass，或确保 Pod 调度到与云盘同可用区的节点 |
| PVC Pending（动态创建） | `kubectl describe pvc PVC_NAME` 查看 Events | CBS-CSI 未安装或 StorageClass 不存在 | 确认 CBS-CSI `Phase: "Succeeded"`，且 StorageClass 已创建 |

## 下一步

- [StorageClass 管理云硬盘模板](../StorageClass管理云硬盘模板/tccli%20操作.md)
- [PV 和 PVC 管理云硬盘](../PV和PVC管理云硬盘/tccli%20操作.md)
- [PV 和 PVC 的绑定规则](../../PV和PVC的绑定规则/tccli%20操作.md)

## 控制台替代

[控制台 → 集群 → 组件管理](https://console.cloud.tencent.com/tke2/cluster) 安装 CBS-CSI 组件；[CBS 控制台](https://console.cloud.tencent.com/cvm/cbs) 管理云硬盘。
