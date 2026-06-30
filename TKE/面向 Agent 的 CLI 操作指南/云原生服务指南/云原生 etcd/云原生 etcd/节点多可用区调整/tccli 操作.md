# 节点多可用区调整

> 对照官方：[节点多可用区调整](https://cloud.tencent.com/document/product/457/109851) · page_id `109851`

## 概述

调整 etcd 集群节点的可用区分布以实现高可用部署。核心操作包括：查看当前可用区分布、添加不同可用区子网、通过 `ResizeEtcdInstance` 重新分配节点到多可用区。扩容不影响业务，缩容可能触发 leader 选举，建议低峰期操作。

## 前置条件

- [环境准备](../../../../环境准备.md)
- 已有运行中的 etcd 实例

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置

# 3. 检查 CAM 权限 — 需要 cetcd:ResizeEtcdInstance, vpc:DescribeSubnets
tccli cetcd DescribeEtcdInstances --region <Region>
# expected: exit 0

# 4. 查询可用子网（含不同 AZ）
tccli vpc DescribeSubnets --region <Region> \
    --Filters '[{"Name":"vpc-id","Values":["VPC_ID"]}]' \
    | jq '.SubnetSet[] | {SubnetId, Zone}'
# expected: 至少返回多个不同可用区的子网
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看节点可用区分布 | `DescribeEtcdNodes` | 是 |
| 查看当前子网 AZ | `DescribeEtcdInstance` + `DescribeSubnets` | 是 |
| 添加子网 | `AddEtcdSubnets` | 否 |
| 调整节点数/规格 | `ResizeEtcdInstance` | 否 |

## 操作步骤

### 步骤 1：查看当前节点可用区分布

```bash
tccli cetcd DescribeEtcdNodes --region <Region> \
    --InstanceId INSTANCE_ID
# expected: exit 0，返回各节点的 Zone 信息
```

### 步骤 2：查看当前实例的子网配置

```bash
tccli cetcd DescribeEtcdInstance --region <Region> \
    --InstanceId INSTANCE_ID \
    | jq '{SubnetIds, ServiceSubnetId}'
# expected: 返回当前绑定的子网列表
```

### 步骤 3：添加不同可用区的子网

如果当前子网不覆盖所需可用区，先添加子网：

```bash
tccli cetcd AddEtcdSubnets --region <Region> \
    --InstanceId INSTANCE_ID \
    --SubnetIds '["SUBNET_ID_NEW_1", "SUBNET_ID_NEW_2"]'
# expected: exit 0
```

### 步骤 4：通过扩缩容调整节点 AZ 分布

#### 选择依据

多可用区调整的核心思路是通过 `ResizeEtcdInstance` 先缩容再扩容，使 etcd 自动将新节点分布到不同可用区的子网中。调整策略取决于当前集群节点数和目标 AZ 分布：

| 当前节点数 | 目标 AZ 分布 | 操作 |
|-----------|-------------|------|
| 1 节点 | 3 AZ | 先确保至少 3 个不同 AZ 子网已添加，然后 ResizeEtcdInstance Size=3（扩容至 3 节点） |
| 3 节点 | 3 AZ | 如当前已分布 3 AZ → 无需调整。如仅 1-2 AZ → 添加缺失 AZ 的子网后，Size=5 扩容至 5 节点，再 Size=3 缩容回 3 节点 |
| 5 节点 | 3 AZ | 如当前仅 1-2 AZ → 添加缺失 AZ 的子网后，先缩容至 3 节点再扩容回 5 节点 |

> **关键规则**：扩容不会影响业务，缩容可能触发 leader 选举，建议在低峰期操作。高可用部署要求至少 3 个不同可用区的子网。

#### 操作示例（1 节点扩展到 3 节点 3 AZ）

```bash
# 1. 确认已添加 3 个不同 AZ 的子网
tccli cetcd DescribeEtcdInstance --region <Region> \
    --InstanceId INSTANCE_ID | jq '.SubnetIds'

# 2. 扩容到 3 节点（etcd 自动将新节点分布到不同 AZ 子网）
tccli cetcd ResizeEtcdInstance --region <Region> \
    --InstanceId INSTANCE_ID \
    --Size 3
# expected: exit 0
```

#### 操作示例（3 节点 1 AZ 重新分布到 3 AZ）

```bash
# 1. 添加缺失 AZ 的子网
tccli cetcd AddEtcdSubnets --region <Region> \
    --InstanceId INSTANCE_ID \
    --SubnetIds '["SUBNET_ID_AZ2", "SUBNET_ID_AZ3"]'

# 2. 扩容到 5 节点
tccli cetcd ResizeEtcdInstance --region <Region> \
    --InstanceId INSTANCE_ID \
    --Size 5

# 3. 等待新节点就绪后，缩容回 3 节点（移除旧 AZ 节点）
tccli cetcd ResizeEtcdInstance --region <Region> \
    --InstanceId INSTANCE_ID \
    --Size 3
```

### 步骤 5：验证 AZ 分布

```bash
tccli cetcd DescribeEtcdNodes --region <Region> \
    --InstanceId INSTANCE_ID \
    | jq '.Nodes[] | {NodeId, Zone}'
# expected: 节点分布在 3 个不同 AZ
```

## 验证

| 维度 | 命令 | 预期 |
|------|------|------|
| 状态 | `DescribeEtcdInstance` | `Status: "Running"` |
| 节点数 | 同上 | 与操作后的 Size 一致 |
| AZ 分布 | `DescribeEtcdNodes` 提取 Zone | ≥ 3 个不同 AZ |
| 集群健康 | etcdctl | 全部 endpoint healthy |

## 清理

本页操作调整的是现有实例，如需恢复原状，逆序执行调整操作即可。添加的子网无法通过 CLI 直接移除，可通过控制台操作。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `AddEtcdSubnets` 返回 `InvalidParameter` | 检查 SubnetId 格式和归属 VPC | 子网不属于实例所在 VPC 或已存在 | 选择同 VPC 不同可用区的子网 |
| `ResizeEtcdInstance` 缩容被拒绝 | 检查 Size 是否合法 | 节点数必须为奇数 | 使用奇数（3/5/7）作为 Size 值 |
| 节点分布未达预期 | `DescribeEtcdNodes` 查看实际 Zone 分布 | 子网可能在同一可用区 | 确认添加的子网来自不同 AZ：`tccli vpc DescribeSubnets --SubnetIds '["SUBNET_ID_1","SUBNET_ID_2"]' \| jq '.SubnetSet[] | .Zone'` |

### 操作成功但 AZ 分布未变化

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 扩容后新节点仍在同一 AZ | `DescribeEtcdNodes` 对比新旧节点 Zone | 新增的子网和原有子网在同一可用区 | 确认添加的子网确实来自不同可用区后，先缩容再扩容 |

## 下一步

- [节点扩容和升降配](../节点扩容和升降配/tccli%20操作.md) — 调整节点规格
- [监控和告警配置](../监控和告警配置/tccli%20操作.md) — 确认节点监控正常
- [创建集群](../创建集群/tccli%20操作.md) — 新建时直接配置多 AZ

## 控制台替代

访问 [云原生 etcd 控制台](https://console.cloud.tencent.com/tke2/etcd/list)，在实例详情页的 基本信息 页面查看子网并添加不同 AZ 子网，再通过 调整规格 变更节点数。
