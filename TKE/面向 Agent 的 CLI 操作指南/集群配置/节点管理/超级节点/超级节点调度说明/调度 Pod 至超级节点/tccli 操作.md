# 调度 Pod 至超级节点

> 对照官方：[调度 Pod 至超级节点](https://cloud.tencent.com/document/product/457/74016) · page_id `74016` · tccli ≥3.1.107 · API 2018-05-25

## 概述

将 Pod 调度到超级节点上有两种方式：**自动调度**（集群资源不足时按量超级节点自动承接）和**手动调度**（通过 `nodeSelector`、`tolerations` 或 TKE 调度 Annotation 显式指定）。

| 调度方式 | 机制 | 适用场景 |
|---------|------|---------|
| 自动调度 | 按量超级节点池在普通节点资源不足时自动承接 Pod | 弹性场景，无需修改工作负载配置 |
| 手动调度 — nodeSelector | `spec.nodeSelector: {node.kubernetes.io/instance-type: eklet}` | 明确指定 Pod 必须调度到超级节点 |
| 手动调度 — tolerations | 配合超级节点污点，允许 Pod 容忍并调度 | 超级节点配置了 taint 时的必要配置 |
| 手动调度 — affinity | `nodeAffinity` 或 `podAntiAffinity` | 复杂调度策略（跨可用区、互斥等） |

> **注意**：本文档中的 kubectl 命令需要集群端点可达。如因 CAM 策略公网端点被拒绝（见[排障](#排障)），需通过 IOA/VPN/专线接入 VPC 或使用同 VPC 的 CVM 跳板机执行 kubectl 操作。kubectl 命令格式基于 TKE 超级节点标准行为，经控制面 API 间接验证。

## 前置条件

- [环境准备](../../../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
```

**预期输出**：

```text
3.1.107.1
```

```bash
# 2. 检查当前凭据和地域
tccli configure list
```

**预期输出**：

```text
Name    Value
secretId     *********************
secretKey    *********************
region       ap-guangzhou
output       json
```

```bash
# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:DescribeClusterVirtualNodePools, tke:DescribeClusterVirtualNode
#    tke:DescribeClusterKubeconfig, tke:DescribeClusterEndpoints
# 验证：
tccli tke DescribeClusterVirtualNodePools --region <Region> --ClusterId <ClusterId>
# expected: exit 0，返回节点池列表（可为空）
```

```bash
# 4. 检查 kubectl 已安装
kubectl version --client
```

**预期输出**：

```text
Client Version: v1.32.2
Kustomize Version: v5.0.4-0.20230601165947-6ce0bf390ce3
```

### 资源检查

```bash
# 5. 确认超级节点池已创建且状态正常
tccli tke DescribeClusterVirtualNodePools --region <Region> \
    --ClusterId <ClusterId>
# expected: NodePoolSet 非空，目标节点池 LifeState 为 normal
```

**预期输出**：

```json
{
  "TotalCount": 3,
  "NodePoolSet": [
    {
      "NodePoolId": "<NodePoolId>",
      "SubnetIds": [
        "<SubnetId-1>",
        "<SubnetId-2>"
      ],
      "Name": "test-vnp",
      "LifeState": "normal",
      "Labels": [
        {
          "Name": "node.kubernetes.io/instance-type",
          "Value": "eklet"
        }
      ],
      "Taints": []
    }
  ]
}
```

```bash
# 5a. 验证超级节点池关联的子网存在
#     从上一步 NodePoolSet[].SubnetIds 获取子网 ID 列表
tccli vpc DescribeSubnets --region <Region> --SubnetIds '["<SubnetId-1>","<SubnetId-2>"]'
```

**预期输出**：

```json
{
  "TotalCount": 2,
  "SubnetSet": [
    {
      "VpcId": "<VpcId>",
      "SubnetId": "<SubnetId-1>",
      "SubnetName": "<SubnetName>",
      "CidrBlock": "10.0.16.0/20",
      "AvailableIpAddressCount": 4000,
      "Zone": "ap-guangzhou-6",
      "SubnetState": "AVAILABLE"
    }
  ]
}
```

> 若返回空或 `SubnetState` 非 `AVAILABLE`，超级节点无法在该子网内为 Pod 分配网络。需在对应 VPC 下创建子网（参见 [VPC 子网文档](https://cloud.tencent.com/document/product/215)）。

```bash
# 5b. 检查虚拟节点配额余量
#     超级节点池的节点数量受配额限制，Pod 调度时需有可用虚拟节点
tccli tke DescribeClusterVirtualNode --region <Region> \
    --ClusterId <ClusterId> \
    --NodePoolId <NodePoolId>
# expected: TotalCount 未达配额上限，仍有空余（创建 Pod 时系统按需自动创建虚拟节点）
```

```bash
# 6. 确认超级节点已注册且 Running
tccli tke DescribeClusterVirtualNode --region <Region> \
    --ClusterId <ClusterId> \
    --NodePoolId <NodePoolId>
# expected: 至少返回 1 个节点，Phase 为 Running
```

**预期输出**：

```json
{
  "TotalCount": 2,
  "Nodes": [
    {
      "Name": "<NodeName-eklet-xxxx>",
      "SubnetId": "<SubnetId>",
      "Phase": "Running",
      "CreatedTime": "2026-06-23T08:13:53Z"
    },
    {
      "Name": "<NodeName-eklet-xxxx>",
      "SubnetId": "<SubnetId>",
      "Phase": "Running",
      "CreatedTime": "2026-06-23T08:13:52Z"
    }
  ]
}
```

```bash
# 7. 检查集群端点可达性
tccli tke DescribeClusterEndpoints --region <Region> --ClusterId <ClusterId>
# expected: exit 0，确认端点类型和可达性
```

**预期输出**：

```json
{
  "ClusterExternalEndpoint": "",
  "ClusterIntranetEndpoint": "<InternalIP>",
  "ClusterDomain": "<ClusterDomain>",
  "SecurityGroup": ""
}
```

> 若 `ClusterExternalEndpoint` 为空且 `ClusterIntranetEndpoint` 不可从本地访问，kubectl 命令需在 VPC 可达环境下执行（参见[排障](#排障)）。

```bash
# 8. 获取 kubeconfig
tccli tke DescribeClusterKubeconfig --region <Region> \
    --ClusterId <ClusterId> --IsExtranet true \
    | jq -r '.Kubeconfig' > ~/.kube/config-super
# expected: Kubeconfig 写入成功，长度 > 1000 字符
```

**预期输出**（截断）：

```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: <certificate-authority-data>
    server: https://<ClusterDomain>
  name: cls-<ClusterId>
contexts:
- context:
    cluster: cls-<ClusterId>
    user: <UserId>
  name: cls-<ClusterId>-<UserId>-context-default
current-context: cls-<ClusterId>-<UserId>-context-default
kind: Config
preferences: {}
users:
- name: <UserId>
  user:
    client-certificate-data: <client-certificate-data>
    client-key-data: <client-key-data>
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli / kubectl | 幂等 |
|-----------|-----------|:--:|
| 查看超级节点池 | `DescribeClusterVirtualNodePools` | 是 |
| 查看超级节点 | `DescribeClusterVirtualNode` | 是 |
| 获取 kubeconfig | `DescribeClusterKubeconfig` | 是 |
| 查看集群端点 | `DescribeClusterEndpoints` | 是 |
| 查看超级节点（kubectl） | `kubectl get nodes -l node.kubernetes.io/instance-type=eklet` | 是 |
| 创建带节点选择的 Pod | `kubectl apply -f` | 是 |
| 查看 Pod 调度结果 | `kubectl get pod -o wide` | 是 |

## 操作步骤

### 步骤 1：确认超级节点就绪和标签

```bash
tccli tke DescribeClusterVirtualNodePools --region <Region> \
    --ClusterId <ClusterId>
# expected: exit 0，NodePoolSet 中目标节点池 LifeState 为 normal
```

**预期输出**：

```json
{
  "TotalCount": 3,
  "NodePoolSet": [
    {
      "NodePoolId": "<NodePoolId>",
      "SubnetIds": [
        "<SubnetId-1>",
        "<SubnetId-2>"
      ],
      "Name": "test-vnp",
      "LifeState": "normal",
      "Labels": [
        {
          "Name": "node.kubernetes.io/instance-type",
          "Value": "eklet"
        }
      ],
      "Taints": []
    }
  ]
}
```

确认超级节点已 Running：

```bash
tccli tke DescribeClusterVirtualNode --region <Region> \
    --ClusterId <ClusterId> \
    --NodePoolId <NodePoolId>
# expected: 所有节点 Phase 为 Running
```

**预期输出**：

```json
{
  "TotalCount": 6,
  "Nodes": [
    {
      "Name": "<NodeName-eklet-subnet-rdmcho9m-h2qcg3oh>",
      "SubnetId": "<SubnetId>",
      "Phase": "Running",
      "CreatedTime": "2026-06-23T08:13:53Z"
    },
    {
      "Name": "<NodeName-eklet-subnet-rdmcho9m-53ruri19>",
      "SubnetId": "<SubnetId>",
      "Phase": "Running",
      "CreatedTime": "2026-06-23T08:13:52Z"
    },
    {
      "Name": "<NodeName-eklet-subnet-29gwv72o-7rgrxd8v>",
      "SubnetId": "<SubnetId>",
      "Phase": "Running",
      "CreatedTime": "2026-06-23T08:13:53Z"
    },
    {
      "Name": "<NodeName-eklet-subnet-29gwv72o-4opla8eb>",
      "SubnetId": "<SubnetId>",
      "Phase": "Running",
      "CreatedTime": "2026-06-23T08:13:53Z"
    },
    {
      "Name": "<NodeName-eklet-subnet-29gwv72o-c8m2u45l>",
      "SubnetId": "<SubnetId>",
      "Phase": "Running",
      "CreatedTime": "2026-06-23T08:13:52Z"
    },
    {
      "Name": "<NodeName-eklet-subnet-rdmcho9m-nbt2r1j9>",
      "SubnetId": "<SubnetId>",
      "Phase": "Running",
      "CreatedTime": "2026-06-23T08:13:52Z"
    }
  ]
}
```

```bash
kubectl --kubeconfig ~/.kube/config-super get nodes -l node.kubernetes.io/instance-type=eklet
# expected: 返回至少 1 个超级节点
```

**预期输出**：

```text
NAME                                STATUS   ROLES    AGE   VERSION
<NodeName-eklet-xxxx>   Ready    <none>   10m   v1.32.2-tke.1
<NodeName-eklet-xxxx>   Ready    <none>   10m   v1.32.2-tke.1
<NodeName-eklet-xxxx>   Ready    <none>   10m   v1.32.2-tke.1
```

```bash
kubectl --kubeconfig ~/.kube/config-super get nodes <NodeName> --show-labels
# expected: 显示节点完整标签列表，确认 node.kubernetes.io/instance-type=eklet 存在
```

**预期输出**：

```text
NAME                                STATUS   ROLES    AGE   VERSION            LABELS
<NodeName-eklet-xxxx>   Ready    <none>   10m   v1.32.2-tke.1   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,node.kubernetes.io/instance-type=eklet,...
```

### 步骤 2：通过 nodeSelector 调度 Pod 到超级节点

#### 选择依据

- **nodeSelector**：最简单的手动调度方式，适合明确指定 Pod 只跑在超级节点的场景。超级节点默认带有 `node.kubernetes.io/instance-type=eklet` 标签。
- **tolerations**：仅当超级节点配置了特殊 taint 时才需要（新建的超级节点池通常无额外 taint）。
- **自动调度**：如果按量超级节点池已创建且 Pod 未指定 nodeSelector，普通节点资源不足时自动调度到超级节点——无需修改 YAML。
- **资源 Annotation**：超级节点上的 Pod **必须**通过 Annotation（`eks.tke.cloud.tencent.com/cpu` 和 `eks.tke.cloud.tencent.com/mem`）指定资源规格，不同于普通节点的 `resources.requests`。未指定会导致 Pod 无法创建。

**超级节点资源 Annotation 关键字段说明**：

| 字段名 | 类型 | 必填 | 取值与约束 | 错误后果 |
|--------|------|:--:|---------|---------|
| `eks.tke.cloud.tencent.com/cpu` | String (Annotation) | 是 | CPU 核数，字符串格式，如 `"1"` `"2"` `"4"`。须为整数或 0.25 的倍数，不超过超级节点单 Pod CPU 上限 | Pod 创建失败，Events 提示资源规格未指定或超出允许范围 |
| `eks.tke.cloud.tencent.com/mem` | String (Annotation) | 是 | 内存量，K8s 资源量格式，如 `"2Gi"` `"4Gi"` `"8Gi"`。须符合 K8s 资源量单位（Ki/Mi/Gi），CPU 与内存比例有上限约束 | Pod 创建失败，Events 提示资源规格未指定或比例违规 |

> **重要**：超级节点上的 Pod **不**使用 `spec.containers[].resources.requests` 作为调度依据。上述两个 Annotation 同时指定才可成功创建 Pod。Annotation 的值同时影响计费——超级节点按 Annotation 中声明的 CPU 和内存规格计费（而非实际使用量）。

#### 最小示例（nodeSelector + 资源 Annotation）

`pod-schedule-minimal.yaml`：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: <PodName>
  annotations:
    eks.tke.cloud.tencent.com/cpu: "1"
    eks.tke.cloud.tencent.com/mem: "2Gi"
spec:
  nodeSelector:
    node.kubernetes.io/instance-type: eklet
  containers:
  - name: <ContainerName>
    image: nginx:1.25
    resources:
      requests:
        cpu: "1"
        memory: "2Gi"
```

> **计费提示**：超级节点上的 Pod 按 vCPU·秒和 GiB·秒计费，Pod 进入 Running 状态即持续产生费用。超级节点按 Pod Annotation 中声明的 CPU/内存规格（而非实际使用量）计费。测试完成后请及时执行[清理](#清理)。

```bash
kubectl --kubeconfig ~/.kube/config-super apply -f pod-schedule-minimal.yaml
# expected: pod/<PodName> created
```

#### 增强示例（nodeSelector + affinity + tolerations）

`pod-schedule-enhanced.yaml`：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: <PodName>
  annotations:
    eks.tke.cloud.tencent.com/cpu: "2"
    eks.tke.cloud.tencent.com/mem: "4Gi"
spec:
  nodeSelector:
    node.kubernetes.io/instance-type: eklet
  tolerations:
  # 此 toleration 仅在超级节点配置了匹配 taint 时生效。
  # 当前新建的超级节点池通常无额外 taint，未匹配时不影响调度。
  - key: "eks.tke.cloud.tencent.com/eklet"
    operator: "Exists"
    effect: "NoSchedule"
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: app
              operator: In
              values:
              - <AppLabel>
          topologyKey: kubernetes.io/hostname
  containers:
  - name: <ContainerName>
    image: nginx:1.25
    resources:
      requests:
        cpu: "2"
        memory: "4Gi"
      limits:
        cpu: "2"
        memory: "4Gi"
      # 计费注意：超级节点按 resources.limits 而非 resources.requests 计费。
      # 上例中 limits 与 requests 相等（2C4G），不相等时以 limits 值为计费基准。
```

> **计费提示**：增强示例中 `limits` 设为 2C4G，超级节点将以此规格计费（约 2 vCPU·秒 + 4 GiB·秒）。若降低 `requests` 为 1C2G 但保留 `limits` 为 2C4G，计费仍按 2C4G 计算。

```bash
kubectl --kubeconfig ~/.kube/config-super apply -f pod-schedule-enhanced.yaml
# expected: pod/<PodName> created
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<PodName>` | Pod 名称 | 须遵循 K8s 命名规范 | 自定义 |
| `<ContainerName>` | 容器名称 | 须遵循 K8s 命名规范 | 自定义 |
| `<AppLabel>` | 应用标签 | 用于 podAntiAffinity | 自定义 |
| `<NodeName>` | 超级节点名称 | 以 `eklet` 开头 | `kubectl get nodes -l node.kubernetes.io/instance-type=eklet` |

## 验证

### 控制面（tccli）

```bash
# 确认超级节点池状态
tccli tke DescribeClusterVirtualNodePools --region <Region> \
    --ClusterId <ClusterId>
# expected: NodePoolSet 非空，LifeState 为 normal
```

**预期输出**：

```json
{
  "TotalCount": 3,
  "NodePoolSet": [
    {
      "NodePoolId": "<NodePoolId>",
      "SubnetIds": [
        "<SubnetId-1>",
        "<SubnetId-2>"
      ],
      "Name": "test-vnp",
      "LifeState": "normal",
      "Labels": [
        {
          "Name": "node.kubernetes.io/instance-type",
          "Value": "eklet"
        }
      ],
      "Taints": []
    }
  ]
}
```

```bash
# 确认超级节点 Running
tccli tke DescribeClusterVirtualNode --region <Region> \
    --ClusterId <ClusterId> \
    --NodePoolId <NodePoolId>
# expected: 所有超级节点 Phase 为 Running
```

**预期输出**：

```json
{
  "TotalCount": 6,
  "Nodes": [
    {
      "Name": "<NodeName-eklet-xxxx>",
      "SubnetId": "<SubnetId>",
      "Phase": "Running",
      "CreatedTime": "2026-06-23T08:13:53Z"
    },
    {
      "Name": "<NodeName-eklet-xxxx>",
      "SubnetId": "<SubnetId>",
      "Phase": "Running",
      "CreatedTime": "2026-06-23T08:13:52Z"
    }
  ]
}
```

### 数据面（kubectl）

> **注意**：以下 kubectl 命令需要集群端点可达。kubectl 命令格式基于 TKE 超级节点标准行为，经控制面 API 间接验证。

```bash
# 验证 Pod 调度到超级节点
kubectl --kubeconfig ~/.kube/config-super get pod <PodName> -o wide
```

**预期输出**：

```text
NAME        READY   STATUS    RESTARTS   AGE   IP           NODE                                NOMINATED NODE   READINESS GATES
<PodName>   1/1     Running   0          2m    10.0.16.51   <NodeName-eklet-subnet-rdmcho9m-h2qcg3oh>   <none>           <none>
```

```bash
# 验证 Pod 详情 — nodeSelector 生效
kubectl --kubeconfig ~/.kube/config-super describe pod <PodName> | grep "Node-Selectors"
```

**预期输出**：

```text
Node-Selectors:  node.kubernetes.io/instance-type=eklet
```

```bash
# 验证 Pod 资源 Annotation 已设置
kubectl --kubeconfig ~/.kube/config-super describe pod <PodName> | grep -A5 Annotations
```

**预期输出**：

```text
Annotations:      eks.tke.cloud.tencent.com/cpu: 1
                  eks.tke.cloud.tencent.com/mem: 2Gi
                  ...
                  kubernetes.io/psp: privileged
                  ...
```

```bash
# 验证 Pod 未调度到普通节点
kubectl --kubeconfig ~/.kube/config-super get pod <PodName> -o json \
    | jq '.spec.nodeName'
```

**预期输出**：

```text
"<NodeName-eklet-subnet-rdmcho9m-h2qcg3oh>"
```

| 验证维度 | 命令 | 预期 |
|---------|------|------|
| 节点池就绪 | `DescribeClusterVirtualNodePools --ClusterId <ClusterId>` | `LifeState: "normal"` |
| 超级节点 Running | `DescribeClusterVirtualNode --ClusterId <ClusterId> --NodePoolId <NodePoolId>` | 所有节点 `Phase: "Running"` |
| Pod 状态 | `kubectl get pod <PodName>` | `STATUS: "Running"` |
| Pod 节点 | `kubectl get pod <PodName> -o wide` | NODE 以 `eklet` 开头 |
| nodeSelector | `kubectl describe pod <PodName> \| grep "Node-Selectors"` | 含 `node.kubernetes.io/instance-type=eklet` |
| 资源 Annotation | `kubectl describe pod <PodName> \| grep -A5 Annotations` | 含 `eks.tke.cloud.tencent.com/cpu` 和 `mem` |
| 调度策略 | `kubectl get pod <PodName> -o json \| jq '.spec.nodeSelector'` | 声明的 nodeSelector 均匹配 |

## 清理

> **警告**：删除 Pod 将清除该 Pod 的运行时数据。如 Pod 为 Deployment/DaemonSet/StatefulSet 管理，直接删除 Pod 后控制器会自动重建——需先删除上层控制器或缩容至 0。
> **kubectl 环境要求**：以下 kubectl 命令需集群端点可达。若公网端点被 CAM 策略拒绝且无 VPC 接入，需通过 IOA/VPN/专线 或同 VPC CVM 跳板机执行（参见[排障](#排障)）。

```bash
# 清理前确认 Pod 状态
kubectl --kubeconfig ~/.kube/config-super get pod <PodName>
```

**预期输出**：

```text
NAME        READY   STATUS    RESTARTS   AGE
<PodName>   1/1     Running   0          5m
```

```bash
# 删除测试 Pod
kubectl --kubeconfig ~/.kube/config-super delete pod <PodName>
```

**预期输出**：

```text
pod "<PodName>" deleted
```

```bash
# 验证 Pod 已删除
kubectl --kubeconfig ~/.kube/config-super get pod <PodName>
```

**预期输出**：

```text
Error from server (NotFound): pods "<PodName>" not found
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `CreateClusterEndpoint --IsExtranet true` 返回 `InvalidParameter.Param`，error 含 `strategyId:240463971` 和 `tke:clusterExtranetEndpoint=true` | 查看错误详情中的 CAM 策略拒绝信息 | 组织级 CAM 策略 `strategyId:240463971` 对 `tke:clusterExtranetEndpoint=true` 设置了硬拒绝（effect: deny）。更换安全组无法绕过——策略在 CAM 层生效 | 改用内网端点（`--IsExtranet false`），kubectl 需通过 IOA/VPN/专线接入 VPC 或使用同 VPC 的 CVM 跳板机。此限制为环境级组织策略，非命令错误 |
| `kubectl apply` 返回 `Error from server (Forbidden)` | `kubectl --kubeconfig ~/.kube/config-super auth can-i create pods` | 子账号缺少 Pod 创建权限 | 联系集群管理员授予 namespace 级别的 Pod 创建权限 |
| `kubectl get nodes -l node.kubernetes.io/instance-type=eklet` 无节点 | `tccli tke DescribeClusterVirtualNodePools --region <Region> --ClusterId <ClusterId>` | 超级节点池未创建或节点未注册 | 先执行 [新建超级节点](../../使用超级节点/新建超级节点/tccli%20操作.md) |
| `DescribeClusterKubeconfig` 返回 `InvalidParameter.ClusterId` | `tccli tke DescribeClusters --region <Region>` 列出集群 | 集群 ID 错误或不存在 | 用 `tccli tke DescribeClusters --region <Region>` 获取正确 ClusterId |

### 创建成功但状态异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| kubectl 连接集群超时或无响应 | `tccli tke DescribeClusterEndpoints --region <Region> --ClusterId <ClusterId>` 检查端点配置 | 公网端点被 CAM 策略拒绝（`strategyId:240463971`），内网端点仅 VPC 内可达。本地 macOS 无法访问 VPC 内网 IP | 通过 IOA/VPN/专线接入 VPC；或使用同 VPC 的 CVM 跳板机执行 kubectl；或联系管理员调整 CAM 策略 `strategyId:240463971` 中 `tke:clusterExtranetEndpoint` 条件 |
| Pod 调度到了普通节点 | `kubectl get pod <PodName> -o wide` 查看 NODE | 未设置 `nodeSelector` 或 `nodeSelector` 键名不匹配 | 检查 YAML 中 `nodeSelector` 的键名是否为 `node.kubernetes.io/instance-type`，值为 `eklet`。验证标签名称：`kubectl get nodes --show-labels \| grep instance-type` |
| Pod Pending，Events 显示 `0/1 nodes are available: 1 node(s) didn't match node selector` | `kubectl describe pod <PodName> \| tail -20` | nodeSelector 标签不存在于任何节点上 | 检查超级节点实际标签：`kubectl get nodes -l node.kubernetes.io/instance-type=eklet` 确认标签名是否正确 |
| Pod Pending，Events 显示 `node(s) had untolerated taint` | `kubectl describe node <NodeName> \| grep Taints` | 超级节点带有额外 taint，Pod 未声明对应 tolerations | 在 Pod spec 中增加对超级节点 taint 的 tolerations 配置 |
| 自动调度未触发（普通节点资源充足时 Pod 未调度到按量超级节点） | `kubectl describe pod <PodName> \| tail -20` | 此为目标行为：自动调度仅在普通节点资源不足时触发 | 如需强制调度到超级节点，添加 `nodeSelector: {node.kubernetes.io/instance-type: eklet}` |
| 按量超级节点上 Pod 被优先缩容 | `kubectl get pod -o wide` 查看 Pod 分布 | 此为设计行为：资源充足时按量超级节点上的 Pod 优先被缩容 | 如不希望 Pod 被优先缩容，考虑使用包年包月超级节点或添加 Pod Disruption Budget |
| Pod 创建失败，Events 提示资源规格未指定 | `kubectl describe pod <PodName> \| grep -A10 Events` | 超级节点上 Pod 缺少 `eks.tke.cloud.tencent.com/cpu` 和 `eks.tke.cloud.tencent.com/mem` Annotation | 在 Pod `metadata.annotations` 中添加 `eks.tke.cloud.tencent.com/cpu` 和 `eks.tke.cloud.tencent.com/mem`，值分别为 CPU 核数（如 `"1"`）和内存量（如 `"2Gi"`） |

## 下一步

- [超级节点 Annotation 说明](https://cloud.tencent.com/document/product/457/44173) — 完整 Annotation 列表
- [超级节点可调度 Pod 说明](https://cloud.tencent.com/document/product/457/74015) — 可调度规格范围
- [指定资源规格](https://cloud.tencent.com/document/product/457/44174) — CPU/内存/GPU 规格配置
- [超级节点常见问题](https://cloud.tencent.com/document/product/457/60411) — 常见 FAQ
- [超级节点上支持运行 DaemonSet](https://cloud.tencent.com/document/product/457/98730) — DaemonSet 调度配置

## 控制台替代

[TKE 控制台 → 工作负载 → 新建/编辑 → 调度策略](https://console.cloud.tencent.com/tke2/cluster)：在调度策略中选择「调度到超级节点」。
