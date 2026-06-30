# 普通节点支持的 CVM 机型（tccli）

> 对照官方：[普通节点支持的 CVM 机型](https://cloud.tencent.com/document/product/457/84660) · page_id `84660`

## 概述

普通节点池可选机型受**可用区库存**、**GPU/非 GPU 混用规则**等限制。本文介绍如何通过 tccli 查询可用区可售卖机型、可加入集群的已有 CVM 实例、TKE 支持的 OS 镜像列表，帮助你在创建节点池前完成选型。

关键约束：

- 同一节点池内 GPU 机型与 CPU 机型**不可混用**
- 同一节点池最多可选 **10 种机型**
- 机型可售卖状态取决于可用区库存，需确认 `Status=SELL` 且 `StatusCategory` 为 `EnoughStock` 或 `NormalStock`
- 可用 `DescribeExistedInstances` 查看可加入集群的已有 CVM 实例，结合节点池 `LaunchConfiguration`/`GPUArgs` 与 [CVM 实例规格](https://cloud.tencent.com/document/product/213/11518) 选型

## 前置条件

- [环境准备](../../../../环境准备.md) · [tccli 专页（TKE）](../../../../tccli 专页（TKE）.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置，region 为目标地域

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:DescribeClusters, tke:DescribeExistedInstances, tke:DescribeOSImages
#    cvm:DescribeZoneInstanceConfigInfos
# 验证：执行 DescribeClusters 确认 TKE 权限
tccli tke DescribeClusters --region ap-guangzhou
# expected: exit 0，返回集群列表（可为空）

# 验证 CVM 权限
tccli cvm DescribeZoneInstanceConfigInfos --region ap-guangzhou --Filters '[{"Name":"zone","Values":["ap-guangzhou-6"]}]'
# expected: exit 0，返回 InstanceTypeQuotaSet
```

### 资源检查

```bash
# 4. 确认目标集群状态
tccli tke DescribeClusters --ClusterIds '["<ClusterId>"]' --region ap-guangzhou
# expected: ClusterStatus: "Running"

# 5. 获取集群所在可用区（从子网信息推导）
tccli tke DescribeClusterNodePools --ClusterId <ClusterId> --region ap-guangzhou
# expected: 返回 NodePoolSet，含 SubnetIds 字段
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查询可用区可售卖机型 | `cvm DescribeZoneInstanceConfigInfos` | 是 |
| 按实例族筛选机型 | `cvm DescribeZoneInstanceConfigInfos --Filters instance-family` | 是 |
| 查询可加入集群的已有实例 | `tke DescribeExistedInstances` | 是 |
| 查询 TKE 支持的 OS 镜像 | `tke DescribeOSImages` | 是 |
| 查看集群状态 | `tke DescribeClusters` | 是 |
| 查看节点池机型与 GPU 配置 | `tke DescribeClusterNodePoolDetail` | 是 |

## 操作步骤

### 选择依据

**可用区选择**：选择集群所在可用区进行机型查询，确保查询结果与实际可部署节点一致。可用区决定了哪些实例族有库存。

检查集群可用区：
```bash
tccli tke DescribeClusterNodePools --ClusterId <ClusterId> --region ap-guangzhou
# expected: 返回 NodePoolSet，含 SubnetIds 字段
```
或从 `DescribeClusters` 的 `ClusterNetworkSettings` 获取子网信息，再通过 `tccli vpc DescribeSubnets` 确认可用区。

**实例族选择**：标准型（S/SA 系列）通用工作负载、性价比最高，是 TKE 普通节点最常用选择。计算型 C 适合 CPU 密集型（如编译、渲染），内存型 M/MA 适合内存密集型（如缓存、数据库）。GPU 机型需额外参数配置。

> **反模式**：GPU 机型（GN/PTX/GT 系列）在节点池创建时需额外参数（如 `GPUArgs.Driver`），不可与 CPU 机型在同一节点池混用。GPU 节点池和 CPU 节点池须分开创建，普通节点池不使用 `GPUArgs` 参数。

**OS 镜像选择**：TencentOS 与 TKE 集成最好（预装容器运行时），Ubuntu 22.04 LTS 长期支持稳定、社区活跃。建议优先使用 TencentOS（tlinux3）或 Ubuntu 22.04。

检查可用 OS 镜像：
```bash
tccli tke DescribeOSImages --region ap-guangzhou
# expected: 返回 OSImageSeriesSet，全部 Status 为 online
```

### 1. 查询可用区全部 CVM 机型

推荐先查全量，了解可用实例族范围：

```bash
tccli cvm DescribeZoneInstanceConfigInfos --region ap-guangzhou \
    --Filters '[{"Name":"zone","Values":["ap-guangzhou-6"]}]'
# expected: exit 0，返回 TotalCount 和 InstanceTypeQuotaSet
```

输出（2143 条机型，669 种规格，90 个实例族，此处展示样例记录）：

```json
{
  "TotalCount": 2143,
  "InstanceTypeQuotaSet": [
    {
      "Zone": "ap-guangzhou-6",
      "InstanceType": "SA9.MEDIUM2",
      "InstanceFamily": "SA9",
      "TypeName": "标准型SA9",
      "Cpu": 2,
      "Memory": 2,
      "Gpu": 0,
      "CpuType": "AMD EPYC Turin-D",
      "Frequency": "-/3.4GHz",
      "Status": "SELL",
      "StatusCategory": "EnoughStock",
      "NetworkCard": 200
    }
  ],
  "RequestId": "<RequestId>"
}
```

主要实例族分布（ap-guangzhou-6）：

| 类别 | 实例族（规格数） |
|------|----------------|
| 标准型 | S5(35), S6(16), S7(11), S8(15), S9(16), S9e(15), S9pro(15), SA2(24), SA3(24), SA4(31), SA5(22), SA9(18), SA9e(16) |
| 计算型 | C4(6), C5(14), C6(8) |
| 内存型 | M5(8), M6(6), M8(7), M9(6), MA2(3), MA3(5), MA4(5), MA5(5), MA9(6), M6ce(7), M9e(6), M9pro(6) |
| GPU 型 | GN7(17), GN10X(8), GN10Xp(12), GN7vw(13), PNV4(12), PNV5b(25), PNV6(5), PTX2(18), GT4(12) |
| 其他 | IT5(4), ITA4(5), ITA5(4), DA4m(17) 等共 90 族 |

> **提示**：全量查询返回 2000+ 条记录，输出量大。建议先查全量了解范围，再用实例族过滤器缩小结果（见步骤 2）。

### 2. 按实例族筛选常用 TKE 机型

聚焦目标实例族，缩减输出：

```bash
tccli cvm DescribeZoneInstanceConfigInfos --region ap-guangzhou \
    --Filters '[{"Name":"zone","Values":["ap-guangzhou-6"]},{"Name":"instance-family","Values":["S5","SA5","SA9","S8","S9","C6","M6","M9","MA5","MA9"]}]'
# expected: exit 0，TotalCount 约 100，覆盖 10 个实例族
```

输出（使用 `instance-family` 过滤器将输出从 2143 条缩减至 106 条）：

```json
{
  "TotalCount": 106,
  "InstanceTypeQuotaSet": [
    {
      "Zone": "ap-guangzhou-6",
      "InstanceType": "S5.MEDIUM2",
      "InstanceFamily": "S5",
      "TypeName": "标准型S5",
      "Cpu": 2,
      "Memory": 2,
      "Status": "SELL",
      "StatusCategory": "EnoughStock"
    }
  ],
  "RequestId": "<RequestId>"
}
```

找到的实例族：S5, SA5, SA9, S8, S9, C6, M6, M9, MA5, MA9。

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `ap-guangzhou-6` | 可用区 ID | 集群所在可用区 | `DescribeClusters` → 子网 → `vpc DescribeSubnets` |
| `S5` / `SA9` 等 | 实例族代号 | 大小写敏感，需与 `InstanceFamily` 字段一致 | 步骤 1 全量查询结果 |

### 3. 查询可加入集群的已有 CVM 实例

```bash
tccli tke DescribeExistedInstances --ClusterId <ClusterId> --region ap-guangzhou
# expected: exit 0，返回 ExistedInstanceSet
```

输出（2 条实例，此处展示样例）：

```json
{
  "TotalCount": 2,
  "ExistedInstanceSet": [
    {
      "Usable": false,
      "UnusableReason": "already in Cluster",
      "AlreadyInCluster": "cls-example",
      "InstanceId": "ins-example",
      "InstanceName": "example-instance",
      "InstanceType": "S5.MEDIUM2",
      "CPU": 2,
      "Memory": 2,
      "OsName": "Ubuntu Server 22.04 LTS 64bit",
      "AutoscalingGroupId": "asg-example"
    }
  ],
  "RequestId": "<RequestId>"
}
```

关键字段说明：

| 字段 | 说明 |
|------|------|
| `Usable` | 是否可加入集群。`false` 表示不可用，参见 `UnusableReason` |
| `UnusableReason` | 不可用原因，如 `already in Cluster` |
| `AlreadyInCluster` | 如已在某集群中，显示该集群 ID |
| `InstanceType` | 实例规格，如 `S5.MEDIUM2` |
| `AutoscalingGroupId` | 关联的伸缩组 ID |

> **提示**：如需只查看可加入集群的实例（`Usable: true`），可用 jq 过滤：`tccli tke DescribeExistedInstances --ClusterId <ClusterId> --region ap-guangzhou | jq '.ExistedInstanceSet[] | select(.Usable == true)'`

### 4. 查询 TKE 支持的 OS 镜像列表

```bash
tccli tke DescribeOSImages --region ap-guangzhou
# expected: exit 0，返回 OSImageSeriesSet，全部 Status 为 online
```

输出（60 个镜像，全部 online）：

```json
{
  "TotalCount": 60,
  "OSImageSeriesSet": [
    {
      "OsName": "tlinux3.1x86_64",
      "OsNameShow": "TencentOS Server 3.1",
      "Status": "online"
    },
    {
      "OsName": "ubuntu22.04x86_64",
      "OsNameShow": "Ubuntu Server 22.04 LTS 64bit",
      "Status": "online"
    }
  ],
  "RequestId": "<RequestId>"
}
```

OS 镜像分布（全部 `Status: online`）：

| OS 系列 | 镜像示例（数量） |
|---------|----------------|
| TencentOS | tlinux2.4(tkernel4)x86_64_HCC, tlinux3.1x86_64, tlinux4_x86_64_public_uefi 等 20 个 |
| Ubuntu | ubuntu16.04.1 LTSx86_64, ubuntu18.04.1x86_64, ubuntu20.04x86_64, ubuntu22.04x86_64, ubuntu24.04x86_64 等 12 个 |
| CentOS | centos7.2x86_64, centos7.6.0_x64, centos8.0x86_64 等 13 个 |
| Debian | debian11.11x86_64, debian12.8x86_64（2 个） |
| RedHat | redhat7.9x86_64, redhat8.9x86_64, redhat9.5x86_64（3 个） |
| 其他 | rockylinux9.x(2), opencloudos9.0_x86_64(1), kylin10x86_64(1) |

> **建议**：优先使用 TencentOS 或 Ubuntu 22.04+。

### 5. 确认集群状态（上下文验证）

```bash
tccli tke DescribeClusters --ClusterIds '["<ClusterId>"]' --region ap-guangzhou
# expected: exit 0，ClusterStatus: "Running"
```

```json
{
  "TotalCount": 1,
  "Clusters": [
    {
      "ClusterId": "cls-example",
      "ClusterStatus": "Running",
      "ClusterVersion": "1.32.2",
      "ClusterType": "MANAGED_CLUSTER",
      "ClusterNetworkSettings": {
        "VpcId": "vpc-example"
      }
    }
  ],
  "RequestId": "<RequestId>"
}
```

## 验证

| 维度 | 命令 | 预期 |
|------|------|------|
| 机型可售卖 | `cvm DescribeZoneInstanceConfigInfos` 检查目标机型 | `Status: "SELL"` 且 `StatusCategory: "EnoughStock"` 或 `"NormalStock"` |
| 实例可加入 | `tke DescribeExistedInstances` 检查 `Usable` | `Usable: true`；若 `false` 查看 `UnusableReason` |
| OS 镜像可用 | `tke DescribeOSImages` 检查 `Status` | `Status: "online"` |
| 集群状态 | `tke DescribeClusters` 检查 `ClusterStatus` | `"Running"` |

## 清理

本页全部为只读查询操作，无资源创建，无需清理。

## 排障

### 查询结果异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DescribeZoneInstanceConfigInfos` 返回 2000+ 条记录，输出量太大难以阅读 | 检查是否未加 `instance-family` 过滤器 | 全量查询返回所有实例族机型 | 加 `instance-family` 过滤器：`--Filters '[{"Name":"instance-family","Values":["S5","SA9"]}]'`，可将输出从 2143 条缩减至约 100 条 |
| 创建节点池时报实例规格不可用或售罄 | `tccli cvm DescribeZoneInstanceConfigInfos --region ap-guangzhou --Filters '[{"Name":"zone","Values":["ap-guangzhou-6"]},{"Name":"instance-family","Values":["S5"]}]'` 检查目标机型 `Status` | 未提前确认机型在目标可用区是否可售卖 | 换有库存的机型或可用区。确认 `Status=SELL` 且 `StatusCategory=EnoughStock` 或 `NormalStock` 后再创建节点池 |
| 节点池 `GPUArgs` 与普通机型冲突 | `tccli tke DescribeClusterNodePoolDetail --ClusterId <ClusterId> --NodePoolId <NodePoolId> --region ap-guangzhou` 检查节点池 `LaunchConfiguration` 中机型列表 | GPU 机型（GN/PTX/GT 系列）与 CPU 机型在同一节点池混用 | GPU 节点池和 CPU 节点池分开创建。普通节点池不使用 `GPUArgs` 参数 |

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DescribeExistedInstances` 返回空列表 | `tccli tke DescribeClusters --ClusterIds '["<ClusterId>"]' --region ap-guangzhou` 确认集群存在且 Running | 集群 ID 错误或集群状态异常 | 确认 `ClusterId` 正确，集群状态为 `Running` |
| `DescribeZoneInstanceConfigInfos` 返回 `UnauthorizedOperation` | 检查 CAM 权限是否含 `cvm:DescribeZoneInstanceConfigInfos` | 缺少 CVM 读权限（环境限制，非命令错误） | 在 CAM 策略中添加 `cvm:DescribeZoneInstanceConfigInfos` Action |
| `DescribeOSImages` 返回空列表 | 确认 `--region` 参数与集群地域一致 | 地域不匹配 | 使用与集群相同的 `--region` 参数 |

## 下一步

- [创建节点池](../创建节点池/tccli 操作.md)
- [新增节点](../../常用操作/新增节点/tccli 操作.md)
- [CVM 实例规格](https://cloud.tencent.com/document/product/213/11518)

## 控制台替代

容器服务控制台 → **集群** → 目标集群 → **节点管理**，按[官方文档](https://cloud.tencent.com/document/product/457/84660)对应菜单操作。
