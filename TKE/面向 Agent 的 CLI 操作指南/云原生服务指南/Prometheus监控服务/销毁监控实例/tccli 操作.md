# 销毁监控实例

> 对照官方：[销毁监控实例](https://cloud.tencent.com/document/product/457/71906) · page_id `71906`

## 概述

通过 `DestroyPrometheusInstance` 销毁指定的 Prometheus 监控实例。销毁操作不可逆，实例关联的所有监控数据、告警规则、采集配置将被永久删除。销毁前务必确认已备份必要的配置和告警规则。

## 前置条件

- [环境准备](../../../环境准备.md)
- 已有 Prometheus 监控实例，且确认不再需要

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    monitor:DestroyPrometheusInstance, monitor:DescribePrometheusInstances
#    monitor:DescribePrometheusClusterAgents
# 验证
tccli monitor DescribePrometheusInstances --region <Region>
# expected: exit 0，返回实例列表
```

### 资源检查

```bash
# 4. 确认目标实例信息
tccli monitor DescribePrometheusInstances --region <Region> \
    --InstanceIds '["<InstanceId>"]'
# expected: 返回实例详情，确认 InstanceId、InstanceName、InstanceStatus

# 5. 查看实例关联的集群
tccli monitor DescribePrometheusClusterAgents --region <Region> \
    --InstanceId '<InstanceId>'
# expected: 了解哪些集群依赖此实例的监控（如不传 ClusterId，需确认 API 是否支持）
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看实例 | `tccli monitor DescribePrometheusInstances --region <Region> --InstanceIds '["<InstanceId>"]'` | 是 |
| 销毁实例 | `tccli monitor DestroyPrometheusInstance --region <Region> --InstanceId '<InstanceId>'` | 是 |

## 操作步骤

### 步骤：销毁 Prometheus 实例

#### 选择依据

- **销毁时机**：确认实例不再需要，且已备份告警规则、采集配置和必要的监控数据（如需要）
- **销毁后影响**：关联的 TKE 集群将失去监控数据上报目标，agent 组件会停止工作但不自动卸载
- **不可逆**：销毁后数据和配置无法恢复。如有疑虑，可先通过 [精简监控指标](../精简监控指标/tccli%20操作.md) 降低使用量观察一段时间

#### 销毁实例

```bash
tccli monitor DestroyPrometheusInstance --region <Region> \
    --InstanceId '<InstanceId>'
# expected: exit 0，返回 RequestId
```

预期输出：

```json
{
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

| 占位符 | 说明 | 获取方式 |
|--------|------|---------|
| `<InstanceId>` | Prometheus 实例 ID | `tccli monitor DescribePrometheusInstances --region <Region>` |
| `REGION` | 地域 | `tccli configure list` |

## 验证

| 维度 | 命令 | 预期 |
|------|------|------|
| 实例已删除 | `tccli monitor DescribePrometheusInstances --region <Region> --InstanceIds '["<InstanceId>"]'` | `ResourceNotFound` 或 `TotalCount: 0` |
| 不在实例列表中 | `tccli monitor DescribePrometheusInstances --region <Region>` | 实例列表中不再包含该 InstanceId |

```bash
# 验证实例已彻底删除
tccli monitor DescribePrometheusInstances --region <Region> \
    --InstanceIds '["<InstanceId>"]'
# expected: ResourceNotFound 或 TotalCount: 0
```

预期输出（已删除）：

```json
{
    "InstanceSet": [],
    "TotalCount": 0,
    "RequestId": "..."
}
```

```bash
# 二次确认：在完整实例列表中搜索
tccli monitor DescribePrometheusInstances --region <Region> | jq '.InstanceSet[] | select(.InstanceId == "prom-example")'
# expected: 无输出（实例不存在）
```

## 清理

`DestroyPrometheusInstance` 本身就是清理操作，执行后即告完成。以下为善后工作。

> **计费提醒**：实例销毁后立即停止计费。按量计费销毁后不会再产生费用，包年包月未到期销毁按比例退款。
> **副作用警告**：销毁操作**不可逆**。实例下的所有监控数据、告警规则、采集配置、关联的 Grafana 集成（如配置）将被永久删除。关联的 TKE 集群上的 agent 组件不会被自动卸载，需手动在集群中清理。

### 1. 清理 agent 残留（可选）

关联的 TKE 集群中 agent 组件会在实例销毁后停止上报但不会自动卸载。如需彻底清理，登录 TKE 集群执行：

```bash
# 查看 agent 相关命名空间和资源
kubectl get ns | grep tke-monitor
# expected: 可能返回 tke-monitor 等命名空间

kubectl get all -n tke-monitor
# expected: 列出残留的 agent 组件 Pod/Deployment/DaemonSet
```

```text
NAME  STATUS  AGE
...
```

清理集群内的 agent 残留需通过控制台或 kubectl 手动删除 `tke-monitor` 命名空间中的资源。

### 2. 确认不再产生监控相关费用

```bash
tccli monitor DescribePrometheusInstances --region <Region>
# expected: 已销毁的实例不再出现于列表中
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DestroyPrometheusInstance` 返回 `ResourceNotFound` | `tccli monitor DescribePrometheusInstances --region <Region> --InstanceIds '["<InstanceId>"]'` 验证实例 | InstanceId 不存在或已被销毁 | 确认 InstanceId 正确，实例是否已被其他操作删除 |
| `DestroyPrometheusInstance` 返回 `UnauthorizedOperation.CamNoAuth` | 检查 CAM 策略 | 缺少 `monitor:DestroyPrometheusInstance` 权限 | 联系管理员授权 `monitor:DestroyPrometheusInstance` |
| `DestroyPrometheusInstance` 返回 `FailedOperation.InstanceNotEmpty` | 检查实例关联状态 | 实例仍有活跃的集群关联 | 先在控制台解除所有集群关联，再销毁实例 |
| `DestroyPrometheusInstance` 返回 `FailedOperation.InstanceIsolated` | 检查实例状态 | 实例处于欠费隔离状态，系统将自动在 7 天后销毁 | 无需操作，或充值恢复后手动销毁 |
| `DestroyPrometheusInstance` 返回 `InvalidParameter.InstanceId` | 检查 InstanceId 格式 | InstanceId 格式错误 | InstanceId 格式为 `prom-xxxxxxxx`，`tccli monitor DescribePrometheusInstances` 获取正确 ID |

### 操作成功但异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 销毁成功但 Describe 仍能看到实例 | `tccli monitor DescribePrometheusInstances --region <Region> --InstanceIds '["<InstanceId>"]'` | 实例可能处于 `Terminating` 状态，异步销毁中 | 等待 1-2 分钟后重新 Describe |
| 销毁后集群仍尝试上报数据 | 检查集群中 agent Pod 状态 | agent 组件未随实例销毁而自动清理 | 手动清理集群中 `tke-monitor` 命名空间的资源 |
| 销毁后包年包月实例未退款 | 检查账户余额变动 | 包年包月退款处理有延迟 | 等待 1-2 个工作日，未收到退款则提交工单 |
| 想恢复已销毁的实例 | — | 销毁不可逆 | 无法恢复。重新 [创建监控实例](../创建监控实例/tccli%20操作.md) 并重新配置 |

## 下一步

- [创建监控实例](../创建监控实例/tccli%20操作.md) — 重新创建 Prometheus 监控实例
- [Prometheus监控概述](../Prometheus监控概述/tccli%20操作.md) — 了解整体架构

## 控制台替代

控制台：容器服务控制台 → 运维功能 → Prometheus 监控 → 实例列表 → 选择实例 → 更多 → 销毁。
