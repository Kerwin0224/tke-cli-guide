---
doc_type: How-to
subtype: 6A
fused: true
---
# 镜像加速

> 为 TCR 企业版实例开启镜像加速服务，加速镜像拉取。
> 控制台: [容器镜像服务 - 镜像加速](https://console.cloud.tencent.com/tcr/acceleration)

> ⚠️ **类型纠偏（P2 实测）**：设计期（D34 最小片内设计）曾判此为 6B Configure「开关型」，实测 `CreateImageAccelerationService` 含 6+ 参数（VpcId/SubnetId/StorageType/PGroupId/Zone）且创建 CFS 存储资源，多失败模式，应升为 **Fused 6A**。本篇按 Fused 6A 规格。

## 概述

镜像加速服务为 TCR 实例创建一个 CFS（云文件存储）加速后端，镜像层缓存在 CFS 中，拉取时从 CFS 就近读取，降低大镜像拉取延迟。`CFSVIP` 是加速后端访问入口。

## 决策依据

### 是否需要加速

| 场景 | 是否启用 | 说明 |
|:-----|:--------:|:-----|
| 大镜像频繁拉取（>1GB） | ✅ | CFS 缓存显著降低延迟 |
| 小镜像 / 拉取频次低 | ❌ | CFS 有存储成本，收益不显 |
| VPC 内网拉取为主 | ⚠️ | 内网本身较快，加速收益有限 |

### CFS 存储选型

`CreateImageAccelerationService` 的 `StorageType` 决定 CFS 类型：

| StorageType | 含义 | 适用 |
|:------------|:-----|:-----|
| `SD` | 通用标准型 | 通用场景，性价比高 |
| `HP` | 性能型 | 高吞吐需求 |

> `PGroupId` 是 CFS 权限组 ID，需先在 CFS 服务创建权限组。`Zone` 必须与 `SubnetId` 所在可用区一致。

## 关键字段

| 参数 | 所属 Action | 必填 | 说明 |
|:-----|:-----------|:----:|:-----|
| `RegistryId` | 全部 | 是 | TCR 实例 ID |
| `VpcId` | Create | 是 | VPC ID（CFS 所在） |
| `SubnetId` | Create | 是 | 子网 ID（与 Zone 一致） |
| `StorageType` | Create | 是 | CFS 类型 SD/HP |
| `PGroupId` | Create | 是 | CFS 权限组 ID |
| `Zone` | Create | 是 | 可用区（与 SubnetId 一致） |

> 参数名实测自各 Action `--generate-cli-skeleton`（P7）。`--PGroupId` 缺失报 `the following arguments are required: --PGroupId`（exit 252）。

## 操作步骤

### 步骤 1：查询加速服务现状

```bash
tccli tcr DescribeImageAccelerateService --region <REGION> --RegistryId <REGISTRY_ID>
# expected: exit 0, IsEnable 反映当前状态
```
```json
{
    "Status": "",
    "CFSVIP": "",
    "IsEnable": false,
    "RequestId": "..."
}
```

> `IsEnable=false` 表示未开启。开启后 `CFSVIP` 返回加速后端 VIP。

### 步骤 2：创建加速服务

```bash
tccli tcr CreateImageAccelerationService --region <REGION> \
  --RegistryId <REGISTRY_ID> \
  --VpcId <VPC_ID> \
  --SubnetId <SUBNET_ID> \
  --StorageType SD \
  --PGroupId <PGROUP_ID> \
  --Zone <ZONE>
# expected: exit 0
```

### 步骤 3：查询确认开启

```bash
tccli tcr DescribeImageAccelerateService --region <REGION> --RegistryId <REGISTRY_ID>
# expected: exit 0, IsEnable=true, CFSVIP 非空
```

## 验证

| 维度 | 命令 | 期望 |
|:-----|:-----|:-----|
| 服务已开启 | `DescribeImageAccelerateService` | `IsEnable=true` |
| 加速后端就绪 | 同上 | `CFSVIP` 非空，`Status` 正常 |
| 加速生效 | 拉取大镜像对比延迟 | 加速后延迟下降 |

| 占位符 | 含义 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<REGISTRY_ID>` | TCR 实例 ID | 企业版 | `tccli tcr DescribeInstances --region <REGION>` |
| `<REGION>` | 地域 | 如 `ap-guangzhou` | `tccli tcr DescribeRegions` |
| `<VPC_ID>` | VPC ID | CFS 所在 VPC | `tccli vpc DescribeVpcs` |
| `<SUBNET_ID>` | 子网 ID | 与 Zone 同可用区 | `tccli vpc DescribeSubnets` |
| `<PGROUP_ID>` | CFS 权限组 ID | 已创建 | CFS 服务创建 |
| `<ZONE>` | 可用区 | 与 SubnetId 一致 | `tccli tcr DescribeRegions` / `tccli cbs DescribeZones` |

## 清理

```bash
tccli tcr DeleteImageAccelerateService --region <REGION> --RegistryId <REGISTRY_ID>
# expected: exit 0
```

> `DeleteImageAccelerateService` 仅需 `RegistryId`。删除后 CFS 加速后端释放，镜像回到普通拉取。

## 副作用

- **创建加速服务**会在指定 VPC/子网创建 CFS 文件存储，产生 CFS 存储费用（P8 透明性）。
- **CFS 占用子网 IP**：需确保子网有可用 IP。
- **删除加速服务**会释放 CFS，缓存镜像层丢失，下次拉取回退到普通模式。

## 故障恢复

| 现象 | 根因 | 修复 |
|:-----|:-----|:-----|
| `the following arguments are required: --PGroupId`（exit 252） | 缺 CFS 权限组 ID | 先在 CFS 服务创建权限组，传 `--PGroupId` |
| VPC/子网不存在 | `VpcId`/`SubnetId` 错误 | `tccli vpc DescribeVpcs` / `DescribeSubnets` 查真实 ID |
| 可用区与子网不一致 | `Zone` 与 `SubnetId` 所在可用区不同 | 改 `Zone` 与子网可用区一致 |
| `StorageType` 无效 | 非 SD/HP | 用 `SD` 或 `HP` |
| `IsEnable` 始终 false | 创建异步未完成 | 轮询 `DescribeImageAccelerateService`，等 `Status` 正常 |
| 拉取未加速 | 客户端未走 CFSVIP | 确认拉取走 `CFSVIP`，非默认域名 |

> 实测参数校验样本：缺 `--PGroupId` → `tccli: error: the following arguments are required: --PGroupId`（exit 252，客户端解析错误）。

## 下一步

- 镜像推送拉取（加速的对象）：[推送与拉取镜像](push-pull.md)
- 实例访问配置：[访问管理](../instances/manage-access.md)
- 实例同步（跨地域）：[实例同步](../replication/manage.md)

## Action 清单

| Action | 类型 | 跨产品 | 说明 |
|:-------|:-----|:------:|:-----|
| `CreateImageAccelerationService` | 主操作 | vpc/cfs | 创建加速服务（VpcId/SubnetId/StorageType/PGroupId/Zone） |
| `DescribeImageAccelerateService` | 验证 | — | 查询状态（IsEnable/CFSVIP/Status） |
| `DeleteImageAccelerateService` | 清理 | — | 删除加速服务（仅 RegistryId） |
