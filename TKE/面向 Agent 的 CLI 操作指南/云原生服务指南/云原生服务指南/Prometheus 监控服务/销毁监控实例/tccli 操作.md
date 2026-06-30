# 销毁监控实例（tccli）

> 对照官方：[销毁监控实例](https://cloud.tencent.com/document/product/457/71906) · page_id `71906`

## 概述

通过 `tccli monitor DestroyPrometheusInstance` 销毁不再需要的 Prometheus 监控实例。销毁操作**不可逆**——实例及其所有关联资源（TKE Serverless 集群、CLB 等）将被删除，已关联的集群自动解除关联，监控数据无法恢复。

## 前置条件

- [环境准备](../../../../环境准备.md)
- 确认目标实例 `InstanceId`（格式：`prom-xxxxxxxx`）
- 确认实例不再需要，已备份重要监控数据或告警规则（如有需要）
- 确认无业务依赖该监控实例的告警

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 销毁实例 | `tccli monitor DestroyPrometheusInstance --InstanceId <InstanceId>` | 否（重复调用对已删除实例报错） |
| 确认销毁结果 | `tccli monitor DescribePrometheusInstances --InstanceIds '["<InstanceId>"]'` | 是 |

## 操作步骤

### 1. 确认目标实例信息

```bash
tccli monitor DescribePrometheusInstances --region ap-guangzhou --InstanceIds '["prom-exampledel"]' --filter "InstanceSet[0].{InstanceId:InstanceId,InstanceName:InstanceName,InstanceStatus:InstanceStatus}"
```

```json
{
    "InstanceId": "prom-exampledel",
    "InstanceName": "demo-monitoring",
    "InstanceStatus": 2
}
```

确认实例状态为 `2`（运行中）且 `InstanceId` 正确无误。

### 2. 查看已关联集群（了解销毁影响范围）

```bash
tccli monitor DescribePrometheusClusterAgents --region ap-guangzhou --InstanceId prom-exampledel --filter "Agents[].ClusterId, Agents[].ClusterName"
```

```json
{
    "Agents": [
        {"ClusterId": "cls-example", "ClusterName": "demo-cluster"}
    ]
}
```

确认销毁后哪些集群将失去监控。

### 3. 销毁实例

```bash
tccli monitor DestroyPrometheusInstance --region ap-guangzhou --InstanceId prom-exampledel
```

```json
{
    "Response": {
        "RequestId": "abc123-..."
    }
}
```

### 4. 确认销毁完成

```bash
# 查询已销毁的实例，应返回空或状态为非 2
tccli monitor DescribePrometheusInstances --region ap-guangzhou --InstanceIds '["prom-exampledel"]' --filter "InstanceSet[0].InstanceStatus"
```

实例销毁后 `InstanceStatus` 变为 `5`（销毁中） → 最终从列表消失，或 `TotalCount` 为 `0`。

### 安全提示

销毁操作会将以下资源一并删除：

| 关联资源 | 销毁行为 |
|---------|---------|
| TKE Serverless 集群 | 随实例删除（集群名等于 TMP 实例 ID） |
| 内网 CLB | 随实例删除 |
| 公网 CLB | 随实例删除 |
| 集群关联 | 自动解除，采集插件被卸载 |
| 采集配置 / 告警策略 | 数据不可恢复 |

## 验证

```bash
# 再次查询确认实例已不存于列表
tccli monitor DescribePrometheusInstances --region ap-guangzhou --InstanceIds '["prom-exampledel"]' --filter "TotalCount"
```

期望结果为 `TotalCount: 0`。

## 清理

实例已销毁，无需额外清理。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 销毁请求返回权限错误 `UnauthorizedOperation` | `tccli configure list` 检查当前凭据 | 子账号缺少 `monitor:DestroyPrometheusInstance` 权限（此为环境限制，非命令错误） | 联系主账号授予 `QcloudMonitorFullAccess` 或最小权限 `monitor:DestroyPrometheusInstance` |

### 销毁成功但状态延迟

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 销毁后实例仍在列表中 | `tccli monitor DescribePrometheusInstances --InstanceIds '["<InstanceId>"]'` 检查 `InstanceStatus` | 实例从「销毁中（状态 5）」到完全删除有短暂延迟（正常现象） | 等待 3-5 分钟后重新查询，`TotalCount` 应为 `0` |
| 误删实例 | —（不可逆） | 销毁操作不可逆，关联的 TKE Serverless 集群、CLB、采集配置、告警规则等数据无法恢复 | 对关键实例启用 [MFA](https://cloud.tencent.com/document/product/378/8394) 二次确认；销毁前先 `DescribePrometheusInstanceDetail` 确认 InstanceId |
| 关联的 eks 集群仍可见 | `tccli monitor DescribePrometheusInstances` 确认实例已不在列表 | TKE Serverless 集群随实例删除，资源回收有延迟（环境回收） | 等待数分钟资源回收完成，若持续存在且实例已删，联系工单确认 |

## 下一步

- [创建监控实例](../创建监控实例/tccli%20操作.md) 如需重新创建实例
- [Prometheus 监控概述](../Prometheus%20监控概述/tccli%20操作.md) 返回总览页

## 控制台替代

控制台：Prometheus 监控 → 实例列表 → 点击目标实例右侧「销毁/退还」→ 确认弹窗中验证实例信息 → 确定。
