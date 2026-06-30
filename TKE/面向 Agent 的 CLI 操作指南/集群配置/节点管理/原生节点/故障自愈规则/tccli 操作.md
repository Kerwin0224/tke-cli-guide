# 故障自愈规则（tccli）

> 对照官方：[故障自愈规则](https://cloud.tencent.com/document/product/457/78650) · page_id `78650`

## 概述

通过 `CreateNodePoolHealthCheckPolicy` 为原生节点池创建**故障自愈规则**，定义节点健康检查类型和修复策略。当节点出现指定故障（如磁盘只读、Kubelet 异常、网络不可达等）时，系统自动执行修复操作（如重启节点、Cordon 节点并重新创建）。

健康检查策略基于多维检测（云探 + 节点自检），支持多种检查类型和修复策略组合。可通过 `DescribeNodePoolHealthCheckPolicy` 查询已有规则，通过 `ModifyNodePoolHealthCheckPolicy` 修改规则，通过 `DeleteNodePoolHealthCheckPolicy` 删除规则。

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
#    tke:CreateNodePoolHealthCheckPolicy, tke:DescribeNodePoolHealthCheckPolicy
#    tke:ModifyNodePoolHealthCheckPolicy, tke:DeleteNodePoolHealthCheckPolicy
#    tke:DescribeClusterNodePoolDetail
tccli tke DescribeClusterNodePools --region <Region> --ClusterId CLUSTER_ID
# expected: exit 0，返回节点池列表
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

### 资源检查

```bash
# 4. 确认节点池存在且为原生类型
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId CLUSTER_ID \
    --NodePoolId NODE_POOL_ID
# expected: Type 为 "Native"，LifeState 为 "normal"

# 5. 查看节点池内节点
tccli tke DescribeClusterInstances --region <Region> \
    --ClusterId CLUSTER_ID \
    --Filters '[{"Name":"node-pool-id","Values":["NODE_POOL_ID"]}]'
# expected: 返回节点列表，确认节点已就绪
```

## 控制台与 CLI 参数映射

| 控制台操作 | CLI 命令 | 幂等 |
|-----------|---------|:--:|
| 创建故障自愈规则 | `CreateNodePoolHealthCheckPolicy` | 否 |

## 关键字段说明

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `ClusterId` | String | 是 | 集群 ID，格式 `cls-` 开头 | 集群不存在 → `ResourceNotFound.ClusterNotFound` |
| `NodePoolId` | String | 是 | 节点池 ID，格式 `np-` 开头 | 节点池不存在 → `ResourceNotFound.NodePoolNotFound` |
| `Name` | String | 是 | 策略名称，2-64 字符 | 重名 → `InvalidParameterValue` |
| `HealthCheckPolicy` | Object | 是 | 健康检查规则配置 | 配置错误 → 策略不生效 |
| `AutoRepair` | Bool | 否 | 是否自动修复，默认 `true` | 设为 `false` 则仅检测不修复 |

### HealthCheckPolicy 结构

| 子字段 | 类型 | 说明 |
|--------|------|------|
| `Rules` | Object[] | 健康检查规则列表 |
| `Rules[].Name` | String | 规则名称 |
| `Rules[].Enabled` | Bool | 是否启用该规则 |
| `Rules[].CheckType` | String | 检查类型：`ReadonlyFilesystem`, `KubeletProblem`, `ContainerRuntimeProblem`, `NetworkUnavailable` 等 |
| `Rules[].RepairAction` | String | 修复动作：`RestartNode`, `CordonAndReplace`, `ReportOnly` |
| `Rules[].Duration` | String | 异常持续时间阈值，如 `"5m"` |
| `Rules[].Interval` | String | 检查间隔，如 `"1m"` |

### 检查类型（CheckType）

| CheckType | 含义 | 适用场景 |
|-----------|------|---------|
| `ReadonlyFilesystem` | 文件系统只读 | 磁盘故障、挂载异常 |
| `KubeletProblem` | Kubelet 异常 | Kubelet 进程僵死或未运行 |
| `ContainerRuntimeProblem` | 容器运行时异常 | containerd 异常 |
| `NetworkUnavailable` | 节点网络不可达 | 网络故障、CNI 异常 |
| `DiskPressure` | 磁盘压力（使用率过高） | 磁盘空间不足 |
| `MemoryPressure` | 内存压力 | 节点内存不足 |

### 修复动作（RepairAction）

| RepairAction | 含义 | 是否重建节点 |
|-------------|------|:--:|
| `RestartNode` | 重启节点 | 否 |
| `CordonAndReplace` | Cordon 节点并重新创建新节点替代 | 是 |
| `ReportOnly` | 仅上报事件，不执行修复 | 否 |

## 操作步骤

### 步骤1：创建故障自愈规则

#### 选择依据

- **`CheckType` 选择**：`ReadonlyFilesystem` + `KubeletProblem` 覆盖最常见的两类节点故障；`NetworkUnavailable` 适合对网络 SLA 要求高的场景。
- **`RepairAction` 选择**：`RestartNode` 适合可恢复的软件故障（快速，不新建节点）；`CordonAndReplace` 适合硬件相关或不可恢复的故障（可靠，会重建节点）；`ReportOnly` 适合需要人工判断的场景。
- **`Duration` 阈值**：建议设为 `"5m"`，避免瞬时抖动触发误修复；生产环境可缩短至 `"3m"`。

#### 最小创建（核心自愈规则）

`create-health-policy-minimal.json`：

```json
{
  "ClusterId": "CLUSTER_ID",
  "NodePoolId": "NODE_POOL_ID",
  "Name": "default-health-policy",
  "AutoRepair": true,
  "HealthCheckPolicy": {
    "Rules": [
      {
        "Name": "readonly-fs-check",
        "Enabled": true,
        "CheckType": "ReadonlyFilesystem",
        "RepairAction": "CordonAndReplace",
        "Duration": "5m",
        "Interval": "1m"
      },
      {
        "Name": "kubelet-check",
        "Enabled": true,
        "CheckType": "KubeletProblem",
        "RepairAction": "RestartNode",
        "Duration": "5m",
        "Interval": "1m"
      }
    ]
  }
}
```

```bash
tccli tke CreateNodePoolHealthCheckPolicy --region <Region> \
    --cli-input-json file://create-health-policy-minimal.json
# expected: exit 0，返回 HealthPolicyId
```

**预期输出**：

```json
{
    "Response": {
        "HealthPolicyId": "hp-example",
        "RequestId": "xxxx-xxxx-xxxx-xxxx"
    }
}
```

#### 增强配置（覆盖更多故障类型）

`create-health-policy-enhanced.json`：

```json
{
  "ClusterId": "CLUSTER_ID",
  "NodePoolId": "NODE_POOL_ID",
  "Name": "production-health-policy",
  "AutoRepair": true,
  "HealthCheckPolicy": {
    "Rules": [
      {
        "Name": "readonly-fs-check",
        "Enabled": true,
        "CheckType": "ReadonlyFilesystem",
        "RepairAction": "CordonAndReplace",
        "Duration": "3m",
        "Interval": "1m"
      },
      {
        "Name": "kubelet-check",
        "Enabled": true,
        "CheckType": "KubeletProblem",
        "RepairAction": "RestartNode",
        "Duration": "5m",
        "Interval": "1m"
      },
      {
        "Name": "runtime-check",
        "Enabled": true,
        "CheckType": "ContainerRuntimeProblem",
        "RepairAction": "RestartNode",
        "Duration": "5m",
        "Interval": "1m"
      },
      {
        "Name": "network-check",
        "Enabled": true,
        "CheckType": "NetworkUnavailable",
        "RepairAction": "CordonAndReplace",
        "Duration": "5m",
        "Interval": "1m"
      },
      {
        "Name": "disk-pressure-check",
        "Enabled": true,
        "CheckType": "DiskPressure",
        "RepairAction": "ReportOnly",
        "Duration": "10m",
        "Interval": "2m"
      }
    ]
  }
}
```

```bash
tccli tke CreateNodePoolHealthCheckPolicy --region <Region> \
    --cli-input-json file://create-health-policy-enhanced.json
# expected: exit 0，返回 HealthPolicyId
```

**预期输出**：

```json
{
    "Response": {
        "HealthPolicyId": "hp-example",
        "RequestId": "xxxx-xxxx-xxxx-xxxx"
    }
}
```

### 步骤2：查询已有自愈规则

```bash
tccli tke DescribeNodePoolHealthCheckPolicy --region <Region> \
    --ClusterId CLUSTER_ID \
    --NodePoolId NODE_POOL_ID
# expected: exit 0，返回节点池绑定的健康检查策略列表
```

**预期输出**：

```json
{
    "Response": {
        "TotalCount": 1,
        "HealthCheckPolicySet": [
            {
                "HealthPolicyId": "hp-example",
                "Name": "production-health-policy",
                "ClusterId": "cls-example",
                "NodePoolId": "np-example",
                "AutoRepair": true,
                "HealthCheckPolicy": {
                    "Rules": [
                        {
                            "Name": "readonly-fs-check",
                            "Enabled": true,
                            "CheckType": "ReadonlyFilesystem",
                            "RepairAction": "CordonAndReplace",
                            "Duration": "3m",
                            "Interval": "1m"
                        }
                    ]
                }
            }
        ],
        "RequestId": "xxxx-xxxx-xxxx-xxxx"
    }
}
```

### 步骤3：修改自愈规则

`modify-health-policy.json`：

```json
{
  "ClusterId": "CLUSTER_ID",
  "NodePoolId": "NODE_POOL_ID",
  "HealthPolicyId": "HEALTH_POLICY_ID",
  "HealthCheckPolicy": {
    "Rules": [
      {
        "Name": "readonly-fs-check",
        "Enabled": true,
        "CheckType": "ReadonlyFilesystem",
        "RepairAction": "CordonAndReplace",
        "Duration": "2m",
        "Interval": "30s"
      }
    ]
  }
}
```

```bash
tccli tke ModifyNodePoolHealthCheckPolicy --region <Region> \
    --cli-input-json file://modify-health-policy.json
# expected: exit 0，返回 RequestId
```

**预期输出**：

```json
{
    "Response": {
        "RequestId": "xxxx-xxxx-xxxx-xxxx"
    }
}
```

### 步骤4：删除自愈规则

```bash
tccli tke DeleteNodePoolHealthCheckPolicy --region <Region> \
    --ClusterId CLUSTER_ID \
    --NodePoolId NODE_POOL_ID \
    --HealthPolicyId HEALTH_POLICY_ID
# expected: exit 0，返回 RequestId
```

**预期输出**：

```json
{
    "Response": {
        "RequestId": "xxxx-xxxx-xxxx-xxxx"
    }
}
```

### 步骤5：确认规则已删除

```bash
tccli tke DescribeNodePoolHealthCheckPolicy --region <Region> \
    --ClusterId CLUSTER_ID \
    --NodePoolId NODE_POOL_ID
# expected: TotalCount 为 0，规则已清空
```

```json
{"RequestId": "..."}
```

## 验证

| 维度 | 检查内容 | 预期 |
|------|---------|------|
| 策略存在性 | `DescribeNodePoolHealthCheckPolicy` → `TotalCount` | >= 1 |
| 策略名称 | `DescribeNodePoolHealthCheckPolicy` → `Name` | 与创建值一致 |
| 规则启用 | `DescribeNodePoolHealthCheckPolicy` → `HealthCheckPolicy.Rules[].Enabled` | 所有规则为 `true` |
| AutoRepair | `DescribeNodePoolHealthCheckPolicy` → `AutoRepair` | `true` |
| 节点池关联 | `DescribeNodePoolHealthCheckPolicy` → `NodePoolId` | 与 NODE_POOL_ID 一致 |

### 验证自愈规则生效（模拟场景）

当节点发生匹配的故障时，可通过以下方式观察自愈行为：

```bash
# 观察节点事件
kubectl --kubeconfig KUBECONFIG_PATH get events -A --field-selector involvedObject.name=NODE_NAME
# expected: 出现 ReadonlyFilesystem 或 KubeletProblem 相关事件

# 观察节点状态变化
kubectl --kubeconfig KUBECONFIG_PATH get node NODE_NAME -w
# expected: 节点先变为 NotReady，随后可能被 Cordon

# 控制面确认修复动作
tccli tke DescribeClusterInstances --region <Region> \
    --ClusterId CLUSTER_ID
# expected: InstanceState 显示 repair 或 replacing 等修复相关状态
```

```json
{
  "TotalCount": "<TotalCount>",
  "InstanceSet": "<InstanceSet>",
  "InstanceId": "<InstanceId>",
  "InstanceRole": "<InstanceRole>",
  "FailedReason": "<FailedReason>",
  "InstanceState": "<InstanceState>"
}
```

## 清理

```bash
# 删除自愈规则
tccli tke DeleteNodePoolHealthCheckPolicy --region <Region> \
    --ClusterId CLUSTER_ID \
    --NodePoolId NODE_POOL_ID \
    --HealthPolicyId HEALTH_POLICY_ID
# expected: exit 0

# 确认已删除
tccli tke DescribeNodePoolHealthCheckPolicy --region <Region> \
    --ClusterId CLUSTER_ID \
    --NodePoolId NODE_POOL_ID
# expected: TotalCount 为 0
```

```json
{"RequestId": "..."}
```

> **警告**：删除自愈规则后，节点故障将不会被自动检测和修复。建议在维护窗口内操作，并在操作后重新创建规则。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `CreateNodePoolHealthCheckPolicy` 返回 `ResourceNotFound.NodePoolNotFound` | `tccli tke DescribeClusterNodePoolDetail --region <Region> --ClusterId CLUSTER_ID --NodePoolId NODE_POOL_ID` | NodePoolId 不存在或非原生类型 | 确认节点池存在且 Type 为 `Native` |
| 返回 `InvalidParameterValue` — CheckType | 检查 `CheckType` 是否为支持的取值 | 使用了不支持的检查类型 | 使用 `ReadonlyFilesystem`、`KubeletProblem` 等支持的枚举值 |
| 返回 `InvalidParameterValue` — Duration | 确认 `Duration` 格式为 `"<数字><单位>"`，如 `"5m"` | Duration 格式错误 | 使用 `"30s"`、`"1m"`、`"5m"`、`"10m"` 等格式 |
| 返回 `LimitExceeded.HealthCheckPolicy` | 查询已存在的策略数量 | 每个节点池的自愈规则数量超限 | 删除不再需要的旧规则 |

### 创建成功但策略不生效

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 节点故障发生后未触发自愈 | `tccli tke DescribeNodePoolHealthCheckPolicy` 确认规则 `Enabled` | 规则 `Enabled` 为 `false` | 修改规则将 `Enabled` 设为 `true` |
| 自愈动作仅上报不修复 | `DescribeNodePoolHealthCheckPolicy` → `RepairAction` | `RepairAction` 设为 `ReportOnly` | 修改为 `RestartNode` 或 `CordonAndReplace` |
| 故障持续时间短但策略未触发 | `DescribeNodePoolHealthCheckPolicy` → `Duration` | 故障持续时间未达到 `Duration` 阈值 | 降低 `Duration` 阈值或等待更长时间 |
| 自愈后节点未恢复 | `kubectl --kubeconfig KUBECONFIG_PATH describe node NODE_NAME` 查看 Conditions | 自愈动作不足以修复故障（如仅重启，但磁盘硬件故障） | 将 `RepairAction` 改为 `CordonAndReplace` |

## 下一步

- [原生节点扩缩容](../原生节点扩缩容/tccli%20操作.md) — page_id `78648`
- [原生节点底层宿主机异常告警](../原生节点底层宿主机异常告警/tccli%20操作.md)
- [修改原生节点](../修改原生节点/tccli%20操作.md) — page_id `103599`

## 控制台替代

[容器服务控制台 → 集群 → 节点管理 → 原生节点 → 故障自愈](https://console.cloud.tencent.com/tke2/cluster)
