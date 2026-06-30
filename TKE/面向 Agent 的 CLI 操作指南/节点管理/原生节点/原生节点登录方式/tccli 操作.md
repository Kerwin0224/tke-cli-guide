# 登录原生节点

> 对照官方：[原生节点登录方式](https://cloud.tencent.com/document/product/457/88696) · page_id `88696` · tccli ≥3.1.107 · API 2017-03-12

## 概述

登录原生节点（Native Node）有四种方式，适用于不同场景。本文档介绍如何通过 tccli 和 kubectl 完成每种登录方式的全流程操作。

| 登录方式 | 适用场景 | 是否需要公网 IP | 是否需要 SSH 端口 |
|---------|---------|:---:|:---:|
| SSH 密钥对登录 | 日常运维、交互式操作，推荐生产环境使用 | 是 | 是（22） |
| TAT 远程命令执行 | 批量运维、自动化脚本，无需 SSH 服务 | 否 | 否 |
| VNC Web 控制台 | 紧急排障（网络配置错误、SSH 异常） | 否 | 否 |
| kubectl 数据面访问 | 查看节点状态、标签、污点等 K8s 信息 | 否 | 否 |

- **SSH 密钥对登录**：安全性最高，支持密钥轮换。创建密钥时私钥仅返回一次，需安全保存。
- **TAT 自动化助手**：无需开放 SSH 端口、无需分发密钥，适合批量运维和自动化场景。支持 `SESSION_MANAGER` 交互式会话。
- **VNC Web 控制台**：适用 SSH 无法连接的紧急排障场景。URL 含临时令牌，有效时间有限。
- **kubectl 数据面访问**：需先获取集群 kubeconfig，通过 `kubectl` 命令查看节点层面的 K8s 信息。

## 前置条件

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置，region 为目标地域

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:DescribeClusters, tke:DescribeClusterInstances
#    cvm:DescribeInstances, cvm:CreateKeyPair, cvm:DescribeKeyPairs
#    cvm:AssociateInstancesKeyPairs, cvm:DescribeInstanceVncUrl
#    tat:DescribeAutomationAgentStatus, tat:RunCommand, tat:DescribeInvocationTasks
# 验证：执行 DescribeClusters 确认权限
tccli tke DescribeClusters --region <Region>
# expected: exit 0，返回集群列表（可为空）

# 验证 CVM 权限
tccli cvm DescribeInstances --region <Region>
# expected: exit 0，返回 CVM 列表（可为空）

# 验证 TAT 权限
tccli tat DescribeAutomationAgentStatus --region <Region> \
    --InstanceIds '["<InstanceId>"]'
# expected: exit 0，返回 Agent 状态
```

**环境检查预期输出**（ID 均为虚构示例）：

- `tccli --version`：输出类似 `tccli version 1.0.0`
- `tccli configure list`：

```text
secretId     AKID********************************
secretKey    ********************************
region       ap-guangzhou
```

- `tccli tke DescribeClusters --region <Region>`（权限验证通过）：

```json
{
    "TotalCount": 0,
    "Clusters": [],
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

- `tccli cvm DescribeInstances --region <Region>`（权限验证通过）：

```json
{
    "TotalCount": 0,
    "InstanceSet": [],
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

- `tccli tat DescribeAutomationAgentStatus`（权限验证通过）：

```json
{
    "AutomationAgentSet": [
        {
            "InstanceId": "ins-example",
            "AgentStatus": "Online",
            "SupportFeatures": ["SESSION_MANAGER"]
        }
    ],
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

### 资源检查

```bash
# 4. 确认目标集群和节点存在
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["<ClusterId>"]'
# expected: ClusterStatus 为 Running

tccli tke DescribeClusterInstances --region <Region> \
    --ClusterId <ClusterId>
# expected: 返回 InstanceSet，至少 1 个节点 InstanceState 为 running

# 5. 确认节点 CVM 实例详情
tccli cvm DescribeInstances --region <Region> \
    --InstanceIds '["<InstanceId>"]'
# expected: 返回实例详情，含 LoginSettings 和 PrivateIpAddresses
```

**资源检查预期输出**（ID 均为虚构示例）：

- `tccli tke DescribeClusters --ClusterIds '["<ClusterId>"]'`：

```json
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "cls-example",
            "ClusterName": "example-cluster",
            "ClusterStatus": "Running"
        }
    ],
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

| 关键字段 | 说明 |
|---------|------|
| `ClusterStatus` | `Running` 表示集群正常运行，可进行后续操作 |

- `tccli tke DescribeClusterInstances --ClusterId <ClusterId>`：

```json
{
    "TotalCount": 1,
    "InstanceSet": [
        {
            "InstanceId": "ins-example",
            "InstanceRole": "WORKER",
            "InstanceState": "running",
            "LanIP": "172.16.0.10"
        }
    ],
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

- `tccli cvm DescribeInstances --InstanceIds '["<InstanceId>"]'`：

```json
{
    "TotalCount": 1,
    "InstanceSet": [
        {
            "InstanceId": "ins-example",
            "InstanceState": "RUNNING",
            "PrivateIpAddresses": ["172.16.0.10"],
            "PublicIpAddresses": ["1.2.3.4"],
            "LoginSettings": {
                "KeyIds": ["skey-example"]
            }
        }
    ],
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

### 网络与前置资源

- **SSH 密钥对登录**：节点须有公网 IP，安全组放行 22 端口。或通过 IOA/VPN/专线 连接 VPC 后使用内网 IP 登录。
- **TAT 远程执行**：节点须安装 TAT Agent（腾讯云公共镜像默认预装）。可通过 `DescribeAutomationAgentStatus` 确认 Agent 状态为 `Online`。
- **VNC 控制台**：无需网络前置条件。VNC URL 通过 `DescribeInstanceVncUrl` 获取，含临时令牌，每次调用生成新 URL。
- **kubectl 数据面访问**：需安装 `kubectl`。集群 kubeconfig 中的 server 地址可能为内网 IP，需 IOA/VPN/专线 连接 VPC 才能 kubectl 可达。

```bash
# 检查 kubectl 版本（如需数据面访问）
kubectl version --client
# expected: Client Version >= v1.28.0
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli/kubectl 命令 | 幂等 |
|-----------|-------------------|:--:|
| 查看集群列表 | `tccli tke DescribeClusters` | 是 |
| 查看节点列表 | `tccli tke DescribeClusterInstances` | 是 |
| 查看 CVM 实例详情 | `tccli cvm DescribeInstances` | 是 |
| 创建 SSH 密钥对 | `tccli cvm CreateKeyPair` | 否 |
| 查询密钥对列表 | `tccli cvm DescribeKeyPairs` | 是 |
| 绑定密钥到节点 | `tccli cvm AssociateInstancesKeyPairs` | 是 |
| 获取 VNC 登录 URL | `tccli cvm DescribeInstanceVncUrl` | 是 |
| 查询 TAT Agent 状态 | `tccli tat DescribeAutomationAgentStatus` | 是 |
| 通过 TAT 执行命令 | `tccli tat RunCommand` | 否 |
| 查询 TAT 执行结果 | `tccli tat DescribeInvocationTasks` | 是 |
| 获取集群 kubeconfig | `tccli tke DescribeClusterKubeconfig` | 是 |
| 查看节点 K8s 信息 | `kubectl describe node <node-name>` | 是 |

### 关键字段说明

以下说明 `AssociateInstancesKeyPairs` 的主要参数。完整参数定义见 `tccli cvm AssociateInstancesKeyPairs --help`。

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `InstanceIds` | Array of String | 是 | 目标 CVM 实例 ID 列表，格式 `ins-xxxxxxxx`。通过 `tccli cvm DescribeInstances` 获取 | 实例不存在 → `InvalidInstanceId.NotFound` |
| `KeyIds` | Array of String | 是 | SSH 密钥 ID 列表，格式 `skey-xxxxxxxx`。通过 `tccli cvm DescribeKeyPairs` 获取 | 密钥不存在 → `InvalidKeyPairId.NotFound` |
| `ForceStop` | Boolean | 否 | 是否强制关机。`true` 时强制重启实例以完成绑定（Running 状态实例需此参数）。默认 `false` | 不传或 `false` 且实例 Running → `UnsupportedOperation.InstanceStateRunning` |

以下说明 `CreateKeyPair` 的 `KeyName` 参数约束：

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `KeyName` | String | 是 | 密钥名称，最长 25 字符，只能包含字母、数字和下划线 `[a-zA-Z0-9_]`。**不支持连字符(-)** | 含连字符或超 25 字符 → `InvalidParameterValue`：包含不合法的字符或超过 25 字符 |

## 操作步骤

### SSH 密钥对登录

#### 选择依据

- **登录方式**：选择 SSH 密钥对而非密码登录。密钥对安全性更高，支持密钥轮换，推荐用于生产环境节点登录。
- **密钥命名**：`KeyName` 仅使用字母、数字、下划线，最长 25 字符。连字符(-)是常见踩坑点——CVM CreateKeyPair 不接受含 `-` 的 KeyName。
- **绑定策略**：绑定密钥到 Running 状态实例时，必须传 `--ForceStop true`。该参数会强制重启实例以完成密钥绑定，实例重启期间服务会短暂中断。

#### 步骤 1：创建 SSH 密钥对

```bash
tccli cvm CreateKeyPair --region <Region> \
    --KeyName <KeyName> \
    --ProjectId 0
# expected: exit 0，返回 KeyId 和 PrivateKey
```

**预期输出**：

```json
{
    "KeyPair": {
        "KeyId": "skey-example",
        "KeyName": "my_login_key",
        "PrivateKey": "-----BEGIN RSA PRIVATE KEY-----\n...\n-----END RSA PRIVATE KEY-----"
    },
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

| 关键字段 | 说明 |
|---------|------|
| `KeyId` | 密钥对 ID，格式 `skey-xxxxxxxx`。后续绑定、查询、删除均需使用 |
| `PrivateKey` | 私钥内容（PEM 格式）。**仅创建时返回一次**，务必立即保存到安全位置（如 `~/.ssh/`），文件权限设为 `600` |

> **注意**：`KeyName` 最长 25 字符，只能包含字母、数字和下划线 `[a-zA-Z0-9_]`。含连字符(-)或超长度会返回 `InvalidParameterValue`。

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<Region>` | 地域 | 如 `ap-guangzhou` | `tccli configure list` 查看当前配置 |
| `<KeyName>` | 密钥名称 | 最长 25 字符，`[a-zA-Z0-9_]` | 自定义，如 `my_login_key` |

#### 步骤 2：查询密钥对

```bash
tccli cvm DescribeKeyPairs --region <Region> \
    --Filters '[{"Name":"key-name","Values":["<KeyName>"]}]'
# expected: exit 0，返回匹配的密钥对
```

**预期输出**：

```json
{
    "KeyPairSet": [
        {
            "KeyId": "skey-example",
            "KeyName": "my_login_key",
            "AssociatedInstanceIds": [],
            "CreatedTime": "2026-01-01T00:00:00Z"
        }
    ]
}
```

| 关键字段 | 说明 |
|---------|------|
| `AssociatedInstanceIds` | 已绑定该密钥的实例列表。空数组表示尚未绑定 |
| `CreatedTime` | 密钥创建时间 |

> **注意**：`DescribeKeyPairs` 支持的过滤器名为 `key-name`，使用 `key-id` 过滤会返回 `InvalidFilter`。

#### 步骤 3：绑定密钥到节点

```bash
cat > associate-keypair.json <<'EOF'
{
    "InstanceIds": ["<InstanceId>"],
    "KeyIds": ["<KeyId>"],
    "ForceStop": true
}
EOF

tccli cvm AssociateInstancesKeyPairs --region <Region> \
    --cli-input-json file://associate-keypair.json
# expected: exit 0，返回 RequestId
```

**预期输出**：

```json
{
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

> **注意**：`--ForceStop true` 会强制重启实例以完成密钥绑定。实例重启期间服务会短暂中断。Running 状态实例不传 `ForceStop` 或传 `ForceStop false` 会返回 `UnsupportedOperation.InstanceStateRunning`。
> 
> ⚠️ 绑定密钥后，CVM 的密码登录设置（`LoginSettings.Password`）可能被覆盖。如果实例之前配置了密码登录，绑定密钥后密码登录可能失效。建议另行记录密码，或同时保留密钥和密码两种登录方式。

绑定完成后验证密钥绑定状态：

```bash
tccli cvm DescribeKeyPairs --region <Region> \
    --Filters '[{"Name":"key-name","Values":["<KeyName>"]}]'
# expected: AssociatedInstanceIds 包含目标实例 ID
```

**预期输出**：

```json
{
    "KeyPairSet": [
        {
            "KeyId": "skey-example",
            "KeyName": "my_login_key",
            "AssociatedInstanceIds": ["ins-example"],
            "CreatedTime": "2026-01-01T00:00:00Z"
        }
    ]
}
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<InstanceId>` | 目标 CVM 实例 ID | 格式 `ins-xxxxxxxx` | `tccli tke DescribeClusterInstances` |
| `<KeyId>` | SSH 密钥对 ID | 格式 `skey-xxxxxxxx` | `tccli cvm DescribeKeyPairs` |

### TAT 远程命令执行

#### 选择依据

- **运维方式**：选择 TAT 自动化助手而非 SSH。TAT 无需 SSH 端口开放、无需密钥分发，适合批量运维和自动化场景。Agent 支持 `SESSION_MANAGER` 功能可实现交互式会话。
- **命令编码**：`RunCommand` 的 `Content` 参数必须为 Base64 编码。未编码传入原始命令会返回 `InvalidParameterValue`。

#### 步骤 1：检查 TAT Agent 状态

```bash
tccli tat DescribeAutomationAgentStatus --region <Region> \
    --InstanceIds '["<InstanceId>"]'
# expected: AgentStatus 为 Online，SupportFeatures 含 SESSION_MANAGER
```

**预期输出**：

```json
{
    "AutomationAgentSet": [
        {
            "InstanceId": "ins-example",
            "Version": "1.1.11",
            "AgentStatus": "Online",
            "Environment": "Linux",
            "SupportFeatures": ["SESSION_MANAGER"]
        }
    ]
}
```

| 关键字段 | 说明 |
|---------|------|
| `AgentStatus` | `Online` 表示 Agent 正常运行，可接收命令。`Offline` 需排查 Agent 安装或启动状态 |
| `SupportFeatures` | `SESSION_MANAGER` 表示支持交互式会话。如不含此项，仅支持单次命令执行 |

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<Region>` | 地域 | 如 `ap-guangzhou` | 与 CVM 实例所在地域一致 |
| `<InstanceId>` | 目标 CVM 实例 ID | 格式 `ins-xxxxxxxx` | `tccli tke DescribeClusterInstances` |

#### 步骤 2：通过 TAT 执行命令

先将命令内容做 Base64 编码，再传入：

```bash
# 编码命令内容
COMMAND_B64=$(echo -n 'uname -a && whoami && uptime' | base64)

cat > tat-run.json <<EOF
{
    "InstanceIds": ["<InstanceId>"],
    "CommandType": "SHELL",
    "Content": "$COMMAND_B64"
}
EOF

tccli tat RunCommand --region <Region> \
    --cli-input-json file://tat-run.json
# expected: exit 0，返回 CommandId 和 InvocationId
```

**预期输出**：

```json
{
    "CommandId": "cmd-example",
    "InvocationId": "inv-example",
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

| 关键字段 | 说明 |
|---------|------|
| `CommandId` | 命令 ID，可用于查询命令详情 |
| `InvocationId` | 执行 ID，用于查询该次执行的详细结果。后续通过 `DescribeInvocationTasks` 使用此 ID 查询 |

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<InstanceId>` | 目标 CVM 实例 ID | 格式 `ins-xxxxxxxx` | `tccli tke DescribeClusterInstances` |
| `<Region>` | 地域 | 必须与实例所在地域一致 | `tccli configure list` |

#### 步骤 3：查询命令执行结果

```bash
tccli tat DescribeInvocationTasks --region <Region> \
    --Filters '[{"Name":"invocation-id","Values":["<InvocationId>"]}]'
# expected: TaskStatus 为 SUCCESS，TaskResult.Output 含命令输出（Base64）
```

**预期输出**：

```json
{
    "InvocationTaskSet": [
        {
            "InvocationTaskId": "invt-example",
            "TaskStatus": "SUCCESS",
            "TaskResult": {
                "Output": "TGludXggVk0tMC0zNC11YnVudHUgNS4xNS4wLTEzMS1nZW5lcmljICMxND..." 
            }
        }
    ]
}
```

| 关键字段 | 说明 |
|---------|------|
| `TaskStatus` | `SUCCESS` 表示命令执行成功。`FAILED` 需检查命令内容或 Agent 状态 |
| `TaskResult.Output` | 命令的标准输出，为 **Base64 编码**。需 `echo '<Output>' | base64 -d` 解码后阅读 |

解码输出示例：

```bash
echo 'TGludXggVk0tMC0zNC11YnVudHUg...' | base64 -d
# 输出示例：
# Linux VM-0-34-ubuntu 5.15.0-131-generic #141-Ubuntu ...
# root
#  12:00:00 up 10 min,  0 users,  load average: 0.08, 0.12, 0.09
```

### VNC Web 控制台登录

#### 选择依据

- **使用时机**：VNC 适用于 SSH 无法连接时的紧急排障场景（如网络配置错误、SSH 服务异常、安全组误改导致端口不通）。VNC URL 含临时令牌，有效时间有限（通常数分钟），过期需重新获取。
- **限制**：VNC 只能提供基础的控制台访问，不支持文件传输、端口转发等 SSH 高级功能。

#### 步骤 1：获取 VNC URL

```bash
tccli cvm DescribeInstanceVncUrl --region <Region> \
    --InstanceId <InstanceId>
# expected: exit 0，返回 InstanceVncUrl
```

**预期输出**：

```json
{
    "InstanceVncUrl": "wss://gzvnc.qcloud.com:26789/vnc?s=<token>&password=&isWindows=false&isUbuntu=true",
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

| 关键字段 | 说明 |
|---------|------|
| `InstanceVncUrl` | VNC Web 控制台地址（WebSocket 协议）。在浏览器中直接打开即可进入控制台。URL 含临时令牌，过期需重新调用 `DescribeInstanceVncUrl` 获取 |

> **注意**：进入 VNC 控制台后需输入用户名（通常为 `root`）和密码（若实例已设置密码）。VNC URL 为临时令牌，有效时间有限（通常数分钟），过期需重新获取。

### kubectl 数据面访问

#### 选择依据

- **访问层级**：kubectl 是 K8s 数据面访问方式，用于查看节点的 K8s 层面信息（标签、污点、状态、资源容量、Conditions 等），而非操作系统层面。需要操作系统交互时，应使用 SSH 密钥对或 TAT。
- **可达性**：集群 kubeconfig 中 `server` 地址可能为**内网 IP**，需通过 IOA/VPN/专线 连接 VPC 才能 kubectl 可达。如无法连接，考虑在 VPC 内的 CVM 上执行 kubectl 命令。

#### 步骤 1：获取集群 Kubeconfig

```bash
tccli tke DescribeClusterKubeconfig --region <Region> \
    --ClusterId <ClusterId>
# expected: exit 0，返回 Kubeconfig 内容
```

**预期输出**：

```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTi...
    server: https://172.24.0.12
  name: cls-example
contexts:
- context:
    cluster: cls-example
    user: "100012345678"
  name: cls-example-100012345678-context-default
current-context: cls-example-100012345678-context-default
kind: Config
preferences: {}
users:
- name: "100012345678"
  user:
    client-certificate-data: LS0tLS1CRUdJTi...
    client-key-data: LS0tLS1CRUdJTi...
```

将 kubeconfig 保存到本地并使用 kubectl 查看节点信息：

> **警告**：以下命令会直接覆盖 `~/.kube/config`。如果已有其他集群的 kubeconfig，请先备份：
> ```bash
> cp ~/.kube/config ~/.kube/config.bak.$(date +%s)
> ```

```bash
# 保存 kubeconfig
tccli tke DescribeClusterKubeconfig --region <Region> \
    --ClusterId <ClusterId> | jq -r '.Kubeconfig' > ~/.kube/config

# 查看节点列表
kubectl get nodes
# expected: 返回节点列表

# 查看节点详情
kubectl describe node <node-name>
# expected: 返回节点详细信息（标签、污点、容量、Conditions 等）
```

**kubectl get nodes 预期输出**：

```text
NAME            STATUS   ROLES    AGE   VERSION
172.24.0.34     Ready    <none>   10m   v1.32.2-tke.1
```

| 字段 | 说明 |
|------|------|
| `NAME` | 节点名称，通常为节点的内网 IP |
| `STATUS` | `Ready` 表示节点健康可用 |
| `VERSION` | Kubelet 版本，需与集群控制面版本匹配 |

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<ClusterId>` | 集群 ID | 格式 `cls-xxxxxxxx` | `tccli tke DescribeClusters` |
| `<Region>` | 地域 | 与集群所在地域一致 | `tccli configure list` |
| `<node-name>` | K8s 节点名称 | 通常为节点内网 IP | `kubectl get nodes` |

## 验证

### 控制面（tccli）

验证各登录方式的控制面配置正确：

```bash
# 1. 验证 SSH 密钥已绑定
tccli cvm DescribeKeyPairs --region <Region> \
    --Filters '[{"Name":"key-name","Values":["<KeyName>"]}]'
# expected: AssociatedInstanceIds 包含目标 InstanceId

# 2. 验证 TAT Agent 在线
tccli tat DescribeAutomationAgentStatus --region <Region> \
    --InstanceIds '["<InstanceId>"]'
# expected: AgentStatus: "Online", SupportFeatures 含 SESSION_MANAGER

# 3. 验证 VNC URL 可获取
tccli cvm DescribeInstanceVncUrl --region <Region> \
    --InstanceId <InstanceId>
# expected: InstanceVncUrl 非空，以 wss:// 开头

# 4. 验证 kubeconfig 可获取
tccli tke DescribeClusterKubeconfig --region <Region> \
    --ClusterId <ClusterId> | jq -r '.Kubeconfig' | head -1
# expected: "apiVersion: v1"
```

**控制面验证预期输出**（ID 均为虚构示例）：

- `DescribeKeyPairs`（密钥已绑定）：

```json
{
    "KeyPairSet": [
        {
            "KeyId": "skey-example",
            "KeyName": "my_login_key",
            "AssociatedInstanceIds": ["ins-example"],
            "CreatedTime": "2026-01-01T00:00:00Z"
        }
    ],
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

- `DescribeAutomationAgentStatus`（Agent 在线）：

```json
{
    "AutomationAgentSet": [
        {
            "InstanceId": "ins-example",
            "AgentStatus": "Online",
            "SupportFeatures": ["SESSION_MANAGER"]
        }
    ],
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

- `DescribeInstanceVncUrl`（VNC URL 可获取）：

```json
{
    "InstanceVncUrl": "wss://gzvnc.qcloud.com:26789/vnc?s=<token>&password=&isWindows=false&isUbuntu=true",
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

- `DescribeClusterKubeconfig`（kubeconfig 首行）：

```text
apiVersion: v1
```

### 数据面（kubectl）

```bash
# 验证 kubectl 连通性
kubectl cluster-info
# expected: Kubernetes control plane is running at https://...

# 验证节点可访问
kubectl get nodes -o wide
# expected: 目标节点 STATUS 为 Ready

# 验证节点详情可查
kubectl describe node <node-name> | head -20
# expected: 返回节点 Conditions、Capacity、Allocatable 等信息
```

## 清理

> **警告**：密钥绑定期间实例通过 `--ForceStop true` 会重启，服务短暂中断。密钥解绑操作（`DisassociateInstancesKeyPairs`）可能被 CAM 策略拒绝——需确认有 `cvm:DisassociateInstancesKeyPairs` 权限且满足标签条件。已绑定实例的密钥无法直接删除——需先解绑所有关联实例。

### 数据面

kubectl 数据面访问使用本地 kubeconfig 文件，无需销毁云端资源。清理时移除本地 kubeconfig 即可：

```bash
# 移除本地 kubeconfig（如保存到默认路径）
rm -f ~/.kube/config
# expected: 本地 kubeconfig 已删除
```

### 控制面（tccli）

#### 清理前状态检查

```bash
# 确认密钥绑定状态
tccli cvm DescribeKeyPairs --region <Region> \
    --Filters '[{"Name":"key-name","Values":["<KeyName>"]}]'
# 记录 AssociatedInstanceIds，确认待解绑的实例
```

**清理前状态检查预期输出**（ID 均为虚构示例）：

```json
{
    "KeyPairSet": [
        {
            "KeyId": "skey-example",
            "KeyName": "my_login_key",
            "AssociatedInstanceIds": ["ins-example"],
            "CreatedTime": "2026-01-01T00:00:00Z"
        }
    ],
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

确认 `AssociatedInstanceIds` 非空：密钥仍绑定在实例上，需先解绑。

#### 解绑密钥

```bash
cat > disassociate-keypair.json <<'EOF'
{
    "InstanceIds": ["<InstanceId>"],
    "KeyIds": ["<KeyId>"],
    "ForceStop": true
}
EOF

tccli cvm DisassociateInstancesKeyPairs --region <Region> \
    --cli-input-json file://disassociate-keypair.json
# expected: exit 0，返回 RequestId
# ⚠️ 该操作可能被 CAM 策略拒绝——参见 [排障](#排障) 节
```

#### 删除密钥

```bash
# ⚠️ 仅当密钥已解绑所有关联实例后才能删除
tccli cvm DeleteKeyPairs --region <Region> \
    --KeyIds '["<KeyId>"]'
# expected: exit 0
```

#### 验证已清理

```bash
tccli cvm DescribeKeyPairs --region <Region> \
    --Filters '[{"Name":"key-name","Values":["<KeyName>"]}]'
# expected: KeyPairSet 为空，或密钥列表不含目标密钥
```

**验证已清理预期输出**（密钥已删除）：

```json
{
    "KeyPairSet": [],
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

`KeyPairSet` 为空表示密钥已成功删除。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `CreateKeyPair` 返回 `InvalidParameterValue`，提示包含不合法的字符或超过 25 字符 | 检查 `KeyName` 参数值 | KeyName 含连字符(-)或超过 25 字符，或包含非法字符 | 改用短名称，仅用字母、数字、下划线 `[a-zA-Z0-9_]`。示例：`my_login_key` 非 `my-login-key` |
| `AssociateInstancesKeyPairs` 返回 `UnsupportedOperation.InstanceStateRunning` | 检查实例状态：`tccli cvm DescribeInstances --region <Region> --InstanceIds '["<InstanceId>"]'` | 实例正在运行，绑定密钥需要重启实例但 `ForceStop` 未设为 `true` | 添加 `--ForceStop true` 参数（会强制重启实例）。注意：重启期间服务会短暂中断 |
| `RunCommand` 返回 `InvalidParameterValue`，提示 Content 参数值不合法 | 检查传入的 Content 值是否为纯文本 | Content 未做 Base64 编码，直接传入了原始命令文本 | 先 Base64 编码后传入：`echo -n '<命令>' \| base64` |
| `DescribeKeyPairs` 返回 `InvalidFilter`，提示不支持 `key-id` 过滤字段 | 检查 Filters 参数中的 Name 字段 | `DescribeKeyPairs` 不支持 `key-id` 过滤器 | 改用 `key-name` 过滤器。或省略 Filters 查询全部密钥 |
| `DescribeAutomationAgentStatus` 返回 `AgentStatus: "Offline"` | 确认 TAT Agent 是否安装：在节点上执行 `systemctl status tat_agent` | Agent 未安装或未启动。公共镜像默认预装，自定义镜像可能缺失 | 登录节点安装/启动 Agent：`systemctl start tat_agent && systemctl enable tat_agent`。如未安装，参见 [自动化助手安装指引](https://cloud.tencent.com/document/product/1340/51945) |
| `DisassociateInstancesKeyPairs` 返回 `UnauthorizedOperation` | 查看错误消息中的 CAM 策略信息 | CAM 硬策略（如 `strategyId:274251632`）以 `qcs:request_tag` 标签条件拒绝 `cvm:DisassociateInstancesKeyPairs`。即使资源打了对应标签仍不满足（该 API 需要 `request_tag`，非 `resource_tag`） | 解绑密钥请使用控制台操作，或联系 CAM 管理员申请临时 `cvm:DisassociateInstancesKeyPairs` 权限 |
| `DeleteKeyPairs` 返回 `InvalidParameterValue.KeyPairNotSupported`，提示密钥关联了实例 | 查询密钥绑定状态：`tccli cvm DescribeKeyPairs --region <Region> --Filters '[{"Name":"key-name","Values":["<KeyName>"]}]'` 查看 `AssociatedInstanceIds` | 密钥仍绑定在实例上。删除密钥前必须先解绑所有关联实例，但解绑操作被 CAM 拒绝导致级联受阻 | 密钥将随实例销毁自动清理。如需主动删除：先通过控制台解绑密钥，再删除密钥；或销毁关联实例后删除密钥 |

### 登录方式无法使用

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| SSH 无法连接（`Connection refused` / `Connection timed out`） | 检查安全组：`tccli vpc DescribeSecurityGroupPolicies --region <Region> --SecurityGroupId <SecurityGroupId>` | 安全组未放行 22 端口，或节点无公网 IP | 安全组添加入站规则放行 TCP 22；无公网 IP 则通过 IOA/VPN/专线 连接 VPC 后使用内网 IP，或改用 TAT/VNC 登录 |
| VNC URL 打开后无法连接 | 重新获取 VNC URL：`tccli cvm DescribeInstanceVncUrl --region <Region> --InstanceId <InstanceId>` | VNC URL 含临时令牌，已过期（有效时间通常数分钟） | 重新调用 `DescribeInstanceVncUrl` 获取新 URL |
| kubectl 报 `Unable to connect to the server: dial tcp ... i/o timeout` | 检查 kubeconfig 中的 server 地址：`kubectl config view \| grep server` | server 地址为内网 IP（如 `172.x.x.x`），本地机器不在同一 VPC 内 | 通过 IOA/VPN/专线 连接 VPC 后重试，或在同 VPC 的 CVM 上执行 kubectl 命令 |
| TAT 命令执行结果 `TaskStatus: "FAILED"` | 查看 TaskResult.Output 解码后的错误信息 | 命令内容有语法错误，或目标节点缺少执行环境 | 修正命令内容后重新执行。检查命令的 Base64 编码是否正确 |

## 下一步

- [SSH 密钥对登录](#ssh-密钥对登录) — 日常运维推荐方式
- [TAT 远程命令执行](#tat-远程命令执行) — 批量运维和自动化场景
- [VNC Web 控制台登录](#vnc-web-控制台登录) — 紧急排障备用方案
- [kubectl 数据面访问](#kubectl-数据面访问) — 查看 K8s 层面节点信息
- TAT 自动化助手 API 文档：<https://cloud.tencent.com/document/api/1340>
- TKE 集群 Kubeconfig 管理：<https://cloud.tencent.com/document/product/457/32191>

## 控制台替代

登录 [TKE 控制台](https://console.cloud.tencent.com/tke2/cluster) → 选择集群 → 节点管理 → 原生节点 → 选择节点 → 登录。
