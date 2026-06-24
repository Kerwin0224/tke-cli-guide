# 版本变更说明

> 对照官方：[版本变更说明](https://cloud.tencent.com/document/product/457/83193) · page_id `83193`

## 概述

云原生 etcd 在社区版本基础上提供了 TKE 定制修复版本（带 `-tke.x` 后缀），增强数据安全、审计和性能。本文对照各版本的变更内容，帮助用户选择合适版本。

当前可用版本：

```json
{"Versions": [
  {"Version": "v3.5.4"},
  {"Version": "v3.4.13-tke.3"},
  {"Version": "v3.4.13-tke.5"},
  {"Version": "v3.5.4-tke.5"},
  {"Version": "v3.5.12-tke.1"}
]}
```

## 前置条件

- [环境准备](../../../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置

# 3. 检查 CAM 权限 — 需要 cetcd:DescribeEtcdAvailableVersions
tccli cetcd DescribeEtcdAvailableVersions --region <Region>
# expected: exit 0
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看可用版本 | `DescribeEtcdAvailableVersions` | 是 |
| 查看实例版本 | `DescribeEtcdInstance` | 是 |

## 关键字段说明

以下说明 `CreateEtcdInstance` 中与版本选择相关的参数：

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `EtcdVersion` | String | 是 | 如 `v3.5.12-tke.1`。通过 `DescribeEtcdAvailableVersions` 获取可用版本列表。不带后缀的为社区版，带 `tke.x` 的为 TKE 定制修复版 | 填不可用版本 → `InvalidParameter.EtcdVersion` |

## 操作步骤

### 查看版本列表与变更说明

```bash
tccli cetcd DescribeEtcdAvailableVersions --region <Region>
# expected: exit 0，返回 5 个版本
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

### v3.4.13 TKE 修订记录

| 日期 | 版本 | 更新内容 |
|------|------|---------|
| 2022-09-02 | v3.4.13-tke.5 | 审计日志记录请求错误码 |
| 2022-08-01 | v3.4.13-tke.4 | **数据安全**：全量删除拦截（`ETCD_FORBID_DELETING_ALL_KEYS=true`）和批量删除拦截（`ETCD_MAX_DELETE_KEY_NUM=1000`），默认关闭。**运维能力**：审计覆盖删除请求和高延迟请求 |
| 2022-07-16 | v3.4.13-tke.3 | 性能优化：支持配置最大并发流参数 |
| 2020-11-02 | v3.4.13-tke.2 | count-only 场景性能优化；高延迟请求详细输出；limit 参数下推到索引层 |

### v3.5.4 TKE 修订记录

| 日期 | 版本 | 更新内容 |
|------|------|---------|
| 2024-08-15 | v3.5.4-tke.5 | 移植社区 PR14419，修复告警列表操作可能触发数据不一致的 bug |
| 2022-11-24 | v3.5.4-tke.4 | 移植社区 PR14733，修复 defrag 操作可能触发数据不一致的 bug |
| 2022-09-02 | v3.5.4-tke.3 | 审计日志记录请求错误码 |
| 2022-08-25 | v3.5.4-tke.2 | **数据安全**：全量删除拦截和批量删除拦截（默认关闭）。**运维能力**：审计覆盖删除请求和高延迟请求 |
| 2022-07-19 | v3.5.4-tke.1 | 性能优化：移植社区 PR14219，支持最大并发流参数配置 |

### v3.5.12 TKE 修订

| 日期 | 版本 | 更新内容 |
|------|------|---------|
| - | v3.5.12-tke.1 | 基于社区 v3.5.12 的 TKE 定制版本，支持当前创建集群 |

## 验证

- `DescribeEtcdAvailableVersions` 返回 ≥ 5 个版本，覆盖 v3.4 和 v3.5 系列
- 新版本 `v3.5.12-tke.1` 可用，推荐用于新建集群

## 清理

本页面为只读操作，无需清理。

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 特定版本不可用 | `tccli cetcd DescribeEtcdAvailableVersions --region <Region>` | 旧版本已停止提供新建 | 使用最新可用版本创建集群 |
| 版本选择错误导致创建失败 | 检查请求中的 `EtcdVersion` 值 | 填了不存在的版本号 | 用 `DescribeEtcdAvailableVersions` 获取精确版本字符串 |

## 下一步

- [版本维护机制](../版本维护机制/tccli%20操作.md) — 了解版本支持策略
- [创建集群](../../创建集群/tccli%20操作.md) — 使用最新版本创建集群
- [etcd 社区发布说明](https://github.com/etcd-io/etcd/releases) — 上游变更详情

## 控制台替代

访问 [云原生 etcd 控制台](https://console.cloud.tencent.com/tke2/etcd/list)，创建集群时查看可选版本。
