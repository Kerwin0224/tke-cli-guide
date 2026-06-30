# Pod 原地升降配

> 对照官方：[Pod 原地升降配](https://cloud.tencent.com/document/product/457/79697) · page_id `79697` · tccli ≥3.1.107 · API 2018-05-25

## 概述

Pod 原地升降配是 TKE 原生节点的特性，允许在**不重启 Pod** 的情况下修改 Pod 的 CPU 和内存配置。通过 `ModifyClusterNodePool` 设置节点池注解（Annotations）启用该能力，然后通过 kubectl 调整 Pod 资源限制实现原地更新。该功能对延迟敏感或状态保持型应用特别有价值，可避免因滚动更新带来的服务中断。

| 节点池类型 | 是否支持原地升降配 | 备注 |
|-----------|:---:|------|
| 原生节点池 | 是 | 需设置 `node.tke.cloud.tencent.com/in-place-upgrade=true` 注解 |
| 普通节点池（CVM 弹性伸缩） | 否 | 修改 Pod 资源会触发重建 |

## 前置条件

- [环境准备](../../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 3.1.107

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置，region 为目标地域

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:DescribeClusters, tke:DescribeClusterNodePools
#    tke:DescribeClusterNodePoolDetail, tke:ModifyClusterNodePool
#    tke:DescribeClusterKubeconfig, tke:DescribeClusterSecurity
#    tke:DescribeClusterEndpointStatus, tke:DescribeClusterEndpoints
# 验证：执行 DescribeClusters 确认权限
tccli tke DescribeClusters --region <Region>
# expected: exit 0，返回集群列表（可为空）
```

### 资源检查

```bash
# 4. 检查集群存在且状态正常
tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'
# expected: exit 0，ClusterStatus: "Running"，ClusterType: "MANAGED_CLUSTER"
```

预期输出：

```json
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "cls-example",
            "ClusterName": "example-cluster",
            "ClusterStatus": "Running",
            "ClusterType": "MANAGED_CLUSTER",
            "ClusterVersion": "1.28.3",
            "ClusterNodeNum": 3
        }
    ],
    "RequestId": "xxxx-xxxx-xxxx-xxxx"
}
```

```bash
# 5. 检查节点池类型 — 必须为原生节点池
tccli tke DescribeClusterNodePools --region <Region> --ClusterId <ClusterId>
# expected: 返回至少一个原生节点池
```

预期输出：

```json
{
    "TotalCount": 2,
    "NodePoolSet": [
        {
            "NodePoolId": "np-example01",
            "Name": "native-pool-1",
            "ClusterInstanceId": "cls-example",
            "LifeState": "normal",
            "NodePoolType": "Native",
            "NodeCountSummary": {
                "ManuallyAdded": {"Total": 2},
                "AutoscalingAdded": {"Total": 1}
            }
        },
        {
            "NodePoolId": "np-example02",
            "Name": "regular-pool-1",
            "ClusterInstanceId": "cls-example",
            "LifeState": "normal",
            "NodePoolType": "Regular"
        }
    ],
    "RequestId": "xxxx-xxxx-xxxx-xxxx"
}
```

```bash
# 6. 查看目标节点池详情（确认注解和节点池类型）
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId <ClusterId> \
    --NodePoolId <NodePoolId>
# expected: 返回节点池详细信息，确认 NodePoolType 为原生节点池
```

预期输出：

```json
{
    "NodePool": {
        "NodePoolId": "np-example01",
        "Name": "native-pool-1",
        "ClusterInstanceId": "cls-example",
        "LifeState": "normal",
        "NodePoolType": "Native",
        "DesiredNodesNum": 2,
        "MinNodesNum": 1,
        "MaxNodesNum": 5,
        "Annotations": []
    },
    "RequestId": "xxxx-xxxx-xxxx-xxxx"
}
```

### kubectl 访问环境

```bash
# 7. 获取集群 kubeconfig
tccli tke DescribeClusterKubeconfig --region <Region> --ClusterId <ClusterId> | jq -r '.Kubeconfig' > ~/.kube/config
# expected: Kubeconfig 内容完整，含集群 API Server 地址
```

预期输出：

```json
{
    "Kubeconfig": "YXBpVmVyc2lvbjogdjEKY2x1c3RlcnM6Ci0gY2x1c3RlcjoKICAgIGNlcnRpZmljYXRlLWF1dGhvcml0eS1kYXRhOiBMUzB0TFM...",
    "RequestId": "xxxx-xxxx-xxxx-xxxx"
}
```

```bash
# 8. 验证 kubectl 可达
kubectl cluster-info
# expected: Kubernetes control plane is running at ...

# 9. 如果集群无公网端点，检查集群安全信息确认端点类型
tccli tke DescribeClusterSecurity --region <Region> --ClusterId <ClusterId>
# expected: ClusterExternalEndpoint 或 ClusterIntranetEndpoint 至少一个非空
```

预期输出：

```json
{
    "UserName": "admin",
    "Password": "",
    "CertificationAuthority": "",
    "ClusterExternalEndpoint": "https://cls-example.ccs.tencent-cloud.com",
    "Domain": "cls-example.ccs.tencent-cloud.com",
    "PgwEndpoint": "",
    "SecurityPolicy": [],
    "Kubeconfig": "",
    "JnsGwEndpoint": "",
    "RequestId": "xxxx-xxxx-xxxx-xxxx"
}
```

```bash
# 10. 确认端点状态
tccli tke DescribeClusterEndpointStatus --region <Region> --ClusterId <ClusterId>
# expected: Status: "Created"
```

预期输出：

```json
{
    "Status": "Created",
    "RequestId": "xxxx-xxxx-xxxx-xxxx"
}
```

> **注意**：kubectl 命令需在集群端点可达的环境下执行。若集群仅内网端点可用，需通过 IOA/VPN/专线接入集群 VPC，或在同 VPC 内的 CVM 上执行。

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看集群列表 | `DescribeClusters` | 是 |
| 查看节点池列表 | `DescribeClusterNodePools` | 是 |
| 查看节点池详情 | `DescribeClusterNodePoolDetail` | 是 |
| 启用原地升降配（修改节点池注解） | `ModifyClusterNodePool` | 否 |
| 获取 kubeconfig | `DescribeClusterKubeconfig` | 是 |
| 查看集群安全信息 | `DescribeClusterSecurity` | 是 |
| 查看集群端点状态 | `DescribeClusterEndpointStatus` | 是 |

## 控制台与 CLI 参数映射

以下说明 `ModifyClusterNodePool` 启用原地升降配相关的主要参数。完整参数定义见 `tccli tke ModifyClusterNodePool --generate-cli-skeleton`。

| 控制台参数 | CLI 参数 | 类型 | 必填 | 取值与约束 | 幂等 | 错误后果 |
|-----------|---------|------|:--:|------|:--:|------|
| 集群 ID | `ClusterId` | String | 是 | 目标集群 ID，格式 `cls-xxxxxxxx`。`tccli tke DescribeClusters` 获取 | 是 | 集群不存在 → `ResourceNotFound.ClusterNotFound` |
| 节点池 ID | `NodePoolId` | String | 是 | 目标节点池 ID，格式 `np-xxxxxxxx`。`tccli tke DescribeClusterNodePools` 获取 | 是 | 节点池不存在 → `ResourceNotFound.NodePoolNotFound` |
| 原地升级开关注解 | `Annotations[].Name` | String | 是 | `node.tke.cloud.tencent.com/in-place-upgrade`（原地升级开关） | 否（PUT 整体替换） | 注解名错误 → 功能不生效，无直接报错 |
| 原地升级开关值 | `Annotations[].Value` | String | 是 | `"true"` 或 `"false"` | 是（相同值多次执行幂等） | 格式错误 → 注解无效，Pod 原地升级不工作 |
| QoS 注解 | `Annotations[].Name` | String | 否 | `node.tke.cloud.tencent.com/qos`（QoS 配置） | 否（PUT 整体替换） | 注解名错误 → 功能不生效 |
| QoS 配置值 | `Annotations[].Value` | String | 否 | JSON 字符串格式，含 `memoryQos`/`cpuQos` | 是（相同值多次执行幂等） | 格式错误 → 注解无效 |

> **重要行为说明**：`ModifyClusterNodePool` 的 `Annotations` 参数是 **PUT 操作（整体替换）**，不是 PATCH（追加）。每次调用需传入完整的 `Annotations` 列表，否则新调用会覆盖已有注解。建议流程：先 `DescribeClusterNodePoolDetail` 获取当前注解 → 合并新注解 → 一次性完整写入。

## 操作步骤

### 步骤 1：加入集群（如有必要）

若尚未加入集群，请先通过以下方式将集群加入本地 kubeconfig 上下文。需要集群端点可达才能完成后续 kubectl 操作。

```bash
tccli tke DescribeClusterKubeconfig --region <Region> --ClusterId <ClusterId> | jq -r '.Kubeconfig' > ~/.kube/config
# expected: Kubeconfig 写入成功
```

预期输出：

```text
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: ...
    server: https://<ClusterId>.ccs.tencent-cloud.com
  name: <ClusterId>
contexts:
- context:
    cluster: <ClusterId>
    user: admin
  name: <ClusterId>-context
current-context: <ClusterId>-context
kind: Config
users:
- name: admin
  user:
    client-certificate-data: ...
    client-key-data: ...
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<ClusterId>` | 集群 ID | 格式 `cls-xxxxxxxx` | `tccli tke DescribeClusters` |
| `<Region>` | 地域 | 如 `ap-guangzhou` | `tccli configure list` |

### 步骤 2：启用节点池原地升级注解

#### 选择依据

*为什么选这个注解值和操作方式：*

- **原地升级开关注解**（`node.tke.cloud.tencent.com/in-place-upgrade`）：设为 `"true"` 后，节点池内的 Pod 可通过 kubectl patch 调整 CPU/内存请求而不重启。这是启用原地升降配的核心开关。
- **节点池类型要求**：原地升级仅对原生节点池（Type=Native）生效。普通节点池（CVM 弹性伸缩）不支持修改 Pod 资源不重启。通过 `DescribeClusterNodePoolDetail` 确认节点池类型。
- **QoS 注解**（`node.tke.cloud.tencent.com/qos`）：可选高阶配置。内存页缓存限制（`memPageCacheLimit`）可减少 Pod 内存碎片，CPU 绑定（`podCPUBinding`）可保障延迟敏感型 Pod 性能。如不需 QoS 控制可跳过。
- **PUT 行为**：`ModifyClusterNodePool` 的 `Annotations` 参数整体替换而非追加。建议先 `DescribeClusterNodePoolDetail` 获取当前注解列表，合并新注解后一次性完整传入，避免覆盖已有注解。

> **请根据需求选择以下方案之一（互斥）**：
> - **方案 A（最小修改）**：仅启用原地升级开关，不配置 QoS。适用于基需要开启原地升级能力的场景。
> - **方案 B（增强配置）**：同时启用原地升级开关 + QoS 内存压缩 + CPU 绑定。适用于需要精细化资源控制的场景。
>
> 只需执行其中一种方案。若先执行方案 A 再执行方案 B，需等待节点池 LifeState 恢复为 `normal`（约 1-2 分钟）后再执行，否则会收到 `OperationDenied` 错误。

#### 方案 A：最小修改（仅启用原地升级开关）

`inplace-upgrade-minimal.json`：

```json
{
  "ClusterId": "<ClusterId>",
  "NodePoolId": "<NodePoolId>",
  "Annotations": [
    {
      "Name": "node.tke.cloud.tencent.com/in-place-upgrade",
      "Value": "true"
    }
  ]
}
```

```bash
tccli tke ModifyClusterNodePool --cli-input-json file://inplace-upgrade-minimal.json --region <Region>
# expected: exit 0，返回 RequestId
```

预期输出：

```json
{
    "RequestId": "b78454da-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

#### 方案 B：增强配置（加 QoS 注解 — 内存压缩 + CPU 绑定）

> **注意**：若已执行方案 A，需等待节点池 LifeState 恢复为 `normal` 后再执行方案 B。

`inplace-upgrade-enhanced.json`：

```json
{
  "ClusterId": "<ClusterId>",
  "NodePoolId": "<NodePoolId>",
  "Annotations": [
    {
      "Name": "node.tke.cloud.tencent.com/in-place-upgrade",
      "Value": "true"
    },
    {
      "Name": "node.tke.cloud.tencent.com/qos",
      "Value": "{\"memoryQos\":{\"memPageCacheLimit\":{\"enabled\":\"true\"}},\"cpuQos\":{\"podCPUBinding\":{\"enabled\":\"true\"}}}"
    }
  ]
}
```

```bash
tccli tke ModifyClusterNodePool --cli-input-json file://inplace-upgrade-enhanced.json --region <Region>
# expected: exit 0，返回 RequestId
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<ClusterId>` | 集群 ID | 格式 `cls-xxxxxxxx` | `tccli tke DescribeClusters` |
| `<NodePoolId>` | 节点池 ID，必须为原生节点池 | 格式 `np-xxxxxxxx` | `tccli tke DescribeClusterNodePools` |
| `<Region>` | 地域 | 如 `ap-guangzhou` | `tccli configure list` |

> **注意**：增强配置中的 `Annotations` 同时包含原地升级开关和 QoS 注解。若后续需要新增注解，必须将现有注解与新增注解合并后整体传入，否则已有注解会被清除。

### 步骤 3：通过 kubectl 调整 Pod 资源限制（示例）

以下 kubectl 命令为示例形式，展示原地升降配的 kubectl 侧操作。执行前需确认集群端点可达。

```bash
# 修改 Pod 的 CPU 请求（原地生效，不重启 Pod）
kubectl patch pod <PodName> -n <Namespace> --patch '
spec:
  containers:
  - name: <ContainerName>
    resources:
      requests:
        cpu: "500m"
        memory: "256Mi"
      limits:
        cpu: "500m"
        memory: "256Mi"
'
# expected: pod/<PodName> patched
```

预期输出：

```text
pod/<PodName> patched
```

```bash
# 通过 annotation 方式修改 Pod 资源（另一种原地更新方式）
kubectl annotate pod <PodName> -n <Namespace> \
    tke.cloud.tencent.com/resource-status='{"resources":[{"name":"containerd","limits":{"cpu":"500m","memory":"256Mi"},"requests":{"cpu":"250m","memory":"128Mi"}}]}' \
    --overwrite
# expected: pod/<PodName> annotated
```

期望输出：

```text
pod/<PodName> annotated
```

| 占位符 | 说明 | 获取方式 |
|--------|------|---------|
| `<PodName>` | 目标 Pod 名称 | `kubectl get pods -n <Namespace>` |
| `<Namespace>` | Pod 所在命名空间 | `kubectl get ns` |
| `<ContainerName>` | Pod 中容器名称 | `kubectl describe pod <PodName> -n <Namespace>` |

> **注意**：以上 kubectl 命令需在集群端点可达的环境下执行（同 VPC CVM 或已通过 IOA/VPN/专线接入）。若集群无公网端点且本地不可达，可通过云控制台 Web Shell 或跳板机执行。

## 验证

### 控制面（tccli）

验证节点池注解已正确生效：

```bash
# 查看节点池详情，确认 Annotations 已包含原地升级注解
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId <ClusterId> \
    --NodePoolId <NodePoolId>
# expected: Annotations 中含 node.tke.cloud.tencent.com/in-place-upgrade: "true"
```

预期输出：

```json
{
    "NodePool": {
        "NodePoolId": "np-example",
        "Name": "example-node-pool",
        "LifeState": "normal",
        "Annotations": [
            {
                "Name": "node.tke.cloud.tencent.com/in-place-upgrade",
                "Value": "true"
            },
            {
                "Name": "node.tke.cloud.tencent.com/qos",
                "Value": "{\"memoryQos\":{\"memPageCacheLimit\":{\"enabled\":\"true\"}},\"cpuQos\":{\"podCPUBinding\":{\"enabled\":\"true\"}}}"
            }
        ]
    }
}
```

验证维度：

| 维度 | 命令 | 预期 |
|------|------|------|
| 注解存在 | `DescribeClusterNodePoolDetail` | `Annotations` 含 `in-place-upgrade: "true"` |
| 注解完整 | 同上 | QoS 注解也存在（如设置过） |
| 节点池状态 | 同上，检查 `LifeState` | `normal` |
| 节点就绪 | `DescribeClusterInstances` | 节点 `InstanceState: "running"` |

```bash
# 检查节点池下节点状态
tccli tke DescribeClusterInstances --region <Region> \
    --ClusterId <ClusterId> \
    --Limit 5
# expected: 节点 InstanceState 均为 "running"
```

预期输出：

```json
{
    "TotalCount": 2,
    "InstanceSet": [
        {
            "InstanceId": "ins-example01",
            "InstanceRole": "WORKER",
            "InstanceState": "running",
            "LanIP": "10.0.0.10",
            "NodePoolId": "np-example"
        },
        {
            "InstanceId": "ins-example02",
            "InstanceRole": "WORKER",
            "InstanceState": "running",
            "LanIP": "10.0.0.11",
            "NodePoolId": "np-example"
        }
    ]
}
```

### 数据面（kubectl）

> 以下 kubectl 命令需在集群端点可达的环境下执行。

```bash
# 验证节点注解已下发到节点
kubectl get nodes -l pool-name=<NodePoolLabel> -o json | jq '.items[].metadata.annotations'
# expected: 节点注解中含 node.tke.cloud.tencent.com/in-place-upgrade: "true"
```

预期输出：

```json
{
  "node.tke.cloud.tencent.com/in-place-upgrade": "true",
  "node.tke.cloud.tencent.com/qos": "{\"memoryQos\":{\"memPageCacheLimit\":{\"enabled\":\"true\"}},...}"
}
```

```bash
# 部署测试 Pod 并验证原地升级能力
kubectl describe pod <PodName> -n <Namespace>
# expected: Pod Status 为 Running，Conditions 中 Ready=True
```

## 清理

> **注意**：本页仅修改节点池注解，不创建新资源，无计费影响。如需恢复节点池原始配置，按以下步骤操作。

### 数据面（kubectl）

> 以下 kubectl 命令需在集群端点可达的环境下执行。

```bash
# 1. 如需删除测试 Pod（先清理数据面资源）
kubectl delete pod <PodName> -n <Namespace>
# expected: pod "<PodName>" deleted
```

### 控制面（tccli）

#### 清理前状态检查

```bash
# 2. 查看当前节点池注解
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId <ClusterId> \
    --NodePoolId <NodePoolId>
# 记录当前 Annotations 列表
```

预期输出：

```json
{
    "NodePool": {
        "NodePoolId": "np-example",
        "Name": "example-node-pool",
        "LifeState": "normal",
        "Annotations": [
            {
                "Name": "node.tke.cloud.tencent.com/in-place-upgrade",
                "Value": "true"
            },
            {
                "Name": "node.tke.cloud.tencent.com/qos",
                "Value": "{\"memoryQos\":{\"memPageCacheLimit\":{\"enabled\":\"true\"}},\"cpuQos\":{\"podCPUBinding\":{\"enabled\":\"true\"}}}"
            }
        ]
    }
}
```

#### 恢复节点池原始配置（移除原地升级注解）

```bash
# 3. 将 Annotations 设回原始配置（如原本为空则传空数组）
tccli tke ModifyClusterNodePool --cli-input-json file://inplace-cleanup.json --region <Region>
# expected: exit 0，返回 RequestId
```

`inplace-cleanup.json`：

```json
{
  "ClusterId": "<ClusterId>",
  "NodePoolId": "<NodePoolId>",
  "Annotations": []
}
```

> **⚠️ 警告**：`ModifyClusterNodePool` 调用会立即生效，可能影响正在运行的 Pod 的原地升级能力。建议在低峰期操作，并确认当前无进行中的 Pod 原地升降配任务。

#### 验证已清理

```bash
# 4. 确认注解已移除
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId <ClusterId> \
    --NodePoolId <NodePoolId>
# expected: Annotations 为空或恢复为操作前状态
```

预期输出：

```json
{
    "NodePool": {
        "NodePoolId": "np-example",
        "Name": "example-node-pool",
        "LifeState": "normal",
        "Annotations": []
    }
}
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `tccli tke DescribeClusterKubeconfig` 获取 kubeconfig 成功，但 `kubectl cluster-info` 返回 DNS 解析失败 | `kubectl cluster-info` 报 `dial tcp: lookup <ClusterId>.ccs.tencent-cloud.com: no such host` | 集群仅内网端点可用。公网端点被组织级 CAM 策略以 `tke:clusterExtranetEndpoint=true` 条件硬拒绝（`effect:deny`） | 1) 在同 VPC 内的 CVM 上执行 kubectl 命令；2) 通过 IOA/VPN/专线接入后，使用 `DescribeClusterSecurity` 返回的内网 IP 作为 API Server 地址（`--server https://<内网IP>`）。`kubectl --insecure-skip-tls-verify --server https://<内网IP> cluster-info` |
| `CreateClusterEndpoint --IsExtranet true` 返回 `InvalidParameter.Param` | 检查错误消息含 `CAM deny, condition tke:clusterExtranetEndpoint=true, effect:deny` | 组织级 CAM 策略硬拒绝所有公网端点创建请求 | 此为环境限制，非命令错误。使用内网端点替代方案：IOA/VPN/专线接入集群 VPC |
| `CreateClusterEndpoint --IsExtranet true` 返回 `InvalidParameter`，提示 `SecurityGroup can not be empty` | 命令中未传 `--SecurityGroup` 参数 | 公网端点创建需指定安全组 | 传入安全组 ID（如通过 `vpc DescribeSecurityGroups` 获取），但注意 CAM 策略仍可能拒绝 |

### 修改已提交但原地升级未生效

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `ModifyClusterNodePool` 返回成功，但 Pod 原地升级不生效（Pod 被重建） | `tccli tke DescribeClusterNodePoolDetail --region <Region> --ClusterId <ClusterId> --NodePoolId <NodePoolId>` 查看 `Annotations` | 节点池不是原生节点池——普通节点池（CVM 弹性伸缩）不支持原地升级 | 确认节点池类型为原生节点池。如非原生：需创建原生节点池替代普通节点池 |
| `ModifyClusterNodePool` 第二次调用后，原地升级注解丢失 | `tccli tke DescribeClusterNodePoolDetail` 查看 Annotations，发现 `in-place-upgrade` 注解不在列表中 | `Annotations` 参数是 PUT（整体替换），第二次调用仅传新注解时覆盖了旧注解 | 每次调用前先 `DescribeClusterNodePoolDetail` 获取完整 `Annotations` → 合并新旧注解 → 一次性传入完整数组 |
| Pod `kubectl describe` 仍显示旧资源规格 | `kubectl describe pod <PodName> -n <Namespace>` 查看 `Containers[].Limits/Requests` | 1) 节点池注解未正确设置；2) Pod 不在目标节点池的节点上 | 1) `DescribeClusterNodePoolDetail` 确认注解已生效；2) `kubectl get pod <PodName> -o wide` 确认 Pod 所在节点；3) `DescribeClusterInstances` 确认该节点属于目标节点池 |
| 修改节点池注解后，已有 Pod 自动重启 | 观察 Pod 年龄 `kubectl get pod <PodName> -n <Namespace>` 的 `AGE` 字段 | 某些节点配置变更可能触发节点 taint/evict，导致 Pod 重建 | 此为节点池配置变更的正常行为，非原地升级失败。建议在低峰期操作，并确保 Pod 有合适的反亲和性策略 |

> **保留信息以备工单**：遇到无法解决的错误时，保留 `region`、`ClusterId`、`NodePoolId`、`RequestId` 和完整的请求 JSON，可通过 [提交工单](https://console.cloud.tencent.com/workorder) 获取支持。

## 下一步

- [节点管理概述](https://cloud.tencent.com/document/product/457/79795)
- [创建原生节点池](https://cloud.tencent.com/document/product/457/79798)
- [Kubernetes 官方文档：为 Pod 和容器管理资源](https://kubernetes.io/zh-cn/docs/concepts/configuration/manage-resources-containers/)
- [TKE 原生节点 QoS 配置说明](https://cloud.tencent.com/document/product/457/79796)

## 控制台替代

参见 [Pod 原地升降配 - 控制台操作](https://console.cloud.tencent.com/tke2/native-node-pool)
