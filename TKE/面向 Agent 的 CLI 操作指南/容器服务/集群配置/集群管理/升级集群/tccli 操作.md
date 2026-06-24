# 升级集群

> 对照官方：[升级集群](https://cloud.tencent.com/document/product/457/32192) · page_id `32192` · tccli ≥3.1.107 · API 2018-05-25

## 概述

通过 `UpdateClusterVersion` 升级 TKE 集群的 Kubernetes 控制面版本。该 API 仅升级 Master 控制面组件，不升级工作节点（节点升级需单独执行 `UpgradeClusterInstances`）。

升级是**异步、不可逆**操作：调用后立即返回 `RequestId`，集群进入 `Upgrading` 状态，控制面组件实际升级需 10-30 分钟。升级期间控制面 API Server 可能短暂不可达，数据面（运行中的 Pod）不受影响。

| 维度 | 说明 |
|------|------|
| 升级范围 | 仅控制面（Master + Etcd） |
| 目标版本 | 只能选择 `DescribeAvailableClusterVersion` 返回的版本，不可自由指定 |
| 跨版本升级 | 允许跨代升级（如 1.30→1.32），只要目标版本在返回列表中 |
| 可逆性 | 不可逆，不支持降级 |
| 节点升级 | 需单独执行 `UpgradeClusterInstances` |

## 前置条件

- [环境准备](../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置，region 为集群所在地域

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:DescribeClusters, tke:DescribeAvailableClusterVersion
#    tke:DescribeClusterStatus, tke:UpdateClusterVersion
#    tke:DescribeVersions
# 验证：执行 DescribeClusters 确认权限
tccli tke DescribeClusters --region <Region>
# expected: exit 0，返回集群列表
```

```json
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "cls-example",
            "ClusterName": "<ClusterName>",
            "ClusterVersion": "1.30.0",
            "ClusterStatus": "Running"
        }
    ],
    "RequestId": "..."
}
```

### 资源检查

```bash
# 4. 确认集群状态和当前版本
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["<ClusterId>"]'
# expected: ClusterStatus: "Running"，记录当前 ClusterVersion

# 5. 查询该集群可升级的目标版本（关键步骤）
tccli tke DescribeAvailableClusterVersion --region <Region> \
    --ClusterId <ClusterId>
# expected: Versions 非空，列出当前集群允许的升级目标

# 6. 查询全地域可用版本（参考）
tccli tke DescribeVersions --region <Region>
# expected: exit 0，返回地域支持的 K8s 版本列表

# 7. 检查集群状态详情（确认升级前状态）
tccli tke DescribeClusterStatus --region <Region> \
    --ClusterIds '["<ClusterId>"]'
# expected: ClusterState: "Running"
```

**预期输出（步骤 4，升级前）**：

```json
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "cls-example",
            "ClusterName": "<ClusterName>",
            "ClusterDescription": "<ClusterDescription>",
            "ClusterVersion": "1.30.0",
            "ClusterOs": "tlinux3.1x86_64",
            "ClusterType": "MANAGED_CLUSTER",
            "ClusterNetworkSettings": {
                "ClusterCIDR": "10.0.0.0/16",
                "ServiceCIDR": "10.1.0.0/20",
                "VpcId": "<VpcId>",
                "MaxClusterServiceNum": 4096,
                "MaxNodePodNum": 64,
                "Cni": true
            },
            "ClusterStatus": "Running",
            "ClusterLevel": "L5",
            "AutoUpgradeClusterLevel": false,
            "DeletionProtection": false,
            "ContainerRuntime": "containerd",
            "CreatedTime": "..."
        }
    ],
    "RequestId": "..."
}
```

**预期输出（步骤 5，可升级版本）**：

```json
{
    "Versions": [
        "1.32.2"
    ],
    "Clusters": [
        {
            "ClusterId": "cls-example",
            "Versions": [
                "1.32.2"
            ]
        }
    ],
    "RequestId": "..."
}
```

### 升级前检查清单

| 检查项 | 通过标准 | 验证方式 |
|--------|---------|---------|
| 集群状态 | `ClusterStatus: "Running"` | `DescribeClusters` |
| 目标版本可用 | 在 `DescribeAvailableClusterVersion` 返回列表中 | `DescribeAvailableClusterVersion` |
| 节点与 Pod 正常 | 所有节点 `Ready`，业务 Pod 正常运行 | `kubectl get nodes` / `kubectl get pods` |
| ETCD 数据备份 | 已完成 ETCD 快照备份 | `DescribeClusterKubeconfig` → kubectl 访问 → ETCD 快照 |
| API 弃用评估 | 已评估目标版本的 K8s API 弃用影响 | 对照 [Kubernetes API Deprecation Guide](https://kubernetes.io/docs/reference/using-api/deprecation-guide/) |
| 时间窗口 | 业务低峰期，预留 10-30 分钟控制面升级时间 | — |

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看可升级版本 | `DescribeAvailableClusterVersion` | 是 |
| 查询所有可用版本 | `DescribeVersions` | 是 |
| 升级集群 | `UpdateClusterVersion` | 否 |
| 查询升级状态 | `DescribeClusterStatus` | 是 |
| 查看集群详情 | `DescribeClusters` | 是 |

## 关键字段说明

以下说明 `UpdateClusterVersion` 的主要参数。

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `ClusterId` | String | 是 | 格式 `cls-xxxxxxxx`，集群必须为 `Running` 状态 | 不存在 → `InvalidParameter.ClusterId`；非 Running → `ResourceUnavailable.ClusterState` |
| `DstVersion` | String | 是 | 目标 K8s 版本，**必须**在 `DescribeAvailableClusterVersion` 返回列表中。允许跨代升级（如 1.30→1.32） | 版本不在可用列表 → `InvalidParameter` |
| `ExtraArgs` | ClusterExtraArgs | 否 | 自定义控制面组件参数，子字段为 `Array of String`，格式 `["k1=v1","k2=v2"]` | 格式错误 → `InvalidParameter.ExtraArgs` |
| `SkipPreCheck` | Boolean | 否 | 是否跳过预检查，默认 `false` | 设为 `true` 可能导致升级失败后难以定位根因 |
| `MaxNotReadyPercent` | Float | 否 | 升级过程中允许的最大不可用节点比例 | 设置过高可能导致服务不可用未被检测 |

`ExtraArgs` 子字段：

| 子字段 | 说明 |
|--------|------|
| `KubeAPIServer` | kube-apiserver 启动参数 |
| `KubeControllerManager` | kube-controller-manager 启动参数 |
| `KubeScheduler` | kube-scheduler 启动参数 |
| `Etcd` | etcd 启动参数 |

## 操作步骤

### 步骤 1：查询当前版本和可升级版本

```bash
# 当前版本与集群状态
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["<ClusterId>"]'
# expected: ClusterVersion, ClusterStatus: "Running"
```

**预期输出**（升级前，当前版本 1.30.0）：

```json
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "cls-example",
            "ClusterName": "<ClusterName>",
            "ClusterDescription": "<ClusterDescription>",
            "ClusterVersion": "1.30.0",
            "ClusterOs": "tlinux3.1x86_64",
            "ClusterType": "MANAGED_CLUSTER",
            "ClusterStatus": "Running",
            "ClusterLevel": "L5",
            "DeletionProtection": false,
            "ContainerRuntime": "containerd"
        }
    ],
    "RequestId": "..."
}
```

```bash
# 可升级版本（升级路径查询，关键步骤）
tccli tke DescribeAvailableClusterVersion --region <Region> \
    --ClusterId <ClusterId>
# expected: Versions 列表，非空
```

**预期输出**（1.30.0 可升级到 1.32.2，跨两代版本）：

```json
{
    "Versions": [
        "1.32.2"
    ],
    "Clusters": [
        {
            "ClusterId": "cls-example",
            "Versions": [
                "1.32.2"
            ]
        }
    ],
    "RequestId": "..."
}
```

```bash
# 集群状态详情（确认升级前无异常）
tccli tke DescribeClusterStatus --region <Region> \
    --ClusterIds '["<ClusterId>"]'
# expected: ClusterState: "Running"
```

**预期输出**：

```json
{
    "ClusterStatusSet": [
        {
            "ClusterId": "cls-example",
            "ClusterState": "Running",
            "ClusterInstanceState": "-",
            "ClusterBMonitor": false,
            "ClusterInitNodeNum": 0,
            "ClusterRunningNodeNum": 0,
            "ClusterFailedNodeNum": 0,
            "ClusterClosedNodeNum": 0,
            "ClusterClosingNodeNum": 0,
            "ClusterDeletionProtection": false,
            "ClusterAuditEnabled": false
        }
    ],
    "TotalCount": 1,
    "RequestId": "..."
}
```

### 步骤 2：执行升级

#### 选择依据

- **目标版本**：选择 `1.32.2`。该版本是 `DescribeAvailableClusterVersion` 查询到的唯一可升级版本。TKE 托管集群仅支持升级到该 API 返回的版本，不支持自由选择中间版本。当前集群 `1.30.0` 直接升级到 `1.32.2`（跨 1.30→1.32 两代版本），这是 TKE 允许的升级路径。
- **升级策略**：仅升级控制面（Master 组件），不升级工作节点。节点升级需在控制面升级完成后单独执行 `UpgradeClusterInstances`。
- **不可逆性**：升级不可逆，不支持降级。升级前已确认业务兼容性并完成 ETCD 备份。
- **异步行为**：`UpdateClusterVersion` 是异步 API，调用后立即返回 `RequestId`，不返回 `TaskId`。集群进入 `Upgrading` 状态，`Clusters[].ClusterVersion` 字段即刻变为目标版本，但控制面组件实际升级需 10-30 分钟。
- **升级时机**：业务低峰期或维护窗口执行。

> **常见误区**：不要误以为可选任意 K8s 小版本（如 `1.31.x`）。TKE 只开放经过验证的特定版本作为升级目标，必须以 `DescribeAvailableClusterVersion` 返回的列表为准。

#### 最小升级

```bash
tccli tke UpdateClusterVersion --region <Region> \
    --ClusterId <ClusterId> \
    --DstVersion <TargetVersion>
# expected: exit 0，返回 RequestId。异步操作，集群立即进入 Upgrading 状态。
```

**预期输出**（API 接受请求，仅返回 RequestId，无 TaskId）：

```json
{
    "RequestId": "..."
}
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<ClusterId>` | 集群 ID | 格式 `cls-xxxxxxxx`，须为 `Running` 状态 | `tccli tke DescribeClusters --region <Region>` |
| `<TargetVersion>` | 目标 K8s 版本 | 必须在 `DescribeAvailableClusterVersion` 返回列表中 | `tccli tke DescribeAvailableClusterVersion --region <Region> --ClusterId <ClusterId>` |
| `<Region>` | 地域 | 集群所在地域，如 `ap-guangzhou` | `tccli configure list` |

> 参数名是 `--DstVersion`（不是 `--ClusterVersion` 也不是 `--Version`）。返回的 `RequestId` 即表示命令被接受，升级已触发。

#### 增强配置（自定义控制面参数 + 预检查控制）

`upgrade-enhanced.json`：

```json
{
    "ClusterId": "<ClusterId>",
    "DstVersion": "<TargetVersion>",
    "ExtraArgs": {
        "KubeAPIServer": ["max-requests-inflight=500"],
        "KubeControllerManager": [],
        "KubeScheduler": [],
        "Etcd": []
    },
    "MaxNotReadyPercent": 0.1,
    "SkipPreCheck": false
}
```

```bash
tccli tke UpdateClusterVersion --region <Region> \
    --cli-input-json file://upgrade-enhanced.json
# expected: exit 0，返回 RequestId
```

### 步骤 3：轮询升级进度

升级为异步操作。使用 `DescribeClusterStatus` 配合 `--waiter` 参数自动轮询，或手动轮询。

```bash
# 方式一：自动轮询等待至 Running（推荐）
tccli tke DescribeClusterStatus --region <Region> \
    --ClusterIds '["<ClusterId>"]' \
    --waiter expr='ClusterStatusSet[0].ClusterState',to=Running,timeout=1800,interval=30
# expected: 超时 1800 秒，每 30 秒轮询一次，ClusterState 变为 Running 后返回
```

**预期输出（升级进行中）**：

```json
{
    "ClusterStatusSet": [
        {
            "ClusterId": "cls-example",
            "ClusterState": "Upgrading",
            "ClusterInstanceState": "-",
            "ClusterBMonitor": false,
            "ClusterInitNodeNum": 0,
            "ClusterRunningNodeNum": 0,
            "ClusterFailedNodeNum": 0,
            "ClusterClosedNodeNum": 0,
            "ClusterClosingNodeNum": 0,
            "ClusterDeletionProtection": false,
            "ClusterAuditEnabled": false
        }
    ],
    "TotalCount": 1,
    "RequestId": "..."
}
```

```bash
# 方式二：手动轮询集群详情
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["<ClusterId>"]'
# expected: ClusterStatus: "Upgrading" → 等待 → "Running"
```

**预期输出（升级进行中）**：

```json
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "cls-example",
            "ClusterName": "<ClusterName>",
            "ClusterVersion": "1.32.2",
            "ClusterStatus": "Upgrading"
        }
    ],
    "RequestId": "..."
}
```

> **注意**：`ClusterVersion` 字段在 API 返回后立即可见为目标版本（如 `1.32.2`），但这不表示升级已完成。必须等待 `ClusterStatus` 变为 `Running` 后才算升级完成。

**预期输出（升级完成）**：

```json
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "cls-example",
            "ClusterName": "<ClusterName>",
            "ClusterVersion": "1.32.2",
            "ClusterStatus": "Running",
            "ClusterType": "MANAGED_CLUSTER"
        }
    ],
    "RequestId": "..."
}
```

## 验证

### 控制面（tccli）

```bash
# 确认版本与状态
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["<ClusterId>"]' \
    | jq '.Clusters[0] | {ClusterVersion, ClusterStatus, ClusterType, ContainerRuntime, DeletionProtection}'
# expected: ClusterVersion: "<TargetVersion>", ClusterStatus: "Running"
```

**预期输出（升级完成）**：

```json
{
    "ClusterVersion": "1.32.2",
    "ClusterStatus": "Running",
    "ClusterType": "MANAGED_CLUSTER",
    "ContainerRuntime": "containerd",
    "DeletionProtection": false
}
```

| 验证维度 | 命令 | 预期 |
|---------|------|------|
| 集群状态 | `DescribeClusters` 检查 `ClusterStatus` | `Running`（不再是 `Upgrading`） |
| 集群版本 | `DescribeClusters` 检查 `ClusterVersion` | 为 `<TargetVersion>`（如 `1.32.2`） |
| 集群类型 | `DescribeClusters` 检查 `ClusterType` | `MANAGED_CLUSTER`（未变） |
| 运行时 | `DescribeClusters` 检查 `ContainerRuntime` | `containerd`（未变） |
| 删除保护 | `DescribeClusters` 检查 `DeletionProtection` | 与升级前一致（未变） |
| 升级状态 | `DescribeClusterStatus` 检查 `ClusterState` | `Running` |

> 直到所有维度确认无误后继续下一步。最长等待约 30 分钟。超时参见 [排障](#排障)。

### 数据面（kubectl）

集群就绪后，获取 kubeconfig 并验证控制面连通性：

```bash
tccli tke DescribeClusterKubeconfig --region <Region> \
    --ClusterId <ClusterId> | jq -r '.Kubeconfig' > ~/.kube/config
kubectl cluster-info
# expected: Kubernetes control plane is running at https://...
```

```text
Kubernetes control plane is running at https://<ClusterId>.ccs.tencent-cloudsdk.com:443
CoreDNS is running at https://<ClusterId>.ccs.tencent-cloudsdk.com:443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
```

```bash
kubectl get ns
# expected: 返回 default 等系统命名空间，确认 kubectl 可达
```

```text
NAME              STATUS   AGE
default           Active   30d
kube-node-lease   Active   30d
kube-public       Active   30d
kube-system       Active   30d
```

```bash
# 检查节点状态（节点 kubelet 版本仍为旧版，属预期）
kubectl get nodes -o wide
# expected: 所有节点 Ready，KubeletVersion 仍为旧版（需后续 UpgradeClusterInstances 升级）
```

```text
NAME             STATUS   ROLES    AGE   VERSION    INTERNAL-IP   OS-IMAGE
node-xxxxx       Ready    <none>   30d   v1.30.0    10.0.0.2      TencentOS Server 3.1
```

> 升级后节点 kubelet 不会自动更新。旧版 kubelet 一般可与新版控制面兼容（Kubernetes 版本偏差策略允许 kubelet 低于控制面最多 2 个次版本），建议通过 `UpgradeClusterInstances` 逐步升级节点。

## 清理

> **⚠️ 警告**：`UpdateClusterVersion` 升级不可逆，不支持版本回退或降级。升级前务必确认业务兼容性并完成 ETCD 备份。
>
> 异步升级期间，控制面 API Server 可能短暂不可达，kubectl 操作会受影响。数据面（运行中的 Pod）不受影响。

本页操作不创建新资源，升级完成后无需清理。如果升级后发现业务不兼容：

1. 保留 region、ClusterId、RequestId，登录 [TKE 控制台](https://console.cloud.tencent.com/tke2/cluster) 查看 `ClusterStatus` 和 `StatusReason`
2. 评估是否需要将业务迁移至新版本集群（新建目标版本集群 → 迁移业务 → 删除旧集群）
3. 删除旧集群前需先关闭删除保护并按依赖倒序清理资源

```bash
# 升级后状态确认（无需清理，仅验证）
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["<ClusterId>"]'
# expected: ClusterStatus: "Running", ClusterVersion: "<TargetVersion>"
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `UpdateClusterVersion` 仅返回 `RequestId`，无 `TaskId` | 这是正常行为 | `UpdateClusterVersion` 是异步 API，不返回 `TaskId`，`RequestId` 即表示命令被接受 | 检查 `DescribeClusterStatus` 确认升级进度，参见 [步骤 3](#步骤-3轮询升级进度) |
| `UpdateClusterVersion` 返回 `InvalidParameter`（版本不可升级） | 确认 `--DstVersion` 是否在 `DescribeAvailableClusterVersion` 返回列表中 | 未确认可升级版本就调用 `UpdateClusterVersion`，目标版本不在可用列表中（如 1.30 跳到未开放的 1.34） | 先执行 `DescribeAvailableClusterVersion --ClusterId <ClusterId>` 获取可用版本列表，再选择目标版本 |
| `UpdateClusterVersion` 返回 `UnknownParameter: DstVersion` | 检查参数拼写 | 参数名是 `--DstVersion`，不是 `--ClusterVersion` 也不是 `--Version` | 改为 `--DstVersion <TargetVersion>` |
| `UpdateClusterVersion` 返回 `ResourceUnavailable.ClusterState` | `tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'` 查看 `ClusterStatus` | 集群状态不是 `Running`（可能已在升级中或其他状态） | 等待恢复 `Running` 后重试 |
| `UpdateClusterVersion` 返回 `InvalidParameter.ExtraArgs` | 检查 `ExtraArgs` 格式 | 子字段必须为 `Array of String`，格式 `["k1=v1","k2=v2"]` | 调整为字符串数组格式 |
| `DescribeAvailableClusterVersion` 返回 `UnknownAction` | 检查 API 拼写 | API 名为 `DescribeAvailableClusterVersion`（非 `DescribeClusterAvailableVersion`） | 使用正确 API 名 |
| `DescribeAvailableClusterVersion` 返回空列表 | `tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'` 确认集群状态；`tccli tke DescribeVersions --region <Region>` 查看全局版本 | 当前集群版本可能已是最新可升级版本，或集群状态非 `Running` | 检查集群状态为 `Running`，确认当前版本非最新 |
| `DescribeClusters` 返回 `UnauthorizedOperation` | `tccli configure list` 检查凭据 | 子账号缺少 `tke:DescribeClusters` 权限（环境限制，非命令错误） | 联系主账号授予 `QcloudTKEFullAccess` 或对应 Action 权限 |

### 升级已提交但状态异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 升级后立即假设升级完成，kubectl 不可用 | `tccli tke DescribeClusterStatus --region <Region> --ClusterIds '["<ClusterId>"]'` 查看 `ClusterState` | 集群状态仍为 `Upgrading`，控制面 API Server 升级中短暂不可达 | 使用 `DescribeClusterStatus` 轮询至 `Running` 状态后再执行 kubectl 操作 |
| 返回了 `RequestId` 但 10 分钟后仍 `Upgrading` | `tccli tke DescribeClusterStatus --region <Region> --ClusterIds '["<ClusterId>"]' --waiter expr='ClusterStatusSet[0].ClusterState',to=Running,timeout=1800,interval=30` 持续轮询 | 升级过程正常但缓慢（控制面组件升级需 10-30 分钟） | 继续等待；超过 30 分钟则保留 region、ClusterId、RequestId → 登录控制台查看详细状态 → 仍无法解决则提交工单 |
| `ClusterVersion` 已显示为新版本但 `ClusterStatus` 仍为 `Upgrading` | `DescribeClusters` 检查 `ClusterStatus` | `ClusterVersion` 字段在 API 接受请求后即刻更新，但控制面组件实际仍在升级 | 以 `ClusterStatus: "Running"` 为准，不要以 `ClusterVersion` 判断升级完成 |
| 升级后 `ClusterStatus: "Abnormal"` | `tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'` 查看 `StatusReason` | 版本兼容性问题或组件故障 | 保留 region、ClusterId、RequestId → 登录控制台。紧急处理需迁移业务至新集群 |
| 升级后 kubectl 不可达 | 重新获取 kubeconfig：`tccli tke DescribeClusterKubeconfig --region <Region> --ClusterId <ClusterId>` | API Server 重启中或凭证过期 | 等待 2-5 分钟重试 |
| 升级后节点 `NotReady` | `kubectl get nodes` 检查节点状态 | kubelet 与控制面版本偏差过大 | 通过 `UpgradeClusterInstances` 升级节点 |
| 只升级了控制面，节点版本不一致 | `kubectl get nodes -o wide` 查看 `KubeletVersion` | `UpdateClusterVersion` 只升级控制面，忘记升级后也要升级节点 | 控制面升级完成后，执行 `UpgradeClusterInstances` 升级工作节点 |
| 升级期间执行其他修改操作失败 | `DescribeClusterStatus` 确认 `ClusterState` | 升级期间其他修改操作可能冲突 | 升级期间避免对集群做其他修改操作，等待 `Running` 后再执行 |

> 涉及 RequestId 的排查路径 → 提醒保留 region、ClusterId、RequestId 及升级参数 JSON，以备工单/日志查询。

## 下一步

- [集群生命周期](https://cloud.tencent.com/document/product/457/67792) — 集群状态流转与生命周期管理
- [TKE Kubernetes 大版本更新说明](https://cloud.tencent.com/document/product/457/47714) — 各版本关键变更与 API 弃用
- [Kubernetes API Deprecation Guide](https://kubernetes.io/docs/reference/using-api/deprecation-guide/) — 评估升级后的 API 弃用影响
- [连接集群](../连接集群/tccli%20操作.md) — 升级后获取 kubeconfig 并验证连通性
- [集群管理模式说明](../集群管理模式说明/tccli%20操作.md) — 托管集群与独立集群的能力差异

## 控制台替代

[TKE 控制台 → 集群详情 → 基本信息 → 集群版本 → 升级](https://console.cloud.tencent.com/tke2/cluster)
