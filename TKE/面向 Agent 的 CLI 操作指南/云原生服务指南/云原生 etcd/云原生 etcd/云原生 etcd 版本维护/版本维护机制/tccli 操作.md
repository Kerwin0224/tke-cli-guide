# 版本维护机制

> 对照官方：[版本维护机制](https://cloud.tencent.com/document/product/457/83192) · page_id `83192`

## 概述

云原生 etcd 遵循语义化版本规范（x.y.z）。平台支持**两个最新大版本**的集群创建和运维保障。新集群推荐选择最新大版本（如 v3.5.x），已有集群建议及时升级以保持技术支持和 Bugfix 覆盖。

| 版本系列 | 状态 | 建议 |
|---------|------|------|
| v3.5.x | 当前最新大版本，推荐 | 新集群首选，性能优化最多 |
| v3.4.x | 仍受支持，但后续可能停止新建 | 已有集群继续使用，计划升级 |
| 更早版本 | 已过期 | 尽快升级到受支持版本 |

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
| 查看实例列表（含版本信息） | `DescribeEtcdInstances` | 是 |
| 升级版本 | `UpgradeEtcdClusterVersion` | 否 |

## 操作步骤

### 查看平台当前支持的版本

```bash
tccli cetcd DescribeEtcdAvailableVersions --region <Region>
# expected: exit 0，返回 V3.4 和 V3.5 系列
```

**预期输出**（5 个版本可用）：

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

> 不带 `-tke.x` 后缀的版本等同于社区开源版本。带 `tke.x` 后缀表示腾讯云定制修复版。

### 查看已有实例的版本分布

```bash
tccli cetcd DescribeEtcdInstances --region <Region> \
    | jq '.InstanceSet[] | {Name: .InstanceName, Version: .EtcdVersion}'
# expected: 显示各实例名称和版本
```

## 验证

- `DescribeEtcdAvailableVersions` 返回 ≥ 1 个 v3.5.x 版本
- 确认新集群创建时可选择最新版本

## 清理

本页面为只读操作，无需清理。

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DescribeEtcdAvailableVersions` 返回空列表 | 确认 region 参数正确 | 地域可能不支持 cetcd 服务 | 切换到支持的地域（如 ap-guangzhou），`tccli cetcd DescribeEtcdRegions` 查看支持地域 |
| 实例版本过期无法新建集群 | `tccli cetcd DescribeEtcdAvailableVersions --region <Region>` 查看可用版本 | 旧大版本已停止支持新建 | 使用新大版本创建集群 |
| 实例版本过期无法升级 | 检查当前版本与目标版本差距 | 跨多个大版本可能不支持直接升级（此为平台限制） | 逐个大版本升级，或创建新集群后通过数据同步迁移 |

## 下一步

- [版本变更说明](../版本变更说明/tccli%20操作.md) — 查看各版本 TKE 定制修复详情
- [创建集群](../../创建集群/tccli%20操作.md) — 使用最新版本创建 etcd 集群
- [数据同步](../../数据同步/tccli%20操作.md) — 跨版本数据迁移

## 控制台替代

访问 [云原生 etcd 控制台](https://console.cloud.tencent.com/tke2/etcd/list)，在创建集群页面查看可选版本。
