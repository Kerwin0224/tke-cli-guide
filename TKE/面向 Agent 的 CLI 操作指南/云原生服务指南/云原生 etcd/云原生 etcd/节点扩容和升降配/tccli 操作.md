# 节点扩容和升降配

> 对照官方：[节点扩容和升降配](https://cloud.tencent.com/document/product/457/58180) · page_id `58180`

## 概述

对已有 etcd 集群进行节点数量调整（扩容）和节点规格变更（升降配）。节点扩容不影响正常业务，节点升降配会触发滚动更新，建议在业务低峰期操作。

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

# 3. 检查 CAM 权限 — 需要 cetcd:ResizeEtcdInstance, cetcd:DescribeEtcdInstances
tccli cetcd DescribeEtcdInstances --region <Region>
# expected: exit 0

# 4. 确认目标实例运行中
tccli cetcd DescribeEtcdInstances --region <Region> \
    | jq '.InstanceSet[] | select(.InstanceId=="INSTANCE_ID") | .Status'
# expected: "Running"
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 调整规格 | `ResizeEtcdInstance` | 否 |
| 查看实例列表 | `DescribeEtcdInstances` | 是 |
| 查看节点管理 | `DescribeEtcdNodes` | 是 |

## 关键字段说明

以下说明 `ResizeEtcdInstance` 的主要参数：

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `InstanceId` | String | 是 | 目标 etcd 实例 ID | 不存在 → `InvalidParameter` |
| `Size` | Integer | 否 | 节点数量。奇数（3/5/7）。不填则保持不变 | - |
| `Cpu` | Integer | 否 | CPU 核数。不填则保持不变 | - |
| `Memo` | Integer | 否 | 内存大小（GB）。不填则保持不变 | - |

## 操作步骤

### 步骤 1：查看当前配置

```bash
tccli cetcd DescribeEtcdInstance --region <Region> \
    --InstanceId INSTANCE_ID
# expected: exit 0，返回当前 Size 和规格
```

**预期输出**：

```json
{
    "InstanceId": "etcd-example",
    "InstanceName": "my-etcd",
    "Status": "Running",
    "Size": 3,
    "Cpu": 2,
    "Memo": 4,
    "DiskSize": 50
}
```

### 步骤 2：执行节点扩容/升降配

#### 选择依据

- **扩容**：仅调整 Size。节点扩容不影响正常业务，Raft 协议要求奇数节点数。从 3 扩到 5 可增强高可用性。
- **升降配**：调整 Cpu/Memo。触发滚动更新，每个节点依次重启。建议业务低峰期执行。
- **同时扩容和升降配**：一次调用中同时调整 Size 和规格。先扩容再升配更安全。

#### 最小变更（仅调整节点数，从 3 到 5）

`resize-etcd-minimal.json`：

```json
{
    "InstanceId": "INSTANCE_ID",
    "Size": 5
}
```

```bash
tccli cetcd ResizeEtcdInstance --region <Region> \
    --cli-input-json file://resize-etcd-minimal.json
# expected: exit 0
```

#### 增强变更（同时调整规格和节点数）

`resize-etcd-enhanced.json`：

```json
{
    "InstanceId": "INSTANCE_ID",
    "Size": 5,
    "Cpu": 4,
    "Memo": 8
}
```

```bash
tccli cetcd ResizeEtcdInstance --region <Region> \
    --cli-input-json file://resize-etcd-enhanced.json
# expected: exit 0
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `INSTANCE_ID` | etcd 实例 ID | 格式 `etcd-xxxxxxxx` | `tccli cetcd DescribeEtcdInstances` |

### 步骤 3：轮询等待变更完成

```bash
tccli cetcd DescribeEtcdInstance --region <Region> \
    --InstanceId INSTANCE_ID
# expected: Status 为 "Running"，Size/Cpu/Memo 与变更参数一致
```

## 验证

验证维度：

| 维度 | 命令 | 预期 |
|------|------|------|
| 状态 | `DescribeEtcdInstance` | `Status: "Running"` |
| 节点数 | 同上 | `Size` 与变更参数一致 |
| 规格 | 同上 | `Cpu`/`Memo` 与变更参数一致 |
| 集群健康 | 通过 etcdctl 检查 | `etcdctl endpoint health` 全部 healthy |

```bash
# 确认变更后的配置
tccli cetcd DescribeEtcdInstance --region <Region> \
    --InstanceId INSTANCE_ID \
    | jq '{Size, Cpu, Memo, Status}'
# expected: Size=5, Cpu=4, Memo=8, Status="Running"
```

## 清理

节点缩容（减少 Size）是相同操作，将 `Size` 调整为更小的奇数即可。注意缩容可能触发 leader 选举，建议业务低峰期执行。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `ResizeEtcdInstance` 返回 `InvalidParameter` | 检查 InstanceId 和 Size 合法值 | InstanceId 不存在或 Size 不是奇数 | 用 `DescribeEtcdInstances` 确认 InstanceId，Size 改为 3/5/7 |
| `ResizeEtcdInstance` 返回 `FailedOperation` | 检查实例状态 | 实例非 Running 状态或正在进行其他操作 | 等待实例状态为 `Running` 后重试 |

### 变更已提交但异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 返回了 RequestId 但长时间未完成 | `tccli cetcd DescribeEtcdInstance --region <Region> --InstanceId INSTANCE_ID` 查看 Status | 滚动更新缓慢或资源不足 | 继续等待；超过 30 分钟则保留 region、InstanceId、RequestId → [提交工单](https://console.cloud.tencent.com/workorder) |
| 升级后部分节点不健康 | 执行 `etcdctl endpoint health` 检查各节点 | 单节点可能还在重启中 | 等待全部节点完成重启，确认 `endpoint health` 全部 healthy |

## 下一步

- [节点多可用区调整](../节点多可用区调整/tccli%20操作.md) — 调整节点可用区分布
- [监控和告警配置](../监控和告警配置/tccli%20操作.md) — 配置监控
- [自动压缩管理](../自动压缩管理/tccli%20操作.md) — 配置自动压缩

## 控制台替代

访问 [云原生 etcd 控制台](https://console.cloud.tencent.com/tke2/etcd/list)，在实例右侧点击 调整规格。
