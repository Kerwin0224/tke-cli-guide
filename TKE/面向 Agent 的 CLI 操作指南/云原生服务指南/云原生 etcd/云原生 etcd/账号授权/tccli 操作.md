# 账号授权

> 对照官方：[账号授权](https://cloud.tencent.com/document/product/457/73029) · page_id `73029`

## 概述

云原生 etcd 已接入访问管理（CAM），支持主子账号权限控制。授权分为两类：**子用户授权**（授予子用户访问 etcd 服务）和**服务授权**（授予 etcd 服务操作其他云资源如标签、监控）。

CAM 内置两个预设策略：`QcloudCEtcdFullAccess`（全读写）和 `QcloudCEtcdReadOnlyaccess`（只读）。

## 前置条件

- [环境准备](../../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置（需主账号或有 CAM 管理权限的子账号）

# 3. 检查 CAM 权限 — 需要 cam:ListPolicies, cam:AttachUserPolicy
# 验证：执行 DescribeEtcdCredentials 确认 etcd 凭证是否已配置
tccli cetcd DescribeEtcdCredentials --region <Region> --InstanceId INSTANCE_ID
# expected: exit 0，返回 etcd 访问凭证信息
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看 etcd 凭证 | `DescribeEtcdCredentials` | 是 |
| 查看 CAM 预设策略 | `tccli cam ListPolicies --Keyword QcloudCEtcd` | 是 |
| 子用户授权 | `tccli cam AttachUserPolicy` | 否 |
| 子用户解绑权限 | `tccli cam DetachUserPolicy` | 否 |

## 操作步骤

### 步骤 1：查看 etcd 访问凭证

```bash
tccli cetcd DescribeEtcdCredentials --region <Region> \
    --InstanceId INSTANCE_ID
# expected: exit 0，返回 etcd 服务凭证（证书、端点等）
```

**预期输出**：

```json
{
    "Credentials": {
        "EtcdEndpoint": "https://INSTANCE_ID.etcd.tencentcloudapi.com:2379",
        "CaCert": "-----BEGIN CERTIFICATE-----\n...",
        "ClientCert": "-----BEGIN CERTIFICATE-----\n...",
        "ClientKey": "-----BEGIN RSA PRIVATE KEY-----\n..."
    }
}
```

### 步骤 2：查看 CAM 预设策略

```bash
tccli cam ListPolicies --Keyword QcloudCEtcd
# expected: exit 0，返回 QcloudCEtcdFullAccess 和 QcloudCEtcdReadOnlyaccess
```

**预期输出**：

```json
{
    "List": [
        {"PolicyName": "QcloudCEtcdFullAccess", "PolicyId": 12345678},
        {"PolicyName": "QcloudCEtcdReadOnlyaccess", "PolicyId": 12345679}
    ]
}
```

### 步骤 3：为子用户授权

```bash
tccli cam AttachUserPolicy --UserName SUB_USER_NAME \
    --PolicyName QcloudCEtcdFullAccess
# expected: exit 0
```

> 将 `SUB_USER_NAME` 替换为子用户名。用 `QcloudCEtcdReadOnlyaccess` 替代 `QcloudCEtcdFullAccess` 仅授予只读权限。

## 验证

```bash
# 确认策略已绑定到子用户
tccli cam ListAttachedUserPolicies --UserName SUB_USER_NAME
# expected: 返回列表中包含 QcloudCEtcdFullAccess
```

## 清理

如需撤销授权：

```bash
tccli cam DetachUserPolicy --UserName SUB_USER_NAME \
    --PolicyName QcloudCEtcdFullAccess
# expected: exit 0
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `AttachUserPolicy` 返回 `AuthFailure` | `tccli configure list` 检查当前凭据 | 当前账号无 CAM 管理权限 | 使用主账号或具有 `cam:AttachUserPolicy` 权限的子账号操作 |
| `AttachUserPolicy` 返回 `InvalidParameter.UserNotExist` | `tccli cam GetUser --UserName SUB_USER_NAME` 检查用户是否存在 | 子用户名不存在 | 先通过 `tccli cam AddUser` 创建子用户 |
| `DescribeEtcdCredentials` 返回 `InvalidParameter` | 检查 InstanceId 是否正确 | InstanceId 不存在或格式错误 | 用 `tccli cetcd DescribeEtcdInstances --region <Region>` 确认 InstanceId |

## 下一步

- [创建集群](../创建集群/tccli%20操作.md) — 授权后创建 etcd 实例
- [CAM 访问管理文档](https://cloud.tencent.com/document/product/598) — 了解更多 CAM 权限管理
- [监控和告警配置](../监控和告警配置/tccli%20操作.md) — 配置 etcd 监控

## 控制台替代

访问 [访问管理控制台](https://console.cloud.tencent.com/cam)，在用户列表中为子用户关联策略 `QcloudCEtcdFullAccess`。
