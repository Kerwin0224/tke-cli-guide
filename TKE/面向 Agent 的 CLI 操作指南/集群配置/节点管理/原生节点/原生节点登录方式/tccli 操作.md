# 原生节点登录方式

> 对照官方：[原生节点登录方式](https://cloud.tencent.com/document/product/457/88696) · page_id `88696`

## 概述

原生节点支持两种登录方式，通过 `LoginSettings` 参数在创建节点池时指定，后续可通过 `ModifyClusterNodePool` 修改：

| 登录方式 | 配置参数 | 安全性 | 适用场景 |
|---------|---------|:--:|---------|
| SSH 密钥登录 | `LoginSettings.KeyIds` | 高（推荐） | 生产环境、自动化运维、多人协作 |
| 密码登录 | `LoginSettings.Password` | 中 | 临时调试、密钥不便管理的特殊场景 |

**建议**：生产环境优先使用 SSH 密钥登录。密码登录的密码会通过 API 明文传输，存在泄露风险。原生节点创建后，同一节点池内所有新建节点使用相同登录配置。

> **注意**：原生节点默认禁止 root 密码登录。如确有需要，可在控制台或通过 `LoginSettings.KeepImageLogin` 参数保留镜像默认登录配置。

## 前置条件

- [环境准备](../../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:CreateClusterNodePool, tke:ModifyClusterNodePool
#    tke:DescribeClusterNodePoolDetail, cvm:DescribeKeyPairs
# 验证：执行 DescribeKeyPairs 确认密钥查询权限
tccli cvm DescribeKeyPairs --region <Region>
# expected: exit 0，返回密钥对列表（可为空）
```

### 资源检查

```bash
# 4. 查询已有 SSH 密钥对
tccli cvm DescribeKeyPairs --region <Region>
# expected: 返回 KeyPairSet，含 KeyId 和 KeyName

# 5. 确认目标集群和节点池
tccli tke DescribeClusterNodePools --region <Region> --ClusterId CLUSTER_ID
# expected: 返回节点池列表（含 Type=Native 的原生节点池）

# 6. （如需修改已有节点池登录方式）查看当前登录配置
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId CLUSTER_ID \
    --NodePoolId NODE_POOL_ID \
    | jq '.NodePool.Native.LoginSettings'
# expected: 如已配置则返回 LoginSettings 对象
```

```json
{
  "NodePoolSet": "<NodePoolSet>",
  "NodePoolId": "<NodePoolId>",
  "Name": "<Name>",
  "ClusterInstanceId": "<ClusterInstanceId>",
  "LifeState": "<LifeState>",
  "LaunchConfigurationId": "<LaunchConfigurationId>"
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看可用 SSH 密钥 | `cvm DescribeKeyPairs` | 是 |
| 创建节点池时指定登录方式 | `CreateClusterNodePool` → `LoginSettings` | 否 |
| 修改节点池登录方式 | `ModifyClusterNodePool` → `LoginSettings` | 否 |
| 查看节点池当前登录配置 | `DescribeClusterNodePoolDetail` → `Native.LoginSettings` | 是 |

## 关键字段说明

`LoginSettings` 是 `Native` 结构内的可选参数，在 `CreateClusterNodePool` 和 `ModifyClusterNodePool` 中使用：

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `LoginSettings.KeyIds` | Array of String | 否 | SSH 密钥 ID 列表，如 `["skey-xxxxxxxx"]`。通过 `cvm DescribeKeyPairs` 查询。**KeyIds 和 Password 互斥，只能选其一** | 密钥 ID 不存在或不属于当前账号 → `InvalidParameter.KeyIdNotFound` |
| `LoginSettings.Password` | String | 否 | Linux 节点登录密码，需满足复杂度要求：8-30 位，含大小写字母、数字和特殊字符中的至少三种 | 不满足复杂度 → `InvalidParameter.PasswordNotMeetComplexity`；明文传输有泄露风险 |
| `LoginSettings.KeepImageLogin` | Boolean | 否 | 默认 `false`。`true` 表示保留镜像原始登录配置（忽略 KeyIds 和 Password） | 设为 `true` 时无法通过 API 管理登录配置 |

## 操作步骤

### 步骤 1：查询可用 SSH 密钥对

#### 选择依据

SSH 密钥对是推荐方式。先查询已有密钥，如无可用密钥，需通过 `cvm CreateKeyPair` 或在控制台创建新密钥对后再使用。

```bash
tccli cvm DescribeKeyPairs --region <Region>
# expected: exit 0，返回所有密钥对
```

**预期输出**：

```json
{
    "TotalCount": 2,
    "KeyPairSet": [
        {
            "KeyId": "skey-example01",
            "KeyName": "prod-admin-key",
            "ProjectId": 0,
            "AssociatedInstanceIds": ["ins-example-001"],
            "CreatedTime": "2025-03-15T02:00:00Z"
        },
        {
            "KeyId": "skey-example02",
            "KeyName": "dev-team-key",
            "ProjectId": 0,
            "AssociatedInstanceIds": [],
            "CreatedTime": "2025-06-01T08:00:00Z"
        }
    ],
    "RequestId": "..."
}
```

### 步骤 2：创建含密钥登录的原生节点池（最小配置）

```bash
cat > native-pool-ssh-login.json <<'EOF'
{
  "ClusterId": "CLUSTER_ID",
  "Name": "NODE_POOL_NAME",
  "Type": "Native",
  "Native": {
    "InstanceChargeType": "POSTPAID_BY_HOUR",
    "SubnetIds": ["SUBNET_ID"],
    "InstanceTypes": ["CXM_INSTANCE_TYPE"],
    "LoginSettings": {
      "KeyIds": ["SSH_KEY_ID"]
    }
  }
}
EOF

tccli tke CreateClusterNodePool --region <Region> \
    --cli-input-json file://native-pool-ssh-login.json
# expected: exit 0，返回 NodePoolId
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `CLUSTER_ID` | 目标集群 ID | 托管集群，K8s >= 1.24 | `tccli tke DescribeClusters --region <Region>` |
| `NODE_POOL_NAME` | 节点池名称 | 长度 1-60 | 自定义 |
| `SUBNET_ID` | 子网 ID | 必须属于集群所在 VPC | `tccli vpc DescribeSubnets --region <Region>` |
| `CXM_INSTANCE_TYPE` | CXM 机型 | 原生节点必须用 CXM 机型 | `tccli cvm DescribeInstanceTypeConfigs` |
| `SSH_KEY_ID` | SSH 密钥 ID | 格式 `skey-xxxxxxxx` | `tccli cvm DescribeKeyPairs --region <Region>` |

### 步骤 3：使用密码登录（增强配置）

```bash
cat > native-pool-password-login.json <<'EOF'
{
  "ClusterId": "CLUSTER_ID",
  "Name": "NODE_POOL_NAME",
  "Type": "Native",
  "Native": {
    "InstanceChargeType": "POSTPAID_BY_HOUR",
    "SubnetIds": ["SUBNET_ID"],
    "InstanceTypes": ["CXM_INSTANCE_TYPE"],
    "LoginSettings": {
      "Password": "YOUR_COMPLEX_PASSWORD"
    }
  }
}
EOF

tccli tke CreateClusterNodePool --region <Region> \
    --cli-input-json file://native-pool-password-login.json
# expected: exit 0，返回 NodePoolId

# ⚠️ 密码明文在 JSON 文件中，执行后建议删除此文件
```

### 步骤 4：查看节点池的登录配置

```bash
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId CLUSTER_ID \
    --NodePoolId NODE_POOL_ID \
    | jq '.NodePool.Native.LoginSettings'
# expected: 返回 KeyIds 或 KeepImageLogin 字段（Password 不返回）
```

**预期输出**：

```json
{
  "KeyIds": ["skey-example01"],
  "KeepImageLogin": false
}
```

> **注意**：出于安全考虑，`DescribeClusterNodePoolDetail` 返回的 `LoginSettings` 不会包含 `Password` 字段，仅返回 `KeyIds` 和 `KeepImageLogin`。

### 步骤 5：修改已有节点池的登录方式

```bash
cat > modify-nodepool-login.json <<'EOF'
{
  "ClusterId": "CLUSTER_ID",
  "NodePoolId": "NODE_POOL_ID",
  "Native": {
    "LoginSettings": {
      "KeyIds": ["NEW_SSH_KEY_ID"]
    }
  }
}
EOF

tccli tke ModifyClusterNodePool --region <Region> \
    --cli-input-json file://modify-nodepool-login.json
# expected: exit 0

# 注意：修改 LoginSettings 仅影响后续新建的节点，不会影响存量节点
```

## 验证

### 验证登录配置

```bash
# 确认节点池登录方式已按预期配置
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId CLUSTER_ID \
    --NodePoolId NODE_POOL_ID \
    | jq '.NodePool.Native.LoginSettings'
# expected: KeyIds 数组包含指定密钥 ID
```

```json
{
  "NodePool": "<NodePool>",
  "NodePoolId": "<NodePoolId>",
  "Name": "<Name>",
  "ClusterInstanceId": "<ClusterInstanceId>",
  "LifeState": "<LifeState>",
  "LaunchConfigurationId": "<LaunchConfigurationId>"
}
```

### 验证密钥可用性

```bash
# 确认密钥仍存在且未被删除
tccli cvm DescribeKeyPairs --region <Region> \
    --KeyIds '["SSH_KEY_ID"]'
# expected: TotalCount = 1，密钥状态正常
```

### 登录验证（需密钥文件）

```bash
# 使用 SSH 密钥登录节点验证
ssh -i PATH_TO_PRIVATE_KEY root@NODE_PUBLIC_IP
# expected: 成功登录节点命令行
```

| 验证维度 | 命令 | 预期 |
|---------|------|------|
| 登录配置 | `DescribeClusterNodePoolDetail` → `LoginSettings` | KeyIds 含指定密钥 |
| 密钥存在 | `cvm DescribeKeyPairs --KeyIds` | TotalCount >= 1 |
| SSH 可达 | `ssh -i key root@ip` | 登录成功（需节点有公网 IP） |

## 清理

本页为概念说明页，无资源需清理。

> **安全提醒**：如使用密码登录方式创建节点池，建议执行后立即删除包含明文密码的 JSON 文件。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `CreateClusterNodePool` 返回 `InvalidParameter.KeyIdNotFound` | `tccli cvm DescribeKeyPairs --region <Region> --KeyIds '["SSH_KEY_ID"]'` 检查密钥是否存在 | 填写的密钥 ID 不存在或属于其他账号 | 用正确密钥 ID 重新创建；`tccli cvm DescribeKeyPairs --region <Region>` 列出所有可用密钥 |
| `CreateClusterNodePool` 返回 `InvalidParameter.PasswordNotMeetComplexity` | 检查密码长度和复杂度 | 密码不满足腾讯云复杂度要求（8-30 位，含大小写、数字、特殊字符中的至少三种） | 修改密码使其满足要求，例如 `"MyP@ssw0rd2025"` |
| `CreateClusterNodePool` 同时传了 KeyIds 和 Password | 检查请求 JSON 中的 `LoginSettings` | KeyIds 和 Password 互斥，不能同时设置 | 选择其一：选 KeyIds 用 SSH 密钥，选 Password 用密码（推荐 KeyIds） |

### 登录失败

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| SSH 密钥登录报 `Permission denied` | 检查公网 IP 和密钥文件：`ssh -vvv -i key root@IP` 查看详细日志 | 密钥不匹配、节点未配置对应公钥、或私钥文件权限不正确 | 确保私钥文件权限为 600（`chmod 600 key`）；确认节点池 LoginSettings 的 KeyIds 与当前私钥匹配 |
| 密码始终登录失败 | 检查节点池配置中 `LoginSettings.KeepImageLogin` 是否为 `true` | `KeepImageLogin=true` 时，KeyIds 和 Password 均无效，节点使用镜像默认登录 | 将 `KeepImageLogin` 改为 `false`，重新设置 KeyIds 或 Password |
| 修改登录配置后新建节点仍使用旧配置 | `tccli tke DescribeClusterNodePoolDetail --region <Region> --ClusterId CLUSTER_ID --NodePoolId NODE_POOL_ID` 查看 `LoginSettings` | API 成功返回但不等于节点已创建（异步操作） | 确认节点池 `LifeState` 为 `normal` 后，手动扩容新增节点验证登录配置 |

## 下一步

- [新建原生节点](../新建原生节点/tccli%20操作.md) — 完整的原生节点池创建流程（含登录方式选择）
- [密钥管理](https://console.cloud.tencent.com/cvm/sshkey) — 创建、导入和管理 SSH 密钥对
- [原生节点开启公网访问](../原生节点开启公网访问/tccli%20操作.md) — page_id `82334`
- [声明式操作实践](../声明式操作实践/tccli%20操作.md) — page_id `78649`
- [ModifyClusterNodePool API](https://cloud.tencent.com/document/api/457/67726) — API 完整参数定义

## 控制台替代

[TKE 控制台 → 集群 → 节点管理 → 新建节点池](https://console.cloud.tencent.com/tke2/cluster)：创建原生节点池时，在「登录方式」步骤选择 SSH 密钥或密码。
