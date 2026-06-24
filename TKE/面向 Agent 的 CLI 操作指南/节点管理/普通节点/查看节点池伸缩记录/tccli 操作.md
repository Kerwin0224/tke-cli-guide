# 查看节点池伸缩记录

> 对照官方：[查看节点池伸缩记录](https://cloud.tencent.com/document/product/457/48538) · page_id `48538` · tccli ≥3.1.107 · API 2018-05-25

## 概述

查看节点池伸缩活动记录，适用于业务流量复盘、成本管理和扩缩容失败排查。提供两条查询路径：

| 路径 | 数据来源 | 适用场景 | 查询方式 |
|------|---------|---------|---------|
| **全局伸缩记录** | CA (Cluster Autoscaler) 事件（Kubernetes events / CLS 持久化） | 多节点池场景，关心 CA 的扩缩容决策过程 | `kubectl get events` + CLS 检索 |
| **特定节点池伸缩记录** | AS (Auto Scaling) 产品 API | 关心单个节点池的扩缩容活动详情（状态、原因、实例） | `tccli as DescribeAutoScalingActivities` |

> **重要发现**：`tccli tke DescribeClusterAsGroups` 在节点池关联 AS 组时可能返回 `TotalCount: 0`（实测行为），不能作为查询伸缩活动的唯一手段。推荐通过 AS 产品 API 直接查询。

## 前置条件

- [环境准备](../../../环境准备.md)：`tccli` 已配置，具备 TKE 只读权限和 AS 只读权限
- 已存在集群和节点池
- （全局记录）集群事件持久化已开启，或节点池已发生过扩缩容活动

### 环境检查

```bash
# 1. 检查 tccli 和 kubectl 版本
tccli --version
# expected: tccli version >= 1.0.0
kubectl version --client
# expected: Client Version >= v1.28.0

# 2. 检查 CAM 权限
#    需要：tke:DescribeClusterNodePools, tke:DescribeClusterNodePoolDetail
#          tke:DescribeClusterEndpoints, tke:DescribeClusterKubeconfig
#          as:DescribeAutoScalingGroups, as:DescribeAutoScalingActivities
#          as:DescribeAutoScalingGroupLastActivities
tccli tke DescribeClusterNodePools --ClusterId <ClusterId> --region <Region>
# expected: exit 0，返回节点池列表
```

**预期输出**：

```json
{
    "NodePoolSet": [
        {
            "NodePoolId": "<NodePoolId>",
            "Name": "<NodePoolName>",
            "LifeState": "normal",
            "AutoscalingGroupId": "<AutoscalingGroupId>"
        }
    ],
    "TotalCount": 1
}
```

### 资源检查

```bash
# 3. 确认集群和节点池存在
tccli tke DescribeClusters --ClusterIds '["<ClusterId>"]' --region <Region>
# expected: ClusterStatus: "Running"

tccli tke DescribeClusterNodePools --ClusterId <ClusterId> --region <Region>
# expected: TotalCount >= 1，记录目标 NodePoolId

# 4. 获取节点池关联的 AS Group ID
tccli tke DescribeClusterNodePoolDetail --ClusterId <ClusterId> --NodePoolId <NodePoolId> --region <Region>
# expected: AutoscalingGroupId 非空（格式 asg-xxxxxxxx）
```

**DescribeClusters 预期输出**：

```json
{
    "Clusters": [
        {
            "ClusterId": "<ClusterId>",
            "ClusterName": "<ClusterName>",
            "ClusterStatus": "Running",
            "ClusterVersion": "<K8sVersion>",
            "ClusterNodeNum": 1
        }
    ],
    "TotalCount": 1
}
```

**DescribeClusterNodePools 预期输出**：

```json
{
    "NodePoolSet": [
        {
            "NodePoolId": "<NodePoolId>",
            "Name": "<NodePoolName>",
            "LifeState": "normal",
            "AutoscalingGroupId": "<AutoscalingGroupId>"
        }
    ],
    "TotalCount": 1
}
```

**DescribeClusterNodePoolDetail 预期输出**：

```json
{
    "NodePool": {
        "NodePoolId": "<NodePoolId>",
        "AutoscalingGroupId": "<AutoscalingGroupId>",
        "LifeState": "normal",
        "DesiredNodesNum": 1
    },
    "RequestId": "<RequestId>"
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 查看节点池列表 | `tccli tke DescribeClusterNodePools --ClusterId <ClusterId>` | 是 |
| 查看节点池详情（含 AS Group ID） | `tccli tke DescribeClusterNodePoolDetail --NodePoolId <NodePoolId>` | 是 |
| 查看伸缩组信息 | `tccli as DescribeAutoScalingGroups --AutoScalingGroupIds '["<AsGroupId>"]'` | 是 |
| 查看伸缩活动记录（近期） | `tccli as DescribeAutoScalingGroupLastActivities --AutoScalingGroupIds '["<AsGroupId>"]'` | 是 |
| 查看伸缩活动记录（完整） | `tccli as DescribeAutoScalingActivities --Filters '[{"Name":"auto-scaling-group-id","Values":["<AsGroupId>"]}]'` | 是 |
| 查看 CA 全局伸缩事件 | `kubectl get events -n kube-system --field-selector source=cluster-autoscaler` | 是 |
| 查看集群端点状态 | `tccli tke DescribeClusterEndpoints --ClusterId <ClusterId>` | 是 |

## 操作步骤

### 路径一：查看特定节点池伸缩记录（AS 产品 API）

#### 选择依据

- **为什么用 AS API 而非 TKE API**：`DescribeClusterAsGroups` 在节点池关联 AS 组时可能返回 `TotalCount: 0`（实测行为）。AS 产品 API 直接查询 AS 组数据，返回完整的活动记录、实例状态和时间信息，更可靠。
- **两步法**：先通过 TKE API 获取 `AutoscalingGroupId`，再用 AS API 查询活动。`AutoscalingGroupId` 是连接 TKE 节点池和 AS 伸缩组的桥梁。

#### 步骤 1：获取节点池的 AS Group ID

```bash
tccli tke DescribeClusterNodePoolDetail \
    --ClusterId <ClusterId> \
    --NodePoolId <NodePoolId> \
    --region <Region>
# expected: exit 0，返回 AutoscalingGroupId
```

**预期输出**：

```json
{
    "NodePool": {
        "NodePoolId": "<NodePoolId>",
        "Name": "<NodePoolName>",
        "MaxNodesNum": 1,
        "MinNodesNum": 0,
        "DesiredNodesNum": 1,
        "LifeState": "normal",
        "AutoscalingGroupId": "<AutoscalingGroupId>",
        "LaunchConfigurationId": "<LaunchConfigurationId>",
        "NodeCountSummary": {
            "AutoscalingAdded": {
                "Total": 1,
                "Normal": 1
            }
        }
    },
    "RequestId": "<RequestId>"
}
```

| 关键字段 | 说明 |
|---------|------|
| `AutoscalingGroupId` | AS 伸缩组 ID，格式 `asg-xxxxxxxx`。用于后续 AS API 查询 |
| `DesiredNodesNum` | 期望节点数，对应 AS 组的 `DesiredCapacity` |
| `NodeCountSummary` | 当前节点数汇总（手动添加 vs 自动伸缩添加） |

#### 步骤 2：查看 AS 伸缩组详情

```bash
tccli as DescribeAutoScalingGroups \
    --AutoScalingGroupIds '["<AutoscalingGroupId>"]' \
    --region <Region>
# expected: exit 0，返回伸缩组状态和容量信息
```

**预期输出**：

```json
{
    "TotalCount": 1,
    "AutoScalingGroupSet": [
        {
            "AutoScalingGroupId": "<AutoscalingGroupId>",
            "AutoScalingGroupName": "tke-<NodePoolId>",
            "AutoScalingGroupStatus": "NORMAL",
            "DesiredCapacity": 1,
            "MaxSize": 1,
            "MinSize": 0,
            "CreatedTime": "2026-06-23T08:17:11Z",
            "VpcId": "<VpcId>",
            "SubnetIdSet": [
                "<SubnetId>"
            ]
        }
    ]
}
```

| 字段 | 说明 |
|------|------|
| `AutoScalingGroupStatus` | `NORMAL`（正常）/ `DISABLED`（停用） |
| `DesiredCapacity` | 期望容量，对应节点池 `DesiredNodesNum` |
| `MaxSize` / `MinSize` | 伸缩组容量边界 |

#### 步骤 3：查看近期伸缩活动

```bash
tccli as DescribeAutoScalingGroupLastActivities \
    --AutoScalingGroupIds '["<AutoscalingGroupId>"]' \
    --region <Region>
# expected: exit 0，返回最近一次伸缩活动记录
```

**预期输出**（有活动记录时）：

```json
{
    "ActivitySet": [
        {
            "ActivityId": "<ActivityId>",
            "AutoScalingGroupId": "<AutoscalingGroupId>",
            "StatusCode": "SUCCESSFUL",
            "StatusMessage": "伸缩活动成功。",
            "Cause": "因匹配期望实例数",
            "Description": "因匹配期望实例数，扩容1台",
            "StartTime": "2026-06-23T08:12:41Z",
            "EndTime": "2026-06-23T08:13:18Z"
        }
    ]
}
```

**预期输出**（无活动记录时）：

```json
{
    "ActivitySet": []
}
```

#### 步骤 4：查看完整伸缩活动历史

```bash
tccli as DescribeAutoScalingActivities \
    --Filters '[{"Name":"auto-scaling-group-id","Values":["<AutoscalingGroupId>"]}]' \
    --region <Region>
# expected: exit 0，返回近两年的完整活动记录
```

**预期输出**：

```json
{
    "TotalCount": 1,
    "ActivitySet": [
        {
            "ActivityId": "<ActivityId>",
            "AutoScalingGroupId": "<AutoscalingGroupId>",
            "StatusCode": "SUCCESSFUL",
            "StatusMessage": "伸缩活动成功。",
            "Cause": "因匹配期望实例数",
            "Description": "因匹配期望实例数，扩容1台",
            "StartTime": "2026-06-23T08:12:41Z",
            "EndTime": "2026-06-23T08:13:18Z",
            "ActivityRelatedInstanceSet": [
                {
                    "InstanceId": "<InstanceId>",
                    "InstanceStatus": "SUCCESSFUL"
                }
            ]
        }
    ]
}
```

| 关键字段 | 说明 |
|---------|------|
| `StatusCode` | `SUCCESSFUL`（成功）/ `FAILED`（失败）/ `IN_PROGRESS`（进行中）/ `CANCELLED`（已取消） |
| `Cause` | 触发原因：`因匹配期望实例数`、`因定时任务触发`、`因告警策略触发` |
| `Description` | 活动描述，如"扩容1台"、"缩容2台" |
| `ActivityRelatedInstanceSet` | 本次活动涉及的 CVM 实例及各自状态 |
| `StartTime` / `EndTime` | 活动起止时间（精确到秒） |

#### 步骤 5（可选）：按时间范围过滤

```bash
tccli as DescribeAutoScalingActivities \
    --Filters '[{"Name":"auto-scaling-group-id","Values":["<AutoscalingGroupId>"]}]' \
    --StartTime "2026-06-01T00:00:00Z" \
    --EndTime "2026-06-23T23:59:59Z" \
    --region <Region>
# expected: exit 0，返回指定时间范围内的活动
```

**预期输出**：

```json
{
    "TotalCount": 1,
    "ActivitySet": [
        {
            "AutoScalingGroupId": "<AutoscalingGroupId>",
            "ActivityId": "<ActivityId>",
            "ActivityType": "SCALE_OUT",
            "StatusCode": "SUCCESSFUL",
            "StatusMessage": "伸缩活动成功。",
            "Cause": "因匹配期望实例数",
            "Description": "因匹配期望实例数，扩容1台",
            "StartTime": "2026-06-23T08:12:41Z",
            "EndTime": "2026-06-23T08:13:18Z",
            "ActivityRelatedInstanceSet": [
                {
                    "InstanceId": "<InstanceId>",
                    "InstanceStatus": "SUCCESSFUL"
                }
            ]
        }
    ]
}
```

### 路径二：查看全局伸缩记录（CA 事件）

> **执行前提**：以下 kubectl 命令需要集群端点可达。如集群仅有内网端点，需通过 IOA、VPN、专线或在同 VPC 的 CVM 上执行。

#### 步骤 1：检查集群端点可达性

```bash
tccli tke DescribeClusterEndpoints \
    --ClusterId <ClusterId> \
    --region <Region>
# expected: ClusterExternalEndpoint 或 ClusterIntranetEndpoint 非空
```

**预期输出**：

```json
{
    "ClusterExternalEndpoint": "",
    "ClusterIntranetEndpoint": "<InternalIP>",
    "ClusterDomain": "cls-xxxxxxxx.ccs.tencent-cloud.com"
}
```

#### 步骤 2：获取 kubeconfig

```bash
tccli tke DescribeClusterKubeconfig \
    --ClusterId <ClusterId> \
    --IsExtranet true \
    --region <Region> \
    | python3 -c "import json,sys; print(json.load(sys.stdin)['Kubeconfig'])" \
    > ~/.kube/config-<ClusterId>
# expected: kubeconfig 写入成功
```

**预期输出**（原始 API 响应，kubeconfig 内容经 base64 编码）：

```json
{
    "Kubeconfig": "<base64-encoded-kubeconfig>",
    "RequestId": "<RequestId>"
}
```

#### 步骤 3：查看 CA 伸缩事件

```bash
KUBECONFIG=~/.kube/config-<ClusterId> \
    kubectl get events -n kube-system \
    --field-selector source=cluster-autoscaler \
    --sort-by='.lastTimestamp'
# expected: 返回 CA 扩缩容事件列表
```

**预期输出**：

```text
LAST SEEN   TYPE      REASON                 OBJECT                        MESSAGE
2m          Normal    TriggeredScaleUp       pod/nginx-7d8b9f6c4-xxxx      pod triggered scale-up
2m          Normal    ScaledUpGroup          node/<NodeName>               Scale-up: group <AsGroupId> size set to 2
30m         Normal    ScaleDown              node/<NodeName>               Scale-down: removing node
```

| CA Reason | 含义 |
|-----------|------|
| `TriggeredScaleUp` | CA 检测到 Pod 无法调度，触发扩容 |
| `NotTriggerScaleUp` | CA 评估后决定不扩容（如 Pod 可调度到已有节点） |
| `ScaledUpGroup` | 扩容成功，AS 组增加节点 |
| `FailedToScaleUpGroup` | 扩容失败（需查看详细错误） |
| `ScaleDown` | 缩容成功，节点正在被移除 |
| `ScaleDownFailed` | 缩容失败 |
| `ScaleDownEmpty` | 节点空闲，符合缩容条件 |

> **注意**：Kubernetes events 默认仅保留 1 小时。如需长期存储和检索，建议为集群开启事件持久化功能（`tccli tke EnableEventPersistence`），将事件写入 CLS（日志服务）。

#### 步骤 4（可选）：通过 CLS 检索历史 CA 事件

开启事件持久化后，可在 CLS 控制台使用以下检索语句：

```text
event.source.component:cluster-autoscaler
```

## 验证

### 控制面（tccli）

```bash
# 1. 验证节点池详情中的 AS Group ID 与 AS API 一致
tccli tke DescribeClusterNodePoolDetail \
    --ClusterId <ClusterId> \
    --NodePoolId <NodePoolId> \
    --region <Region> \
    | python3 -c "import json,sys; d=json.load(sys.stdin); print('ASG ID:', d['NodePool']['AutoscalingGroupId'])"
# expected: 输出 asg-xxxxxxxx

tccli as DescribeAutoScalingGroups \
    --AutoScalingGroupIds '["<AutoscalingGroupId>"]' \
    --region <Region> \
    | python3 -c "import json,sys; d=json.load(sys.stdin); print('Status:', d['AutoScalingGroupSet'][0]['AutoScalingGroupStatus'])"
# expected: Status: NORMAL
```

```bash
# 2. 验证伸缩活动记录可查询
tccli as DescribeAutoScalingActivities \
    --Filters '[{"Name":"auto-scaling-group-id","Values":["<AutoscalingGroupId>"]}]' \
    --region <Region> \
    | python3 -c "import json,sys; d=json.load(sys.stdin); print('TotalCount:', d.get('TotalCount', 0))"
# expected: 返回活动总数（可为 0，表示尚无伸缩活动）
```

| 维度 | 检查内容 | 预期 |
|------|---------|------|
| AS Group 存在性 | `DescribeClusterNodePoolDetail` 的 `AutoscalingGroupId` | 非空，格式 `asg-xxxxxxxx` |
| AS Group 状态 | `DescribeAutoScalingGroups` 的 `AutoScalingGroupStatus` | `NORMAL`（正常） |
| 活动记录 | `DescribeAutoScalingActivities` 的 `TotalCount` | >= 0（0 表示尚无伸缩活动，非异常） |
| 活动状态 | 最新活动的 `StatusCode` | `SUCCESSFUL` / `FAILED` / `IN_PROGRESS` |

### 数据面（kubectl）

```bash
# 验证 CA 事件可查询（需集群端点可达）
KUBECONFIG=~/.kube/config-<ClusterId> \
    kubectl get events -n kube-system \
    --field-selector source=cluster-autoscaler
# expected: 返回事件列表（可为空，表示近期无伸缩活动）
```

> 如 kubectl 不可达（集群仅有内网端点），此验证跳过，见 [排障](#排障) 了解替代方案。

## 清理

查看操作无需清理。如创建了专用节点池来测试伸缩活动，需删除节点池：

```bash
tccli tke DeleteClusterNodePool \
    --ClusterId <ClusterId> \
    --NodePoolIds '["<NodePoolId>"]' \
    --KeepInstance false \
    --region <Region>
# ⚠️ KeepInstance false 会级联删除该节点池关联的 AS 组和启动配置
```

> **注意**：自建安全组不会随节点池自动删除，需手动清理 `tccli vpc DeleteSecurityGroup`。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DescribeClusterAsGroups` 返回 `TotalCount: 0`，但节点池明确存在 | `tccli tke DescribeClusterNodePoolDetail` 获取 `AutoscalingGroupId` → `tccli as DescribeAutoScalingGroups` 验证 | TKE API 在节点池未显式开启弹性伸缩时可能不返回 AS 组信息（实测行为） | 改用 AS 产品 API：`tccli as DescribeAutoScalingGroups --AutoScalingGroupIds '["<AutoscalingGroupId>"]'` |
| `DescribeClusterNodePoolDetail` 返回 `AutoscalingGroupId` 为空 | `tccli tke DescribeClusterNodePools` 检查 `LifeState` | 节点池可能正在删除中（`LifeState: "deleting"`）或创建失败 | 等待节点池 `LifeState` 变为 `normal` 后重试 |
| `DescribeAutoScalingActivities` 返回 `UnauthorizedOperation` | 检查当前账号是否有 AS 只读权限 | CAM 缺少 `as:DescribeAutoScalingActivities` 权限 | 联系管理员添加 AS 只读权限（`QcloudASReadOnlyAccess`） |
| kubectl 执行 `get events` 报 DNS 解析错误 `no such host` | `tccli tke DescribeClusterEndpoints` 检查 `ClusterExternalEndpoint` | 集群无公网端点，当前环境无法解析内网域名 | 见下方 [kubectl 不可达](#kubectl-不可达) 解决路径 |

### kubectl 不可达

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| kubectl 执行报 `dial tcp: lookup ... no such host` | `DescribeClusterEndpoints` → `ClusterExternalEndpoint` 为空，`ClusterIntranetEndpoint` 为内网 IP | 集群仅有内网端点，外部网络无法解析内网域名 | 方案 1：创建公网端点 `tccli tke CreateClusterEndpoint --IsExtranet true`。方案 2：通过 IOA/VPN/专线 连接到集群 VPC。方案 3：在同 VPC 内启动一台 CVM 作为跳板机执行 kubectl 命令 |
| 公网端点已创建但 kubectl 仍不通 | `DescribeClusterEndpoints` 确认 `ClusterExternalEndpoint` 非空 → 检查安全组入站规则 | 安全组未放行 443 端口 | `tccli vpc CreateSecurityGroupPolicies` 添加 TCP 443 放行规则 |

### 创建节点池时的参数错误（如需自建节点池测试）

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `InvalidParameter.Param: autoscaling group name can not be set` | `AutoScalingGroupPara` JSON 中含 `AutoScalingGroupName` 字段 | TKE 自动生成 AS 组名，不支持手动设置 | 移除 `LaunchConfigurePara` 中的 `AutoScalingGroupName` |
| `InvalidParameter.Param: image id can not be set` | `LaunchConfigurePara` JSON 中含 `ImageId` 字段 | TKE 根据 `NodePoolOs` 自动选择镜像 | 移除 `LaunchConfigurePara` 中的 `ImageId` 字段 |
| `InvalidParameter.Param: security group ids is not set` | `LaunchConfigurePara` 缺少 `SecurityGroupIds` | 安全组为必填字段 | 在 `LaunchConfigurePara` 中添加 `"SecurityGroupIds": ["<SecurityGroupId>"]` |
| `InvalidParameter.Param: InstanceChargeType must be set` | `LaunchConfigurePara` 缺少 `InstanceChargeType` | 计费类型为必填 | 添加 `"InstanceChargeType": "POSTPAID_BY_HOUR"` |
| `FailedOperation.PolicyServerRuntimeNotMatchK8sVersionError` | 检查 `RuntimeVersion` 是否与 K8s 版本匹配 | 指定的容器运行时版本不兼容当前 K8s 版本（如 containerd 1.6.28 不兼容 K8s 1.32） | **不指定 `RuntimeVersion`**，让 TKE 自动选择匹配的运行时版本。也可通过 `tccli tke DescribeSupportedRuntime` 查询兼容版本 |

> **创建节点池的快速参考**：`LaunchConfigurePara` 中只需要 `InstanceChargeType`、`InstanceType`、`SystemDisk`、`InternetAccessible`、`SecurityGroupIds`。不要设置 `AutoScalingGroupName`、`ImageId`、`LaunchConfigurationName`——TKE 会自动填充。

## 下一步

- [创建节点池](https://cloud.tencent.com/document/product/457/43735) — 创建新的节点池并启用弹性伸缩
- [查看节点池](https://cloud.tencent.com/document/product/457/43736) — 查看节点池当前配置和状态
- [调整节点池](https://cloud.tencent.com/document/product/457/43737) — 修改节点池规模和弹性伸缩配置
- [删除节点池](https://cloud.tencent.com/document/product/457/43738) — 删除不再需要的节点池

## 控制台替代

[容器服务控制台 → 集群 → 节点管理 → 节点池 → 伸缩记录](https://console.cloud.tencent.com/tke2/nodepool) — 查看节点池弹性伸缩活动详情。全局伸缩记录见 [日志 → 事件日志 → 检索分析](https://console.cloud.tencent.com/tke2/log)。
