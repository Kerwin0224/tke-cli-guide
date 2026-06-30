# 云原生 etcd 概述

> 对照官方：[云原生 etcd 概述](https://cloud.tencent.com/document/product/457/58176) · page_id `58176`

## 概述

云原生 etcd（Tencent Cloud Native etcd）是腾讯云容器团队基于开源 etcd 提供的高可用、可观测、免运维的托管 etcd 服务。完全兼容开源 etcd 的分布式键值存储能力，主要用于服务发现、分布式选举和元数据存储等场景。

**方案对比：**

| 方案 | 适用场景 | 运维成本 |
|------|---------|---------|
| 云原生 etcd | 需要高可用 etcd 且不想运维 | 零运维，腾讯云负责部署、版本更新、故障处理 |
| 自建 etcd | 已有运维团队，需完全掌控 | 需自行维护监控、备份、故障恢复 |

> 云原生 etcd 已于 2022 年 6 月 8 日结束内测并正式商业化计费。详见 [购买指南](购买指南/tccli%20操作.md)。

## 前置条件

- [环境准备](../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置

# 3. 检查 CAM 权限 — cetcd 服务需要以下 Action 名
#    cetcd:DescribeEtcdInstances
# 验证：执行 DescribeEtcdInstances 确认权限
tccli cetcd DescribeEtcdInstances --region <Region>
# expected: exit 0，返回实例列表（可为空）
```

### 资源检查

```bash
# 4. 查询已有 etcd 实例
tccli cetcd DescribeEtcdInstances --region <Region>
# expected: 返回 TotalCount 和 InstanceSet 数组
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看 etcd 实例列表 | `DescribeEtcdInstances` | 是 |
| 查看 etcd 实例详情 | `DescribeEtcdInstance` | 是 |
| 查看可用版本 | `DescribeEtcdAvailableVersions` | 是 |
| 查看地域列表 | `DescribeEtcdRegions` | 是 |
| 查看配额 | `DescribeEtcdQuota` | 是 |

## 操作步骤

### 查看 etcd 实例列表

```bash
tccli cetcd DescribeEtcdInstances --region <Region>
# expected: exit 0，返回 TotalCount 和实例列表
```

**预期输出**：

```json
{
    "TotalCount": 17,
    "InstanceSet": [
        {
            "InstanceId": "etcd-example",
            "InstanceName": "my-etcd-cluster",
            "Status": "Running",
            "EtcdVersion": "v3.5.12-tke.1"
        }
    ]
}
```

### 查看可用版本

```bash
tccli cetcd DescribeEtcdAvailableVersions --region <Region>
# expected: exit 0，返回可用版本列表
```

**预期输出**：

```json
{
    "Versions": [
        {"Version": "v3.5.4"},
        {"Version": "v3.4.13-tke.3"},
        {"Version": "v3.4.13-tke.5"},
        {"Version": "v3.5.4-tke.5"},
        {"Version": "v3.5.12-tke.1"}
    ]
}
```

### 查看配额

```bash
tccli cetcd DescribeEtcdQuota --region <Region>
# expected: exit 0，返回 QuotaLimit
```

**预期输出**：

```json
{
    "QuotaLimit": 21
}
```

### 查看支持的地域

```bash
tccli cetcd DescribeEtcdRegions --region <Region>
# expected: exit 0，返回地域列表
```

## 验证

`DescribeEtcdInstances` 返回 `exit 0` 且 `TotalCount` 与预期一致即可确认 API 连通性正常。

## 清理

本页面为只读操作，无需清理。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DescribeEtcdInstances` 返回 `AuthFailure` | 执行 `tccli configure list` 检查凭据 | CAM 权限不足或凭据过期 | 确认已授予 `cetcd:DescribeEtcdInstances` 权限，检查 secretId/secretKey 是否有效 |
| `DescribeEtcdInstances` 返回空结果 | 切换地域重试 `tccli cetcd DescribeEtcdInstances --region ap-guangzhou` | 当前地域无 etcd 实例 | 切换到有实例的地域，或先创建实例 |

## 下一步

- [创建集群](创建集群/tccli%20操作.md) — 创建新的 etcd 集群
- [购买指南](购买指南/tccli%20操作.md) — 了解计费模式与规格
- [版本维护机制](云原生%20etcd%20版本维护/版本维护机制/tccli%20操作.md) — 了解版本支持策略
- [官方 etcd 文档](https://etcd.io/docs/) — 开源 etcd 参考

## 控制台替代

访问 [云原生 etcd 控制台](https://console.cloud.tencent.com/tke2/etcd/list)。
