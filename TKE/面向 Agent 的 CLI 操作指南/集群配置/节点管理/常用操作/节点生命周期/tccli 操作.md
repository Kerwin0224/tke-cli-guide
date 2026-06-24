# 查询节点生命周期状态

> 对照官方：[节点生命周期](https://cloud.tencent.com/document/product/457/32202) · page_id `32202`

## 概述

TKE 节点在其生命周期中经历多个状态阶段。本页说明如何用 tccli（控制面）和 kubectl（数据面）观察节点状态，并解读控制台「健康 / 异常 / 已封锁 / 驱逐中」标签与底层字段的对应关系。普通节点底层对应 CVM 实例，其生命周期也受 CVM 生命周期影响（参考 [云服务器生命周期](https://cloud.tencent.com/document/product/457/32202)）。

| 控制台状态 | tccli 字段（`DescribeClusterInstances`） | kubectl 对应 | 是否可调度 |
|-----------|------------------------------------------|-------------|:----------:|
| 健康 | `InstanceState: running` 且 `FailedReason` 为空或含 `Ready:True` | `Ready=True` | 是 |
| 异常 | `FailedReason` 非空（如 `KubeletNotReady`） | `Ready=False` | 视情况 |
| 已封锁 | `unschedulable` 为 true（控制台 cordon 后） | `SchedulingDisabled` | 否 |
| 驱逐中 | `DrainStatus: draining` | `unschedulable: true` + Pod 迁移 | 否 |
| 移出中 | `InstanceState: deleting` | 节点对象消失 | 否 |

> **kubectl 数据面可达性**：`kubectl get nodes` 需 APIServer 端点可达。公网端点可能受 CAM 策略限制（如 strategyId:240463971 拒绝 `tke:clusterExtranetEndpoint`），内网端点需 VPN/IOA。若 kubectl 不可达，`DescribeClusterInstances` 控制面状态可替代。

## 前置条件

- [环境准备](../../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId、secretKey、region 均已配置，region 为目标地域

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:DescribeClusters, tke:DescribeClusterInstances
# 验证：执行 DescribeClusterInstances 确认权限
tccli tke DescribeClusterInstances --region <Region> \
    --ClusterId <ClusterId> \
    --InstanceRole WORKER
# expected: exit 0，返回 InstanceSet 列表（可为空）

# 4. 检查 kubectl（可选，仅数据面验证需要）
kubectl version --client
# expected: Client Version v1.28.x+
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
# 5. 确认目标集群存在且状态为 Running
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["<ClusterId>"]'
# expected: TotalCount >= 1，ClusterStatus "Running"

# 6. 确认集群下有节点
tccli tke DescribeClusterInstances --region <Region> \
    --ClusterId <ClusterId> \
    --InstanceRole WORKER
# expected: 返回 InstanceSet，TotalCount >= 1
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

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `REGION` | 地域 | 如 `ap-guangzhou`、`ap-beijing` | `tccli configure list` |
| `<ClusterId>` | 集群 ID | 格式 `cls-xxxxxxxx` | `tccli tke DescribeClusters --region <Region>` |
| `<NodeName>` | 节点名（kubectl 侧） | kubernetes 节点名，非 CVM ID | `kubectl get nodes` |
| `<InstanceId>` | CVM 实例 ID | 格式 `ins-xxxxxxxx` | `DescribeClusterInstances` 的 `InstanceId` 字段 |

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看节点列表 | `DescribeClusterInstances --ClusterId` | 是 |
| 按角色筛选节点 | `DescribeClusterInstances --ClusterId --InstanceRole WORKER` | 是 |
| 按实例 ID 查询 | `DescribeClusterInstances --ClusterId --InstanceIds` | 是 |
| 查看节点详情（控制面） | `DescribeClusterInstances --ClusterId` 检查 `FailedReason`/`DrainStatus`/`InstanceState` | 是 |
| 查看节点条件（数据面） | `kubectl describe node <NodeName>` | 是 |
| 查看节点封锁状态（数据面） | `kubectl get node <NodeName>` | 是 |

## 操作步骤

### 步骤 1：查询集群所有 WORKER 节点的控制面状态

控制面查询不依赖 kubectl，是 kubectl 不可达时的首选观测手段。

```bash
tccli tke DescribeClusterInstances --region <Region> \
    --ClusterId <ClusterId> \
    --InstanceRole WORKER
# expected: exit 0，返回 InstanceSet 数组
```

**预期输出**（截取关键字段）：

```json
{
    "TotalCount": 3,
    "InstanceSet": [
        {
            "InstanceId": "ins-example-1",
            "InstanceRole": "WORKER",
            "InstanceState": "running",
            "DrainStatus": "",
            "FailedReason": "=Ready:True",
            "InstanceName": "node-example-1",
            "LanIP": "172.24.0.10"
        },
        {
            "InstanceId": "ins-example-2",
            "InstanceRole": "WORKER",
            "InstanceState": "running",
            "DrainStatus": "draining",
            "FailedReason": "",
            "InstanceName": "node-example-2",
            "LanIP": "172.24.0.11"
        },
        {
            "InstanceId": "ins-example-3",
            "InstanceRole": "WORKER",
            "InstanceState": "running",
            "DrainStatus": "",
            "FailedReason": "KubeletNotReady: PLEG is not healthy",
            "InstanceName": "node-example-3",
            "LanIP": "172.24.0.12"
        }
    ]
}
```

字段解读：

| 字段 | 健康值 | 异常值含义 |
|------|--------|-----------|
| `InstanceId` | `ins-xxxxxxxx` | CVM 实例 ID，用于跨服务关联 |
| `InstanceState` | `running` | `initializing`（初始化中）、`terminating`（移出中）、`stopped`（关机） |
| `DrainStatus` | `""`（空） | `draining`（驱逐中）、`drained`（驱逐完成） |
| `FailedReason` | 空 或 `=Ready:True` | 非空字符串表示节点异常原因（见排障） |

状态判定逻辑：

- `ins-example-1`：**健康** — `InstanceState: running`，`DrainStatus` 空，`FailedReason` 含 `Ready:True`。
- `ins-example-2`：**驱逐中** — `DrainStatus: draining`。
- `ins-example-3`：**异常** — `FailedReason` 含 `KubeletNotReady`。

### 步骤 2：筛选非健康节点

```bash
tccli tke DescribeClusterInstances --region <Region> \
    --ClusterId <ClusterId> \
    --InstanceRole WORKER
# expected: exit 0，从返回结果中筛选 FailedReason 非空且不含 Ready:True 的节点
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

在返回的 `InstanceSet` 中查找 `FailedReason` 字段非空且不含 `=Ready:True` 的条目，即异常节点。如需按实例 ID 精确查询：

```bash
tccli tke DescribeClusterInstances --region <Region> \
    --ClusterId <ClusterId> \
    --InstanceIds '["<InstanceId>"]'
# expected: exit 0，返回单个实例的状态详情
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

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<InstanceId>` | CVM 实例 ID | 格式 `ins-xxxxxxxx` | `DescribeClusterInstances` 的 `InstanceId` 字段 |

### 步骤 3：数据面查看节点条件（须 VPN/IOA）

在可访问 APIServer 的环境中，kubectl 提供更细粒度的节点状态信息：

```bash
kubectl get nodes -o wide
# expected: exit 0，列出所有节点 STATUS、INTERNAL-IP
```

**预期输出**：

```text
NAME             STATUS   ROLES    AGE   VERSION   INTERNAL-IP
node-example-1   Ready    <none>   2d    v1.32.2   172.24.0.10
node-example-2   Ready,SchedulingDisabled   <none>   2d    v1.32.2   172.24.0.11
node-example-3   NotReady <none>   1d    v1.32.2   172.24.0.12
```

查看节点详细条件：

```bash
kubectl describe node <NodeName>
# expected: exit 0，输出 Conditions 段含 Ready/MemoryPressure/DiskPressure/PIDPressure
```

**预期输出**（关键字段）：

```text
Conditions:
  Type             Status  LastHeartbeatTime   LastTransitionTime  Reason
  Ready            True    recent              recent              KubeletReady
  MemoryPressure   False   recent              recent              KubeletHasSufficientMemory
  DiskPressure     False   recent              recent              KubeletHasNoDiskPressure
  PIDPressure      False   recent              recent              KubeletHasSufficientPID
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<NodeName>` | kubernetes 节点名 | 非 CVM ID | `kubectl get nodes` |

检查封锁状态：

```bash
kubectl get node <NodeName> -o custom-columns=NAME:.metadata.name,SCHEDULABLE:.spec.unschedulable
# expected: SCHEDULABLE 为 true 表示已封锁（unschedulable=true）
```

**预期输出**：

```text
NAME             SCHEDULABLE
node-example-2   true
```

> `SCHEDULABLE: true` 表示 `spec.unschedulable` 字段为 true，即节点已被封锁。kubectl `STATUS` 列会显示 `SchedulingDisabled`。

### 步骤 4：关联 CVM 实例状态

普通节点底层是 CVM 实例，CVM 状态会影响节点状态。通过 CVM API 查询底层实例：

```bash
tccli cvm DescribeInstances --region <Region> \
    --InstanceIds '["<InstanceId>"]'
# expected: exit 0，返回 CVM 实例状态
```

**预期输出**（截取关键字段）：

```json
{
    "TotalCount": 1,
    "InstanceSet": [
        {
            "InstanceId": "ins-example-1",
            "InstanceName": "node-example-1",
            "InstanceState": "RUNNING"
        }
    ]
}
```

> **生命周期关联**：CVM `InstanceState: STOPPED` 对应 TKE 节点 `NotReady`；CVM `InstanceState: TERMINATING` 对应 TKE 节点 `terminating`。更多 CVM 状态见 [云服务器生命周期](https://cloud.tencent.com/document/product/213/4856)。

## 验证

### 控制面（tccli）

```bash
tccli tke DescribeClusterInstances --region <Region> \
    --ClusterId <ClusterId> \
    --InstanceRole WORKER
# expected: exit 0，返回 InstanceSet，InstanceState/FailedReason/DrainStatus 字段可读
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

验证维度：

| 维度 | 命令 | 预期 |
|------|------|------|
| 状态 | `DescribeClusterInstances` 检查 `InstanceState` | `running` 表示 CVM 运行中 |
| 健康 | 检查 `FailedReason` | 空 或 `=Ready:True` 表示健康 |
| 驱逐 | 检查 `DrainStatus` | `""` 空表示无驱逐，`draining`/`drained` 表示驱逐中/完成 |
| 关联 CVM | `cvm DescribeInstances --InstanceIds` | `InstanceState` 为 `RUNNING` 时节点才有 `running` |

### 数据面（须 VPN/IOA）

```bash
kubectl get nodes
# expected: 所有节点 STATUS 列为 Ready 或已知异常状态

kubectl describe node <NodeName>
# expected: Conditions 中 Ready=True 且 MemoryPressure/DiskPressure/PIDPressure=False
```

```text
NAME  STATUS  AGE
...
```

## 清理

只读操作，无资源清理。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DescribeClusterInstances` 返回 `InvalidParameter` | `tccli configure list` 检查 region 与集群所在地域 | `region` 与集群地域不一致，或 `ClusterId` 格式错误 | 用 `tccli tke DescribeClusters --region <Region>` 确认集群 ID 与地域；`--InstanceRole` 仅取 `WORKER`/`MASTER_ETCD` |
| `DescribeClusterInstances` 返回 `UnauthorizedOperation` | `tccli configure list` 检查当前凭据 | 子账号缺少 `tke:DescribeClusterInstances` 权限（此为环境限制，非命令错误） | 联系主账号授予 `QcloudTKEFullAccess` 或最小权限 `tke:DescribeClusterInstances` |
| `kubectl get nodes` 返回 `Unable to connect to the server` | `kubectl cluster-info` 检查 APIServer 可达性 | 公网端点受 CAM 策略拒绝（如 strategyId:240463971）或内网端点未连通 | 通过 VPN/IOA 连接内网端点；或回退使用 `DescribeClusterInstances` 控制面观测 |
| `kubectl describe node` 返回 `NotFound` | `kubectl get nodes` 确认节点名 | `<NodeName>` 拼写错误或节点已移出 | 从 `kubectl get nodes` 输出复制节点名；已移出的节点用 `DescribeClusterInstances` 查询历史状态 |

### 查询成功但结果异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `FailedReason: KubeletNotReady` | `kubectl describe node <NodeName>` 查看 `Conditions`；SSH 登录 CVM 执行 `systemctl status kubelet` | kubelet 进程异常或资源耗尽 | `systemctl restart kubelet`；若持续异常，检查 `journalctl -u kubelet` 日志 |
| `FailedReason: PLEG is not healthy` | SSH 登录 CVM 执行 `systemctl status containerd` 和 `crictl ps` | 容器运行时（containerd）异常导致 PLEG 卡死 | `systemctl restart containerd`；检查磁盘 IO 是否异常 |
| `FailedReason: NodeHasDiskPressure` | SSH 登录 CVM 执行 `df -h` 查看磁盘使用率 | 节点磁盘空间不足 | 清理日志/镜像：`crictl rmi --prune`、`journalctl --vacuum-size=100M`；必要时扩容磁盘 |
| `FailedReason: NodeHasInsufficientMemory` | `kubectl describe node <NodeName>` 查看 `Allocatable` 与已分配量 | 节点内存不足，Pod 调度超卖 | 迁移部分 Pod 到其他节点，或封锁节点后扩容 CVM |
| `DrainStatus: draining` 长时间不变成 `drained` | `kubectl get pods -A --field-selector spec.nodeName=<NodeName>` 查看残留 Pod | PodDisruptionBudget 阻止驱逐，或 DaemonSet Pod 无法迁移 | 见 [驱逐或封锁节点](../驱逐或封锁节点/tccli%20操作.md) 排障节 |
| `InstanceState: stopped` 但节点显示 NotReady | `tccli cvm DescribeInstances --region <Region> --InstanceIds '["<InstanceId>"]'` 查 CVM 状态 | CVM 被手动关机或欠费停机 | 启动 CVM：`tccli cvm StartInstances --InstanceIds '["<InstanceId>"]'`；检查欠费状态 |

## 下一步

- [驱逐或封锁节点](https://cloud.tencent.com/document/product/457/32205) — 封锁/驱逐节点的操作步骤
- [移出节点](https://cloud.tencent.com/document/product/457/32208) — 将节点移出集群
- [新增节点](https://cloud.tencent.com/document/product/457/32200) — 向集群添加工作节点
- [节点资源预留说明](https://cloud.tencent.com/document/product/457/76057) — 节点可分配资源计算公式
- [云服务器生命周期](https://cloud.tencent.com/document/product/213/4856) — CVM 实例生命周期与 TKE 节点的关联

## 控制台替代

[容器服务控制台 → 节点管理](https://console.cloud.tencent.com/tke2/cluster?tab=instance)
