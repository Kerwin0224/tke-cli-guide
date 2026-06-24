# 数据同步

> 对照官方：[数据同步](https://cloud.tencent.com/document/product/457/90280) · page_id `90280`

## 概述

将源 etcd 集群的数据同步到云原生 etcd 集群。通过创建同步任务（Snapshot 一次性导入方式），将源集群数据自动导入目标实例。前提是源集群和目标集群之间网络可达。

| 同步方式 | 适用场景 | 注意事项 |
|---------|---------|---------|
| Snapshot 导入（同步任务） | 一次性全量迁移 | 目标集群需停止业务写入，源集群和目标集群需在同一 VPC |
| 手动快照导入 | 跨 VPC 或外部集群迁移 | 参见 [快照管理 → 导入快照文件](../快照管理/tccli%20操作.md) |

## 前置条件

- [环境准备](../../../../环境准备.md)
- 已有运行中的目标 etcd 实例
- 源 etcd 集群与目标集群在同一 VPC（或已建立网络互通）

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置

# 3. 检查 CAM 权限 — 需要 cetcd:DescribeEtcdInstances
tccli cetcd DescribeEtcdInstances --region <Region>
# expected: exit 0

# 4. 确认目标实例运行中
tccli cetcd DescribeEtcdInstance --region <Region> --InstanceId INSTANCE_ID | jq '.Status'
# expected: "Running"

# 5. 获取目标实例访问凭证
tccli cetcd DescribeEtcdCredentials --region <Region> \
    --InstanceId INSTANCE_ID
# expected: 返回证书和端点信息
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看目标实例信息 | `DescribeEtcdInstance` | 是 |
| 获取实例凭证 | `DescribeEtcdCredentials` | 是 |
| 创建同步任务 | `CreateEtcdSyncTask` | 否 |
| 查看同步任务状态 | `DescribeEtcdSyncTasks` | 是 |

## 关键字段说明

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `InstanceId` | String | 是 | 目标 etcd 实例 ID | 不存在 → `InvalidParameter` |
| `SourceEndpoint` | String | 是 | 源 etcd 集群访问地址，含端口（如 `https://SOURCE_IP:2379`） | 网络不可达 → 连接测试失败 |
| `SourceAuth` | Object | 是 | 源集群认证信息（证书或密钥），取决于源集群认证方式 | 认证不匹配 → 连接测试失败 |

## 操作步骤

### 步骤 1：获取目标实例凭证

```bash
tccli cetcd DescribeEtcdCredentials --region <Region> \
    --InstanceId INSTANCE_ID
# expected: exit 0，返回 CaCert、ClientCert、ClientKey、Endpoints
```

### 步骤 2：创建同步任务

#### 选择依据

- 同步任务是一次性全量导入操作。适合迁移场景，不适合持续同步。
- 源集群与目标集群必须在同一 VPC，否则网络不可达。
- 建议暂停业务对源集群的写入后再执行同步，确保数据一致性。
- 同步过程预计约 5 分钟完成。

`sync-task.json`：

```json
{
    "InstanceId": "INSTANCE_ID",
    "SourceEndpoint": "https://SOURCE_ADDRESS:2379",
    "SourceAuth": {
        "CaCert": "CA_CERT_CONTENT",
        "ClientCert": "CLIENT_CERT_CONTENT",
        "ClientKey": "CLIENT_KEY_CONTENT"
    }
}
```

```bash
tccli cetcd CreateEtcdSyncTask --region <Region> \
    --cli-input-json file://sync-task.json
# expected: exit 0，返回 TaskId
```

**预期输出**：

```json
{
    "TaskId": "sync-task-example",
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

### 步骤 3：查看同步任务状态

```bash
tccli cetcd DescribeEtcdSyncTasks --region <Region> \
    --InstanceId INSTANCE_ID
# expected: exit 0，返回任务列表和状态
```

> 后台会依次执行四项检查：网络可达性、权限正确性、版本支持性、数据大小。全部通过后开始同步，预计约 5 分钟完成。

### 步骤 4：验证同步结果

```bash
# 使用 etcdctl 检查目标集群 key 数量
etcdctl --endpoints=TARGET_ENDPOINT \
    --cacert=/path/to/ca.crt \
    --cert=/path/to/client.crt \
    --key=/path/to/client.key \
    get "" --from-key --keys-only | wc -l
# expected: key 数量与源集群一致
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `INSTANCE_ID` | 目标 etcd 实例 ID | 格式 `etcd-xxxxxxxx` | `tccli cetcd DescribeEtcdInstances` |
| `SOURCE_ADDRESS` | 源 etcd 集群地址 | 与目标实例同 VPC | 源集群基本信息页 |
| `CA_CERT_CONTENT` | CA 证书内容 | PEM 格式 | 源集群凭证信息 |

## 验证

| 维度 | 命令 | 预期 |
|------|------|------|
| 同步完成 | `DescribeEtcdSyncTasks` | 状态为 `Success` |
| 数据完整性 | etcdctl get "" --from-key --keys-only | key 数量与源一致 |
| 集群健康 | etcdctl endpoint health | 全部 healthy |

## 清理

同步任务为一次性操作，完成后无需清理。如需删除目标实例中已同步的数据：

```bash
etcdctl --endpoints=TARGET_ENDPOINT \
    --cacert=/path/to/ca.crt \
    --cert=/path/to/client.crt \
    --key=/path/to/client.key \
    del "" --prefix
# ⚠️ 删除集群中所有 key-value 数据，不可逆
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `CreateEtcdSyncTask` 连接测试失败：网络不可达 | 确认源集群与目标集群 VPC | 源集群与目标不在同一 VPC | 使用云联网打通 VPC，或改用 [快照导入](../快照管理/tccli%20操作.md) 方式 |
| `CreateEtcdSyncTask` 连接测试失败：权限错误 | 检查源集群认证信息是否正确 | 证书或密钥不匹配 | 从源集群重新获取正确凭证 |
| `CreateEtcdSyncTask` 连接测试失败：版本不支持 | `tccli cetcd DescribeEtcdAvailableVersions` 查看支持范围 | 源集群 etcd 版本不在平台支持范围内 | 确认源集群版本属于 v3.4.x 或 v3.5.x |
| 同步任务长时间未完成 | `DescribeEtcdSyncTasks` 查询状态 | 源集群数据量较大 | 大数据量同步可能需要更长时间 |

## 下一步

- [快照管理](../快照管理/tccli%20操作.md) — 通过快照实现跨 VPC 数据迁移
- [监控和告警配置](../监控和告警配置/tccli%20操作.md) — 监控同步后的集群状态
- [删除集群](../删除集群/tccli%20操作.md) — 同步完成后删除源集群

## 控制台替代

访问 [云原生 etcd 控制台](https://console.cloud.tencent.com/tke2/etcd/list)，点击实例进入 数据同步 页面，点击 创建同步任务。
