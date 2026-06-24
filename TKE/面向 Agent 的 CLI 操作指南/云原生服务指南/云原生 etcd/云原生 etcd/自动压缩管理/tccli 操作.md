# 自动压缩管理

> 对照官方：[自动压缩管理](https://cloud.tencent.com/document/product/457/58181) · page_id `58181`

## 概述

etcd 会记录所有键值对的更新操作，历史数据不断累积会导致性能退化和存储空间耗尽。云原生 etcd 支持自动压缩（Auto Compaction），定期清理旧版本数据。两种模式：**周期性压缩**（按时间间隔）和**按 Revision 压缩**（按保留的版本数量）。

| 模式 | 适用场景 | 配置方式 |
|------|---------|---------|
| 周期性压缩（periodic） | 数据写入速率稳定，希望按固定间隔清理 | 设置压缩周期（时:分:秒） |
| 按 Revision 压缩（revision） | 数据写入速率不固定，希望限制保留的版本数 | 设置最大保留版本数 |

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

# 3. 检查 CAM 权限 — 需要 cetcd:ModifyEtcdConfiguration
tccli cetcd DescribeEtcdInstances --region <Region>
# expected: exit 0

# 4. 确认目标实例运行中
tccli cetcd DescribeEtcdInstance --region <Region> --InstanceId INSTANCE_ID | jq '.Status'
# expected: "Running"
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 配置压缩参数 | `ModifyEtcdConfiguration` (AutoCompactionSettings) | 否 |
| 查看当前配置 | `DescribeEtcdInstance` | 是 |

## 关键字段说明

以下说明 `ModifyEtcdConfiguration` 中 `AutoCompactionSettings` 的参数：

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `AutoCompactionSettings.Mode` | String | 是 | `periodic`（周期性压缩）或 `revision`（按版本数压缩） | 填无效值 → `InvalidParameter` |
| `AutoCompactionSettings.Period` | String | periodic 时必填 | 压缩周期，如 `1h30m`（1 小时 30 分钟） | - |
| `AutoCompactionSettings.Revision` | Integer | revision 时必填 | 保留的版本数 | - |

## 操作步骤

### 步骤 1：查看当前压缩配置

```bash
tccli cetcd DescribeEtcdInstance --region <Region> \
    --InstanceId INSTANCE_ID \
    | jq '.AutoCompactionSettings'
# expected: 返回当前压缩配置（可能为空）
```

### 步骤 2：配置自动压缩

#### 选择依据

- **周期性压缩**：适合写入速率均匀的场景。周期设置建议 ≥ 1 小时，避免压缩过于频繁影响性能。
- **按 Revision 压缩**：适合写入速率波动大的场景。保留版本数以业务可接受的回滚窗口为准。

#### 配置周期性压缩

`auto-compaction-periodic.json`：

```json
{
    "InstanceId": "INSTANCE_ID",
    "AutoCompactionSettings": {
        "Mode": "periodic",
        "Period": "1h"
    }
}
```

```bash
tccli cetcd ModifyEtcdConfiguration --region <Region> \
    --cli-input-json file://auto-compaction-periodic.json
# expected: exit 0
```

#### 配置按 Revision 压缩

`auto-compaction-revision.json`：

```json
{
    "InstanceId": "INSTANCE_ID",
    "AutoCompactionSettings": {
        "Mode": "revision",
        "Revision": 1000
    }
}
```

```bash
tccli cetcd ModifyEtcdConfiguration --region <Region> \
    --cli-input-json file://auto-compaction-revision.json
# expected: exit 0
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `INSTANCE_ID` | etcd 实例 ID | 格式 `etcd-xxxxxxxx` | `tccli cetcd DescribeEtcdInstances` |

## 验证

```bash
tccli cetcd DescribeEtcdInstance --region <Region> \
    --InstanceId INSTANCE_ID \
    | jq '.AutoCompactionSettings'
# expected: 返回配置的压缩模式、周期或版本数
```

## 清理

如需关闭自动压缩：将 `AutoCompactionSettings` 设为 `{}` 空对象。

```bash
tccli cetcd ModifyEtcdConfiguration --region <Region> \
    --InstanceId INSTANCE_ID \
    --AutoCompactionSettings '{}'
# expected: exit 0
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `ModifyEtcdConfiguration` 返回 `InvalidParameter` | 检查 `Mode` 值 | Mode 取值不正确 | 使用 `periodic` 或 `revision` |
| `ModifyEtcdConfiguration` 返回 `InvalidParameter` | 检查 `Period` 格式 | 周期格式不正确 | 使用标准格式，如 `1h`、`30m`、`1h30m` |

### 配置生效但压缩未触发

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 数据量持续增长，压缩未生效 | 检查监控中的 "数据库 key 数量" 指标 | 压缩周期过长或写入速率超过压缩速率 | 缩短压缩周期或减少保留版本数 |
| 配置后数据库大小未减少 | 检查 etcd 监控中的 "数据库大小" 指标 | 压缩释放的空间不会立即归还 OS（etcd 行为） | 压缩仅标记数据为可回收，磁盘空间不会立即释放属正常现象 |

## 下一步

- [快照管理](../快照管理/tccli%20操作.md) — 配置数据备份
- [监控和告警配置](../监控和告警配置/tccli%20操作.md) — 监控数据库 key 数量和大小
- [数据量异常](../集群排障/数据量异常/tccli%20操作.md) — 排查数据量异常增长

## 控制台替代

访问 [云原生 etcd 控制台](https://console.cloud.tencent.com/tke2/etcd/list)，在实例右侧选择 更多 > 压缩参数配置。
