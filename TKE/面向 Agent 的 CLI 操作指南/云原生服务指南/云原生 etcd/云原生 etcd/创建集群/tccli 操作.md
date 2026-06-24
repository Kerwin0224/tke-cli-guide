# 创建集群

> 对照官方：[创建集群](https://cloud.tencent.com/document/product/457/58178) · page_id `58178`

## 概述

在指定地域与 VPC 内创建托管 etcd 集群。支持包年包月（PREPAID）和按量计费（POSTPAID_BY_HOUR）两种计费模式，支持 HTTPS 双向认证、自动压缩、数据备份和跨可用区高可用部署。

| 计费模式 | 适用场景 | 计费周期 |
|---------|---------|---------|
| PREPAID（包年包月） | 长期稳定业务 | 按月 |
| POSTPAID_BY_HOUR（按量计费） | 临时测试 | 按小时 |

## 前置条件

- [环境准备](../../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置，region 为目标地域

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    cetcd:CreateEtcdInstance, cetcd:DescribeEtcdInstances
#    vpc:DescribeVpcs, vpc:DescribeSubnets
# 验证：执行 DescribeEtcdInstances 确认 etcd 权限
tccli cetcd DescribeEtcdInstances --region <Region>
# expected: exit 0，返回实例列表（可为空）

# 验证 VPC 权限
tccli vpc DescribeVpcs --region <Region>
# expected: exit 0，返回 VPC 列表（可为空）
```

### 资源检查

```bash
# 4. 查询可用 VPC 和子网
tccli vpc DescribeVpcs --region <Region>
# expected: 至少返回 1 个 VPC

tccli vpc DescribeSubnets --region <Region> \
    --Filters '[{"Name":"vpc-id","Values":["VPC_ID"]}]'
# expected: 至少返回 1 个子网，AvailableIpCount ≥ 3

# 5. 查询可用 etcd 版本
tccli cetcd DescribeEtcdAvailableVersions --region <Region>
# expected: 返回可用版本列表

# 6. 查询配额
tccli cetcd DescribeEtcdQuota --region <Region>
# expected: QuotaLimit > 当前实例数
```

## 关键字段说明

以下说明 `CreateEtcdInstance` 的主要参数：

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `Name` | String | 是 | 实例名称，最长 60 字符 | - |
| `VpcId` | String | 是 | 已存在的 VPC ID，格式 `vpc-xxxxxxxx`。`tccli vpc DescribeVpcs` 获取 | VPC 不存在 → `InvalidParameter.VpcId` |
| `ServiceSubnetId` | String | 是 | 服务子网 ID，用于分配 etcd 集群服务地址 | 子网不存在或 IP 不足 → `InvalidParameter.ServiceSubnetId` |
| `SubnetIds` | Array of String | 是 | 业务子网 ID 列表，用于 etcd 节点 IP 分配。高可用部署时需 ≥ 3 个不同可用区子网 | 子网不可用 → `InvalidParameter.SubnetIds` |
| `EtcdVersion` | String | 是 | 如 `v3.5.12-tke.1`。通过 `DescribeEtcdAvailableVersions` 获取 | 版本不存在 → `InvalidParameter.EtcdVersion` |
| `Size` | Integer | 是 | etcd 集群节点数，Raft 协议要求奇数。高可用推荐 ≥ 3 | Size=1 无法保证高可用 |
| `DiskSize` | Integer | 是 | 存储大小（GB），当前固定 50GB | - |
| `Cpu` | Integer | 是 | CPU 核数。1/2/4/8/16/32。**此参数不在 skeleton 顶层** | 不填 → `InvalidParameter: "Cpu: Required value"` |
| `Memo` | Integer | 是 | 内存大小（GB） | - |
| `ChargeType` | String | 是 | `PREPAID` 或 `POSTPAID_BY_HOUR`。注意不是 `POSTPAID` | 填 `POSTPAID` → `InvalidParameter: "supported values: PREPAID, POSTPAID_BY_HOUR"` |
| `ChargePrepaid.Period` | Integer | PREPAID 时必填 | 购买时长（月），1-36 | 不填 → `InvalidParameter` |
| `ChargePrepaid.RenewFlag` | String | PREPAID 时必填 | 自动续费标识 | 不填 → `InvalidParameter` |
| `DeletionProtection` | Boolean | 否 | 删除保护开关，默认 false。生产环境建议 true | 忘记开启 → 可能误删 |
| `Description` | String | 否 | 实例描述 | - |
| `AdvancedSettings.SecuritySettings.Https` | Boolean | 否 | 是否启用 HTTPS | `false` → 数据传输不加密 |
| `AdvancedSettings.SecuritySettings.ClientCertAuth` | Boolean | 否 | HTTPS 下是否启用客户端证书认证 | - |
| `AdvancedSettings.AutoCompactionSettings` | Object | 否 | 自动压缩配置（周期或版本数模式） | 不配置 → 数据不自动清理，可能影响性能 |
| `AdvancedSettings.BackupSettings` | Object | 否 | 备份配置（间隔 + 最大保留数） | 不配置 → 无自动备份 |

## 操作步骤

### 步骤 1：查询可用版本和配额

#### 选择依据

- **etcd 版本**：推荐 `v3.5.12-tke.1`。当前最新 TKE 定制版，包含性能优化和数据安全增强（全量删除拦截、审计等）。
- **节点数**：生产环境推荐 3 个节点。3 节点为 Raft 共识的最小高可用配置。
- **计费模式**：测试选 `POSTPAID_BY_HOUR`（按小时计费，释放即停），生产选 `PREPAID`（包年包月更优惠）。
- **子网规划**：高可用部署需 3 个不同可用区的子网。`ServiceSubnetId` 是 etcd 集群对外暴露的服务地址所在子网，`SubnetIds` 是 etcd 节点 IP 所在子网。

```bash
# 查询可用版本
tccli cetcd DescribeEtcdAvailableVersions --region <Region>

# 查询配额
tccli cetcd DescribeEtcdQuota --region <Region>
```

### 步骤 2：编写配置并执行创建

#### 最小创建（只含必填字段，按量计费）

`create-etcd-minimal.json`：

```json
{
    "Name": "ETCD_NAME",
    "VpcId": "VPC_ID",
    "ServiceSubnetId": "SUBNET_ID_1",
    "SubnetIds": ["SUBNET_ID_1"],
    "EtcdVersion": "v3.5.12-tke.1",
    "Size": 3,
    "DiskSize": 50,
    "Cpu": 2,
    "Memo": 4,
    "ChargeType": "POSTPAID_BY_HOUR"
}
```

```bash
tccli cetcd CreateEtcdInstance --region <Region> \
    --cli-input-json file://create-etcd-minimal.json
# expected: exit 0，返回 InstanceId
```

**预期输出**：

```json
{
    "InstanceId": "etcd-example",
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

#### 增强配置（生产环境推荐：HTTPS + 备份 + 删除保护 + 包年包月）

`create-etcd-enhanced.json`：

```json
{
    "Name": "ETCD_NAME",
    "VpcId": "VPC_ID",
    "ServiceSubnetId": "SUBNET_ID_1",
    "SubnetIds": ["SUBNET_ID_1", "SUBNET_ID_2", "SUBNET_ID_3"],
    "EtcdVersion": "v3.5.12-tke.1",
    "Size": 3,
    "DiskSize": 50,
    "Cpu": 4,
    "Memo": 8,
    "ChargeType": "PREPAID",
    "ChargePrepaid": {
        "Period": 1,
        "RenewFlag": "NOTIFY_AND_MANUAL_RENEW"
    },
    "DeletionProtection": true,
    "Description": "ETCD_DESCRIPTION",
    "AdvancedSettings": {
        "SecuritySettings": {
            "Https": true,
            "ClientCertAuth": true
        },
        "AutoCompactionSettings": {
            "Mode": "periodic",
            "Period": "1h"
        },
        "BackupSettings": {
            "Interval": 6,
            "MaxCount": 100
        }
    }
}
```

```bash
tccli cetcd CreateEtcdInstance --region <Region> \
    --cli-input-json file://create-etcd-enhanced.json
# expected: exit 0，返回 InstanceId
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `ETCD_NAME` | 实例名称 | 最长 60 字符 | 自定义 |
| `VPC_ID` | VPC 实例 ID | 格式 `vpc-xxxxxxxx` | `tccli vpc DescribeVpcs` |
| `SUBNET_ID_1/2/3` | 子网 ID | 格式 `subnet-xxxxxxxx`，高可用需 ≥ 3 个不同 AZ | `tccli vpc DescribeSubnets` |
| `ETCD_DESCRIPTION` | 实例描述 | 可选 | 自定义 |

### 步骤 3：轮询等待集群就绪

创建是异步操作。轮询直到状态为 `Running`：

```bash
tccli cetcd DescribeEtcdInstances --region <Region> \
    | jq '.InstanceSet[] | select(.InstanceId=="INSTANCE_ID")'
# expected: Status 为 "Running"
```

**验证维度**：

| 维度 | 命令 | 预期 |
|------|------|------|
| 状态 | `DescribeEtcdInstances` 过滤 InstanceId | `Status: "Running"` |
| 版本 | 同上，检查 `EtcdVersion` | 与创建参数一致（如 `v3.5.12-tke.1`） |
| 节点数 | 同上 | 与创建 Size 一致 |
| 计费类型 | `DescribeEtcdInstance` | 与 ChargeType 一致 |
| HTTPS | 通过 etcdctl 测试 | `etcdctl --endpoints=https://INSTANCE_ID.etcd.tencentcloudapi.com:2379 endpoint health` |

## 验证

```bash
# 确认实例运行状态
tccli cetcd DescribeEtcdInstance --region <Region> \
    --InstanceId INSTANCE_ID
# expected: Status=Running, EtcdVersion=v3.5.12-tke.1

# 获取访问凭证
tccli cetcd DescribeEtcdCredentials --region <Region> \
    --InstanceId INSTANCE_ID
# expected: 返回 CA 证书、客户端证书和密钥

# 使用 etcdctl 测试连通性（需先保存凭证）
etcdctl --endpoints=ENDPOINT_URL \
    --cacert=/path/to/ca.crt \
    --cert=/path/to/client.crt \
    --key=/path/to/client.key \
    endpoint health
# expected: exit 0, "is healthy"
```

## 清理

> **警告**：删除 etcd 实例不可逆。`DestroyEtcdInstance`（包年包月）/ `DeleteEtcdInstance`（按量计费）会释放所有节点资源。关联的 COS 快照数据（如勾选删除）和 CLB 资源将一并释放。生产环境操作前务必确认。

### 清理前状态检查

```bash
tccli cetcd DescribeEtcdInstances --region <Region>
# 确认是待删除的目标实例，记录 InstanceId、InstanceName、EtcdVersion
```

### 按量计费实例删除

```bash
# 1. 关闭删除保护（如已启用）
tccli cetcd DisableEtcdInstanceDeletionProtection --region <Region> \
    --InstanceId INSTANCE_ID

# 2. 删除实例
tccli cetcd DeleteEtcdInstance --region <Region> \
    --InstanceId INSTANCE_ID
```

### 包年包月实例退还

```bash
tccli cetcd DestroyEtcdInstance --region <Region> \
    --InstanceId INSTANCE_ID
```

### 验证已删除

```bash
tccli cetcd DescribeEtcdInstances --region <Region>
# expected: 目标实例不再出现在列表中
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `CreateEtcdInstance` 返回 `InvalidParameter: "supported values: PREPAID, POSTPAID_BY_HOUR"` | 检查 JSON 中 `ChargeType` 的值 | 填了 `POSTPAID` 而非 `POSTPAID_BY_HOUR` | 改为 `"POSTPAID_BY_HOUR"` |
| `CreateEtcdInstance` 返回 `InvalidParameter: "Cpu: Required value"` | 检查 JSON 中是否缺少 `Cpu` 字段 | `Cpu` 是必填参数，不在 skeleton 顶层定义中 | 在 JSON 中添加 `"Cpu": 2`（按需调整核数） |
| `CreateEtcdInstance` 返回 `InvalidParameter.VpcId` | `tccli vpc DescribeVpcs --region <Region> --VpcIds '["VPC_ID"]'` 检查是否存在 | VPC ID 不存在或不属于当前地域 | 修正为正确的 VPC ID |
| `CreateEtcdInstance` 返回 `InvalidParameter.SubnetIds` | `tccli vpc DescribeSubnets --region <Region> --SubnetIds '["SUBNET_ID"]'` 检查子网 | 子网不存在、IP 不足或可用区与要求不符 | 选择可用 IP ≥ 3 的子网 |
| `CreateEtcdInstance` 返回 `LimitExceeded` | `tccli cetcd DescribeEtcdQuota --region <Region>` 检查配额 | 实例数量达到上限（此为环境限制，非命令错误） | 删除不再使用的实例后重试 |

### 创建已提交但长时间不 Running

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 返回了 InstanceId 但 10 分钟后状态仍非 Running | `tccli cetcd DescribeEtcdInstance --region <Region> --InstanceId INSTANCE_ID` 查看 Status | 后端创建缓慢（正常）或卡住（异常） | 继续等待；超过 15 分钟则保留 region、InstanceId、RequestId、创建 JSON → 登录 [etcd 控制台](https://console.cloud.tencent.com/tke2/etcd/list) 查看详细状态 → 仍无法解决则 [提交工单](https://console.cloud.tencent.com/workorder) |
| 状态为 `Abnormal` 或 `Failed` | 同上，检查 Status 和 StatusReason | 网络配置、资源不足或配额问题 | 记录 StatusReason → DeleteEtcdInstance 重新创建 |

## 下一步

- [监控和告警配置](../监控和告警配置/tccli%20操作.md) — 配置 etcd 监控
- [节点扩容和升降配](../节点扩容和升降配/tccli%20操作.md) — 调整节点规格
- [快照管理](../快照管理/tccli%20操作.md) — 配置数据备份
- [删除集群](../删除集群/tccli%20操作.md) — 删除 etcd 集群

## 控制台替代

访问 [云原生 etcd 控制台](https://console.cloud.tencent.com/tke2/etcd/list)，点击新建，按四步向导完成创建。
