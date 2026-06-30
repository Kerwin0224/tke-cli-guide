# 弹性健康度

> 对照官方：[弹性健康度](https://cloud.tencent.com/document/product/457/132615) · page_id `132615` · tccli ≥3.1.107 · API 2022-05-01

## 概述

弹性健康度是 TKE 原生节点的节点级健康检测与自动修复机制。通过定义**健康检测策略**（HealthCheckPolicy），对节点上运行的 kubelet、containerd、内核、文件系统等进行周期性故障检测，并可选择性地触发自动修复（如重启 kubelet/containerd）。

核心概念：
- **健康检测模板**（DescribeHealthCheckTemplate）：系统预定义的 15 种检测规则，每种规则定义了检测项、严重级别、修复动作
- **健康检测策略**（HealthCheckPolicy）：用户创建的自定义策略，从模板中选择规则并决定是否启用自动修复
- **策略绑定**（DescribeHealthCheckPolicyBindings）：策略是集群级别资源，通过绑定关系关联到节点池后生效

> **注意**：健康检测策略是集群级别资源（通过 `ClusterId` 管理），而非节点池级别。策略创建后需绑定到节点池才能对节点生效。绑定操作可通过控制台或创建节点池时引用策略名完成。

## 前置条件

- [环境准备](../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置，region 为目标地域

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:DescribeHealthCheckTemplate
#    tke:DescribeHealthCheckPolicies
#    tke:CreateHealthCheckPolicy
#    tke:ModifyHealthCheckPolicy
#    tke:DeleteHealthCheckPolicy
#    tke:DescribeHealthCheckPolicyBindings
# 验证：执行 DescribeHealthCheckPolicies 确认权限
tccli tke DescribeHealthCheckPolicies --ClusterId <ClusterId> \
    --version 2022-05-01 --region <Region>
# expected: exit 0，返回策略列表（可为空）

**预期输出**（空集群）：

```json
{
    "HealthCheckPolicies": null,
    "TotalCount": 0,
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

# 验证模板查询权限
tccli tke DescribeHealthCheckTemplate --version 2022-05-01 --region <Region>
# expected: exit 0，返回系统预定义的 15 种检测规则
```

**预期输出**：

```json
{
    "HealthCheckTemplate": {
        "Rules": [
            {"Name": "FDPressure", "Severity": "Low", "RepairAction": "None"},
            {"Name": "RuntimeUnhealthy", "Severity": "Low", "RepairAction": "RestartRuntime"},
            {"Name": "KubeletUnhealthy", "Severity": "Low", "RepairAction": "RestartKubelet"},
            {"Name": "ReadonlyFilesystem", "Severity": "High", "RepairAction": "None"},
            "... (共 15 条规则，详见步骤 1 完整输出)"
        ]
    },
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

### 资源检查

```bash
# 4. 确认目标集群存在且状态正常
tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'
# expected: TotalCount >= 1，ClusterStatus 为 Running
```

**预期输出**：

```json
{
    "Clusters": [
        {
            "ClusterId": "<ClusterId>",
            "ClusterName": "<集群名称>",
            "ClusterStatus": "Running",
            "ClusterVersion": "1.28.3"
        }
    ],
    "TotalCount": 1,
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

> **API 版本说明**：健康检测策略相关 API（CreateHealthCheckPolicy、DescribeHealthCheckPolicies、ModifyHealthCheckPolicy、DeleteHealthCheckPolicy、DescribeHealthCheckPolicyBindings、DescribeHealthCheckTemplate）**仅在 TKE API `2022-05-01` 版本中可用**。v2018-05-25 中不存在这些 API，使用旧版本将返回 `Invalid choice` 错误。所有命令必须显式指定 `--version 2022-05-01`。

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看健康检测规则模板 | `DescribeHealthCheckTemplate` | 是 |
| 查看健康检测策略列表 | `DescribeHealthCheckPolicies` | 是 |
| 创建健康检测策略 | `CreateHealthCheckPolicy` | 否 |
| 修改健康检测策略 | `ModifyHealthCheckPolicy` | 是 |
| 删除健康检测策略 | `DeleteHealthCheckPolicy` | 是 |
| 查看策略绑定关系 | `DescribeHealthCheckPolicyBindings` | 是 |

## 关键字段说明

以下说明 `CreateHealthCheckPolicy` 和 `ModifyHealthCheckPolicy` 的主要参数。

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `ClusterId` | String | 是 | 已存在的集群 ID，格式 `cls-xxxxxxxx`。`tccli tke DescribeClusters` 获取 | 集群不存在 → `FailedOperation` |
| `HealthCheckPolicy.Name` | String | 是 | 策略名称，集群内唯一 | 重名 → `FailedOperation: ...already exists` |
| `HealthCheckPolicy.Rules[].Name` | String | 是 | 必须与 `DescribeHealthCheckTemplate` 返回的模板规则名**精确匹配**。如 `ReadonlyFilesystem`（注意大小写），不可简写为 `ReadonlyFS` | 名称不匹配 → `FailedOperation`（创建成功但规则可能不生效） |
| `HealthCheckPolicy.Rules[].Enabled` | Boolean | 是 | `true` 启用该规则检测，`false` 禁用 | — |
| `HealthCheckPolicy.Rules[].AutoRepairEnabled` | Boolean | 否 | `true` 启用自动修复，`false`（默认）仅告警不修复。**生产环境建议初始设为 `false`，验证后再开启** | 误开启自动修复 → 可能因误检测导致生产节点非预期重启 |

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<ClusterId>` | 集群 ID | 格式 `cls-xxxxxxxx` | `tccli tke DescribeClusters --region <Region>` |
| `<PolicyName>` | 健康检测策略名称 | 集群内唯一，长度 1-63 字符 | 自定义 |
| `<PolicyNameEnhanced>` | 增强配置策略名称 | 集群内唯一，长度 1-63 字符，与最小创建名称不同 | 自定义 |
| `<Region>` | 地域 | 如 `ap-guangzhou` | `tccli configure list` 查看已配置地域 |

## 操作步骤

### 步骤 1：查询健康检测规则模板

在执行创建策略前，先查询系统预定义的检测规则模板，了解可用的规则名称。

```bash
tccli tke DescribeHealthCheckTemplate --version 2022-05-01 --region <Region>
# expected: exit 0，返回 15 种预定义检测规则
```

**预期输出**：

```json
{
    "HealthCheckTemplate": {
        "Rules": [
            {"Name": "FDPressure", "Description": "Too many files opened", "RepairAction": "None", "ShouldEnable": true, "ShouldRepair": false, "Severity": "Low"},
            {"Name": "RuntimeUnhealthy", "Description": "List containerd task failed", "RepairAction": "RestartRuntime", "ShouldEnable": true, "ShouldRepair": false, "Severity": "Low"},
            {"Name": "KubeletUnhealthy", "Description": "Call kubelet healthz failed", "RepairAction": "RestartKubelet", "ShouldEnable": true, "ShouldRepair": false, "Severity": "Low"},
            {"Name": "ReadonlyFilesystem", "Description": "Filesystem is readonly", "RepairAction": "None", "ShouldEnable": true, "ShouldRepair": false, "Severity": "High"},
            {"Name": "OOMKilling", "Description": "Process has been oom-killed", "RepairAction": "None", "ShouldEnable": true, "ShouldRepair": false, "Severity": "High"},
            {"Name": "TaskHung", "Description": "Task blocked more then beyond the threshold", "RepairAction": "None", "ShouldEnable": true, "ShouldRepair": false, "Severity": "High"},
            {"Name": "UnregisterNetDevice", "Description": "Net device unregister", "RepairAction": "None", "ShouldEnable": true, "ShouldRepair": false, "Severity": "High"},
            {"Name": "KernelOopsDivideError", "Description": "Kernel oops with divide error", "RepairAction": "None", "ShouldEnable": true, "ShouldRepair": false, "Severity": "High"},
            {"Name": "KernelOopsNULLPointer", "Description": "Kernel oops with NULL pointer", "RepairAction": "None", "ShouldEnable": true, "ShouldRepair": false, "Severity": "High"},
            {"Name": "Ext4Error", "Description": "Ext4 filesystem error", "RepairAction": "None", "ShouldEnable": true, "ShouldRepair": false, "Severity": "High"},
            {"Name": "Ext4Warning", "Description": "Ext4 filesystem warning", "RepairAction": "None", "ShouldEnable": true, "ShouldRepair": false, "Severity": "High"},
            {"Name": "IOError", "Description": "IOError", "RepairAction": "None", "ShouldEnable": true, "ShouldRepair": false, "Severity": "High"},
            {"Name": "MemoryError", "Description": "MemoryError", "RepairAction": "None", "ShouldEnable": true, "ShouldRepair": false, "Severity": "High"},
            {"Name": "DockerHung", "Description": "Task blocked more then beyond the threshold", "RepairAction": "None", "ShouldEnable": true, "ShouldRepair": false, "Severity": "High"},
            {"Name": "KubeletRestart", "Description": "Kubelet restart", "RepairAction": "None", "ShouldEnable": true, "ShouldRepair": false, "Severity": "Low"}
        ]
    },
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

**关键字段说明**：

| 字段 | 含义 |
|------|------|
| `Name` | 规则唯一标识，创建策略时 `Rules[].Name` 必须与此值精确匹配 |
| `Severity` | `Low`（低严重度）或 `High`（高严重度） |
| `RepairAction` | 系统支持的修复动作。`RestartKubelet` / `RestartRuntime` / `None`（不支持自动修复） |
| `ShouldEnable` | 系统建议是否默认启用该规则 |
| `ShouldRepair` | 系统建议是否默认开启自动修复 |

### 步骤 2：查询已有健康检测策略（初始状态）

```bash
tccli tke DescribeHealthCheckPolicies --ClusterId <ClusterId> \
    --version 2022-05-01 --region <Region>
# expected: exit 0，返回当前集群的策略列表
```

**预期输出**（空集群）：

```json
{
    "HealthCheckPolicies": null,
    "TotalCount": 0,
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

### 步骤 3：创建健康检测策略

#### 选择依据

*为什么选这些规则和参数值：*

- **创建方案**：以下提供最小创建和增强配置两种可选方案，请根据需求**选择其一执行**。两种方案使用不同的策略名称，互不冲突。最小创建仅含 1 条基础规则，用于快速验证流程；增强配置覆盖多个严重级别的规则，适合生产环境。
- **API 版本**：必须使用 `--version 2022-05-01`。v2018-05-25 中不存在健康检测策略相关 API，使用旧版本将返回 `Invalid choice` 错误。
- **策略级别**：健康检测策略是集群级别资源（通过 `ClusterId` 管理），创建时不需要指定 `NodePoolId`。策略创建后通过 `DescribeHealthCheckPolicyBindings` 查看绑定到哪些节点池。
- **规则选择**：从 `DescribeHealthCheckTemplate` 返回的 15 种规则中选取。推荐从低严重度规则开始（如 `KubeletUnhealthy`、`RuntimeUnhealthy`），逐步扩展到高严重度规则。`Rule.Name` 必须与模板中的名称**精确匹配**（如 `ReadonlyFilesystem` 而非 `ReadonlyFS`）。
- **AutoRepairEnabled**：生产环境建议初始设为 `false`，仅告警不自动修复。经过充分验证后再对特定规则手动开启自动修复。错误开启自动修复可能导致因误检测引发生产节点非预期重启。
- **策略名称**：集群内唯一，建议使用有意义的命名（如 `prod-health-check`、`dev-health-check`），避免使用临时名称。最小创建和增强配置使用不同的策略名，避免同名冲突。

#### 最小创建（仅含基础规则）

`health-policy-minimal.json`：

```json
{
    "ClusterId": "<ClusterId>",
    "HealthCheckPolicy": {
        "Name": "<PolicyName>",
        "Rules": [
            {"Name": "ReadonlyFilesystem", "Enabled": true, "AutoRepairEnabled": false}
        ]
    }
}
```

```bash
tccli tke CreateHealthCheckPolicy --version 2022-05-01 --region <Region> \
    --cli-input-json file://health-policy-minimal.json
# expected: exit 0，返回 HealthCheckPolicyName
```

**预期输出**：

```json
{
    "HealthCheckPolicyName": "<PolicyName>",
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

#### 增强配置（多规则覆盖不同严重级别）

`health-policy-enhanced.json`：

```json
{
    "ClusterId": "<ClusterId>",
    "HealthCheckPolicy": {
        "Name": "<PolicyNameEnhanced>",
        "Rules": [
            {"Name": "ReadonlyFilesystem", "Enabled": true, "AutoRepairEnabled": false},
            {"Name": "KubeletUnhealthy", "Enabled": true, "AutoRepairEnabled": false},
            {"Name": "RuntimeUnhealthy", "Enabled": true, "AutoRepairEnabled": false},
            {"Name": "OOMKilling", "Enabled": true, "AutoRepairEnabled": false},
            {"Name": "KernelOopsNULLPointer", "Enabled": true, "AutoRepairEnabled": false}
        ]
    }
}
```

```bash
tccli tke CreateHealthCheckPolicy --version 2022-05-01 --region <Region> \
    --cli-input-json file://health-policy-enhanced.json
# expected: exit 0，返回 HealthCheckPolicyName
```

| 层级 | 包含规则 | 目的 | 策略名称占位符 |
|------|---------|------|------|
| **最小创建** | 1 条规则（`ReadonlyFilesystem`），`AutoRepairEnabled=false` | 最低成本验证创建流程 | `<PolicyName>` |
| **增强配置** | 5 条规则，覆盖 Low + High 严重度，均不开启自动修复 | 生产环境推荐配置，全面检测但不自动介入 | `<PolicyNameEnhanced>` |

> **注意**：最小创建和增强配置使用不同的策略名称占位符，确保两种方案可独立执行，互不冲突。如只需一种方案，请选择其一执行并替换对应的占位符。

### 步骤 4：查询策略（验证创建结果）

```bash
tccli tke DescribeHealthCheckPolicies --ClusterId <ClusterId> \
    --version 2022-05-01 --region <Region>
# expected: exit 0，TotalCount >= 1，返回创建的策略
```

**预期输出**：

```json
{
    "HealthCheckPolicies": [
        {
            "Name": "<PolicyName>",
            "Rules": [
                {"Name": "ReadonlyFilesystem", "Enabled": true, "AutoRepairEnabled": false},
                {"Name": "KubeletUnhealthy", "Enabled": true, "AutoRepairEnabled": false},
                {"Name": "RuntimeUnhealthy", "Enabled": true, "AutoRepairEnabled": false}
            ]
        }
    ],
    "TotalCount": 1,
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

### 步骤 5：查询策略绑定关系

策略创建后需要通过绑定关系确认已关联到哪些节点池。绑定操作可通过控制台或创建节点池时引用策略名完成，tccli 暂未提供独立的绑定 API。

```bash
tccli tke DescribeHealthCheckPolicyBindings --ClusterId <ClusterId> \
    --version 2022-05-01 --region <Region>
# expected: exit 0，返回策略与节点池的绑定关系
```

**预期输出**：

```json
{
    "HealthCheckPolicyBindings": [
        {
            "Name": "<PolicyName>",
            "CreatedAt": "YYYY-MM-DD HH:MM:SS +0800 CST",
            "NodePools": []
        }
    ],
    "TotalCount": 1,
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

> **注意**：`NodePools` 为空列表表示策略尚未绑定到任何节点池。策略只有在绑定到节点池后，才会对节点池中的节点执行健康检测。绑定操作需通过控制台或创建节点池时引用策略名完成。

### 步骤 6：修改健康检测策略

#### 选择依据

- **开启自动修复**：经过验证后，可对特定规则（如 `KubeletUnhealthy`）开启 `AutoRepairEnabled=true`，让系统自动重启异常的 kubelet。不建议对所有规则同时开启自动修复。
- **新增/移除规则**：`ModifyHealthCheckPolicy` 会**全量替换** Rules 列表，不是增量添加。修改时需在请求中包含最终期望的完整规则列表。
- **策略存在性**：修改前必须通过 `DescribeHealthCheckPolicies` 确认策略名存在，修改不存在的策略将返回 `FailedOperation` 错误。

`modify-health-policy.json`：

```json
{
    "ClusterId": "<ClusterId>",
    "HealthCheckPolicy": {
        "Name": "<PolicyName>",
        "Rules": [
            {"Name": "ReadonlyFilesystem", "Enabled": true, "AutoRepairEnabled": true},
            {"Name": "KubeletUnhealthy", "Enabled": true, "AutoRepairEnabled": true},
            {"Name": "RuntimeUnhealthy", "Enabled": true, "AutoRepairEnabled": false},
            {"Name": "OOMKilling", "Enabled": true, "AutoRepairEnabled": false}
        ]
    }
}
```

```bash
tccli tke ModifyHealthCheckPolicy --version 2022-05-01 --region <Region> \
    --cli-input-json file://modify-health-policy.json
# expected: exit 0
```

**预期输出**：

```json
{
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

### 步骤 7：查询策略（验证修改结果）

```bash
tccli tke DescribeHealthCheckPolicies --ClusterId <ClusterId> \
    --version 2022-05-01 --region <Region>
# expected: exit 0，Rules 列表与修改参数一致
```

**预期输出**：

```json
{
    "HealthCheckPolicies": [
        {
            "Name": "<PolicyName>",
            "Rules": [
                {"Name": "ReadonlyFilesystem", "Enabled": true, "AutoRepairEnabled": true},
                {"Name": "KubeletUnhealthy", "Enabled": true, "AutoRepairEnabled": true},
                {"Name": "RuntimeUnhealthy", "Enabled": true, "AutoRepairEnabled": false},
                {"Name": "OOMKilling", "Enabled": true, "AutoRepairEnabled": false}
            ]
        }
    ],
    "TotalCount": 1,
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

### 步骤 8：按名称过滤查询策略

当集群中策略数量较多时，可通过 `Filters` 参数按策略名称过滤。

```bash
tccli tke DescribeHealthCheckPolicies --ClusterId <ClusterId> \
    --version 2022-05-01 --region <Region> \
    --Filters '[{"Name":"HealthCheckPolicyName","Values":["<PolicyName>"]}]'
# expected: exit 0，仅返回名称匹配的策略
```

**预期输出**：

```json
{
    "HealthCheckPolicies": [
        {
            "Name": "<PolicyName>",
            "Rules": [
                {"Name": "ReadonlyFilesystem", "Enabled": true, "AutoRepairEnabled": false}
            ]
        }
    ],
    "TotalCount": 1,
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

### 步骤 9：按名称过滤查询绑定关系

```bash
tccli tke DescribeHealthCheckPolicyBindings --ClusterId <ClusterId> \
    --version 2022-05-01 --region <Region> \
    --Filter '[{"Name":"HealthCheckPolicyName","Values":["<PolicyName>"]}]'
# expected: exit 0，仅返回名称匹配的策略绑定关系
```

**预期输出**：

```json
{
    "HealthCheckPolicyBindings": [
        {
            "Name": "<PolicyName>",
            "CreatedAt": "YYYY-MM-DD HH:MM:SS +0800 CST",
            "NodePools": []
        }
    ],
    "TotalCount": 1,
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

## 验证

### 控制面（tccli）

| 维度 | 命令 | 预期 |
|------|------|------|
| 策略存在性 | `DescribeHealthCheckPolicies --ClusterId <ClusterId>` | `TotalCount >= 1`，策略名与创建时一致 |
| 规则配置 | 同上，检查 `Rules[]` 列表 | 规则数量、`Name`、`Enabled`、`AutoRepairEnabled` 与创建/修改参数一致 |
| 绑定状态 | `DescribeHealthCheckPolicyBindings --ClusterId <ClusterId>` | 策略在列表中，检查 `NodePools` 确认绑定情况 |
| 模板一致性 | `DescribeHealthCheckTemplate` 与策略 `Rules[].Name` 对比 | 策略中所有规则名均来自模板的有效规则列表 |

```bash
# 综合验证：确认策略存在且规则正确
tccli tke DescribeHealthCheckPolicies --ClusterId <ClusterId> \
    --version 2022-05-01 --region <Region>
# expected: HealthCheckPolicies 列表含目标策略，Rules 与配置一致

# 确认绑定关系
tccli tke DescribeHealthCheckPolicyBindings --ClusterId <ClusterId> \
    --version 2022-05-01 --region <Region>
# expected: 目标策略在列表中，NodePools 反映当前绑定状态

# 验证规则名称有效（确认策略中所有规则名均来自模板）
tccli tke DescribeHealthCheckTemplate --version 2022-05-01 --region <Region> \
    | python3 -c "import sys,json; d=json.load(sys.stdin); print(set(r['Name'] for r in d['HealthCheckTemplate']['Rules']))"
# expected: 输出所有有效规则名集合，策略中的 Rule.Name 应全部在此集合中
```

**预期输出**（DescribeHealthCheckPolicies）：

```json
{
    "HealthCheckPolicies": [
        {
            "Name": "<PolicyName>",
            "Rules": [
                {"Name": "ReadonlyFilesystem", "Enabled": true, "AutoRepairEnabled": true},
                {"Name": "KubeletUnhealthy", "Enabled": true, "AutoRepairEnabled": true},
                {"Name": "RuntimeUnhealthy", "Enabled": true, "AutoRepairEnabled": false},
                {"Name": "OOMKilling", "Enabled": true, "AutoRepairEnabled": false}
            ]
        }
    ],
    "TotalCount": 1,
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

**预期输出**（DescribeHealthCheckPolicyBindings）：

```json
{
    "HealthCheckPolicyBindings": [
        {
            "Name": "<PolicyName>",
            "CreatedAt": "YYYY-MM-DD HH:MM:SS +0800 CST",
            "NodePools": []
        }
    ],
    "TotalCount": 1,
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

## 清理

> **警告**：
> - 删除健康检测策略后，已绑定该策略的节点池将**立即停止健康检测和自动修复**。节点故障将不会被自动发现和处理。
> - 删除前务必通过 `DescribeHealthCheckPolicyBindings` 确认策略是否绑定到关键生产节点池。
> - 建议在维护窗口内操作。如需保留检测能力，先创建新策略并绑定后再删除旧策略。

### 1. 清理前状态检查

```bash
# 确认要删除的策略名称和绑定状态
tccli tke DescribeHealthCheckPolicies --ClusterId <ClusterId> \
    --version 2022-05-01 --region <Region>
# 记录待删除的策略 Name

tccli tke DescribeHealthCheckPolicyBindings --ClusterId <ClusterId> \
    --version 2022-05-01 --region <Region>
# 确认策略绑定的节点池列表，评估删除影响
```

**预期输出**（DescribeHealthCheckPolicies）：

```json
{
    "HealthCheckPolicies": [
        {
            "Name": "<PolicyName>",
            "Rules": [
                {"Name": "ReadonlyFilesystem", "Enabled": true, "AutoRepairEnabled": false}
            ]
        }
    ],
    "TotalCount": 1,
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

**预期输出**（DescribeHealthCheckPolicyBindings）：

```json
{
    "HealthCheckPolicyBindings": [
        {
            "Name": "<PolicyName>",
            "CreatedAt": "YYYY-MM-DD HH:MM:SS +0800 CST",
            "NodePools": []
        }
    ],
    "TotalCount": 1,
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

### 2. 删除健康检测策略

```bash
tccli tke DeleteHealthCheckPolicy --ClusterId <ClusterId> \
    --HealthCheckPolicyName <PolicyName> \
    --version 2022-05-01 --region <Region>
# expected: exit 0
```

**预期输出**：

```json
{
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

### 3. 验证已删除

```bash
tccli tke DescribeHealthCheckPolicies --ClusterId <ClusterId> \
    --version 2022-05-01 --region <Region>
# expected: TotalCount 减少，目标策略不再出现
```

**预期输出**：

```json
{
    "HealthCheckPolicies": null,
    "TotalCount": 0,
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `CreateHealthCheckPolicy` 返回 `Invalid choice` | 检查命令中的 `--version` 参数 | 使用了 `v2018-05-25` API 版本，该版本不存在健康检测策略相关 API | 所有命令必须加 `--version 2022-05-01`。`tccli tke CreateHealthCheckPolicy --version 2022-05-01 --region <Region> --cli-input-json file://health-policy-minimal.json` |
| `CreateHealthCheckPolicy` 返回 `FailedOperation: ...already exists` | `tccli tke DescribeHealthCheckPolicies --ClusterId <ClusterId> --version 2022-05-01 --region <Region>` 检查已有策略名 | 策略名在集群内已存在（Kubernetes CRD 同名冲突） | 先删除同名旧策略：`tccli tke DeleteHealthCheckPolicy --ClusterId <ClusterId> --HealthCheckPolicyName <PolicyName> --version 2022-05-01 --region <Region>`，或使用不同 `Name` 创建 |
| `CreateHealthCheckPolicy` 返回 `FailedOperation`（规则不生效） | 检查 `Rules[].Name` 是否与 `DescribeHealthCheckTemplate` 返回的名称一致 | Rule 的 `Name` 填写了非模板定义的名称（如简写 `ReadonlyFS` 而非 `ReadonlyFilesystem`） | 执行 `tccli tke DescribeHealthCheckTemplate --version 2022-05-01 --region <Region>` 获取有效规则名称列表，`Rule.Name` 必须与模板中的 `Name` 精确匹配 |
| `ModifyHealthCheckPolicy` 返回 `FailedOperation: Failed to modify health-check policy` | `tccli tke DescribeHealthCheckPolicies --ClusterId <ClusterId> --version 2022-05-01 --region <Region>` 确认策略是否存在 | 尝试修改不存在的策略 | 先通过 `DescribeHealthCheckPolicies` 确认策略名正确且策略存在，再执行修改 |
| `DeleteHealthCheckPolicy` 返回 `FailedOperation: Failed to get health-check policy` | `tccli tke DescribeHealthCheckPolicies --ClusterId <ClusterId> --version 2022-05-01 --region <Region>` 确认策略是否存在 | 策略已被删除或策略名拼写错误 | 通过 `DescribeHealthCheckPolicies` 确认策略名正确且策略仍存在 |

### 操作成功但状态异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `CreateHealthCheckPolicy` 返回成功但策略未生效 | `tccli tke DescribeHealthCheckPolicyBindings --ClusterId <ClusterId> --version 2022-05-01 --region <Region>` 检查 `NodePools` | 策略创建成功但尚未绑定到任何节点池（`NodePools` 为空） | 策略需绑定到节点池后才会对节点生效。通过控制台或创建节点池时引用策略名完成绑定 |
| `CreateHealthCheckPolicy` 返回成功但节点无检测行为 | `tccli tke DescribeHealthCheckPolicies --ClusterId <ClusterId> --version 2022-05-01 --region <Region>` 检查规则配置 | 所有规则的 `Enabled` 为 `false`，或 `Rules` 列表为空 | 确保至少一条规则的 `Enabled` 为 `true`，且策略已绑定到节点池 |

> 涉及 `FailedOperation` 的排查路径，请保留 `RequestId` 以备工单查询：[提交工单](https://console.cloud.tencent.com/workorder)

## 下一步

- [新建原生节点](../新建原生节点/tccli%20操作.md) — 创建节点池时可引用健康检测策略
- [原生节点概述](../原生节点概述/tccli%20操作.md) — 了解原生节点功能全景
- [TKE API 文档 - 健康检测策略相关接口](https://cloud.tencent.com/document/api/457/103132) — 完整 API 参数说明

## 控制台替代

- [TKE 控制台 - 弹性健康度](https://console.cloud.tencent.com/tke2/node-pool/native/detail?rid=<Region>&clusterId=<ClusterId>)
