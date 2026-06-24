# 移出节点

> 对照官方：[移出节点](https://cloud.tencent.com/document/product/457/32204) · page_id `32204` · tccli ≥3.1.107 · API 2018-05-25

## 概述

通过 `DeleteClusterInstances` 将 Worker 节点从 TKE 集群中移出。提供两种删除策略，由 `InstanceDeleteMode` 控制：

| 模式 | `InstanceDeleteMode` | CVM 实例 | CVM 计费 | 适用场景 |
|------|---------------------|---------|---------|---------|
| 保留 CVM | `retain` | 保留在账号下，仅从集群移除 | 继续计费，需手动释放 | 节点排障、临时下线、计划重新加入集群 |
| 销毁 CVM | `terminate` | 销毁（同时删除云硬盘和公网 IP） | 停止计费 | 缩容、不再需要的节点 |

**建议：生产环境优先使用 `retain` 模式**，确认数据安全后再决定是否销毁 CVM。移出为异步操作，需轮询确认节点已从集群实例列表中消失。

## 前置条件

- [环境准备](../../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置，region 为目标地域

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:DeleteClusterInstances, tke:DescribeClusterInstances
#    tke:DescribeClusters
# 验证：执行 DescribeClusters 确认权限
tccli tke DescribeClusters --region <Region>
# expected: exit 0，返回集群列表（可为空）
```

**预期输出**：

```json
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "cls-example",
            "ClusterName": "example-cluster",
            "ClusterStatus": "Running",
            "ClusterVersion": "1.32.2"
        }
    ],
    "RequestId": "..."
}
```

### 资源检查

```bash
# 4. 确认集群存在且状态正常
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["<ClusterId>"]'
# expected: ClusterStatus 为 "Running"
```

**预期输出**：

```json
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "cls-example",
            "ClusterName": "example-cluster",
            "ClusterStatus": "Running",
            "ClusterVersion": "1.32.2",
            "ClusterNodeNum": 3
        }
    ],
    "RequestId": "..."
}
```

```bash
# 5. 查询集群内节点列表，确认待移出的节点存在
tccli tke DescribeClusterInstances --region <Region> \
    --ClusterId <ClusterId>
# expected: 返回节点列表，确认目标 InstanceId 在列表中
```

**预期输出**：

```json
{
    "TotalCount": 2,
    "InstanceSet": [
        {
            "InstanceId": "ins-example-01",
            "InstanceRole": "WORKER",
            "InstanceState": "running",
            "LanIP": "172.24.0.45",
            "NodePoolId": ""
        },
        {
            "InstanceId": "ins-example-02",
            "InstanceRole": "WORKER",
            "InstanceState": "running",
            "LanIP": "172.24.0.34",
            "NodePoolId": "np-example"
        }
    ],
    "RequestId": "..."
}
```

```bash
# 6. 排空节点上的 Pod（移出前必须）
kubectl drain <InstanceId> \
    --ignore-daemonsets \
    --delete-emptydir-data
# expected: node/<InstanceId> drained

# 确认排空完成
kubectl get pods -A -o wide \
    --field-selector spec.nodeName=<InstanceId>
# expected: 仅剩 DaemonSet Pod，无业务 Pod
```

**预期输出**：

```text
NAMESPACE     NAME                                      READY   STATUS    RESTARTS   AGE     NODE
kube-system   coredns-xxx                              1/1     Running   0          10m     <InstanceId>
kube-system   kube-proxy-xxx                           1/1     Running   0          10m     <InstanceId>
```

## 控制台与 CLI 参数映射

### 关键字段说明

以下说明 `DeleteClusterInstances` 的主要参数。

| 字段 | 类型 | 必填 | 幂等 | 取值与约束 | 错误后果 |
|------|------|:--:|:--:|------|------|
| `ClusterId` | String | 是 | 是 | 集群 ID，格式 `cls-` 开头。`tccli tke DescribeClusters` 获取 | 集群不存在 → `ResourceNotFound.ClusterNotFound` |
| `InstanceIds` | Array of String | 是 | 否 | CVM 实例 ID 列表，格式 `ins-` 开头。必须已在该集群中。若实例属于节点池，移出后节点数不得低于节点池 `MinNodesNum`；否则需先将节点池 `MinNodesNum` 调整为 0（`tccli tke ModifyClusterNodePool`），移出完成后再恢复 | 实例不在集群中 → 出现在 `NotFoundInstanceIds` 中；节点池节点数低于 `MinNodesNum` → `FailedOperation.AsCommon` |
| `InstanceDeleteMode` | String | 是 | 是 | `retain`（仅移出集群，保留 CVM）或 `terminate`（销毁 CVM，仅支持按量计费实例）。推荐 `retain` | 误用 `terminate` → CVM 被销毁，数据不可恢复 |
| `ForceDelete` | Boolean | 否 | 是 | 默认 `false`（正常驱逐 Pod 后移出）。`true` 强制删除（节点初始化异常时可用） | 设为 `true` → Pod 被强制终止，可能导致服务中断 |
| `ResourceDeleteOptions` | Array | 否 | 否 | 关联资源删除策略，目前支持 CBS。`DeleteMode`：`terminate`（销毁）或 `retain`（保留，默认） | 未指定时 CBS 默认保留，需手动清理 |

## 操作步骤

### 步骤 1：排空节点上的 Pod

> **重要**：移出节点前必须先通过 `kubectl drain` 排空 Pod。`ForceDelete=true` 会跳过此步骤，仅限节点不可用（如 kubelet 僵死）的紧急场景。

```bash
# 排空节点，将 Pod 迁移至其他健康节点
kubectl drain <InstanceId> \
    --ignore-daemonsets \
    --delete-emptydir-data
# expected: node/<InstanceId> drained
```

```bash
# 确认排空完成
kubectl get pods -A -o wide \
    --field-selector spec.nodeName=<InstanceId>
# expected: 无残留业务 Pod（DaemonSet 除外）
```

### 步骤 2：移出节点

#### 选择依据

*为什么选这个值而不是其他：*

- **InstanceDeleteMode（保留 vs 销毁）**：选择 `retain`（保留 CVM 实例）而非 `terminate`（销毁 CVM）。仅将节点移出集群，CVM 实例保留在账号下，可重新加入集群或用于其他用途。`terminate` 会级联删除关联的云硬盘和公网 IP，数据不可恢复，仅适用于确认不再需要的节点。
- **ForceDelete（不启用）**：默认不启用（`false`）。生产环境应先用 `kubectl drain` 正常排空 Pod 后再移出。`ForceDelete=true` 会强制删除节点、忽略 Pod 驱逐，仅在节点不可用（如 kubelet 僵死、网络隔离）且无法正常排空时使用。误用可能导致服务中断。

#### 最小移出（retain 模式，仅必填字段）

仅将节点移出集群，CVM 保留不销毁：

```bash
tccli tke DeleteClusterInstances --region <Region> \
    --ClusterId <ClusterId> \
    --InstanceIds '["<InstanceId>"]' \
    --InstanceDeleteMode retain
# expected: exit 0，返回 SuccInstanceIds 含目标实例
```

**预期输出**：

```json
{
    "Response": {
        "SuccInstanceIds": ["ins-example-01"],
        "FailedInstanceIds": [],
        "NotFoundInstanceIds": [],
        "RequestId": "..."
    }
}
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<Region>` | 地域 | 如 `ap-guangzhou` | `tccli configure list` |
| `<ClusterId>` | 集群 ID | 格式 `cls-` 开头 | `tccli tke DescribeClusters` |
| `<InstanceId>` | 待移出的 CVM 实例 ID | 格式 `ins-` 开头 | `tccli tke DescribeClusterInstances` |

#### 增强配置（terminate 模式，销毁 CVM 及关联资源）

> **警告**：`terminate` 模式会**销毁 CVM 实例**，同时级联删除云硬盘和公网 IP，操作不可逆。

`delete-nodes-terminate.json`：

```json
{
  "ClusterId": "<ClusterId>",
  "InstanceIds": ["<InstanceId>"],
  "InstanceDeleteMode": "terminate",
  "ForceDelete": false,
  "ResourceDeleteOptions": [
    {
      "ResourceType": "CBS",
      "DeleteMode": "terminate",
      "SkipDeletionProtection": false
    }
  ]
}
```

```bash
tccli tke DeleteClusterInstances --region <Region> \
    --cli-input-json file://delete-nodes-terminate.json
# expected: exit 0，返回 SuccInstanceIds 含目标实例，CVM 及云硬盘被销毁
```

**预期输出**：

```json
{
    "Response": {
        "SuccInstanceIds": ["ins-example-01"],
        "FailedInstanceIds": [],
        "NotFoundInstanceIds": [],
        "RequestId": "..."
    }
}
```

### 步骤 3：轮询确认移出完成

移出为异步操作，轮询直到目标节点不在集群实例列表中：

```bash
# 查询集群节点列表，确认目标节点已移出
tccli tke DescribeClusterInstances --region <Region> \
    --ClusterId <ClusterId>
# expected: TotalCount 减 1，目标 InstanceId 不在 InstanceSet 中
```

**预期输出**：

```json
{
    "TotalCount": 1,
    "InstanceSet": [
        {
            "InstanceId": "ins-example-02",
            "InstanceRole": "WORKER",
            "InstanceState": "running",
            "LanIP": "172.24.0.34",
            "NodePoolId": "np-example"
        }
    ],
    "RequestId": "..."
}
```

```bash
# 从 Kubernetes 侧确认节点已移除
kubectl get nodes
# expected: 目标节点不再出现在列表中
```

**预期输出**：

```text
NAME            STATUS   ROLES    AGE   VERSION
172.24.0.34     Ready    <none>   30m   v1.32.2
```

## 验证

| 维度 | 命令 | 预期 |
|------|------|------|
| 节点列表 | `tccli tke DescribeClusterInstances --ClusterId <ClusterId>` | `TotalCount` 减 1，目标 `InstanceId` 不在 `InstanceSet` 中 |
| CVM 状态（retain 模式） | `tccli cvm DescribeInstances --region <Region> --InstanceIds '["<InstanceId>"]'` | CVM 仍存在，状态为 `RUNNING`，但不再由 TKE 管理 |
| CVM 状态（terminate 模式） | 同上 | 返回空或不含该实例 |
| Kubernetes 节点 | `kubectl get nodes` | 目标节点不再出现 |
| 集群状态 | `tccli tke DescribeClusters --ClusterIds '["<ClusterId>"]'` | `ClusterStatus` 为 `Running`（不受影响） |

**CVM 状态验证输出（retain 模式）**：

```json
{
    "TotalCount": 1,
    "InstanceSet": [
        {
            "InstanceId": "ins-example-01",
            "InstanceState": "RUNNING",
            "InstanceType": "S5.MEDIUM2",
            "Placement": {
                "Zone": "ap-guangzhou-3"
            }
        }
    ],
    "RequestId": "..."
}
```

> **kubectl 使用说明**：`kubectl get nodes` 等 kubectl 命令需要 kubeconfig 文件。获取方式：
> ```bash
> # 方式 1：公网端点（推荐，无需 IOA/VPN）
> tccli tke DescribeClusterKubeconfig --region <Region> \
>     --ClusterId <ClusterId> --IsExtranet true
>
> # 方式 2：内网端点（需 IOA/VPN/专线）
> tccli tke DescribeClusterSecurity --region <Region> \
>     --ClusterId <ClusterId>
> ```
> 将返回的 kubeconfig 保存为 `~/.kube/config` 后即可使用 kubectl。公网端点须确保安全组已放行 TCP 443 入站规则。

最长等待约 3 分钟。移出以 `SuccInstanceIds` 返回为准；节点从 `DescribeClusterInstances` 结果中消失可能有 1-2 分钟延迟。

## 清理

本页操作本身即为清理操作（从集群移出节点）。以下副作用需注意：

> **副作用警告**：
> - `InstanceDeleteMode=terminate` 会**级联删除**关联的 CVM 实例、云硬盘及公网 IP，数据不可恢复
> - `InstanceDeleteMode=retain` 仅将节点移出集群，CVM 实例保留但不再由 TKE 管理，**CVM 将持续产生费用**，需在 CVM 控制台手动释放
> - 若待移出节点属于节点池，移出后节点池实际节点数可能低于 `MinNodesNum`。Auto Scaling 检测到节点不足时会**自动扩容创建新节点**，导致移出操作被新节点抵消
> - `kubectl drain` 驱逐 Pod 时，目标节点若无充足资源或无法满足 Pod 亲和性要求，被驱逐的 Pod 会调度失败（`Pending` 状态）。建议先确认其他节点资源余量
> - 生产环境移出节点前务必先 `kubectl drain` 排空 Pod

```bash
# 清理前确认：查看移出节点的 CVM 是否仍在运行（retain 模式）
tccli cvm DescribeInstances --region <Region> \
    --InstanceIds '["<InstanceId>"]'
# expected: retain 模式——CVM 仍在 RUNNING 状态，确认后手动销毁
```

**预期输出**：

```json
{
    "TotalCount": 1,
    "InstanceSet": [
        {
            "InstanceId": "ins-example-01",
            "InstanceState": "RUNNING",
            "InstanceType": "S5.MEDIUM2",
            "Placement": {
                "Zone": "ap-guangzhou-3"
            }
        }
    ],
    "RequestId": "..."
}
```

```bash
# retain 模式后续清理：手动销毁 CVM 实例
tccli cvm TerminateInstances --region <Region> \
    --InstanceIds '["<InstanceId>"]'
# expected: exit 0，CVM 实例被销毁，停止计费
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DeleteClusterInstances` 返回 `InvalidParameter.ClusterId` | `tccli tke DescribeClusters --region <Region>` 列出全部集群 | 集群 ID 格式错误或不属于当前账号/地域 | 从 `DescribeClusters` 输出复制正确的 `ClusterId` |
| 目标 `InstanceId` 出现在 `NotFoundInstanceIds` 中 | `tccli tke DescribeClusterInstances --region <Region> --ClusterId <ClusterId>` 确认实例是否在集群中 | 实例已被移出或不属于该集群 | 确认 InstanceId 正确，且实例仍在集群中 |
| `FailedOperation.Param` — 实例已在集群中或状态冲突 | `tccli tke DescribeClusterInstances --region <Region> --ClusterId <ClusterId>` 查看实例状态 | 实例正在初始化、升级或其他操作中 | 等待实例操作完成后重试 |
| `InvalidParameterValue.InvalidImageState` — 镜像状态异常 | `tccli cvm DescribeInstances --region <Region> --InstanceIds '["<InstanceId>"]'` 查看实例详情 | CVM 镜像处于 DELETING 等异常状态（此为环境限制，非命令错误） | 等待镜像状态恢复正常后重试，或更换可用镜像 |
| `FailedOperation.NetworkScaleError` — VPC-CNI IP 不足 | `tccli vpc DescribeSubnets --region <Region>` 检查对应 zone 子网的 `AvailableIpCount` | 该 zone 的 VPC-CNI Pod IP 不足，影响网络相关操作 | 为对应 zone 添加子网，或使用 `--SkipValidateOptions '["VpcCniCIDRCheck"]'` 跳过 IP 检查 |
| `FailedOperation.AsCommon` — DesiredCapacity 低于 MinSize | `tccli tke DescribeClusterNodePools --region <Region> --ClusterId <ClusterId>` 查看节点池 `MinNodesNum` 和 `DesiredNodesNum` | 移出后节点池节点数低于最小节点数限制。Auto Scaling 拒绝移出操作 | 将节点池 `MinNodesNum` 调整为 0（`tccli tke ModifyClusterNodePool --ClusterId <ClusterId> --NodePoolId <NodePoolId> --MinNodesNum 0`），移出完成后再恢复 `MinNodesNum` 为预期值 |
| `UnauthorizedOperation` — 权限不足 | `tccli configure list` 检查当前凭据 | 子账号缺少 `tke:DeleteClusterInstances` 权限（此为环境限制，非命令错误） | 联系主账号授予 `QcloudTKEFullAccess` 或自定义策略 |

### 移出成功但存在残留

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| retain 模式移出后 CVM 持续计费 | `tccli cvm DescribeInstances --region <Region> --InstanceIds '["<InstanceId>"]'` 确认 CVM 仍在运行 | retain 模式仅移出集群，CVM 仍按量计费 | 确认不需要后，`tccli cvm TerminateInstances --region <Region> --InstanceIds '["<InstanceId>"]'` |
| 节点移出后 kubectl 仍短暂显示该节点 | `kubectl get nodes` 查看 | kubelet 心跳超时有默认宽限期（约 5 分钟），节点 NotReady 后才会从 Kubernetes 中移除 | 等待约 5 分钟，节点自动消失 |
| CVM 销毁后云硬盘未释放 | `tccli cbs DescribeDisks --region <Region>` 过滤检查 | 独立云硬盘（非系统盘）开启了随实例释放保护或独立计费 | 手动 `tccli cbs TerminateDisks --region <Region> --DiskIds '["<DiskId>"]'` |
| 异步延迟 — 移出操作返回成功但 DescribeClusterInstances 仍短暂显示节点 | 再次 `tccli tke DescribeClusterInstances --region <Region> --ClusterId <ClusterId>` | API 返回与数据面同步有 1-2 分钟延迟 | 等待 2 分钟后重新查询。超过 5 分钟仍未消失则保留 region、ClusterId、InstanceId、RequestId → 登录 [TKE 控制台](https://console.cloud.tencent.com/tke2/cluster) 查看 → 仍无法解决则 [提交工单](https://console.cloud.tencent.com/workorder) |

### 移出后重新加入集群的注意事项

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `AddExistedInstances` 报 `FailedOperation.NetworkScaleError` | `tccli vpc DescribeSubnets --region <Region>` 检查 zone 子网 IP | VPC-CNI 模式下该 zone 的 Pod IP 不足 | 为对应 zone 添加子网，或 `--SkipValidateOptions '["VpcCniCIDRCheck"]'` 跳过 IP 检查 |
| `AddExistedInstances` 指定 `--ImageId` 导致 CVM 被新实例替换 | `tccli cvm DescribeInstances --region <Region>` 查看实例 ID 是否变化 | `AddExistedInstances` 指定 `--ImageId` 会触发 CVM 系统重装，可能改变实例 ID | 重新加入时不指定 `--ImageId`（使用 CVM 现有镜像），除非明确需要重装系统 |
| `AddExistedInstances` 报 `FailedOperation.Param` — 实例已在集群中 | `tccli tke DescribeClusterInstances --region <Region> --ClusterId <ClusterId>` 检查 | CVM 实例仍在集群中（未完全移出） | 等待移出完成后重试 |

## 下一步

- [驱逐或封锁节点](../驱逐或封锁节点/tccli%20操作.md) — 封锁和驱逐的详细操作（page_id `32205`）
- [添加节点](https://cloud.tencent.com/document/product/457/32203) — TKE 控制台添加节点
- [创建节点池](https://cloud.tencent.com/document/product/457/32201) — 通过节点池批量管理节点
- [集群节点管理概述](https://cloud.tencent.com/document/product/457/30715) — 节点管理全景

## 控制台替代

[TKE 控制台 → 集群 → 节点管理 → 移出](https://console.cloud.tencent.com/tke2/cluster)
