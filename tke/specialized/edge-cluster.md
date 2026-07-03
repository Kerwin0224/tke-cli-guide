---
doc_type: How-to
subtype: 6A
fused: true
---
# 边缘集群 (TKEEdge)

> 创建和管理 TKE 边缘集群 —— 适合边缘计算场景的轻量级 K8s 集群。
> 控制台: [容器服务 - 边缘集群](https://console.cloud.tencent.com/tke2/edge)

## 概述

边缘集群是 TKE 为边缘计算场景提供的 K8s 集群类型，运行在边缘节点上。与标准集群的主要区别:

| 特性 | 边缘集群 (TKEEdge) | 标准集群 (TKE) |
|------|:---:|:---:|
| 节点位置 | 边缘/IDC/门店 | 腾讯云 CVM |
| 网络 | 公网/弱网自适应 | VPC 内网 |
| Master | 托管（云端） | 托管或独立 |
| 适用场景 | IoT、边缘 AI、CDN | 标准 Web 服务 |

## 关键操作

### 创建边缘集群

```bash
tccli tke CreateTKEEdgeCluster \
  --region ap-guangzhou \
  --ClusterName "<NAME>-edge" \
  --ClusterVersion "1.28.3" \
  --VpcId "<VPC_ID>" \
  --SubnetId "<SUBNET_ID>"
# expected: { "ClusterId": "cls-xxxxxxxx" }
```

### 查询边缘集群

```bash
# 列出所有边缘集群
tccli tke DescribeTKEEdgeClusters --region ap-guangzhou
# expected: { "TotalCount": ..., "Clusters": [...] }

# 查询状态
tccli tke DescribeTKEEdgeClusterStatus --region ap-guangzhou --ClusterId "<CLUSTER_ID>"
# expected: ClusterState: "Running"

# 获取凭证
tccli tke DescribeTKEEdgeClusterCredential --region ap-guangzhou --ClusterId "<CLUSTER_ID>"
# expected: 返回 kubeconfig
```

### 获取注册脚本 (添加边缘节点)

边缘节点通过脚本注册到集群，而非通过 CVM:

```bash
tccli tke DescribeTKEEdgeScript --region ap-guangzhou --ClusterId "<CLUSTER_ID>"
# expected: 返回边缘节点安装脚本
```

在边缘节点上执行返回的脚本来注册节点。

### 升级边缘集群

```bash
# 1. 查询可升级版本
tccli tke DescribeEdgeClusterUpgradeInfo --region ap-guangzhou --ClusterId "<CLUSTER_ID>"
# expected: 返回可升级的目标版本列表

# 2. 执行升级
tccli tke UpdateEdgeClusterVersion --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" \
  --Version "<TARGET_VERSION>"
# expected: exit 0
```

### 边缘集群日志

```bash
# 创建日志配置
tccli tke CreateEdgeLogConfig --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" \
  --LogConfig "<LOG_CONFIG_JSON>"

# 查询日志开关
tccli tke DescribeEdgeLogSwitches --region ap-guangzhou --ClusterId "<CLUSTER_ID>"

# 安装日志 Agent
tccli tke InstallEdgeLogAgent --region ap-guangzhou --ClusterId "<CLUSTER_ID>"
```

## 清理

```bash
# 1. 删除边缘集群
tccli tke DeleteTKEEdgeCluster --region ap-guangzhou --ClusterId "<CLUSTER_ID>"
# expected: exit 0

# 2. 验证
tccli tke DescribeTKEEdgeClusters --region ap-guangzhou
# expected: TotalCount 减少，目标集群不在列表中
```

## API 参考

完整的边缘集群 API 共 21 个操作:

| 分类 | API | 说明 |
|------|-----|------|
| 生命周期 | `CreateTKEEdgeCluster` / `DeleteTKEEdgeCluster` / `UpdateTKEEdgeCluster` | 创建/删除/更新 |
| 查询 | `DescribeTKEEdgeClusters` / `DescribeTKEEdgeClusterStatus` | 列表/状态 |
| 凭证 | `DescribeTKEEdgeClusterCredential` / `DescribeTKEEdgeExternalKubeconfig` | kubeconfig |
| 节点 | `DescribeEdgeClusterInstances` / `DescribeTKEEdgeScript` | 节点/注册脚本 |
| 升级 | `DescribeEdgeClusterUpgradeInfo` / `UpdateEdgeClusterVersion` / `DescribeAvailableTKEEdgeVersion` | 版本管理 |
| 日志 | `CreateEdgeLogConfig` / `DescribeEdgeLogSwitches` / `InstallEdgeLogAgent` / `UninstallEdgeLogAgent` | 日志采集 |
| 其他 | `ForwardTKEEdgeApplicationRequestV3` / `CheckEdgeClusterCIDR` | 应用转发/CIDR 检查 |
| 参数 | `DescribeEdgeAvailableExtraArgs` / `DescribeEdgeClusterExtraArgs` | 集群参数 |

## 集群更新与诊断

> 更新边缘集群属性、查可用版本与额外参数、外部 kubeconfig、CIDR 冲突检查、日志 Agent 卸载。参数以 `--generate-cli-skeleton` 为准（注意 Edge 域 `ClusterID` 大写 vs `ClusterId` 小写不一致）。
>
> ⚠️ Edge 域多数 action 在 ap-guangzhou 返回 `UnsupportedRegion`（`DescribeEdgeClusterExtraArgs`/`CheckEdgeClusterCIDR`/`DescribeEdgeClusterInstances`）或 CAM 拦截（`DescribeEdgeAvailableExtraArgs`/`UpdateTKEEdgeCluster`/`UninstallEdgeLogAgent` 返回 `UnauthorizedOperation`/`AuthFailure.UnauthorizedOperation`）。仅 `DescribeAvailableTKEEdgeVersion` 可用（但 action 已废弃，edge 产品已下线）。下方命令参数名已验证（错误是地域/CAM/资源，非参数名）。

```bash
# 更新边缘集群属性（ClusterId 小写 + ClusterName/ClusterDesc/PodCIDR 等覆盖式）
tccli tke UpdateTKEEdgeCluster --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" --ClusterName "<NEW_NAME>" --ClusterDesc "<NEW_DESC>"
# expected: CAM 拦截 AuthFailure.UnauthorizedOperation（参数已验证）；授权后 exit 0

# 查询可升级到的边缘集群版本（无必填入参；action 已废弃，edge 产品下线）
tccli tke DescribeAvailableTKEEdgeVersion --region ap-guangzhou
# expected: exit 0，返回 Versions[]+EdgeVersionLatest+EdgeVersionCurrent（版本如 1.20.6/1.22.5/1.18.2）

# 查询集群已生效的额外参数（ClusterId 定位）
tccli tke DescribeEdgeClusterExtraArgs --region ap-guangzhou --ClusterId "<CLUSTER_ID>"
# expected:  UnsupportedRegion（ap-guangzhou 不支持）；Edge 地域可用时返回 ClusterExtraArgs

# 查询某版本可用的额外参数（ClusterVersion 定位，建集群/升级前用）
tccli tke DescribeEdgeAvailableExtraArgs --region ap-guangzhou --ClusterVersion "<VERSION>"
# expected: CAM 拦截 UnauthorizedOperation（参数已验证）；授权后返回 ClusterVersion+AvailableExtraArgs

# 获取外部 kubeconfig（ClusterId 定位，区别于 DescribeTKEEdgeClusterCredential）
tccli tke DescribeTKEEdgeExternalKubeconfig --region ap-guangzhou --ClusterId "<CLUSTER_ID>"
# expected: 返回 UnknownParameter: failed to connect to user cluster（非边缘集群连不上）；边缘集群可用时返回 Kubeconfig

# CIDR 冲突检查（建集群前用，VpcId + PodCIDR + ServiceCIDR 三参数）
tccli tke CheckEdgeClusterCIDR --region ap-guangzhou \
  --VpcId "<VPC_ID>" --PodCIDR "<POD_CIDR>" --ServiceCIDR "<SERVICE_CIDR>"
# expected:  UnsupportedRegion（ap-guangzhou 不支持）；Edge 地域可用时返回 ConflictCode+ConflictMsg

# 卸载边缘日志 Agent（ClusterId 定位，停止边缘日志采集）
tccli tke UninstallEdgeLogAgent --region ap-guangzhou --ClusterId "<CLUSTER_ID>"
# expected: CAM 拦截 AuthFailure.UnauthorizedOperation（参数已验证）；授权后 exit 0
```

```bash
# 查询边缘集群节点列表（注意 ClusterID 大写 + Filters/Offset/Limit）
tccli tke DescribeEdgeClusterInstances --ClusterID "<CLUSTER_ID>" --region <REGION> \
  --Offset 0 --Limit 20
# expected:  UnsupportedRegion（ap-guangzhou 不支持，需 Edge 地域）；可用时返回 TotalCount+InstanceInfoSet
```

> ⚠️ `DescribeEdgeClusterInstances` 用大写 `ClusterID`（同 `CreateEdgeCVMInstances`/`DescribeEdgeCVMInstances`），区别于 `UpdateTKEEdgeCluster`/`DescribeAvailableTKEEdgeVersion` 等的小写 `ClusterId`——Edge 域大小写不一致是真实契约，切换接口前用 `--generate-cli-skeleton` 核对。`CheckEdgeClusterCIDR` 不绑集群（建集群前用），需 `VpcId`+`PodCIDR`+`ServiceCIDR` 三参数。

## Edge 节点实例管理

> Edge 集群的 CVM 节点与 ECM（边缘计算模块）实例管理。⚠️ 这些 Action 用 `--ClusterID`（大写 ID），区别于其他 TKE 接口的 `--ClusterId`。

### Edge CVM 实例

```bash
# 创建 Edge CVM 实例 (RunInstancePara 透传 CVM JSON, CvmRegion/CvmCount)
tccli tke CreateEdgeCVMInstances --ClusterID "<CLUSTER_ID>" --region <REGION> \
  --RunInstancePara "<CVM_JSON>" --CvmRegion "<CVM_REGION>" --CvmCount 1
# expected: exit 0

# 查询 Edge CVM 实例 (Filters 过滤)
tccli tke DescribeEdgeCVMInstances --ClusterID "<CLUSTER_ID>" --region <REGION> \
  --Filters '[{"Name":"zone","Values":["<ZONE>"]}]'
# expected: exit 0, 实例列表

# 删除 Edge CVM 实例 (CvmIdSet[] 批量)
tccli tke DeleteEdgeCVMInstances --ClusterID "<CLUSTER_ID>" --region <REGION> --CvmIdSet '["<CVM_ID>"]'
# expected: exit 0

# 删除 Edge 集群节点 (InstanceIds[], 注意此接口用 ClusterId 小写)
tccli tke DeleteEdgeClusterInstances --ClusterId "<CLUSTER_ID>" --region <REGION> --InstanceIds '["<INSTANCE_ID>"]'
# expected: exit 0
```

> `CreateEdgeCVMInstances` 用 `ClusterID`（大写）+ `CvmRegion`（CVM 地域）+ `CvmCount`。`DeleteEdgeClusterInstances` 用 `ClusterId`（小写）+ `InstanceIds[]`——同域大小写不一致，切换接口时核对。

### ECM 实例（边缘计算模块）

```bash
# 创建 ECM 实例 (ModuleId + ZoneInstanceCountISPSet[] 按可用区/ISP 分配)
tccli tke CreateECMInstances --ClusterID "<CLUSTER_ID>" --region <REGION> \
  --ModuleId "<MODULE_ID>" \
  --ZoneInstanceCountISPSet '[{"Zone":"<ZONE>","InstanceCount":1,"ISP":"<ISP>"}]'
# expected: exit 0

# 查询 ECM 实例 (Filters 过滤)
tccli tke DescribeECMInstances --ClusterID "<CLUSTER_ID>" --region <REGION> \
  --Filters '[{"Name":"zone","Values":["<ZONE>"]}]'
# expected: exit 0

# 删除 ECM 实例 (EcmIdSet[] 批量)
tccli tke DeleteECMInstances --ClusterID "<CLUSTER_ID>" --region <REGION> --EcmIdSet '["<ECM_ID>"]'
# expected: exit 0
```

> ECM 用 `ModuleId`（边缘计算模块 ID）+ `ZoneInstanceCountISPSet[]`（按可用区+ISP 运营商分配）。`EcmIdSet[]`/`CvmIdSet[]` 是实例 ID 数组。

## 下一步

- [EKS 弹性集群](eks-cluster.md) — Serverless 集群与容器实例（对比边缘）
- [虚拟节点 (超级节点)](../nodes/virtual-nodes.md) — 标准集群内的 Serverless 节点
- [专用工作负载概览](index.md) — 边缘/EKS 选型
- [标准集群概览](../clusters/index.md) — 对比标准集群
