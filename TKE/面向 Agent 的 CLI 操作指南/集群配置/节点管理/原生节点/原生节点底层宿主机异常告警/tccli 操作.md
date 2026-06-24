# 原生节点底层宿主机异常告警

> 对照官方：[原生节点底层宿主机异常告警](https://cloud.tencent.com/document/product/457/121135) · page_id `121135`

## 概述

原生节点的底层宿主机（物理机）异常时，腾讯云监控系统会自动检测并上报告警事件。与普通节点不同，原生节点配备了**节点管家**组件，检测到宿主机异常后会联动[故障自愈规则](../故障自愈规则/tccli%20操作.md)触发自动迁移或修复动作，无需人工介入。

异常检测覆盖以下层面：

| 异常类型 | 触发条件 | 联动动作 |
|---------|---------|---------|
| 宿主机硬件故障 | 磁盘 I/O 错误、内存 ECC 错误、网卡故障 | 自动迁移节点到健康宿主机 |
| 宿主机性能降级 | CPU 降频、磁盘性能骤降 | 告警通知 + 可配置自愈策略 |
| 宿主机网络隔离 | 交换机故障导致宿主机失联 | 自动重调度 Pod 到健康节点 |
| 内核/系统故障 | Kernel panic、OOM killer 异常 | 自动重启或迁移节点 |

**核心价值**：原生节点通过节点管家实现从检测到修复的自动化闭环，减少故障恢复时间（MTTR）。可通过 API 查询节点健康状态和宿主机异常告警记录。

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
#    tke:DescribeClusterInstances, tke:DescribeClusterNodePoolDetail
#    tke:DescribeClusterNodePools, monitor:DescribeAlarmHistories
# 验证：执行 DescribeClusterInstances 确认权限
tccli tke DescribeClusterInstances --region <Region> --ClusterId CLUSTER_ID
# expected: exit 0，返回集群节点列表（可为空）
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

### 资源检查

```bash
# 4. 确认目标集群存在且为托管集群
tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterType 为 MANAGED_CLUSTER，ClusterStatus 为 Running

# 5. 确认已有原生节点池
tccli tke DescribeClusterNodePools --region <Region> --ClusterId CLUSTER_ID
# expected: 返回含 Type=Native 的节点池列表

# 6. 确认故障自愈规则已配置（如需要自愈联动）
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId CLUSTER_ID \
    --NodePoolId NODE_POOL_ID \
    | jq '.NodePool.Native.Management.AutoRepair'
# expected: true（如已开启自愈）
```

```json
{
  "TotalCount": "<TotalCount>",
  "Clusters": "<Clusters>",
  "ClusterId": "<ClusterId>",
  "ClusterName": "<ClusterName>",
  "ClusterDescription": "<ClusterDescription>",
  "ClusterVersion": "<ClusterVersion>"
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看节点健康状态 | `DescribeClusterInstances` | 是 |
| 查看节点池详情（含告警策略） | `DescribeClusterNodePoolDetail` | 是 |
| 查看节点池列表 | `DescribeClusterNodePools` | 是 |
| 查看告警历史 | `monitor DescribeAlarmHistories` | 是 |

## 操作步骤

### 步骤 1：查询所有节点的实例状态和异常信息

```bash
tccli tke DescribeClusterInstances --region <Region> \
    --ClusterId CLUSTER_ID
# expected: exit 0，返回 InstanceSet 含所有节点实例状态
```

**预期输出**：

```json
{
    "TotalCount": 3,
    "InstanceSet": [
        {
            "InstanceId": "ins-example-001",
            "InstanceRole": "WORKER",
            "InstanceState": "running",
            "FailedReason": "",
            "NodePoolId": "np-example",
            "CreatedTime": "2025-06-10T08:00:00Z"
        },
        {
            "InstanceId": "ins-example-002",
            "InstanceRole": "WORKER",
            "InstanceState": "failed",
            "FailedReason": "HostHardwareFailure: Disk I/O error on physical host",
            "NodePoolId": "np-example",
            "CreatedTime": "2025-06-10T08:00:00Z"
        },
        {
            "InstanceId": "ins-example-003",
            "InstanceRole": "WORKER",
            "InstanceState": "running",
            "FailedReason": "",
            "NodePoolId": "np-example",
            "CreatedTime": "2025-06-10T08:00:00Z"
        }
    ],
    "RequestId": "..."
}
```

关键字段解读：

| 字段 | 含义 | 正常值 | 异常值示例 |
|------|------|--------|-----------|
| `InstanceState` | 节点实例状态 | `"running"` | `"failed"`, `"shutdown"` |
| `FailedReason` | 失败原因 | `""`（空字符串） | `"HostHardwareFailure: ..."`, `"HostNetworkUnreachable: ..."` |

### 步骤 2：按状态筛选异常节点

```bash
tccli tke DescribeClusterInstances --region <Region> \
    --ClusterId CLUSTER_ID \
    --Filters '[{"Name":"instance-state","Values":["failed"]}]'
# expected: 仅返回 InstanceState 为 failed 的节点
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

### 步骤 3：查看节点池维度的健康状态

```bash
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId CLUSTER_ID \
    --NodePoolId NODE_POOL_ID
# expected: 含节点池状态和内部节点健康汇总
```

**预期输出**（关键字段节选）：

```json
{
    "NodePool": {
        "NodePoolId": "np-example",
        "Name": "native-prod-pool",
        "LifeState": "normal",
        "NodeCountSummary": {
            "ManuallyAdded": {"Total": 0},
            "AutoscalingAdded": {
                "Total": 3,
                "Running": 2,
                "Failed": 1,
                "Starting": 0
            }
        },
        "Native": {
            "Management": {
                "AutoRepair": true,
                "HealthCheckPolicyName": "host-hardware-failure-policy"
            }
        }
    },
    "RequestId": "..."
}
```

### 步骤 4：检查告警历史记录

```bash
tccli monitor DescribeAlarmHistories --region <Region> \
    --StartTime START_TIME \
    --EndTime END_TIME \
    --MonitorTypes '["MT_QCE"]' \
    --Namespaces '[{"MonitorType":"MT_QCE","Namespace":"QCE/TKE"}]'
# expected: 返回告警历史列表（如有宿主机异常告警）
```

### 步骤 5：查看故障自愈规则联动状态

```bash
# 查看节点池的自愈配置
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId CLUSTER_ID \
    --NodePoolId NODE_POOL_ID \
    | jq '.NodePool.Native.Management'
# expected: 查看 AutoRepair 和 HealthCheckPolicyName
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

## 验证

### 验证异常节点识别

```bash
# 列出所有非 Running 状态的节点
tccli tke DescribeClusterInstances --region <Region> \
    --ClusterId CLUSTER_ID \
    | jq '.InstanceSet[] | select(.InstanceState != "running") | {InstanceId: .InstanceId, InstanceState: .InstanceState, FailedReason: .FailedReason}'
# expected: 如有异常节点，列出详细失败原因
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

### 验证自愈联动状态

```bash
# 确认自愈已启用且策略正确
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId CLUSTER_ID \
    --NodePoolId NODE_POOL_ID \
    | jq '{AutoRepair: .NodePool.Native.Management.AutoRepair, Policy: .NodePool.Native.Management.HealthCheckPolicyName}'
# expected: AutoRepair 为 true，Policy 非空
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

| 验证维度 | 命令 | 预期 |
|---------|------|------|
| 节点健康状态 | `DescribeClusterInstances` → `InstanceState` | 正常节点为 `running` |
| 失败原因 | `DescribeClusterInstances` → `FailedReason` | 空为正常；非空含宿主机异常描述 |
| 节点池健康汇总 | `DescribeClusterNodePoolDetail` → `NodeCountSummary.AutoscalingAdded` | Running/Failed 数量与实际一致 |
| 自愈开关 | `DescribeClusterNodePoolDetail` → `Management.AutoRepair` | 生产环境建议 `true` |

## 清理

本页为概念说明页，无资源需清理。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DescribeClusterInstances` 返回 `InvalidParameter.ClusterId` | 检查 `--ClusterId` 参数值 | 集群 ID 格式错误或集群不属于当前账号/地域 | `tccli tke DescribeClusters --region <Region>` 列出全部集群确认正确 ID |
| `DescribeClusterInstances` 返回 `UnauthorizedOperation` | `tccli configure list` 检查凭据；确认 CAM 策略 | 子账号缺少 `tke:DescribeClusterInstances` 权限（此为环境限制，非命令错误） | 联系主账号授予 `QcloudTKEFullAccess` 或最小权限 `tke:DescribeClusterInstances` |

### 节点状态异常判断

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 节点 `InstanceState=failed` 且 `FailedReason` 含 `HostHardwareFailure` | `tccli tke DescribeClusterInstances --region <Region> --ClusterId CLUSTER_ID` 查看 `FailedReason` 详情 | 宿主机硬件故障（磁盘、内存、网卡等），此为云平台侧异常 | 若 `AutoRepair=true`，节点管家会自动迁移；若未开启自愈，需手动删除异常节点，节点池自动补充新节点 |
| 节点 `InstanceState=failed` 但 `FailedReason` 为空 | 同上述命令 + `kubectl describe node NODE_NAME` 查看 K8s 层 Conditions | 可能是节点内软件故障而非宿主机问题（如 kubelet 异常） | 检查 kubelet 日志；如为软件问题，Cordon + Drain 后重启 kubelet |
| 节点健康但 `DescribeClusterNodePoolDetail` 显示部分失败数 | 同上 + `kubectl get nodes -o wide` 交叉验证 | 统计延迟：节点已恢复但 API 统计未更新 | 等待 1-2 分钟重新查询；如持续不一致，保留 RequestId 提交工单 |

## 下一步

- [故障自愈规则](../故障自愈规则/tccli%20操作.md) — page_id `78650`
- [声明式操作实践](../声明式操作实践/tccli%20操作.md) — page_id `78649`
- [原生节点扩缩容](../原生节点扩缩容/tccli%20操作.md) — page_id `78648`
- [Management 参数介绍](../Management%20参数介绍/tccli%20操作.md) — page_id `79698`
- [云监控告警配置](https://console.cloud.tencent.com/monitor/alarm2/policy) — 自定义告警策略监控宿主机指标

## 控制台替代

[TKE 控制台 → 集群 → 节点管理 → 原生节点](https://console.cloud.tencent.com/tke2/cluster)：节点列表「状态」列展示节点健康状态，「监控」按钮查看宿主机级别监控指标。
