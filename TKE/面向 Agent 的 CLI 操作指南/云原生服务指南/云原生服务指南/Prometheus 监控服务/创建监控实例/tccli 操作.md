# 创建监控实例（tccli）

> 对照官方：[创建监控实例](https://cloud.tencent.com/document/product/457/71897) · page_id `71897`

## 概述

通过 `tccli monitor` 创建 Prometheus 监控服务（TMP）实例。实例创建后处于「运行中」状态即可关联 TKE 集群，实现跨集群的监控指标联查与统一告警。

首次使用需授权服务角色 `TKE_QCSLinkedRoleInPrometheusService`。

## 前置条件

- [环境准备](../../../../环境准备.md)
- 已确定实例所属 VPC 和子网（TMP 实例必须部署在 VPC 内）
- 已确定数据保留时长（单位：天）
- 首次使用需授权服务角色（控制台或 CAM 控制台完成）

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 购买创建实例 | `tccli monitor CreatePrometheusInstance --cli-input-json file://create.json` | 否 |
| 查询创建进度 | `tccli monitor DescribePrometheusInstances --InstanceIds '["<InstanceId>"]'` | 是 |
| 查看实例详情 | `tccli monitor DescribePrometheusInstanceDetail --InstanceId <InstanceId>` | 是 |
| 查看初始化状态 | `tccli monitor DescribePrometheusInstanceInitStatus --InstanceId <InstanceId>` | 是 |

## 操作步骤

### 1. 确认目标可用区

```bash
tccli monitor DescribePrometheusZones --region ap-guangzhou --filter "ZoneSet[].Zone"
```

### 2. 创建实例

```json
{
    "InstanceName": "demo-monitoring",
    "VpcId": "vpc-example",
    "SubnetId": "subnet-example",
    "DataRetentionTime": 15,
    "Zone": "ap-guangzhou-3",
    "TagSpecification": [
        {
            "ResourceType": "prometheus",
            "Tags": [
                {"Key": "env", "Value": "demo"},
                {"Key": "team", "Value": "platform"}
            ]
        }
    ],
    "GrafanaInstanceId": ""
}
```

主要参数说明：

| 参数 | 必填 | 说明 |
|------|------|------|
| `InstanceName` | 是 | 实例名称，1-60 个字符，支持中英文、数字、下划线 |
| `VpcId` | 是 | 实例所属 VPC ID |
| `SubnetId` | 是 | 实例所属子网 ID |
| `DataRetentionTime` | 是 | 数据保留时长（天），可选 15、30、45、90、180、360 |
| `Zone` | 是 | 可用区，例如 `ap-guangzhou-3` |
| `TagSpecification` | 否 | 标签绑定，便于成本分账 |
| `GrafanaInstanceId` | 否 | 可选绑定已有的 Grafana 可视化实例 ID |

```bash
tccli monitor CreatePrometheusInstance --region ap-guangzhou --cli-input-json file://create.json
```

```json
{
    "Response": {
        "RequestId": "def456-...",
        "InstanceId": "prom-examplenew"
    }
}
```

记录返回的 `InstanceId`（格式：`prom-xxxxxxxx`），后续操作依赖此 ID。

### 3. 等待实例进入「运行中」状态

```bash
# 使用 --waiter 轮询直到 InstanceStatus == 2（运行中）
tccli monitor DescribePrometheusInstances \
    --region ap-guangzhou \
    --InstanceIds '["prom-examplenew"]' \
    --waiter "{\"expr\":\"InstanceSet[0].InstanceStatus\",\"to\":\"2\",\"timeout\":600,\"interval\":10}"
```

也可手动轮询：

```bash
tccli monitor DescribePrometheusInstances --region ap-guangzhou --InstanceIds '["prom-examplenew"]' --filter "InstanceSet[0].InstanceStatus, InstanceSet[0].InstanceName"
```

```json
{
    "InstanceStatus": 2,
    "InstanceName": "demo-monitoring"
}
```

状态码 `2` 表示实例创建成功且正常运行。

## 验证

```bash
# 检查实例详情，确认 VPC、子网、数据保留时长正确
tccli monitor DescribePrometheusInstanceDetail --region ap-guangzhou --InstanceId prom-examplenew --filter "InstanceStatus, InstanceName, VpcId, DataRetentionTime"
```

## 清理

如需删除本实验创建的实例：

```bash
tccli monitor DestroyPrometheusInstance --region ap-guangzhou --InstanceId prom-examplenew
```

参见 [销毁监控实例](../销毁监控实例/tccli%20操作.md)。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `CreatePrometheusInstance` 返回权限错误 `UnauthorizedOperation` | `tccli configure list` 检查当前凭据 | 未授权 `TKE_QCSLinkedRoleInPrometheusService` 服务角色（此为环境限制，非命令错误） | 登录 [CAM 控制台](https://console.cloud.tencent.com/cam/role) 为 TKE 服务授权 `QcloudPrometheusFullAccess` |
| `DataRetentionTime` 参数报错 | 检查传入值 | 数据保留时长取值不在有效范围 | 传入有效取值之一：15、30、45、90、180、360（单位：天） |

### 创建成功但状态异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 创建后长时间处于状态 1（创建中） | `tccli monitor DescribePrometheusInstanceInitStatus --InstanceId <InstanceId>` 检查初始化阶段 | VPC/子网资源不足或后端初始化延迟 | 保留 InstanceId、RequestId；如持续超过 15 分钟异常，登录 [工单](https://console.cloud.tencent.com/workorder) 联系支持 |
| `--waiter` 轮询超时 | `tccli monitor DescribePrometheusInstances --InstanceIds '["<InstanceId>"]'` 检查 InstanceStatus | 后端初始化时间超过 waiter 设置的 timeout | 增大 `timeout` 值（如 900），或手动轮询检查状态 |

## 下一步

- [关联集群](../关联集群/tccli%20操作.md) 将 TKE 集群关联到新建的监控实例
- [数据采集配置](../数据采集配置/tccli%20操作.md) 配置指标采集规则

## 控制台替代

控制台：容器服务控制台 → 云原生服务 → Prometheus → 点击「新建」→ 填写实例参数 → 确认购买。
