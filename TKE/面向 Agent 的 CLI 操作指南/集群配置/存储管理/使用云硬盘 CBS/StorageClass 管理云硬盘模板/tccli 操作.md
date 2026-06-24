# StorageClass 管理云硬盘模板

> 对照官方：[StorageClass 管理云硬盘模板](https://cloud.tencent.com/document/product/457/44239) · page_id `44239` · tccli ≥3.1.107 · API 2017-03-12

## 概述

通过 kubectl 创建 CBS 类型的 StorageClass，自定义云硬盘的参数模板（磁盘类型、计费模式、回收策略等）。后续 PVC 可引用此 StorageClass 动态创建符合模板规格的云硬盘。

TKE 集群默认预装 CBS-CSI 组件，同时提供一个名为 `cbs` 的默认 StorageClass（高性能云硬盘、按量计费、立即绑定）。本文演示如何**自定义 StorageClass** 覆盖默认参数。

**与静态 PV 对比**：

| 维度 | 动态创建（StorageClass） | 静态创建（已有云盘） |
|------|---------------------|----------------|
| 云硬盘来源 | PVC 创建时自动新建 | 需提前在 CBS 创建 |
| 可用区匹配 | `WaitForFirstConsumer` 确保与 Pod 同可用区 | 需手动确认 |
| 管控粒度 | StorageClass 统一模板 | 每个 PV 独立配置 |
| 适合场景 | 按需自动供给 | 存量云盘复用、数据迁移 |

## 前置条件

- [环境准备](../../../../环境准备.md)
- 已安装 CBS-CSI 组件：[云硬盘使用说明](../云硬盘使用说明/tccli%20操作.md)
- kubectl 已连接目标集群

### 环境检查

```bash
# 1. 确认 kubectl 可达
kubectl get ns
# expected: 返回 default 等系统命名空间

# 2. 确认 CBS-CSI 已安装
tccli tke DescribeAddon --region <REGION> --ClusterId <CLUSTER_ID> --AddonName CBS
# expected: Phase: "Succeeded"，AddonVersion 格式如 "1.1.15"
```

**预期输出**：

```json
{
    "Addons": [
        {
            "AddonName": "cbs",
            "AddonVersion": "1.1.15",
            "Phase": "Succeeded"
        }
    ],
    "RequestId": "..."
}
```

### 资源检查

```bash
# 3. 查看默认 cbs StorageClass
kubectl get sc cbs
# expected: 如集群已预装则返回默认 CBS StorageClass

# 4. 查看已存在的自定义 StorageClass（避免命名冲突）
kubectl get sc
# expected: 列出当前集群全部 StorageClass，确认新名称不重复
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli/kubectl 命令 | 幂等 |
|-----------|-------------------|:--:|
| 查看 CBS-CSI 状态 | `DescribeAddon --AddonName CBS` | 是 |
| 查看 StorageClass 列表 | `kubectl get sc` | 是 |
| 创建 StorageClass | `kubectl apply -f cbs-sc.yaml` | 否 |
| 查看 StorageClass 详情 | `kubectl describe sc <SC_NAME>` | 是 |
| 删除 StorageClass | `kubectl delete sc <SC_NAME>` | 是 |

## 关键字段说明

以下说明 CBS StorageClass YAML 中的关键参数。

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `provisioner` | String | 是 | 固定值 `com.tencent.cloud.csi.cbs`（CBS-CSI 组件）或 `cloud.tencent.com/qcloud-cbs`（旧版，1.20 后废弃） | 填错 → PVC 一直 Pending，Events 报 "provisioner not found" |
| `parameters.diskType` | String | 是 | `CLOUD_PREMIUM`（高性能云硬盘）、`CLOUD_SSD`（SSD 云硬盘）、`CLOUD_BSSD`（通用型 SSD 云硬盘）、`CLOUD_HSSD`（增强型 SSD 云硬盘） | 值不存在或地域不支持 → CBS 创建失败，PVC 卡 Pending |
| `parameters.diskChargeType` | String | 否 | `POSTPAID_BY_HOUR`（按量计费，默认）或 `PREPAID`（包年包月） | `PREPAID` 需额外配置 `diskChargePrepaidPeriod` 和 `diskChargePrepaidRenewFlag` |
| `parameters.diskZone` | String | 否 | 可用区标识符，如 `ap-guangzhou-6`。不指定则随机选取 Node 的可用区 | 指定了不存在的可用区 → CBS 创建失败 |
| `reclaimPolicy` | String | 否 | `Delete`（PVC 删则云盘删）或 `Retain`（保留云盘）。默认 `Delete` | `Delete` 误删 PVC → 云盘数据永久丢失，不可恢复 |
| `volumeBindingMode` | String | 否 | `Immediate`（PVC 创建后立即绑定）或 `WaitForFirstConsumer`（Pod 调度后才创建云盘）。默认 `Immediate` | `Immediate` 可能创建在错误的可用区 → Pod 无法挂载（Node 不在同一可用区） |
| `allowVolumeExpansion` | Boolean | 否 | `true`（允许在线扩容）或 `false`（默认） | 设为 `false` 后 PVC 无法扩容，需删除重建 |
| `parameters.aspid` | String | 否 | 快照策略 ID，如 `asp-xxxxxxxx` | 快照策略绑定失败不影响创建 |

## 操作步骤

### 步骤 1：查看已有 StorageClass

创建前先检查当前集群中已有的 StorageClass，避免名称冲突。

```bash
kubectl get sc
# expected: 列出全部 StorageClass，默认含 cbs
```

**预期输出**：

```text
NAME                PROVISIONER                  RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
cbs (default)       com.tencent.cloud.csi.cbs    Delete          Immediate              false                  2d
```

### 步骤 2：创建自定义 StorageClass

#### 选择依据

- **provisioner**：集群已安装 CBS-CSI 组件（通过 `DescribeAddon` 确认），因此使用 `com.tencent.cloud.csi.cbs`。旧版 `cloud.tencent.com/qcloud-cbs` 已在 1.20 后废弃，集群版本 1.32.2 不适用。
- **diskType**：`CLOUD_PREMIUM`（高性能云硬盘）适合大多数通用负载，性价比最高。数据库类低延迟负载选 `CLOUD_SSD`。
- **volumeBindingMode**：强烈推荐 `WaitForFirstConsumer`。`Immediate` 在 PVC 创建时立即分配云盘，但此时 Pod 尚未调度，可能创建在错误的可用区导致 Pod 无法挂载。
- **reclaimPolicy**：推荐 `Retain`。`Delete` 策略下 PVC 被删除时云盘和所有数据同步销毁，不可恢复。
- **allowVolumeExpansion**：推荐 `true`，允许后续在线扩容，无需重建 PVC。

#### 最小配置

`cbs-sc-minimal.yaml`：

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: cbs-kerwinwjyan
provisioner: com.tencent.cloud.csi.cbs
parameters:
  diskType: CLOUD_PREMIUM
reclaimPolicy: Retain
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

```bash
kubectl apply -f cbs-sc-minimal.yaml
# expected: storageclass.storage.k8s.io/cbs-kerwinwjyan created
```

#### 增强配置（指定可用区 + 按量计费 + 快照策略）

`cbs-sc-enhanced.yaml`：

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: cbs-kerwinwjyan-ssd
  labels:
    env: production
provisioner: com.tencent.cloud.csi.cbs
parameters:
  diskType: CLOUD_SSD
  diskZone: <ZONE>
  diskChargeType: POSTPAID_BY_HOUR
  aspid: <ASP_ID>
reclaimPolicy: Retain
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

```bash
kubectl apply -f cbs-sc-enhanced.yaml
# expected: storageclass.storage.k8s.io/cbs-kerwinwjyan-ssd created
```

### 步骤 3：验证 StorageClass 已创建

```bash
kubectl get sc cbs-kerwinwjyan
# expected: NAME: cbs-kerwinwjyan, PROVISIONER: com.tencent.cloud.csi.cbs
```

**预期输出**：

```text
NAME                PROVISIONER                  RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
cbs-kerwinwjyan     com.tencent.cloud.csi.cbs    Retain          WaitForFirstConsumer   true                   5s
```

### 步骤 4：查看 StorageClass 详细配置

```bash
kubectl describe sc cbs-kerwinwjyan
# expected: 显示 Name、Provisioner、Parameters、ReclaimPolicy 等完整信息
```

**预期输出**：

```text
Name:                  cbs-kerwinwjyan
IsDefaultClass:        No
Annotations:           <none>
Provisioner:           com.tencent.cloud.csi.cbs
Parameters:            diskType=CLOUD_PREMIUM
AllowVolumeExpansion:  True
MountOptions:          <none>
ReclaimPolicy:         Retain
VolumeBindingMode:     WaitForFirstConsumer
Events:                <none>
```

## 验证

| 维度 | 命令 | 预期 |
|------|------|------|
| StorageClass 已创建 | `kubectl get sc cbs-kerwinwjyan` | NAME: `cbs-kerwinwjyan`，PROVISIONER: `com.tencent.cloud.csi.cbs` |
| 参数正确 | `kubectl describe sc cbs-kerwinwjyan` | Parameters 含 `diskType=CLOUD_PREMIUM`，ReclaimPolicy: `Retain`，VolumeBindingMode: `WaitForFirstConsumer` |
| CBS-CSI 组件正常 | `tccli tke DescribeAddon --region <REGION> --ClusterId <CLUSTER_ID> --AddonName CBS` | Phase: `Succeeded` |

## 清理

> **警告**：删除 StorageClass 不会影响已通过它创建的 PVC 和 PV——这些资源会继续存在并可正常使用。但后续无法再通过该 StorageClass 创建新的 PVC。
>
> `reclaimPolicy: Delete` 的 StorageClass 下，删除 PVC 会同步删除 CBS 云硬盘及所有数据。`Retain` 策略下删除 PVC 后 PV 状态变为 `Released`，云硬盘保留并持续计费，需在 [CBS 控制台](https://console.cloud.tencent.com/cvm/cbs) 手动退还。

### 1. 清理前状态检查

```bash
kubectl get sc cbs-kerwinwjyan
# 确认目标 StorageClass 存在，且无 PVC 依赖（如无关则直接删）
```

### 2. 删除自定义 StorageClass

```bash
kubectl delete sc cbs-kerwinwjyan
# expected: storageclass.storage.k8s.io "cbs-kerwinwjyan" deleted
```

### 3. 验证已删除

```bash
kubectl get sc cbs-kerwinwjyan 2>&1
# expected: Error from server (NotFound): storageclasses.storage.k8s.io "cbs-kerwinwjyan" not found
```

**预期输出**：

```text
Error from server (NotFound): storageclasses.storage.k8s.io "cbs-kerwinwjyan" not found
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl apply -f cbs-sc.yaml` 返回 "provisioner not found" | `kubectl get csidriver \| grep cbs` 检查 CSI 驱动是否已注册 | CBS-CSI 组件未安装或未正常运行 | 确认 CBS-CSI 已安装：`tccli tke DescribeAddon --region <REGION> --ClusterId <CLUSTER_ID> --AddonName CBS`，确保 Phase 为 `Succeeded` |
| `kubectl apply -f cbs-sc.yaml` 返回 YAML 语法错误 | 检查 YAML 缩进（须用空格，非 Tab）和字段名拼写 | YAML 格式不规范或字段名拼写错误 | 对照 `kubectl explain sc` 检查字段名。缩进用 2 空格，不用 Tab |
| `kubectl get sc cbs-kerwinwjyan` 返回 NotFound | `kubectl get sc` 查看全部 SC 列表，确认是否创建成功 | StorageClass 创建未成功或名称不匹配 | 重新 `kubectl apply -f cbs-sc-minimal.yaml`，确认输出为 "created" |
| kubectl 命令返回 "Unable to connect to the server" | 检查集群端点状态：`tccli tke DescribeClusterEndpoints --region <REGION> --ClusterId <CLUSTER_ID>` | 集群端点不可达——公网端点可能因 CAM 策略被拒绝创建，内网端点需要 IOA/VPN/专线 或同 VPC CVM 才能访问 | 确保在集群端点可达的环境（IOA/VPN/同 VPC CVM）中执行 kubectl 命令。如 extranet 端点被 CAM 策略 `tke:clusterExtranetEndpoint=true → effect:deny` 硬拒绝，需通过内网端点 + VPN/IOA 或同 VPC 跳板机访问 |

### StorageClass 创建后使用时异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 引用此 SC 的 PVC 一直 Pending | `kubectl describe pvc <PVC_NAME>` 查看 Events | Provisioner 未就绪或 CBS 创建失败 | 检查 Events 中错误信息。常见：`diskType not supported`（换可用区或类型）、CBS 配额满（提工单扩容）、计费余额不足（充值） |
| PVC Bound 但 Pod 挂载失败（可用区不匹配） | `kubectl describe pod <POD_NAME>` 查看 Events，检查 Node 可用区与 CBS 可用区是否一致 | `volumeBindingMode: Immediate` 导致云盘创建在非预期的可用区 | 删除 PVC，改用 `WaitForFirstConsumer` 模式重建。`Immediate` 仅在同可用区场景使用 |
| PVC 使用了非预期的 StorageClass | `kubectl get pvc <PVC_NAME> -o yaml \| grep storageClassName` | 未在 PVC 中明确指定 `storageClassName`，或使用了默认 `cbs` StorageClass | 在 PVC YAML 中明确指定 `storageClassName: cbs-kerwinwjyan` |

## 下一步

- [PV 和 PVC 管理云硬盘](../PV%20和%20PVC%20管理云硬盘/tccli%20操作.md)
- [PV 和 PVC 的绑定规则](../../PV%20和%20PVC%20的绑定规则/tccli%20操作.md)
- [使用文件存储 CFS](../../使用文件存储%20CFS/StorageClass%20管理文件存储模板/tccli%20操作.md)

## 控制台替代

[控制台 → 集群 → 存储 → StorageClass](https://console.cloud.tencent.com/tke2/cluster) 新建 StorageClass，选择 CBS 模板，配置磁盘类型、计费模式、回收策略后创建。
