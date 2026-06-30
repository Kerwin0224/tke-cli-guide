# 指定资源规格

> 对照官方：[指定资源规格](https://cloud.tencent.com/document/product/457/44174) · page_id `44174`

## 概述

超级节点上 Pod 的 CPU、内存、GPU 资源通过 `resources.requests` 和 `resources.limits` 指定。资源规格直接影响 Pod 调度和计费，有两条操作路径：

| 路径 | 操作方式 | 适用场景 |
|------|---------|---------|
| **kubectl（数据面）** | 在 Pod YAML 中指定 `resources.requests` 和 `resources.limits`，配合 `nodeSelector` 调度到超级节点 | 单 Pod 粒度，灵活调整 |
| **tccli（控制面）** | 通过 `CreateClusterVirtualNodePool` 的 `VirtualNodes[].Quota` 指定虚拟节点总配额，或 `ModifyClusterVirtualNodePool` 调整已有节点池标签 | 节点池级别资源规划，批量管理 |

**计费模式**：超级节点按 Pod 实际使用的 `resources.requests` 和 `resources.limits` 计费，不指定时使用默认值（0.25 CPU / 256Mi Memory）。

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

# 3. 检查 kubectl 版本（数据面操作）
kubectl version --client
# expected: kubectl client version >= 1.28

# 4. 检查 CAM 权限 — 需要以下 Action 名
#    tke:DescribeClusterVirtualNodePools, tke:DescribeClusterVirtualNode,
#    tke:CreateClusterVirtualNodePool, tke:ModifyClusterVirtualNodePool,
#    tke:DeleteClusterVirtualNodePool
# 验证：执行 DescribeClusterVirtualNodePools 确认权限
tccli tke DescribeClusterVirtualNodePools --region <Region> --ClusterId '<ClusterId>'
# expected: exit 0，返回节点池列表（可为空）
```

```json
{
    "TotalCount": "<TotalCount>",
    "NodePoolSet": "<NodePoolSet>",
    "RequestId": "..."
}
```

### 资源检查

```bash
# 5. 查询目标集群，确认集群已就绪且支持超级节点
tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'
# expected: ClusterStatus 为 "Running"，ClusterType 为 "MANAGED_CLUSTER"
```

```json
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "cls-example",
            "ClusterName": "<ClusterName>",
            "ClusterType": "MANAGED_CLUSTER",
            "ClusterStatus": "Running",
            "ClusterVersion": "1.32.2"
        }
    ],
    "RequestId": "..."
}
```

```bash
# 6. 查询现有超级节点池（如已有）
tccli tke DescribeClusterVirtualNodePools --region <Region> --ClusterId '<ClusterId>'
# expected: 如有节点池，LifeState 应为 "normal"
```

```json
{
    "TotalCount": 4,
    "NodePoolSet": [
        {
            "NodePoolId": "np-example",
            "SubnetIds": ["subnet-example", "subnet-example-2"],
            "Name": "<PoolName>",
            "LifeState": "normal",
            "Labels": [{"Name": "resource-spec", "Value": "demo"}],
            "Taints": []
        }
    ],
    "RequestId": "..."
}
```

```bash
# 7. 查询可用子网（创建节点池需指定）
tccli vpc DescribeSubnets --region <Region> --Filters '[{"Name":"vpc-id","Values":["<VpcId>"]}]'
# expected: 至少返回 1 个子网，AvailableIpCount 充足
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看超级节点池列表 | `DescribeClusterVirtualNodePools` | 是 |
| 查看超级节点池下的虚拟节点 | `DescribeClusterVirtualNode` | 是 |
| 新建超级节点池（含资源配额） | `CreateClusterVirtualNodePool` | 否 |
| 修改节点池标签 | `ModifyClusterVirtualNodePool` | 否 |
| 删除超级节点池 | `DeleteClusterVirtualNodePool` | 是 |
| 在 Pod YAML 中指定资源规格 | `kubectl apply -f pod.yaml`（数据面） | 否 |

## 关键字段说明

以下说明 `CreateClusterVirtualNodePool` 中与资源规格相关的主要参数。

### 节点池基本参数

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `ClusterId` | String | 是 | 已存在的 TKE 集群 ID，格式 `cls-xxxxxxxx` | 集群不存在或非托管集群 → `InvalidParameter.ClusterNotFound` |
| `Name` | String | 是 | 节点池名称，长度 1-60 字符，以字母开头 | 名称冲突 → `InvalidParameter.Name` |
| `SubnetIds` | Array | 是 | 子网 ID 列表，格式 `["subnet-xxxxxxxx"]`。需属于同一 VPC | 子网不存在或不在同一 VPC → `InvalidParameter.SubnetId` |
| `SecurityGroupIds` | Array | 是 | 安全组 ID 列表，格式 `["sg-xxxxxxxx"]` | 安全组不存在 → `InvalidParameter.SecurityGroupId` |
| `Labels` | Array | 否 | 节点池标签，格式 `[{"Name":"key","Value":"val"}]`。用于 Pod nodeSelector 匹配 | 格式错误 → `InvalidParameter.Labels` |
| `DeletionProtection` | Boolean | 否 | 删除保护开关，默认 `false`。`true` 时需先关闭才能删除 | 忘开删除保护 → 可能误删生产节点池 |

### 虚拟节点资源配额（VirtualNodes[].Quota）

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `DisplayName` | String | 是 | 虚拟节点显示名称，用于在 kubectl 中标识节点 | — |
| `SubnetId` | String | 是 | 虚拟节点所属子网，需在 `SubnetIds` 中 | 子网不匹配 → `InvalidParameter.SubnetId` |
| `Quota.Cpu` | Float | 是 | CPU 配额，单位：核。如 `2` 表示 2 核 | 配额过大导致创建失败 → `InvalidParameter.Quota` |
| `Quota.Memory` | Float | 是 | 内存配额，单位：GiB。如 `4` 表示 4 GiB | 同上 |
| `Quota.Gpu` | Float | 否 | GPU 配额，单位：卡。如 `1` 表示 1 卡 GPU | QGPU 未安装 → GPU 不可用 |
| `Quota.Num` | Integer | 是 | 该配置下的虚拟节点数量，如 `1` | — |
| `Quota.QuotaType` | String | 是 | 配额模式：`manual`（手动指定，推荐）或 `auto` | 填错 → `InvalidParameter.QuotaType` |
| `Quota.ChargeType` | String | 是 | 计费模式：`POSTPAID_BY_HOUR`（按量计费） | 不合法值 → `InvalidParameter.ChargeType` |
| `Quota.ResourceType` | String | 是 | 资源类型：`intel`（Intel CPU）、`amd`（AMD CPU）、`gpu`（GPU 实例） | 不合法值 → `InvalidParameter.ResourceType` |
| `Quota.PriceType` | String | 否 | 竞价模式：`minLimitPrice`（最小限价）或不指定 | — |

### Pod YAML 资源字段（kubectl 数据面）

| 字段 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `resources.requests.cpu` | String | Pod 申请的最小 CPU，影响调度和计费。单位：核（小数）或毫核（m） | `0.25`（250m） |
| `resources.requests.memory` | String | Pod 申请的最小内存，影响调度和计费。单位：`Mi` / `Gi` | `256Mi` |
| `resources.limits.cpu` | String | Pod 可使用的 CPU 上限 | 不设上限（等于 requests） |
| `resources.limits.memory` | String | Pod 可使用的内存上限，超出可能 OOMKilled | 不设上限（等于 requests） |
| `spec.nodeSelector` | Object | 指定 `node.kubernetes.io/instance-type: eklet` 将 Pod 调度到超级节点 | 无 |
| `resources.requests.nvidia.com/gpu` | String | GPU 扩展资源请求（标准 GPU） | 无 |
| `resources.requests.tke.cloud.tencent.com/qgpu-core` | String | qGPU 算力请求（需安装 qGPU 组件） | 无 |

## 操作步骤

### 控制面（tccli）

#### 步骤 1：查询超级节点池列表

```bash
tccli tke DescribeClusterVirtualNodePools --region <Region> \
    --ClusterId '<ClusterId>'
# expected: exit 0，返回所有超级节点池及其状态、标签
```

**预期输出**：

```json
{
    "TotalCount": 4,
    "NodePoolSet": [
        {
            "NodePoolId": "np-example-2",
            "SubnetIds": ["subnet-example", "subnet-example-2"],
            "Name": "<PoolName-3>",
            "LifeState": "normal",
            "Labels": [{"Name": "modified", "Value": "true"}],
            "Taints": []
        },
        {
            "NodePoolId": "np-example",
            "SubnetIds": ["subnet-example"],
            "Name": "<PoolName-1>",
            "LifeState": "normal",
            "Labels": [{"Name": "resource-spec", "Value": "demo"}, {"Name": "test", "Value": "rewrite"}],
            "Taints": []
        },
        {
            "NodePoolId": "np-example-3",
            "SubnetIds": ["subnet-example"],
            "Name": "<PoolName-2>",
            "LifeState": "normal",
            "Labels": [],
            "Taints": []
        }
    ],
    "RequestId": "..."
}
```

记录目标节点池的 `NodePoolId` 和 `SubnetIds` 供后续步骤使用。

#### 步骤 2：查询节点池下的虚拟节点

```bash
tccli tke DescribeClusterVirtualNode --region <Region> \
    --ClusterId '<ClusterId>' \
    --NodePoolId '<NodePoolId>'
# expected: exit 0，返回虚拟节点名称、状态和创建时间
```

**预期输出**：

```json
{
    "Nodes": [
        {
            "Name": "eklet-subnet-example-xxxx",
            "SubnetId": "subnet-example",
            "Phase": "Running",
            "CreatedTime": "2026-06-15T00:00:00Z"
        }
    ],
    "TotalCount": 1,
    "RequestId": "..."
}
```

| 字段 | 说明 |
|------|------|
| `Name` | 虚拟节点名称，格式 `eklet-subnet-xxxxxxxx-yyyyyyyy`。后续 kubectl 用此名称执行 `kubectl describe node` |
| `Phase` | `Running` 表示节点就绪，可调度 Pod。`NotReady` 表示不可用 |
| `SubnetId` | 虚拟节点使用的子网，IP 从此子网分配 |

#### 步骤 3：修改节点池标签（为 Pod 调度提供 nodeSelector 依据）

##### 选择依据

- **为什么加标签**：超级节点的 Kubernetes 标签为 `node.kubernetes.io/instance-type=eklet`，这是系统自动注入的。但如需更精细的调度控制（如按环境、团队分组），需在节点池上加自定义 Labels。`ModifyClusterVirtualNodePool --Labels` 会**覆盖**节点池的全部 Labels（非增量更新）。
- **Label 作用**：自定义 Label 会同步到节点池下所有虚拟节点，Pod 可通过 `nodeSelector` 匹配这些标签，实现精确调度。
- **来源**：决策依据来自 `decision_context.node_selector` — 超级节点标签 `node.kubernetes.io/instance-type: eklet` 是系统标签，自定义标签通过节点池 Labels 注入。

##### 操作

```bash
tccli tke ModifyClusterVirtualNodePool --region <Region> \
    --ClusterId '<ClusterId>' \
    --NodePoolId '<NodePoolId>' \
    --Labels '[{"Name":"resource-spec","Value":"demo"},{"Name":"test","Value":"rewrite"}]'
# expected: exit 0，返回 RequestId
```

**预期输出**：

```json
{
    "RequestId": "..."
}
```

> **注意**：`--Labels` 参数执行**全量替换**，非增量添加。务必包含所有需要保留的标签。如节点池原有标签 `env:prod`，上述命令执行后该标签将被移除。

**验证 Labels 生效**：

```bash
tccli tke DescribeClusterVirtualNodePools --region <Region> \
    --ClusterId '<ClusterId>'
# expected: 目标节点池的 Labels 已更新
```

```json
{
    "NodePoolSet": [
        {
            "NodePoolId": "np-example",
            "LifeState": "normal",
            "Labels": [{"Name": "resource-spec", "Value": "demo"}, {"Name": "test", "Value": "rewrite"}]
        }
    ]
}
```

#### 步骤 4：创建带资源配额的超级节点池

##### 选择依据

- **节点池配额 vs Pod resources**：节点池配额（`VirtualNodes[].Quota`）定义了虚拟节点**可分配的总资源上限**，是 Pod `resources.requests` 的上限约束。Pod 的 `requests` 总和不能超过虚拟节点的总配额。
- **QuotaType 选择**：选 `manual`（手动指定）而非 `auto`。手动模式可精确控制 CPU/内存/GPU 配额，适合有明确资源规划的场景。自动模式由系统根据子网规模自动分配。
- **ResourceType 选择**：选 `intel`（Intel CPU）作为通用计算资源。如有 GPU 需求则选 `gpu`，AMD 场景选 `amd`。选择依据来自 `decision_context.quota_configuration`。
- **ChargeType**：超级节点仅支持 `POSTPAID_BY_HOUR`（按量计费），按 Pod 实际使用的 `requests` 和 `limits` 计费。
- **来源**：决策依据来自 `decision_context.quota_configuration` 和 `decision_context.resource_spec_model`。

##### 最小创建（只含必填字段，1 个虚拟节点）

`vnode-pool-minimal.json`：

```json
{
    "ClusterId": "<ClusterId>",
    "Name": "<PoolName>",
    "SecurityGroupIds": ["<SecurityGroupId>"],
    "SubnetIds": ["<SubnetId>"],
    "VirtualNodes": [
        {
            "DisplayName": "vn-1",
            "SubnetId": "<SubnetId>",
            "Quota": {
                "Cpu": 2,
                "Memory": 4,
                "Num": 1,
                "QuotaType": "manual",
                "ChargeType": "POSTPAID_BY_HOUR",
                "ResourceType": "intel",
                "PriceType": "minLimitPrice"
            }
        }
    ]
}
```

```bash
tccli tke CreateClusterVirtualNodePool --region <Region> \
    --cli-input-json file://vnode-pool-minimal.json
# expected: exit 0，返回 NodePoolId
```

**预期输出**：

```json
{
    "NodePoolId": "np-example",
    "RequestId": "..."
}
```

##### 增强配置（加标签、Taint、删除保护）

`vnode-pool-enhanced.json`：

```json
{
    "ClusterId": "<ClusterId>",
    "Name": "<PoolName>",
    "SecurityGroupIds": ["<SecurityGroupId>"],
    "SubnetIds": ["<SubnetId>"],
    "Labels": [
        {"Name": "resource-spec", "Value": "demo"},
        {"Name": "team", "Value": "backend"}
    ],
    "DeletionProtection": true,
    "VirtualNodes": [
        {
            "DisplayName": "vn-1",
            "SubnetId": "<SubnetId>",
            "Quota": {
                "Cpu": 2,
                "Memory": 4,
                "Num": 1,
                "QuotaType": "manual",
                "ChargeType": "POSTPAID_BY_HOUR",
                "ResourceType": "intel",
                "PriceType": "minLimitPrice"
            }
        }
    ]
}
```

```bash
tccli tke CreateClusterVirtualNodePool --region <Region> \
    --cli-input-json file://vnode-pool-enhanced.json
# expected: exit 0，返回 NodePoolId
```

**预期输出**：

```json
{
    "NodePoolId": "np-example-plus",
    "RequestId": "..."
}
```

| 层级 | 包含字段 | 目的 |
|------|---------|------|
| **最小创建** | ClusterId, Name, SecurityGroupIds, SubnetIds, VirtualNodes（含 DisplayName, SubnetId, Quota 必填字段） | 快速创建可用节点池，验证配额模型 |
| **增强配置** | 最小基础上增加 Labels（自定义标签）、DeletionProtection（删除保护） | 生产环境推荐配置，含调度标签和保护开关 |

##### 参数表

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<ClusterId>` | TKE 集群 ID | 格式 `cls-xxxxxxxx` | `tccli tke DescribeClusters --region <Region>` |
| `<PoolName>` | 节点池名称 | 长度 1-60 字符 | 自定义 |
| `<SecurityGroupId>` | 安全组 ID | 格式 `sg-xxxxxxxx` | `tccli vpc DescribeSecurityGroups --region <Region>` |
| `<SubnetId>` | 子网 ID | 格式 `subnet-xxxxxxxx` | `tccli vpc DescribeSubnets --region <Region>` |
| `<NodePoolId>` | 节点池 ID | 格式 `np-xxxxxxxx` | `DescribeClusterVirtualNodePools` 返回 |
| `<Region>` | 地域 | 如 `ap-guangzhou` | `tccli configure list` |

---

### 数据面（kubectl）

> ⚠️ **注意**：以下 kubectl 命令需在集群端点可达的环境中执行。本集群因公网端点创建被组织级 CAM 策略 `strategyId:240463971`（条件 `tke:clusterExtranetEndpoint=true`，effect: `deny`）硬拒绝，内网端点从本地不可达。如需执行，请通过以下方式之一连接集群：(1) IOA/VPN 接入 VPC；(2) 同 VPC 内 CVM 上执行；(3) 专线连接。以下 YAML 语法和命令语句已验证正确。

#### 步骤 5：查看虚拟节点可分配资源

```bash
# 使用步骤 2 中获取的虚拟节点名称
kubectl describe node <VirtualNodeName> | grep -A 10 "Allocatable"
# expected: 显示 CPU、内存、GPU 可分配上限
```

**预期输出**（集群可达时）：

```text
Allocatable:
  cpu:                2
  memory:             4104616Ki
  nvidia.com/gpu:     0
  tke.cloud.tencent.com/qgpu-core:  0
```

| 字段 | 说明 |
|------|------|
| `cpu` | 虚拟节点可分配 CPU，以核为单位。受 `Quota.Cpu` 限制 |
| `memory` | 虚拟节点可分配内存，以 Ki 为单位。受 `Quota.Memory` 限制 |
| `nvidia.com/gpu` | GPU 设备数，如 `Quota.Gpu > 0` 则此处显示对应数量 |
| `tke.cloud.tencent.com/qgpu-core` | qGPU 算力单位（需安装 qGPU 组件） |

#### 步骤 6：创建带资源规格的 Pod

##### 选择依据

- **nodeSelector**：必须指定 `node.kubernetes.io/instance-type: eklet`，将 Pod 调度到超级节点。否则 Pod 可能被调度到普通 CVM 节点，失去超级节点弹性优势。选择依据来自 `decision_context.node_selector`。
- **resources.requests vs limits**：`requests` 决定调度时的最小预留资源和计费基础；`limits` 决定 Pod 可使用的资源上限。不设 `limits` 时，Pod 可爆发到节点配额上限。对生产服务建议同时设置两者。选择依据来自 `decision_context.resource_spec_model`。
- **CPU 单位**：使用小数（如 `0.5` 表示半核）或毫核（如 `500m`）。`1` 表示 1 核。不要写成 `500`（会被解析为 500 核）。常见读者错误见 `reader_pitfalls`。
- **内存单位**：使用 `Mi`（兆字节）或 `Gi`（吉字节）。`512Mi` 表示 512 兆，`1Gi` 表示 1 吉。

##### Pod YAML（最小示例）

`pod-resources-minimal.yaml`：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-resource-demo
  namespace: default
  labels:
    app: nginx-demo
spec:
  containers:
  - name: nginx
    image: nginx:1.21
    resources:
      requests:
        cpu: "0.5"
        memory: "512Mi"
      limits:
        cpu: "1"
        memory: "1Gi"
  nodeSelector:
    node.kubernetes.io/instance-type: eklet
```

```bash
kubectl apply -f pod-resources-minimal.yaml
# expected: pod/nginx-resource-demo created
```

**预期输出**（集群可达时）：

```text
pod/nginx-resource-demo created
```

##### Pod YAML（增强示例：多容器 + GPU）

`pod-resources-enhanced.yaml`：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-inference-demo
  namespace: default
  labels:
    app: inference
spec:
  containers:
  - name: inference-server
    image: nvidia/cuda:11.0-runtime
    resources:
      requests:
        cpu: "2"
        memory: "4Gi"
        nvidia.com/gpu: "1"
      limits:
        cpu: "4"
        memory: "8Gi"
        nvidia.com/gpu: "1"
  - name: sidecar
    image: busybox:1.35
    command: ["sleep", "86400"]
    resources:
      requests:
        cpu: "0.1"
        memory: "64Mi"
      limits:
        cpu: "0.2"
        memory: "128Mi"
  nodeSelector:
    node.kubernetes.io/instance-type: eklet
```

> **注意**：GPU 资源通过 `nvidia.com/gpu` 扩展资源声明。确保虚拟节点配额 `Quota.Gpu` >= Pod 请求的 GPU 数量。qGPU 场景使用 `tke.cloud.tencent.com/qgpu-core` 替代 `nvidia.com/gpu`。

```bash
kubectl apply -f pod-resources-enhanced.yaml
# expected: pod/gpu-inference-demo created
```

#### 步骤 7：验证 Pod 资源配置

```bash
kubectl describe pod nginx-resource-demo | grep -A 8 "Requests:"
# expected: 显示 requests 和 limits 值与创建时一致
```

**预期输出**（集群可达时）：

```text
Requests:
  cpu:     500m
  memory:  512Mi
Limits:
  cpu:     1
  memory:  1Gi
```

## 验证

### 控制面（tccli）

| 验证维度 | 命令 | 预期 |
|---------|------|------|
| 节点池状态 | `tccli tke DescribeClusterVirtualNodePools --region <Region> --ClusterId '<ClusterId>'` | `LifeState: "normal"` |
| 节点池 Labels | 同上，检查目标节点池 `Labels` | 与创建/修改参数一致 |
| 虚拟节点状态 | `tccli tke DescribeClusterVirtualNode --region <Region> --ClusterId '<ClusterId>' --NodePoolId '<NodePoolId>'` | `Phase: "Running"`，节点数量与 `Quota.Num` 一致 |
| 虚拟节点数量 | 同上，检查 `TotalCount` | 等于创建时指定的 `Quota.Num` |

### 数据面（kubectl）

> ⚠️ kubectl 需在集群端点可达的环境中执行。

| 验证维度 | 命令 | 预期 |
|---------|------|------|
| Pod 状态 | `kubectl get pod nginx-resource-demo` | `STATUS: Running` |
| Pod 调度节点 | `kubectl get pod nginx-resource-demo -o wide` | `NODE` 列为虚拟节点名（eklet-* 前缀） |
| 资源配置 | `kubectl describe pod nginx-resource-demo \| grep -A 8 "Requests:"` | cpu/memory requests 和 limits 与 YAML 一致 |
| 节点可分配 | `kubectl describe node <VirtualNodeName> \| grep -A 10 "Allocatable"` | cpu/memory 上限与 Quota 配置一致 |

## 清理

> **⚠️ 警告**：`DeleteClusterVirtualNodePool --Force true` 会**立即删除节点池及其下所有虚拟节点**。节点上运行的所有 Pod 将被驱逐或终止，**数据不可恢复**。节点池删除不可逆，生产环境操作前务必确认节点上无业务 Pod 运行。

### 数据面（kubectl）

> ⚠️ 在集群可达的环境下执行。

```bash
# 1. 删除测试 Pod
kubectl delete pod nginx-resource-demo
# expected: pod "nginx-resource-demo" deleted
```

```bash
# 2. 确认节点上无残留 Pod（生产环境必做）
kubectl get pod --all-namespaces --field-selector spec.nodeName=<VirtualNodeName>
# expected: No resources found
```

### 控制面（tccli）

#### 清理前状态检查

```bash
tccli tke DescribeClusterVirtualNodePools --region <Region> \
    --ClusterId '<ClusterId>'
# expected: 确认目标节点池的 NodePoolId 和 LifeState，记录 Name
```

```json
{
    "NodePoolSet": [
        {
            "NodePoolId": "np-example",
            "Name": "<PoolName>",
            "LifeState": "normal"
        }
    ]
}
```

#### 关闭删除保护（如启用）

```bash
# 检查 DeletionProtection 状态（通过节点池详情 API）
tccli tke DescribeClusterVirtualNodePools --region <Region> \
    --ClusterId '<ClusterId>'
# 如有 DeletionProtection=true，则需要关闭
```

```bash
tccli tke ModifyClusterVirtualNodePool --region <Region> \
    --ClusterId '<ClusterId>' \
    --NodePoolId '<NodePoolId>' \
    --DeletionProtection false
# expected: exit 0
```

#### 删除节点池

```bash
tccli tke DeleteClusterVirtualNodePool --region <Region> \
    --ClusterId '<ClusterId>' \
    --NodePoolIds '["<NodePoolId>"]' \
    --Force true
# ⚠️ --Force true 跳过安全确认，立即删除节点池及所有虚拟节点
# expected: exit 0
```

**预期输出**：

```json
{
    "RequestId": "..."
}
```

#### 验证已删除

```bash
tccli tke DescribeClusterVirtualNodePools --region <Region> \
    --ClusterId '<ClusterId>'
# expected: 目标 NodePoolId 不再出现在 NodePoolSet 中
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `CreateClusterEndpoint` 返回 `InvalidParameter.Param`，含 `tke:clusterExtranetEndpoint=true deny, strategyId:240463971` | `tccli tke DescribeClusterEndpoints --region <Region> --ClusterId '<ClusterId>'` 查看已有端点 | 公网端点被组织级 CAM 策略 `strategyId:240463971` 以条件 `tke:clusterExtranetEndpoint=true` 硬拒绝（effect: deny）。**此为环境限制，非命令错误** | 改用内网端点。通过 IOA/VPN/专线或同 VPC 内 CVM 连接集群后执行 kubectl 命令 |
| `CreateClusterVirtualNodePool` 返回 `InvalidParameter.Quota` | 检查 JSON 中 `Quota.Cpu`、`Quota.Memory`、`Quota.Gpu` 值 | 配额值不合法：可能超过地域上限、单位错误或与 `ResourceType` 不匹配 | 检查值类型（Cpu/Memory 为 Float，Gpu 为 Float），确保 `Quota.Cpu >= 0` 且 `Quota.Memory >= 0`。GPU 配额仅在 `ResourceType=gpu` 时有效 |
| `CreateClusterVirtualNodePool` 返回 `InvalidParameter.SubnetId` | `tccli vpc DescribeSubnets --region <Region> --SubnetIds '["<SubnetId>"]'` 确认子网存在 | 子网不存在、不属于集群 VPC 或已删除 | 用正确的子网 ID 重新创建，确保子网与集群在同一 VPC |
| `ModifyClusterVirtualNodePool` 返回 `InvalidParameter.Labels` | 检查 `--Labels` JSON 格式 | Labels 数组格式错误，如缺少引号、Name/Value 拼写错误 | 确保格式为 `'[{"Name":"key","Value":"val"}]'`，Name 和 Value 均为字符串 |
| `DescribeClusterVirtualNode` 返回空 `Nodes` 数组 | 等待 30 秒后重试 | 节点池刚创建，虚拟节点尚未就绪 | 等待 1-2 分钟后重试 `DescribeClusterVirtualNode`。仍为空则检查 `DescribeClusterVirtualNodePools` 的 `LifeState` 是否 `normal` |

### 创建成功但状态异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 虚拟节点 `Phase` 为 `NotReady` | `tccli tke DescribeClusterVirtualNode --region <Region> --ClusterId '<ClusterId>' --NodePoolId '<NodePoolId>'` 查看节点详情 | 子网 IP 耗尽或无可用 IP | `tccli vpc DescribeSubnets --SubnetIds '["<SubnetId>"]'` 检查 `AvailableIpCount`。如耗尽，更换子网或扩展现有子网 IP 范围 |
| Pod 处于 `Pending` 状态，Events 显示 `insufficient resources` | `kubectl describe pod <PodName> \| grep -A 20 Events` | Pod 的 `resources.requests` 超过虚拟节点可用配额（见 `reader_pitfalls[1]`） | 降低 `requests` 值，或通过 `ModifyClusterVirtualNodePool` 扩容 `Quota.Cpu`/`Quota.Memory`。用 `kubectl describe node <VirtualNodeName>` 确认可分配资源 |
| Pod 被调度到普通 CVM 节点而非超级节点 | `kubectl get pod <PodName> -o wide` 检查 `NODE` 列是否为 `eklet-*` | Pod 缺少 `nodeSelector: node.kubernetes.io/instance-type: eklet`（见 `reader_pitfalls[0]`） | 在 Pod YAML 中补加 `spec.nodeSelector: node.kubernetes.io/instance-type: eklet`，重新创建 Pod |
| Pod CPU 单位填写错误导致调度异常 | `kubectl describe pod <PodName> \| grep Requests` 检查值 | CPU 写成了 `500` 而非 `0.5` 或 `500m`，被解析为 500 核（见 `reader_pitfalls[2]`） | 改为 `"0.5"` 或 `"500m"` |
| Pod 被 `OOMKilled` | `kubectl describe pod <PodName> \| grep -A 5 "State"` 查看终止原因 | 容器内存超过 `limits` 且节点内存压力大（见 `reader_pitfalls[3]`） | 同时设置 `resources.requests` 和 `resources.limits`，确保 `limits.memory >= requests.memory`。对内存密集型服务设置合理的 limits |
| 节点池删除失败（`DeleteClusterVirtualNodePool` 返回错误） | `tccli tke DescribeClusterVirtualNodePools --region <Region> --ClusterId '<ClusterId>'` 检查 `DeletionProtection` | 节点池启用了 `DeletionProtection: true` | 先执行 `ModifyClusterVirtualNodePool --DeletionProtection false`，再重试删除 |

> **提示**：遇到 API 错误时，保留 `RequestId` 和完整的创建 JSON，以备提交工单时提供。

## 下一步

- [新建超级节点池](https://cloud.tencent.com/document/product/457/78328) — 本系列前一页：通过 tccli 创建超级节点池
- [超级节点上支持运行 DaemonSet](https://cloud.tencent.com/document/product/457/98730) — 本系列后一页：在超级节点上部署 DaemonSet
- [超级节点概述](https://cloud.tencent.com/document/product/457/53030) — 超级节点原理、适用场景与限制
- [qGPU 公测说明](https://cloud.tencent.com/document/product/457/61448) — qGPU 组件安装与配置（GPU 资源指定相关）

## 控制台替代

[TKE 控制台 → 集群 → 超级节点 → 指定资源规格](https://console.cloud.tencent.com/tke2/cluster?rid=1)
