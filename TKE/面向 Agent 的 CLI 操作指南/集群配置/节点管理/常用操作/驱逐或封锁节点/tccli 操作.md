# 驱逐或封锁节点（tccli）

> 对照官方：[驱逐或封锁节点](https://cloud.tencent.com/document/product/457/32205) · page_id `32205` · tccli ≥1.0.0 · API 2018-05-25

## 概述

**封锁**（Cordon）使节点不再接受新 Pod 调度，但已有 Pod 继续运行。**驱逐**（Drain）将节点上运行的 Pod 安全迁移到其他节点，常用于节点维护、缩容或故障隔离。驱逐前会自动执行封锁。

本页是 hybrid 页面：封锁/驱逐通过 `kubectl`（数据面）执行，驱逐状态通过 `tccli DescribeClusterInstances`（控制面）观测。`kubectl cordon` 执行后，控制面 `DescribeClusterInstances` 返回的 `InstanceAdvancedSettings.Unschedulable` 标记为 `1`；`kubectl uncordon` 后恢复为 `0`。

| 操作 | 工具 | 作用 | 是否可逆 |
|------|------|------|:--------:|
| 封锁（cordon） | kubectl | 标记节点不可调度，已有 Pod 不动 | 是（uncordon） |
| 取消封锁（uncordon） | kubectl | 恢复节点可调度 | 是 |
| 驱逐（drain） | kubectl | 封锁 + 逐个 evict Pod 到其他节点 | 部分（Pod 已迁移，不可原地回滚） |
| 查看驱逐状态 | tccli | `DescribeClusterInstances` 的 `DrainStatus` 字段 | 只读 |

> **kubectl 数据面可达性**：`kubectl cordon` 和 `kubectl drain` 需连接集群 APIServer。公网端点可能受组织级 CAM 策略限制（strategyId:240463971 以 `tke:clusterExtranetEndpoint=true` 条件硬拒绝 `CreateClusterEndpoint`），内网端点需 VPN/IOA/专线或同 VPC CVM 才能连通。若 kubectl 不可达，控制台仍可执行封锁/驱逐，控制面通过 `DescribeClusterInstances` 的 `DrainStatus` 字段可观测状态。

## 前置条件

- [环境准备](../../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0
```

```text
tccli version 1.0.0
```

```bash
# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId、secretKey、region 均已配置，region 为目标地域
```

```text
secretId:  AKID********************************
secretKey: ********************************
region:    ap-guangzhou
```

```bash
# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:DescribeClusters, tke:DescribeClusterInstances, tke:DescribeClusterKubeconfig, tke:DescribeClusterStatus
# 验证：执行 DescribeClusterInstances 确认权限
tccli tke DescribeClusterInstances --region <Region> \
    --ClusterId <ClusterId> \
    --InstanceRole WORKER
# expected: exit 0，返回 InstanceSet 列表（可为空）
```

```json
{
    "TotalCount": 2,
    "InstanceSet": [
        {
            "InstanceId": "ins-example-1",
            "InstanceRole": "WORKER",
            "InstanceState": "running"
        }
    ]
}
```

```bash
# 4. 检查 kubectl（须 VPN/IOA，数据面操作必需）
kubectl version --client
# expected: Client Version v1.28.x+
```

```text
Client Version: v1.30.0
Kustomize Version: v5.0.4-0.20230601165947-6ce0bf390ce3
```

### 资源检查

```bash
# 5. 确认目标集群状态为 Running
tccli tke DescribeClusterStatus --region <Region> \
    --ClusterIds '["<ClusterId>"]'
# expected: ClusterState "Running"
```

**预期输出**：

```json
{
    "ClusterStatusSet": [
        {
            "ClusterId": "cls-example",
            "ClusterState": "Running",
            "ClusterInstanceState": "PartialAbnormal",
            "ClusterInitNodeNum": 3,
            "ClusterRunningNodeNum": 9,
            "ClusterFailedNodeNum": 0,
            "ClusterDeletionProtection": false
        }
    ],
    "TotalCount": 1
}
```

```bash
# 6. 检查集群端点（确认 kubectl 连接路径）
tccli tke DescribeClusterEndpoints --region <Region> \
    --ClusterId <ClusterId>
# expected: 返回 ClusterIntranetEndpoint（内网端点地址）
```

**预期输出**：

```json
{
    "ClusterExternalEndpoint": "",
    "ClusterIntranetEndpoint": "172.24.0.12",
    "ClusterDomain": "cls-example.ccs.tencent-cloud.com",
    "ClusterExternalDomain": "cls-example.ccs.tencent-cloud.com",
    "ClusterIntranetSubnetId": "subnet-example"
}
```

```bash
# 7. 确认集群下有 WORKER 节点
tccli tke DescribeClusterInstances --region <Region> \
    --ClusterId <ClusterId> \
    --InstanceRole WORKER
# expected: TotalCount >= 1，至少 1 个 InstanceState "running"
```

**预期输出**：

```json
{
    "TotalCount": 2,
    "InstanceSet": [
        {
            "InstanceId": "ins-example-1",
            "InstanceRole": "WORKER",
            "FailedReason": "=Ready:True",
            "InstanceState": "running",
            "DrainStatus": "",
            "InstanceAdvancedSettings": {
                "GPUArgs": {
                    "CUDA": {"Name": "", "Version": ""},
                    "CUDNN": {"Name": "", "Version": ""},
                    "CustomDriver": {"Address": ""},
                    "Driver": {"Name": "", "Version": ""},
                    "MIGEnable": false
                },
                "Taints": null,
                "Unschedulable": 0,
                "Labels": []
            },
            "CreatedTime": "2026-06-23T08:13:42Z",
            "LanIP": "172.24.0.34",
            "NodePoolId": "np-example",
            "AutoscalingGroupId": "asg-example"
        },
        {
            "InstanceId": "ins-example-2",
            "InstanceRole": "WORKER",
            "FailedReason": "=Ready:True",
            "InstanceState": "running",
            "DrainStatus": "",
            "InstanceAdvancedSettings": {
                "GPUArgs": null,
                "Taints": null,
                "Unschedulable": 0,
                "Labels": []
            },
            "CreatedTime": "2026-06-23T08:51:57Z",
            "LanIP": "172.24.0.143",
            "NodePoolId": "",
            "AutoscalingGroupId": ""
        }
    ]
}
```

```bash
# 8. 获取 kubeconfig 并验证 kubectl 可达（须 VPN/IOA）
tccli tke DescribeClusterKubeconfig --region <Region> \
    --ClusterId <ClusterId> \
    --IsExtranet false
# expected: Kubeconfig 字段非空且包含完整的 YAML 内容
```

**预期输出**：

```json
{
    "Kubeconfig": "apiVersion: v1\nclusters:\n- cluster:\n    certificate-authority-data: LS0t...（截断）\n    server: https://172.24.0.12\n  name: cls-example\ncontexts:\n- context:\n    cluster: cls-example\n    user: \"100049208872\"\n  name: cls-example-100049208872-context-default\ncurrent-context: cls-example-100049208872-context-default\nkind: Config\npreferences: {}\nusers:\n- name: \"100049208872\"\n  user:\n    client-certificate-data: LS0t...（截断）\n    client-key-data: LS0t...（截断）\n",
    "RequestId": "52adf34a-bb45-4095-8bcb-a568c0123536"
}
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<Region>` | 地域 | 如 `ap-guangzhou`、`ap-beijing` | `tccli configure list` |
| `<ClusterId>` | 集群 ID | 格式 `cls-xxxxxxxx` | `tccli tke DescribeClusters --region <Region>` |
| `<InstanceId>` | CVM 实例 ID | 格式 `ins-xxxxxxxx` | `DescribeClusterInstances` 的 `InstanceId` 字段 |
| `NODE_NAME` | 节点名 | kubernetes 节点名，非 CVM ID | `kubectl get nodes` |

> `NODE_NAME` 是 kubectl 侧占位符（非 tccli 标准占位符），用于 `kubectl cordon`/`kubectl drain` 等数据面命令，通过 `kubectl get nodes -o wide` 的 `INTERNAL-IP` 列与 `DescribeClusterInstances` 的 `LanIP` 字段交叉对应。管控面统一使用 `<InstanceId>`。

## 控制台与 CLI 参数映射

| 控制台操作 | kubectl / tccli | 幂等 |
|------------|-----------------|:--:|
| 封锁节点 | `kubectl cordon NODE_NAME` | 是 |
| 取消封锁 | `kubectl uncordon NODE_NAME` | 是 |
| 驱逐节点 | `kubectl drain NODE_NAME --ignore-daemonsets --delete-emptydir-data` | 否 |
| 查看封锁状态（数据面） | `kubectl get node NODE_NAME` | 是 |
| 查看驱逐状态（控制面） | `DescribeClusterInstances` 检查 `DrainStatus` | 是 |
| 查询指定实例（控制面） | `DescribeClusterInstances --InstanceIds` | 是 |

> kubectl 使用 kubernetes 节点名（`NODE_NAME`），tccli 使用 CVM 实例 ID（`<InstanceId>`）。两者通过 `kubectl get node -o wide` 的 `INTERNAL-IP` 列与 `DescribeClusterInstances` 的 `LanIP` 字段交叉对应。常见错误：直接对 `ins-xxx` 格式的实例 ID 执行 `kubectl cordon` —— 需先用 `kubectl get nodes` 获取节点名。
>
> **节点状态映射**：`kubectl cordon` → `DescribeClusterInstances` 返回 `InstanceAdvancedSettings.Unschedulable = 1`；`kubectl uncordon` → `Unschedulable = 0`。`DrainStatus` 字段含义：空字符串=无驱逐操作，`draining`=驱逐进行中，`drained`=驱逐已完成。

## 操作步骤

### 操作场景

在以下场景需要封锁或驱逐节点：

- **节点维护**：系统升级、内核参数调整、安全补丁——先封锁再维护，维护后取消封锁。
- **集群缩容**：释放低负载节点——先驱逐迁移 Pod，再 [移出节点](../移出节点/tccli%20操作.md)。
- **故障隔离**：节点异常时迁移工作负载到健康节点——驱逐后移出。
- **下架节点**：完整流程为封锁 → 驱逐 → [移出节点](../移出节点/tccli%20操作.md)。

---

### 步骤 1：定位目标节点

封锁/驱逐前，先确认目标节点的 kubernetes 名称和当前状态。

```bash
# 数据面：列出节点及调度状态（须 VPN/IOA）
kubectl get nodes -o wide
# expected: 返回节点列表，STATUS 列含 Ready，ROLES 列含 worker
```

```text
NAME             STATUS   ROLES    AGE   VERSION    INTERNAL-IP    OS-IMAGE
node-example-1   Ready    worker   12d   v1.30.0    172.24.0.10    TencentOS Server 3.2
node-example-2   Ready    worker   12d   v1.30.0    172.24.0.11    TencentOS Server 3.2
```

控制面定位（kubectl 不可达时）：

```bash
tccli tke DescribeClusterInstances --region <Region> \
    --ClusterId <ClusterId> \
    --InstanceRole WORKER
# expected: exit 0，返回 InstanceSet，记录目标 InstanceId 和 LanIP
```

```json
{
    "TotalCount": 2,
    "InstanceSet": [
        {
            "InstanceId": "ins-example-1",
            "InstanceRole": "WORKER",
            "FailedReason": "=Ready:True",
            "InstanceState": "running",
            "DrainStatus": "",
            "InstanceAdvancedSettings": {
                "GPUArgs": {
                    "CUDA": {"Name": "", "Version": ""},
                    "CUDNN": {"Name": "", "Version": ""},
                    "CustomDriver": {"Address": ""},
                    "Driver": {"Name": "", "Version": ""},
                    "MIGEnable": false
                },
                "Taints": null,
                "Unschedulable": 0,
                "Labels": []
            },
            "CreatedTime": "2026-06-23T08:13:42Z",
            "LanIP": "172.24.0.34",
            "NodePoolId": "np-example",
            "AutoscalingGroupId": "asg-example"
        },
        {
            "InstanceId": "ins-example-2",
            "InstanceRole": "WORKER",
            "FailedReason": "=Ready:True",
            "InstanceState": "running",
            "DrainStatus": "",
            "InstanceAdvancedSettings": {
                "GPUArgs": null,
                "Taints": null,
                "Unschedulable": 0,
                "Labels": []
            },
            "CreatedTime": "2026-06-23T08:51:57Z",
            "LanIP": "172.24.0.143",
            "NodePoolId": "",
            "AutoscalingGroupId": ""
        }
    ]
}
```

选择 `InstanceState` 为 `running` 且 `InstanceAdvancedSettings.Unschedulable` 为 `0` 的节点作为操作目标。`Unschedulable=0` 表示节点当前可调度，可以执行封锁/驱逐。

---

### 步骤 2：封锁节点（须 VPN/IOA）

> **影响范围**：封锁操作标记节点不可调度，已有 Pod 继续运行不受影响。但新 Pod 无法调度到该节点——若该节点是某关键组件（如 Ingress Controller、CoreDNS）的唯一副本，封锁可能导致相关服务的扩容请求失败。封锁**可逆**（`uncordon` 恢复），不影响 Pod 数据和节点生命周期。

封锁后，调度器不再将新 Pod 调度到该节点，已有 Pod 不受影响。

```bash
kubectl cordon NODE_NAME
# expected: exit 0，输出 "node/NODE_NAME cordoned"
```

```text
node/node-example-1 cordoned
```

验证封锁状态：

```bash
kubectl get node NODE_NAME
# expected: STATUS 列显示 Ready,SchedulingDisabled
```

```text
NAME             STATUS                     ROLES    AGE   VERSION
node-example-1   Ready,SchedulingDisabled   worker   12d   v1.30.0
```

---

### 步骤 3：取消封锁节点（须 VPN/IOA）

恢复节点调度能力（维护完成后执行）：

```bash
kubectl uncordon NODE_NAME
# expected: exit 0，输出 "node/NODE_NAME uncordoned"
```

```text
node/node-example-1 uncordoned
```

```bash
kubectl get node NODE_NAME
# expected: STATUS 列恢复为 Ready（无 SchedulingDisabled）
```

```text
NAME             STATUS   ROLES    AGE   VERSION
node-example-1   Ready    worker   12d   v1.30.0
```

---

### 步骤 4：驱逐节点（须 VPN/IOA）

驱逐将节点上的 Pod 安全迁移到其他节点。`kubectl drain` 会自动先执行 cordon（封锁），再逐个 evict Pod。驱逐完成后节点处于封锁状态。

> **影响范围**：驱逐操作将节点上所有 Pod（DaemonSet 除外）安全迁移到其他节点。迁移过程中 Pod 逐个被 evict 并在其他节点重建——每个 Pod 有短暂不可用窗口（约等于 优雅终止时间 + 调度 + 启动时间），直到新 Pod Ready。被驱逐的 Pod 不可原地回滚。集群空余资源不足时驱逐可能失败（Pod Pending），需先扩容再驱逐。
>
> **不可逆操作**：drain 是一个破坏性流程——Pod 已在其他节点重建，原位回滚需要反向 drain 其他节点或手动迁移。`--disable-eviction` + `--force` 组合使用时风险叠加：绕过 PDB 保护且强制删除裸 Pod，可能导致服务可用性大幅下降。

#### 选择依据

*kubectl drain 是数据面操作，无对应 tccli API。`DrainClusterVirtualNode` / `DrainExternalNode` 仅适用于虚拟节点和注册节点，不适用于普通 WORKER 节点。tccli `DescribeClusterInstances` 提供控制面观测能力，通过 `DrainStatus` 字段跟踪节点状态。*

**各参数选择理由**：

- **`--ignore-daemonsets`**（推荐始终使用）：DaemonSet Pod 绑定到特定节点，不会迁移到其他节点。跳过避免 drain 卡住。跳过后果：DaemonSet Pod 留在原节点，随节点移出自动清理。
- **`--delete-emptydir-data`**（推荐使用）：emptyDir 卷数据随 Pod 删除即丢失。确认节点上无依赖 emptyDir 的关键数据后使用。跳过后果：有空 emptyDir 的 Pod 将无法驱逐，drain 卡住。
- **`--force`**（按需使用）：仅当存在裸 Pod（无 ReplicaSet/Deployment/DaemonSet/StatefulSet 管理的 Pod）时使用。风险：强制删除裸 Pod 导致数据丢失。
- **`--disable-eviction`**（按需使用）：用 `delete` 替代 `evict`，绕过 PodDisruptionBudget。仅当 PDB 阻止驱逐且确认可接受中断时使用。风险：可能违反 PDB 约定的可用性要求。
- **`--grace-period`**（可选）：控制 Pod 优雅终止时间（秒），默认 30 秒。设为 0 会强制立即 kill，可能导致数据丢失。
- **`--timeout`**（可选）：drain 超时时间。若 Pod 无法在超时前驱逐完毕则报错退出。不指定则无默认超时（会一直等待）。

#### 最小驱逐（推荐，适用于大多数场景）

```bash
kubectl drain NODE_NAME --ignore-daemonsets --delete-emptydir-data
# expected: exit 0，输出 "node/NODE_NAME drained"
```

```text
node/node-example-1 cordoned
evicting pod default/nginx-deployment-7d8b6c8f5-x8k2l
evicting pod default/redis-6d4b7c8f5-9m2r5
pod/nginx-deployment-7d8b6c8f5-x8k2l evicted
pod/redis-6d4b7c8f5-9m2r5 evicted
node/node-example-1 drained
```

#### 增强驱逐（含超时和强制选项）

```bash
kubectl drain NODE_NAME \
    --ignore-daemonsets \
    --delete-emptydir-data \
    --force \
    --disable-eviction \
    --grace-period=30 \
    --timeout=300s
# expected: exit 0，输出 "node/NODE_NAME drained"
```

**关键字段说明**（聚焦有副作用的参数）：

| 字段名 | 类型 | 必填 | 取值与约束 | 错误后果 |
|--------|------|:----:|-----------|---------|
| `--ignore-daemonsets` | flag | 是 | — | 不传导致 drain 卡死在 DaemonSet Pod 上（驱逐 DaemonSet Pod 后控制器立即重建，无限循环） |
| `--delete-emptydir-data` | flag | 推荐 | 确认 emptyDir 数据可丢弃后使用 | 不传且存在空 emptyDir 的 Pod 时 drain 卡住；传了则 emptyDir 数据永久丢失 |
| `--force` | flag | 按需 | 仅存在裸 Pod（无控制器管理）时使用 | 强制删除裸 Pod 导致数据不可恢复、无自动重建 |
| `--disable-eviction` | flag | 按需 | 仅 PDB 阻止且确认可接受中断时使用 | 绕过 PDB 保护可能导致服务可用性降至不可接受水平 |
| `--grace-period` | int | 可选 | 秒，默认 30。设为 0 则立即 kill | 非零值过小：容器未完成清理 → 数据损坏或锁残留；设为 0：强制 kill，数据丢失风险极高 |
| `--timeout` | duration | 可选 | 如 `300s`、`5m`。不设则无超时，一直等待 | 不设超时：卡住时命令永不返回，阻塞自动化脚本 |

**速查参数表**（简化版，仅保留作用和使用场景）：

| 参数 | 作用 | 何时使用 |
|------|------|----------|
| `--ignore-daemonsets` | 跳过 DaemonSet Pod（不迁移） | 几乎总是需要 |
| `--delete-emptydir-data` | 删除使用 emptyDir 卷的 Pod | 确认 emptyDir 数据可丢弃时 |
| `--force` | 强制删除不受控制器管理的 Pod（裸 Pod） | 有裸 Pod 时 |
| `--disable-eviction` | 用 `delete` 替代 `evict`，绕过 PDB | PDB 阻止且确认可接受中断时 |
| `--grace-period=30` | 优雅终止时间（秒） | 需控制 Pod 清理时间 |
| `--timeout=300s` | drain 超时时间 | 需限制等待时间的场景 |

---

### 步骤 5：控制面查看驱逐状态

不依赖 kubectl 时，通过 tccli 查看 `DrainStatus` 和 `FailedReason` 字段：

```bash
tccli tke DescribeClusterInstances --region <Region> \
    --ClusterId <ClusterId> \
    --InstanceIds '["<InstanceId>"]'
# expected: exit 0，返回 DrainStatus 和 FailedReason 字段
```

```json
{
    "TotalCount": 1,
    "InstanceSet": [
        {
            "InstanceId": "ins-example-1",
            "FailedReason": "=Ready:True",
            "InstanceState": "running",
            "DrainStatus": "",
            "InstanceAdvancedSettings": {
                "DesiredPodNumber": 64,
                "GPUArgs": {
                    "CUDA": {"Name": "", "Version": ""},
                    "CUDNN": {"Name": "", "Version": ""},
                    "CustomDriver": {"Address": ""},
                    "Driver": {"Name": "", "Version": ""},
                    "MIGEnable": false
                },
                "Taints": null,
                "Unschedulable": 0,
                "Labels": []
            },
            "CreatedTime": "2026-06-23T08:13:42Z",
            "LanIP": "172.24.0.34",
            "NodePoolId": "np-example",
            "AutoscalingGroupId": "asg-example"
        }
    ]
}
```

`DrainStatus` 可能值：

| 值 | 含义 | 下一步 |
|----|------|--------|
| `""`（空） | 无驱逐操作 | 节点正常调度 |
| `draining` | 驱逐进行中 | 等待 Pod 迁移完成 |
| `drained` | 驱逐已完成 | 可安全移出节点 |

## 验证

### 控制面（tccli）

```bash
# 1. 确认驱逐状态
tccli tke DescribeClusterInstances --region <Region> \
    --ClusterId <ClusterId> \
    --InstanceIds '["<InstanceId>"]'
# expected: DrainStatus "drained"（驱逐完成）或 "" （仅封锁，未驱逐）
```

```json
{
    "TotalCount": 1,
    "InstanceSet": [
        {
            "InstanceId": "ins-example-1",
            "FailedReason": "",
            "InstanceState": "running",
            "DrainStatus": "drained",
            "InstanceAdvancedSettings": {
                "DesiredPodNumber": 64,
                "GPUArgs": null,
                "Taints": null,
                "Unschedulable": 1,
                "Labels": []
            },
            "CreatedTime": "2026-06-23T08:51:57Z",
            "LanIP": "172.24.0.143",
            "NodePoolId": "",
            "AutoscalingGroupId": ""
        }
    ]
}
```

验证维度：

| 维度 | 命令 | 预期 | kubectl 不可达时的替代检查 |
|------|------|------|---------------------------|
| 封锁状态 | `DescribeClusterInstances --InstanceIds` | `InstanceAdvancedSettings.Unschedulable: 1`（已封锁）/ `0`（未封锁） | 同左（纯控制面字段，不依赖 kubectl） |
| 驱逐状态 | 同上，检查 `DrainStatus` | `drained`（驱逐完成）/ `""`（未驱逐） | 同左 |
| 失败原因 | 同上，检查 `FailedReason` | 正常为空字符串或 `=Ready:True` | 同左 |

> **纯控制面验证路径**：当 kubectl 不可达时（公网端点 CAM 拒绝 + 内网端点不可达），通过 `DescribeClusterInstances` 的以下三个字段即可完成 cordon/drain/uncordon 操作的全量验证：
> - `InstanceAdvancedSettings.Unschedulable`：`1`=已封锁，`0`=可调度。cordon 后变为 `1`，uncordon 后恢复为 `0`。
> - `DrainStatus`：空=无驱逐，`draining`=驱逐进行中，`drained`=驱逐已完成。
> - `FailedReason`：正常为 `=Ready:True` 或空字符串。非空（如 `Ready:False`）表示节点异常，需排查。

### 数据面（须 VPN/IOA）

```bash
# 2. 确认封锁状态（仅封锁时）
kubectl get node NODE_NAME
# expected: STATUS 列显示 Ready,SchedulingDisabled
```

```text
NAME             STATUS                     ROLES    AGE   VERSION
node-example-1   Ready,SchedulingDisabled   worker   12d   v1.30.0
```

```bash
# 3. 确认驱逐完成（驱逐后）
kubectl get node NODE_NAME
# expected: STATUS 列显示 Ready,SchedulingDisabled
```

```text
NAME             STATUS                     ROLES    AGE   VERSION
node-example-1   Ready,SchedulingDisabled   worker   12d   v1.30.0
```

```bash
# 4. 确认节点上仅剩 DaemonSet Pod
kubectl get pods -A -o wide --field-selector spec.nodeName=NODE_NAME
# expected: 仅剩 kube-proxy、kube-flannel 等 DaemonSet Pod，业务 Pod 已迁移
```

```text
NAMESPACE     NAME                             READY   STATUS    NODE
kube-system   kube-proxy-abc123                1/1     Running   node-example-1
kube-system   kube-flannel-ds-xyz789           1/1     Running   node-example-1
```

## 清理

> **注意**：封锁（cordon）和驱逐（drain）不删除 CVM 实例，不产生计费变化。cordon 操作可逆（uncordon 即可恢复），drain 后 Pod 已迁移不可原地回滚。drain 后如需从集群删除节点，使用 `tccli tke DeleteClusterInstances` 或控制台移出节点。

### 数据面（须 VPN/IOA，在控制面之前）

封锁后恢复（如维护完成，不需移出节点）：

```bash
kubectl uncordon NODE_NAME
# expected: exit 0，输出 "node/NODE_NAME uncordoned"
```

驱逐后进一步操作：见 [移出节点](../移出节点/tccli%20操作.md) 从集群删除节点。

### 控制面

无控制面清理操作。确认清理后状态：

```bash
# 确认节点已恢复可调度（uncordon 后）
tccli tke DescribeClusterInstances --region <Region> \
    --ClusterId <ClusterId> \
    --InstanceIds '["<InstanceId>"]'
# expected: Unschedulable: 0
```

```json
{
    "TotalCount": 1,
    "InstanceSet": [
        {
            "InstanceId": "ins-example-1",
            "FailedReason": "=Ready:True",
            "InstanceState": "running",
            "DrainStatus": "",
            "InstanceAdvancedSettings": {
                "DesiredPodNumber": 64,
                "Taints": null,
                "Unschedulable": 0,
                "Labels": []
            },
            "LanIP": "172.24.0.34"
        }
    ]
}
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl cordon` 报 `The connection to the server was refused` | `tccli tke DescribeClusterEndpoints --region <Region> --ClusterId <ClusterId>` 检查 ClusterExternalEndpoint 是否为空；`tccli tke DescribeClusterKubeconfig --region <Region> --ClusterId <ClusterId> --IsExtranet false` 检查内网 kubeconfig | 公网端点被组织级 CAM 策略拒绝（strategyId:240463971，条件 `tke:clusterExtranetEndpoint=true`），集群仅内网端点可用 | 通过 VPN/IOA/专线连接内网端点；或在同 VPC CVM 上执行 kubectl；或使用控制台执行封锁。控制面用 `DescribeClusterInstances` 的 `Unschedulable` 字段确认节点封锁状态 |
| `kubectl cordon` 报 `error: node "ins-xxx" not found` | `kubectl get nodes` 查看实际节点名列表 | 误用了 CVM InstanceId（`ins-xxx`）而非 kubernetes 节点名 | 先执行 `kubectl get nodes` 获取 kubernetes 节点名（如 `172.24.0.34`），或通过 `DescribeClusterInstances` 的 `LanIP` 字段与 `kubectl get nodes -o wide` 的 `INTERNAL-IP` 列交叉对应 |
| `kubectl drain` 卡住不动超过 5 分钟 | `kubectl get pods -A -o wide --field-selector spec.nodeName=NODE_NAME` 查看未驱逐 Pod；`kubectl get pdb -A` 查看是否有 PDB 阻止 | PodDisruptionBudget 阻止驱逐，minAvailable 不满足 | 调整 PDB（调高 minAvailable 或临时删除）；或加 `--disable-eviction` 绕过 PDB（确认影响面后使用） |
| `kubectl drain` 报 `cannot delete Pods not managed by ReplicationController` | `kubectl get pods -A -o wide --field-selector spec.nodeName=NODE_NAME` 查找裸 Pod | 存在裸 Pod（无 Deployment/StatefulSet/DaemonSet 管理的 Pod） | 加 `--force` 强制删除裸 Pod（注意：裸 Pod 数据将丢失） |
| `kubectl drain` 报 `error: unable to drain node due to error: deletion grace period` | `kubectl get pods -A -o wide --field-selector spec.nodeName=NODE_NAME` 查看 Pod 状态 | Pod 优雅终止超时，容器未响应 SIGTERM | 加 `--grace-period=0` 强制立即终止（会丢数据，谨慎使用） |
| `kubectl drain` 卡在 DaemonSet Pod 不退出 | 确认命令是否含 `--ignore-daemonsets` | 未加 `--ignore-daemonsets` 参数，drain 尝试驱逐 DaemonSet Pod 但 DaemonSet 控制器立即重建 | 添加 `--ignore-daemonsets` 参数，DaemonSet Pod 留在原节点，随节点移出自动清理 |
| kubectl 完全不可达（公网端点 CAM 拒绝 + 内网不可达） | `tccli tke DescribeClusterEndpoints --region <Region> --ClusterId <ClusterId>` 确认仅内网端点；`tccli tke DescribeClusterKubeconfig --region <Region> --ClusterId <ClusterId> --IsExtranet false` 获取内网 kubeconfig | 公网端点被组织级 CAM 策略 strategyId:240463971 以 `tke:clusterExtranetEndpoint=true` 条件硬拒绝（自建安全组也无法绕过），且当前环境无内网连通路径 | 控制台操作封锁/驱逐；控制面通过 `DescribeClusterInstances` 的 `DrainStatus` 和 `Unschedulable` 字段跟踪状态 |

### 驱逐成功但状态异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DrainStatus` 长时间 `draining` 不变 `drained` | `tccli tke DescribeClusterInstances --region <Region> --ClusterId <ClusterId> --InstanceIds '["<InstanceId>"]'` 检查 `DrainStatus` 和 `FailedReason` | 部分 Pod 驱逐超时或 PDB 持续阻止 | 通过 VPN/IOA 恢复 kubectl 连接后执行 `kubectl get pods -A -o wide --field-selector spec.nodeName=NODE_NAME` 定位卡住的 Pod，调整 PDB 或加 `--disable-eviction --force` 重新 drain |
| 驱逐后业务 Pod 处于 Pending | `kubectl get pods -A -o wide --field-selector status.phase=Pending` 查看 Pending Pod；`kubectl describe nodes` 检查其他节点剩余资源 | 其他节点资源不足或亲和性/Taint 不匹配，Pod 无法调度 | 检查其他节点资源；调整 Pod 资源请求或亲和性；或先扩容再驱逐：向节点池添加新节点后再执行 drain |
| DaemonSet Pod 仍留在节点 | `kubectl get pods -A -o wide --field-selector spec.nodeName=NODE_NAME` 确认是否为 DaemonSet | 正常现象——`--ignore-daemonsets` 跳过 DaemonSet Pod，它们绑定到节点不迁移 | 无需处理，DaemonSet Pod 随节点移出自动清理 |
| 在 drain 过程中直接重启节点 | — | 部分 Pod 未成功迁移即丢失 | 务必等待 `kubectl drain` 成功退出后再维护节点；如已中断，检查残留 Pod 状态手动迁移 |

## 下一步

- [移出节点](../移出节点/tccli%20操作.md) -- page_id `32204`（驱逐后从集群删除节点）
- [节点生命周期](../节点生命周期/tccli%20操作.md) -- page_id `32202`（理解节点状态流转）
- [连接集群](../../../集群管理/连接集群/tccli%20操作.md)（建立 kubectl 连接，数据面操作前置）
- [腾讯云 TKE 节点生命周期](https://cloud.tencent.com/document/product/457/32202)

## 控制台替代

[容器服务控制台 - 节点管理](https://console.cloud.tencent.com/tke2/cluster)
