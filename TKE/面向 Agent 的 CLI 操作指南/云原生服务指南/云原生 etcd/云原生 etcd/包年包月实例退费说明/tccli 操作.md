# 包年包月实例退费说明

> 对照官方：[包年包月实例退费说明](https://cloud.tencent.com/document/product/457/99318) · page_id `99318`

## 概述

包年包月（PREPAID）计费的 etcd 实例支持自助退还。每个账户主体可享受一次**五天无理由自助退还**，超出则按**普通自助退还**计算退款金额。退款后实例和磁盘数据将被回收，不可恢复。

| 退还类型 | 条件 | 退款金额 |
|---------|------|---------|
| 五天无理由自助退还 | 新购 5 天内（含），每账户主体限 1 个实例 | 全额退还（含现金、收益转入、赠送金） |
| 普通自助退还 | 超出 5 天或无理由额度已用 | 退款金额 = 有效订单金额 + 未开始订单金额 - 资源已使用价值 |

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

# 3. 检查 CAM 权限
tccli cetcd DescribeEtcdInstances --region <Region>
# expected: exit 0
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看实例列表 | `DescribeEtcdInstances` | 是 |
| 销毁/退还实例 | `DestroyEtcdInstance` | 否 |
| 查看实例详情（含计费信息） | `DescribeEtcdInstance` | 是 |

## 操作步骤

### 步骤 1：确认待退还实例

```bash
tccli cetcd DescribeEtcdInstances --region <Region>
# expected: exit 0，列出所有实例及计费类型
```

### 步骤 2：确认实例计费类型和购买时间

```bash
tccli cetcd DescribeEtcdInstance --region <Region> \
    --InstanceId INSTANCE_ID
# expected: 返回实例详情，包含 ChargeType 和 CreateTime
```

### 步骤 3：销毁实例（执行退费）

> **警告**：销毁操作不可逆。实例磁盘数据将被回收且不可恢复。如已开启数据备份，可选择是否同时释放 COS 快照数据。如已开启公网访问，CLB 资源将随实例一并释放。

```bash
tccli cetcd DestroyEtcdInstance --region <Region> \
    --InstanceId INSTANCE_ID
# expected: exit 0
```

## 验证

```bash
tccli cetcd DescribeEtcdInstances --region <Region>
# expected: 目标实例不再出现在列表中
```

## 清理

退还操作本身即为清理，无需额外操作。注意：
- 勾选的关联 COS 快照数据会被删除（桶保留）
- 关联 CLB 资源会被释放
- 退还后不支持续费恢复

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DestroyEtcdInstance` 返回 `InvalidParameter` | 检查 InstanceId 是否正确 | InstanceId 不存在或格式错误 | 用 `DescribeEtcdInstances` 确认 InstanceId |
| `DestroyEtcdInstance` 返回 `FailedOperation` | 检查实例是否开启删除保护 | 实例开启了删除保护 | 先执行 `DisableEtcdInstanceDeletionProtection` 关闭删除保护 |
| 找不到退还入口（超出五天无理由） | 确认购买时间和退费记录 | 超出 5 天或额度已用（此为环境限制，非命令错误） | 可使用普通自助退还，按公式计算退款金额 |

## 下一步

- [创建集群](../创建集群/tccli%20操作.md) — 重新创建 etcd 实例
- [购买指南](../购买指南/tccli%20操作.md) — 了解计费模式
- [删除集群](../删除集群/tccli%20操作.md) — 按量计费实例删除

## 控制台替代

访问 [云原生 etcd 控制台](https://console.cloud.tencent.com/tke2/etcd/list)，在实例右侧选择 更多 > 销毁/退货。
