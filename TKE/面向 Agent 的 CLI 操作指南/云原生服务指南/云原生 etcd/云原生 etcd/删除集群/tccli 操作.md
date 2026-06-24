# 删除集群

> 对照官方：[删除集群](https://cloud.tencent.com/document/product/457/102553) · page_id `102553`

## 概述

删除云原生 etcd 集群。**关键限制**：禁级联删除——必须先清空 etcd 中存储的所有键值数据，才能删除集群。操作不可逆。对于开启密码认证的集群，数据存储状态无法检测，级联删除限制不适用，但仍需谨慎操作。

| 实例计费类型 | 删除 API | 计费停止 |
|------------|---------|---------|
| POSTPAID_BY_HOUR（按量计费） | `DeleteEtcdInstance` | 删除后立即停止计费 |
| PREPAID（包年包月） | `DestroyEtcdInstance` | 按退费规则处理（参见 [包年包月实例退费说明](../包年包月实例退费说明/tccli%20操作.md)） |

## 前置条件

- [环境准备](../../../../环境准备.md)
- 已安装 etcdctl（用于清空数据）
- 已获取实例的访问凭证

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置

# 3. 检查 etcdctl 版本
etcdctl version
# expected: etcdctl version >= 3.4.0

# 4. 检查 CAM 权限 — 需要 cetcd:DeleteEtcdInstance, cetcd:DescribeEtcdInstances
tccli cetcd DescribeEtcdInstances --region <Region>
# expected: exit 0
```

### 资源检查

```bash
# 5. 确认目标实例
tccli cetcd DescribeEtcdInstance --region <Region> --InstanceId INSTANCE_ID
# 确认 InstanceName、EtcdVersion、计费类型，确保是待删除的目标
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看实例列表 | `DescribeEtcdInstances` | 是 |
| 获取访问凭证 | `DescribeEtcdCredentials` | 是 |
| 关闭删除保护 | `DisableEtcdInstanceDeletionProtection` | 否 |
| 删除实例（按量计费） | `DeleteEtcdInstance` | 否 |
| 销毁实例（包年包月） | `DestroyEtcdInstance` | 否 |

## 操作步骤

### 步骤 1：清空存储数据

> **重要**：云原生 etcd 禁止级联删除，必须先清空所有键值数据。执行前务必创建最终快照备份。

#### 选择依据

- 清空数据是用 `etcdctl del "" --prefix` 删除所有 key（`--prefix` 匹配所有前缀）。操作不可逆。
- 先创建快照是为了保留删除前的数据状态，以备需要恢复。

#### 获取凭证并保存

```bash
tccli cetcd DescribeEtcdCredentials --region <Region> \
    --InstanceId INSTANCE_ID
# expected: exit 0，返回 CaCert、ClientCert、ClientKey、Endpoints

# 将凭证保存到文件
echo "CA_CERT_CONTENT" > /tmp/etcd-ca.crt
echo "CLIENT_CERT_CONTENT" > /tmp/etcd-client.crt
echo "CLIENT_KEY_CONTENT" > /tmp/etcd-client.key
```

#### 创建最终快照（备份）

```bash
etcdctl --endpoints=ENDPOINT_URL \
    --cacert=/tmp/etcd-ca.crt \
    --cert=/tmp/etcd-client.crt \
    --key=/tmp/etcd-client.key \
    --command-timeout=30s \
    snapshot save INSTANCE_ID-final.db
# expected: exit 0，生成快照文件

# 验证快照完整性
etcdctl snapshot status INSTANCE_ID-final.db --write-out=table
# expected: 显示 hash、revision、totalKey、totalSize
```

#### 清空所有数据

```bash
etcdctl --endpoints=ENDPOINT_URL \
    --cacert=/tmp/etcd-ca.crt \
    --cert=/tmp/etcd-client.crt \
    --key=/tmp/etcd-client.key \
    del "" --prefix
# expected: exit 0
```

### 步骤 2：关闭删除保护（如已启用）

```bash
tccli cetcd DescribeEtcdInstance --region <Region> \
    --InstanceId INSTANCE_ID \
    | jq '.DeletionProtection'
# expected: true 或 false

# 如果为 true，先关闭
tccli cetcd DisableEtcdInstanceDeletionProtection --region <Region> \
    --InstanceId INSTANCE_ID
# expected: exit 0
```

### 步骤 3：删除实例

#### 按量计费实例

```bash
tccli cetcd DeleteEtcdInstance --region <Region> \
    --InstanceId INSTANCE_ID
# expected: exit 0
```

> **警告**：CLB 资源（如开启公网访问产生）将在云原生 etcd 实例删除时一并释放。

#### 包年包月实例

```bash
tccli cetcd DestroyEtcdInstance --region <Region> \
    --InstanceId INSTANCE_ID
# expected: exit 0
```

> **警告**：销毁后实例磁盘数据被回收、数据丢失、运行服务永久终止且不支持续费恢复。如已开启备份，可选择是否在销毁时同时释放 COS 快照数据。

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `INSTANCE_ID` | etcd 实例 ID | 格式 `etcd-xxxxxxxx` | `tccli cetcd DescribeEtcdInstances` |
| `ENDPOINT_URL` | etcd 集群访问端点 | 含端口 | `tccli cetcd DescribeEtcdCredentials` |

## 验证

```bash
tccli cetcd DescribeEtcdInstances --region <Region>
# expected: 目标实例不再出现在列表中
```

## 清理

删除操作本身即为清理。注意以下资源的状态：

- **COS 快照数据**：默认保留，需在控制台确认是否同时删除
- **CLB 资源**：随实例一并释放
- **COS 桶**：保留不删除

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DeleteEtcdInstance` 返回 `FailedOperation`，提示有存储数据 | etcdctl 检查 key 数量 | etcd 中仍有未清理的数据（级联删除限制） | 执行 `etcdctl del "" --prefix` 清空所有数据后重试 |
| `DisableEtcdInstanceDeletionProtection` 返回 `InvalidParameter` | 检查 InstanceId | InstanceId 不存在或已删除 | 用 `DescribeEtcdInstances` 确认 InstanceId |
| `snapshot save` 超时（30 秒） | 检查实例状态和数据量 | 实例负载过高或数据量过大 | 增加超时时间 `--command-timeout=120s` |
| `etcdctl del "" --prefix` 权限不足 | 检查证书路径和格式 | 证书过期、格式错误或权限不足 | 重新获取凭证：`tccli cetcd DescribeEtcdCredentials --region <Region> --InstanceId INSTANCE_ID` |

### 删除已提交但实例仍在列表中

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 返回了 RequestId 但实例仍未消失 | `tccli cetcd DescribeEtcdInstances --region <Region>` 查看状态 | 删除操作异步执行 | 等待 2-3 分钟后重试 |
| 实例状态为 `Deleting` 持续超过 10 分钟 | 同上 | 删除异常 | 保留 region、InstanceId、RequestId → [提交工单](https://console.cloud.tencent.com/workorder) |

## 下一步

- [创建集群](../创建集群/tccli%20操作.md) — 创建新 etcd 实例
- [包年包月实例退费说明](../包年包月实例退费说明/tccli%20操作.md) — 预付费退还规则
- [快照管理](../快照管理/tccli%20操作.md) — 查看删除前备份的快照

## 控制台替代

访问 [云原生 etcd 控制台](https://console.cloud.tencent.com/tke2/etcd/list)，在实例右侧选择 更多 > 销毁/退货。注意控制台同样要求先清空数据。
