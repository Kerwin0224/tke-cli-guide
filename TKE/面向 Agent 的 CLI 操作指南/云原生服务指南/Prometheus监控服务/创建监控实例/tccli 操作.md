# 创建监控实例

> 对照官方：[创建监控实例](https://cloud.tencent.com/document/product/457/71897) · page_id `71897`

## 概述

通过 `CreatePrometheusInstance` 在指定地域创建一个托管 Prometheus 监控实例。创建完成后可通过 `DescribePrometheusInstances` 查看实例详情，默认状态为 `Running`，才可进行后续的集群关联和采集配置操作。

## 前置条件

- [环境准备](../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置，region 为目标地域

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    monitor:CreatePrometheusInstance, monitor:DescribePrometheusInstances
#    monitor:DestroyPrometheusInstance
# 验证：执行 DescribePrometheusInstances 确认权限
tccli monitor DescribePrometheusInstances --region <Region>
# expected: exit 0，返回实例列表（可为空）
```

### 资源检查

```bash
# 4. 查询 VPC 和子网（实例需绑定 VPC 用于数据上报）
tccli vpc DescribeVpcs --region <Region>
# expected: 至少返回 1 个可用 VPC

tccli vpc DescribeSubnets --region <Region> \
    --Filters '[{"Name":"vpc-id","Values":["VPC_ID"]}]'
# expected: 至少返回 1 个子网

# 5. 查询已有 Prometheus 实例，确认是否需要新建
tccli monitor DescribePrometheusInstances --region <Region>
# expected: 确认当前实例数量和规格
```

### 版本与规格选择

- **实例类型**：`basic`（基础版）/ `standard`（标准版）/ `premium`（高级版），根据数据保留时长和功能需求选择
- **数据保留时长**：由实例类型决定，基础版 15 天、标准版 30 天
- **计费模式**：`PREPAID`（包年包月）或 `POSTPAID`（按量计费）
- **VPC 绑定**：实例需绑定 VPC 和子网，用于 agent 数据上报
- **Grafana**：可选集成 Grafana 实例用于可视化

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 创建实例 | `tccli monitor CreatePrometheusInstance --region <Region> --cli-input-json file://input.json` | 否 |
| 查看实例 | `tccli monitor DescribePrometheusInstances --region <Region> --InstanceIds '["<InstanceId>"]'` | 是 |
| 删除实例 | `tccli monitor DestroyPrometheusInstance --region <Region> --InstanceId '<InstanceId>'` | 是 |

## 操作步骤

### 步骤：创建 Prometheus 实例

#### 选择依据

- **实例类型**：选择 `basic`（基础版）。适合验证 API 和轻度使用场景，成本最低。如需 30 天以上数据保留或更大规格，选择 `standard`（标准版）。
- **计费模式**：选择 `POSTPAID`（按量计费），避免预付费锁定。生产环境长期使用时建议用 `PREPAID`（包年包月）降低成本。
- **VPC**：选择与目标 TKE 集群相同的 VPC，确保 agent 到实例的网络连通性。
- **数据保留时长**：基础版默认 15 天，足够测试使用。

#### 最小创建（基础版按量计费）

`prom-instance-minimal.json`：

```json
{
    "InstanceName": "PROMETHEUS_INSTANCE_NAME",
    "VpcId": "VPC_ID",
    "SubnetId": "SUBNET_ID",
    "DataRetentionTime": 15,
    "Zone": "ZONE"
}
```

```bash
tccli monitor CreatePrometheusInstance --region <Region> \
    --cli-input-json file://prom-instance-minimal.json
# expected: exit 0，返回 InstanceId
```

预期输出：

```json
{
    "InstanceId": "prom-example",
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `PROMETHEUS_INSTANCE_NAME` | 实例名称 | 长度 1-60 字符 | 自定义 |
| `VPC_ID` | VPC 实例 ID | 必须与目标 TKE 集群同 VPC | `tccli vpc DescribeVpcs --region <Region>` |
| `SUBNET_ID` | 子网 ID | 必须是 VPC 下的可用子网 | `tccli vpc DescribeSubnets --region <Region>` |
| `REGION` | 地域 | 如 `ap-guangzhou` | `tccli configure list` |
| `ZONE` | 可用区 | 如 `ap-guangzhou-3` | 需与子网所在可用区一致，`tccli vpc DescribeSubnets --region <Region> --SubnetIds '["SUBNET_ID"]'` 查看 `Zone` |

#### 增强配置（标准版 + Grafana 集成）

`prom-instance-enhanced.json`：

```json
{
    "InstanceName": "PROMETHEUS_INSTANCE_NAME",
    "VpcId": "VPC_ID",
    "SubnetId": "SUBNET_ID",
    "DataRetentionTime": 30,
    "Zone": "ZONE",
    "InstanceChargeType": "PREPAID",
    "InstanceChargePrepaid": {
        "Period": 1,
        "RenewFlag": "NOTIFY_AND_AUTO_RENEW"
    },
    "EnableGrafana": 1,
    "GrafanaInstanceId": "GRAFANA_INSTANCE_ID",
    "TagSpecification": [
        {
            "ResourceType": "prometheus",
            "Tags": [
                {"Key": "env", "Value": "production"},
                {"Key": "team", "Value": "ops"}
            ]
        }
    ]
}
```

```bash
tccli monitor CreatePrometheusInstance --region <Region> \
    --cli-input-json file://prom-instance-enhanced.json
# expected: exit 0，返回 InstanceId
```

### 轮询等待实例就绪

```bash
# 轮询直到状态为 Running，通常需要 2-5 分钟
tccli monitor DescribePrometheusInstances --region <Region> \
    --InstanceIds '["<InstanceId>"]'
# expected: InstanceStatus: 2 (Running)
```

预期输出：

```json
{
    "InstanceSet": [
        {
            "InstanceId": "prom-example",
            "InstanceName": "prom-test-instance",
            "InstanceStatus": 2,
            "VpcId": "vpc-example",
            "SubnetId": "subnet-example",
            "DataRetentionTime": 15,
            "Zone": "ap-guangzhou-3",
            "InstanceChargeType": "POSTPAID"
        }
    ],
    "TotalCount": 1,
    "RequestId": "..."
}
```

| 状态码 | 状态 | 说明 |
|--------|------|------|
| `1` | Creating | 创建中 |
| `2` | Running | 正常运行 |
| `3` | Isolated | 隔离中（欠费） |
| `4` | Terminating | 销毁中 |

## 验证

验证多维度确认实例正常运行，不只检查状态码：

| 维度 | 命令 | 预期 |
|------|------|------|
| 状态 | `tccli monitor DescribePrometheusInstances --region <Region> --InstanceIds '["<InstanceId>"]'` | `InstanceStatus: 2` |
| VPC/子网 | 同上，检查 `VpcId`、`SubnetId` | 与创建参数一致 |
| 数据保留 | 同上，检查 `DataRetentionTime` | 与创建参数一致 |
| API 地址 | 同上，检查 `RemoteWrite` 字段（如有） | 不为空，后续 agent 上报使用 |

```bash
# 综合验证
tccli monitor DescribePrometheusInstances --region <Region> \
    --InstanceIds '["<InstanceId>"]' | jq '.InstanceSet[0] | {InstanceId, InstanceStatus, VpcId, SubnetId, DataRetentionTime}'
# expected: InstanceStatus: 2, VpcId/SubnetId/DataRetentionTime 均与创建参数一致
```

## 清理

> **警告**：`DestroyPrometheusInstance` 会**永久删除**实例及所有历史监控数据，不可恢复。
> **计费提醒**：删除后停止计费。按量计费实例删除后立即停止计费；包年包月实例按比例退款。

### 1. 清理前状态检查

```bash
tccli monitor DescribePrometheusInstances --region <Region> \
    --InstanceIds '["<InstanceId>"]'
# 确认是待删除的目标实例，记录 InstanceId、InstanceName、关联集群数
```

### 2. 销毁实例

```bash
tccli monitor DestroyPrometheusInstance --region <Region> \
    --InstanceId '<InstanceId>'
# expected: exit 0，返回 RequestId
```

### 3. 验证已删除

```bash
tccli monitor DescribePrometheusInstances --region <Region> \
    --InstanceIds '["<InstanceId>"]'
# expected: ResourceNotFound 或 TotalCount: 0
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `CreatePrometheusInstance` 返回 `InvalidParameter` | 检查 input JSON：`python3 -m json.tool prom-instance-minimal.json` | JSON 格式错误或参数值不合法 | 修正 JSON 格式或参数值，参考上方参数表 |
| `CreatePrometheusInstance` 返回 `InvalidParameter.VpcId` | `tccli vpc DescribeVpcs --region <Region> --VpcIds '["VPC_ID"]'` 验证 VPC 存在 | VPC ID 不存在或地域不匹配 | 确认 VpcId 与 region 一致，可通过 `tccli vpc DescribeVpcs` 获取 |
| `CreatePrometheusInstance` 返回 `InvalidParameter.SubnetId` | `tccli vpc DescribeSubnets --region <Region> --SubnetIds '["SUBNET_ID"]'` 验证子网存在 | 子网 ID 不存在或与 VPC 不匹配 | 确认 SubnetId 属于指定的 VpcId |
| `CreatePrometheusInstance` 返回 `LimitExceeded` | `tccli monitor DescribePrometheusInstances --region <Region>` 查当前实例数 | 实例数量达到地域上限（此为环境限制，非命令错误） | 删除不再使用的实例后重试，或联系管理员提升配额 |
| `CreatePrometheusInstance` 返回 `UnauthorizedOperation.CamNoAuth` | 检查 CAM 策略 | 缺少 `monitor:CreatePrometheusInstance` 权限 | 联系管理员授权 `monitor:CreatePrometheusInstance` |
| `CreatePrometheusInstance` 返回 `FailedOperation.TradeFailed` | 检查账户余额 | 账户余额不足（此为环境限制，非命令错误） | 充值或切换为按量计费 |

### 操作成功但异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 返回 InstanceId 但 5 分钟后状态仍为 `1`（Creating） | `tccli monitor DescribePrometheusInstances --region <Region> --InstanceIds '["<InstanceId>"]'` 查看 `InstanceStatus` | 后端创建缓慢（正常）或卡住（异常） | 继续等待；超过 10 分钟则保留 region、InstanceId、RequestId、创建 JSON → 登录控制台查看详细状态 |
| 状态为 `3`（Isolated） | 同上，查看 `InstanceStatus` | 账户欠费导致实例被隔离 | 充值后实例自动恢复；持续欠费 7 天后实例将被销毁 |
| 实例 Running 但 `RemoteWrite` 地址为空 | 同上，检查 `RemoteWrite` 字段 | 后端尚未分配数据上报地址（罕见） | 等待 2-3 分钟重试 Describe |

## 下一步

- [关联集群](../关联集群/tccli%20操作.md) — 将 TKE 集群接入 Prometheus 监控
- [数据采集配置](../数据采集配置/tccli%20操作.md) — 配置指标采集规则
- [精简监控指标](../精简监控指标/tccli%20操作.md) — 过滤不需要的指标
- [计费方式和资源使用](../计费方式和资源使用/tccli%20操作.md) — 了解计费细节

## 控制台替代

控制台：容器服务控制台 → 运维功能 → Prometheus 监控 → 新建。
