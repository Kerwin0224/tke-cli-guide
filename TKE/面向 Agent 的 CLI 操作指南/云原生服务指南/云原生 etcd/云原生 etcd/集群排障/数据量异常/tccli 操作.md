# 数据量异常

> 对照官方：[数据量异常](https://cloud.tencent.com/document/product/457/109850) · page_id `109850`

## 概述

etcd 集群出现数据量异常，Key 或 Lease 数量与预期不符，增加运维风险。需通过 etcdctl 命令行工具导出数据进行对比排查，并考虑将大数据量拆分到多个集群以降低风险。

**风险阈值**：
- Key 数量超过 **100 万** → 运维风险高，建议业务侧拆分
- Lease 数量超过 **10 万** → 运维风险高，建议业务侧拆分

## 前置条件

- [环境准备](../../../../../环境准备.md)
- 已安装 etcdctl（版本 ≥ 3.4.0）
- 已获取实例的访问凭证

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查 etcdctl 版本
etcdctl version
# expected: etcdctl version >= 3.4.0

# 3. 检查 CAM 权限 — 需要 cetcd:DescribeEtcdCredentials, cetcd:DescribeEtcdInstance
tccli cetcd DescribeEtcdInstances --region <Region>
# expected: exit 0

# 4. 获取实例访问凭证
tccli cetcd DescribeEtcdCredentials --region <Region> \
    --InstanceId INSTANCE_ID
# expected: exit 0，返回凭证信息

# 将凭证保存到文件
echo "CA_CERT_CONTENT" > /tmp/etcd-ca.crt
echo "CLIENT_CERT_CONTENT" > /tmp/etcd-client.crt
echo "CLIENT_KEY_CONTENT" > /tmp/etcd-client.key
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看实例详情 | `DescribeEtcdInstance` | 是 |
| 获取凭证 | `DescribeEtcdCredentials` | 是 |
| 查看监控指标 | `DescribeEtcdMonitorData` | 是 |
| 查看 Key 数量 | etcdctl 命令（数据面） | 是 |
| 查看 Lease 数量 | etcdctl 命令（数据面） | 是 |

## 操作步骤

### 步骤 1：查看实例监控中的 Key 数量趋势

```bash
tccli cetcd DescribeEtcdMonitorData --region <Region> \
    --InstanceId INSTANCE_ID \
    --MetricName EtcdKeyCount
# expected: exit 0，返回 Key 数量时序数据
```

### 步骤 2：通过 etcdctl 导出所有 Key 进行对比

```bash
# 导出所有 Key（仅 Key 名，不含 value）
etcdctl --endpoints=ENDPOINT_URL \
    --cacert=/tmp/etcd-ca.crt \
    --cert=/tmp/etcd-client.crt \
    --key=/tmp/etcd-client.key \
    get "" --from-key --keys-only > /tmp/etcd-keys.txt

# 统计 Key 总数
wc -l /tmp/etcd-keys.txt
# expected: 返回 Key 总数
```

### 步骤 3：查看 Lease 数量

```bash
etcdctl --endpoints=ENDPOINT_URL \
    --cacert=/tmp/etcd-ca.crt \
    --cert=/tmp/etcd-client.crt \
    --key=/tmp/etcd-client.key \
    lease list | wc -l
# expected: 返回 Lease 总数
```

### 步骤 4：分析并处理

#### 选择依据

根据排查结果采取不同策略：

| 数据量 | 风险等级 | 建议操作 |
|--------|---------|---------|
| Key < 10 万 | 正常 | 保持现状，定期检查 |
| Key 10 万 ~ 100 万 | 关注 | 排查是否有未清理的过期数据，考虑开启自动压缩 |
| Key > 100 万 | **高风险** | 业务侧拆分数据到多个集群 |
| Lease < 1 万 | 正常 | 保持现状 |
| Lease 1 万 ~ 10 万 | 关注 | 排查是否有未释放的 Lease |
| Lease > 10 万 | **高风险** | 业务侧拆分 Lease 到多个集群 |

#### 定期清理建议（适用于异常增长场景）

```bash
# 查看占用空间最多的 Key 前缀
etcdctl --endpoints=ENDPOINT_URL \
    --cacert=/tmp/etcd-ca.crt \
    --cert=/tmp/etcd-client.crt \
    --key=/tmp/etcd-client.key \
    get "" --from-key --keys-only \
    | cut -d'/' -f1,2 | sort | uniq -c | sort -rn | head -20
# expected: 显示 Key 前缀分布（如 /registry/pods、/registry/configmaps）

# 开启自动压缩（如未开启）
tccli cetcd ModifyEtcdConfiguration --region <Region> \
    --InstanceId INSTANCE_ID
# 参见 自动压缩管理 页面完成完整配置
```

### 步骤 5：持续监控

```bash
# 定期检查 Key 数量趋势
tccli cetcd DescribeEtcdMonitorData --region <Region> \
    --InstanceId INSTANCE_ID \
    --MetricName EtcdKeyCount \
    --StartTime START_TIME \
    --EndTime END_TIME
# expected: Key 数量增长趋势正常或下降
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `INSTANCE_ID` | etcd 实例 ID | 格式 `etcd-xxxxxxxx` | `tccli cetcd DescribeEtcdInstances` |
| `ENDPOINT_URL` | etcd 集群访问端点 | 含端口 | `tccli cetcd DescribeEtcdCredentials` |
| `START_TIME` / `END_TIME` | 监控查询起止时间 | ISO 8601 格式 | 自定义 |

## 验证

| 维度 | 命令 | 预期 |
|------|------|------|
| Key 数量 | etcdctl + monitoring | 在业务预期范围内 |
| Lease 数量 | `etcdctl lease list \| wc -l` | < 10 万 |
| 监控趋势 | `DescribeEtcdMonitorData` | 无明显异常增长 |

## 清理

本页面为排障页，无需清理。如有临时文件，删除即可：

```bash
rm /tmp/etcd-ca.crt /tmp/etcd-client.crt /tmp/etcd-client.key /tmp/etcd-keys.txt
```

## 排障

### 命令执行异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `etcdctl get "" --from-key --keys-only` 返回空 | `etcdctl endpoint health` 检查连接 | 证书过期或端点配置错误 | 重新获取凭证并确认端点地址 |
| `etcdctl get` 响应超时 | 检查 etcd 负载（CPU/内存使用率） | Key 数量过大导致查询缓慢 | 分批查询或使用 `--limit` 参数限制单次返回数量 |
| `etcdctl lease list` 长时间无响应 | 检查 Lease 数量 | Lease 过多导致列表超时 | 增加 `--command-timeout` 参数 |

### 数据量异常排查

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Key 数量远超业务预期 | 按前缀统计 Key 分布 | 业务逻辑未清理过期数据或存在异常写入 | 根据前缀分布定位来源服务，从业务侧修复，手动清理过期 Key |
| 数据库大小远大于预期 | 检查是否开启了自动压缩 | 未配置自动压缩导致历史版本堆积 | 参见 [自动压缩管理](../../自动压缩管理/tccli%20操作.md) 配置压缩策略 |
| Key 被清理但数据库大小未减少 | 检查 etcd 监控 "数据库大小" | etcd 的 MVCC 特性导致空间不会立即可用 | 正常现象，等待自动压缩释放空间，或通过 etcdctl defrag 整理（需联系腾讯云运维） |

## 下一步

- [自动压缩管理](../../自动压缩管理/tccli%20操作.md) — 配置自动压缩防止数据堆积
- [快照管理](../../快照管理/tccli%20操作.md) — 备份当前数据
- [节点扩容和升降配](../../节点扩容和升降配/tccli%20操作.md) — 增加节点规格以承载更多数据
- [监控和告警配置](../../监控和告警配置/tccli%20操作.md) — 配置告警提前发现数据量异常

## 控制台替代

访问 [云原生 etcd 控制台](https://console.cloud.tencent.com/tke2/etcd/list)，点击实例进入 详情页 > 实例监控，查看 "数据库 key 数量" 等业务指标。
