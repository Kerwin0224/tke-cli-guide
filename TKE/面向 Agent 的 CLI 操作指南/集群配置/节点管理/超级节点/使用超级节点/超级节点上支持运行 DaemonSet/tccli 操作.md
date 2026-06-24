# 超级节点上运行 DaemonSet

> 对照官方：[超级节点上支持运行 DaemonSet](https://cloud.tencent.com/document/product/457/98730) · page_id `98730` · tccli ≥3.1.107.1 · API 2026-06-23

## 概述

超级节点通过 **DaemonSet Pod 注入** 机制支持运行 DaemonSet。与传统节点（每个节点独立运行一个 DaemonSet Pod）不同，超级节点上的 DaemonSet Pod 以 **sidecar 容器** 形式注入到已有的业务 Pod 中。被注入的业务 Pod YAML 配置不变，但 Container Status 中会新增注入的 DaemonSet 容器状态。

| 方案 | 机制 | 适用场景 |
|------|------|---------|
| DaemonSet Pod 注入（推荐） | 通过 `eks.tke.cloud.tencent.com/ds-injection: "true"` 注解，DaemonSet Pod 以 sidecar 容器形式注入到超级节点上的业务 Pod | 超级节点上运行日志采集、监控、安全 Agent 等常驻 DaemonSet |
| 传统节点 DaemonSet | 每个节点独立调度一个 Pod | 普通节点/CVM 节点 |

**核心特点**：

- DaemonSet Pod 不独立占用节点资源，以 sidecar 形式注入业务 Pod，**不产生额外计费**
- DaemonSet 中定义的 `resources` 声明**不会生效**（官方明确）
- `kube-system` 命名空间下的 Pod **不会被注入** DaemonSet Pod
- 支持与 Kubernetes 原生一致的滚动更新

## 前置条件

- [环境准备](../../../../../环境准备.md) <!-- TODO: 多平台发布时需将相对链接转换为平台兼容格式（iWiki docid / 写写 nodeId） -->

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查 kubectl 版本（需要 >= 1.16.0；>= 1.18 有容器校验兼容性问题，见排障段）
kubectl version --client
# expected: Client Version: >= v1.16.0

# 3. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置，region 为目标地域

# 4. 检查 CAM 权限 — 需要以下 Action：
#    tke:DescribeClusterVirtualNodePools
#    tke:DescribeClusterVirtualNode
#    tke:DescribeClusterEndpoints
#    tke:DescribeClusterKubeconfig
#    tke:DescribeClusters
# 验证：
tccli tke DescribeClusterVirtualNodePools --region <Region> --ClusterId <ClusterId>
# expected: exit 0，返回节点池列表（可为空）；若返回 UnauthorizedOperation，需在 CAM 控制台为当前子账号授予上述 Action 权限
```

### 资源检查

```bash
# 5. 确认超级节点池已创建且状态正常
tccli tke DescribeClusterVirtualNodePools --region <Region> \
    --ClusterId <ClusterId>
# expected: TotalCount >= 1，目标节点池 LifeState 为 normal
```

```json
{
  "TotalCount": 1,
  "NodePoolSet": [
    {
      "NodePoolId": "np-example",
      "SubnetIds": ["subnet-example"],
      "Name": "example-vnp",
      "LifeState": "normal"
    }
  ]
}
```
<!-- 从以上输出中提取 NodePoolId 字段值，替换后续命令中的 <NodePoolId> 占位符 -->
```bash
# 6. 确认虚拟节点已就绪
tccli tke DescribeClusterVirtualNode --region <Region> \
    --ClusterId <ClusterId> \
    --NodePoolId <NodePoolId>
# expected: Nodes 非空，Phase 为 Running
```

```json
{
  "Nodes": [
    {
      "Name": "eklet-subnet-example",
      "SubnetId": "subnet-example",
      "Phase": "Running"
    }
  ],
  "TotalCount": 1
}
```

### 版本要求

根据官方要求，DaemonSet 注入需要集群控制面和超级节点组件满足最低版本：

| K8s 版本 | 控制面最低版本 | 超级节点组件最低版本 |
|---------|--------------|------------------|
| 1.34 | v1.34.0-tke.1 | v3.4.10 |
| 1.32 | v1.32.0-tke.1 | v2.16.10 |
| 1.30 | v1.30.0-tke.1 | v2.15.10 |
| 1.28 | v1.28.0-tke.1 | v2.10.0 |

> **注意**：超级节点组件版本当前无法通过 CLI 直接查询验证。如不确定版本，请登录 [TKE 控制台](https://console.cloud.tencent.com/tke2/cluster) 查看集群详情确认。

```bash
# 确认集群版本
tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'
# expected: ClusterVersion 满足上表要求
```

```json
{
  "TotalCount": 1,
  "Clusters": [
    {
      "ClusterId": "cls-example",
      "ClusterName": "example-cluster",
      "ClusterVersion": "1.34.0-tke.1",
      "ClusterStatus": "Running"
    }
  ]
}
```

### kubectl 连通性

DaemonSet 操作通过 kubectl 执行，需确保集群端点可达。

```bash
# 获取集群端点信息
tccli tke DescribeClusterEndpoints --region <Region> --ClusterId <ClusterId>
# expected: 返回 ClusterExternalEndpoint 或 ClusterIntranetEndpoint
```

```json
{
  "ClusterExternalEndpoint": "",
  "ClusterIntranetEndpoint": "10.0.0.1",
  "ClusterDomain": "cls-example.ccs.tencent-cloud.com",
  "ClusterExternalDomain": "cls-example.ccs.tencent-cloud.com",
  "ClusterIntranetSubnetId": "subnet-example",
  "SecurityGroup": "sg-example"
}
```

```bash
# 获取 kubeconfig
tccli tke DescribeClusterKubeconfig --region <Region> \
    --ClusterId <ClusterId> \
    --IsExtranet false | jq -r '.Kubeconfig' > ~/.kube/config-super
# 输出已写入 ~/.kube/config-super（标准 kubeconfig YAML 格式，含 clusters/users/contexts 三段结构）

# 验证 kubectl 可达
kubectl --kubeconfig ~/.kube/config-super cluster-info
# expected: Kubernetes control plane is running at ...
```

```text
Kubernetes control plane is running at https://cls-example.ccs.tencent-cloud.com
CoreDNS is running at https://cls-example.ccs.tencent-cloud.com/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
```

> **注意**：若集群仅有内网端点（无公网端点），需在同 VPC CVM 上或通过 IOA/VPN/专线 执行 kubectl 命令。公网端点创建受组织级 CAM 策略限制，详见 [排障](#排障)。

> **若虚拟节点池不存在**：参考[新建超级节点](https://cloud.tencent.com/document/product/457/80229)创建节点池和虚拟节点后再继续本文档。

## 控制台与 CLI 参数映射

| 控制台操作 | tccli / kubectl | 幂等 |
|-----------|-----------|:--:|
| 查看超级节点池 | `tccli tke DescribeClusterVirtualNodePools` | 是 |
| 查看虚拟节点 | `tccli tke DescribeClusterVirtualNode` | 是 |
| 获取集群端点 | `tccli tke DescribeClusterEndpoints` | 是 |
| 获取 kubeconfig | `tccli tke DescribeClusterKubeconfig` | 是 |
| 创建/更新 DaemonSet | `kubectl apply -f` | 是 |
| 查看 DaemonSet 状态 | `kubectl get daemonset` | 是 |
| 查看 DaemonSet Pod | `kubectl get pod -l app=<Label>` | 是 |

以下说明 DaemonSet YAML 中与超级节点注入相关的主要字段。

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `eks.tke.cloud.tencent.com/ds-injection` | String (annotation) | **是** | `"true"`。激活 DaemonSet Pod 注入机制，DaemonSet Pod 以 sidecar 容器注入业务 Pod | 不加此注解 → DaemonSet 正常创建但 Pod 不会注入到超级节点业务 Pod 中 |
| `spec.template.spec.tolerations` | Array | **是** | 必须包含 `{key: eks.tke.cloud.tencent.com/eklet, operator: Exists, effect: NoSchedule}`。超级节点默认带此污点 | 缺 tolerations → DaemonSet Pod 无法创建到超级节点 |
| `spec.template.spec.containers[].resources` | Object | **否（且不推荐）** | 官方明确：调度时 DaemonSet 中定义的 resources 不会有效，也不额外计费。填了也不会报错，但被忽略 | 填了 resources → 静默忽略，无错误但无效果 |
| `eks.tke.cloud.tencent.com/ds-inject-rate` | String (annotation) | 否 | `"50%"` 等百分比值。控制注入速率，仅在滚动更新场景按节点维度生效 | 不加 → 默认行为（由集群 updateStrategy 控制） |
| `spec.updateStrategy.rollingUpdate.maxUnavailable` | Int | 否 | 与 Kubernetes 原生一致。设为 `0` 可暂停发布 | 不设 → 由集群维度默认值控制 |

## 操作步骤

### 步骤 1：确认超级节点池和虚拟节点就绪

```bash
tccli tke DescribeClusterVirtualNodePools --region <Region> \
    --ClusterId <ClusterId>
# expected: exit 0，目标节点池 LifeState 为 normal
```

```json
{
  "TotalCount": 1,
  "NodePoolSet": [
    {
      "NodePoolId": "np-example",
      "SubnetIds": ["subnet-example"],
      "Name": "example-vnp",
      "LifeState": "normal"
    }
  ]
}
```
<!-- 从以上输出中提取 NodePoolId 字段值，替换后续命令中的 <NodePoolId> 占位符 -->
```bash
tccli tke DescribeClusterVirtualNode --region <Region> \
    --ClusterId <ClusterId> \
    --NodePoolId <NodePoolId>
# expected: exit 0，Nodes 非空，Phase 为 Running
```

```json
{
  "Nodes": [
    {"Name": "eklet-subnet-example", "SubnetId": "subnet-example", "Phase": "Running"}
  ],
  "TotalCount": 1
}
```

### 步骤 2：获取 kubeconfig 并验证连通性

```bash
# 获取 kubeconfig
tccli tke DescribeClusterKubeconfig --region <Region> \
    --ClusterId <ClusterId> \
    --IsExtranet false | jq -r '.Kubeconfig' > ~/.kube/config-super

# 验证连通性
kubectl --kubeconfig ~/.kube/config-super cluster-info
# expected: Kubernetes control plane is running at ...
```

> **注意**：若无公网端点，需确保执行环境已通过 IOA/VPN/专线/同 VPC CVM 连接到集群内网端点。

### 步骤 3：创建支持超级节点的 DaemonSet

#### 选择依据

*为什么选这种配置：*

- **DaemonSet Pod 注入机制**：超级节点使用 Nodeless 架构，不支持传统 DaemonSet 调度（每个节点独立运行一个 Pod）。必须通过 `eks.tke.cloud.tencent.com/ds-injection: "true"` 注解激活注入机制，DaemonSet Pod 以 sidecar 容器形式注入到已有业务 Pod 中。这是腾讯云业内唯一的 Nodeless DaemonSet 方案。
- **eklet 污点容忍**：超级节点默认打有污点 `eks.tke.cloud.tencent.com/eklet`（effect: NoSchedule），DaemonSet Pod 必须声明对应 tolerations 才能调度。不加 tolerations 会导致 DaemonSet Pod 无法创建到超级节点。
- **不声明 resources**：官方明确说明，调度时 DaemonSet 中定义的 resources 不会有效，也不会产生额外计费。因此 DaemonSet YAML 中无需声明 `resources.requests` 或 `resources.limits`。
- **容器命名**：DaemonSet 容器名应与业务 Pod 容器名区分，否则 `kubectl exec` 可能优先指向业务 Pod 容器（虽然 DaemonSet 容器仍在运行）。
- **kube-system 例外**：`kube-system` 命名空间下的 Pod 不会被注入 DaemonSet Pod，如果 DaemonSet 预期在 kube-system 下运行需评估替代方案。

#### 最小示例（仅含注入注解 + 容忍 + 基础容器）

`daemonset-inject-minimal.yaml`：

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: <DaemonSetName>
  namespace: <Namespace>
spec:
  selector:
    matchLabels:
      app: <Label>
  template:
    metadata:
      labels:
        app: <Label>
      annotations:
        eks.tke.cloud.tencent.com/ds-injection: "true"
    spec:
      tolerations:
      - key: "eks.tke.cloud.tencent.com/eklet"
        operator: "Exists"
        effect: "NoSchedule"
      containers:
      - name: <ContainerName>
        image: nginx:1.25
```

```bash
kubectl --kubeconfig ~/.kube/config-super apply -f daemonset-inject-minimal.yaml
# expected: exit 0，DaemonSet 已创建
```

```text
daemonset.apps/<DaemonSetName> created
```

#### 增强配置（加滚动更新 + 注入速率控制）

`daemonset-inject-enhanced.yaml`：

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: <DaemonSetName>
  namespace: <Namespace>
spec:
  selector:
    matchLabels:
      app: <Label>
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  template:
    metadata:
      labels:
        app: <Label>
      annotations:
        eks.tke.cloud.tencent.com/ds-injection: "true"
        eks.tke.cloud.tencent.com/ds-inject-rate: "50%"
    spec:
      tolerations:
      - key: "eks.tke.cloud.tencent.com/eklet"
        operator: "Exists"
        effect: "NoSchedule"
      containers:
      - name: <ContainerName>
        image: nginx:1.25
        ports:
        - containerPort: 80
```

```bash
kubectl --kubeconfig ~/.kube/config-super apply -f daemonset-inject-enhanced.yaml
# expected: exit 0，DaemonSet 已更新
```

```text
daemonset.apps/<DaemonSetName> configured
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<DaemonSetName>` | DaemonSet 名称 | 须遵循 K8s 命名规范，建议与业务容器名区分 | 自定义 |
| `<Namespace>` | K8s 命名空间 | 不能为 `kube-system`（该命名空间 Pod 不会被注入） | `kubectl get ns` |
| `<ContainerName>` | 容器名称 | 须遵循 K8s 命名规范，避免与业务 Pod 容器重名 | 自定义 |
| `<Label>` | 应用标签 | 须与 selector 一致 | 自定义 |
| `<Region>` | 地域 | 如 `ap-guangzhou` | `tccli tke DescribeRegions` |
| `<ClusterId>` | 集群 ID | 格式 `cls-xxxxxxxx` | `tccli tke DescribeClusters` |
| `<NodePoolId>` | 超级节点池 ID | 格式 `np-xxxxxxxx` | `tccli tke DescribeClusterVirtualNodePools` |

| 层级 | 包含字段 | 目的 |
|------|---------|------|
| **最小示例** | `ds-injection` 注解 + `tolerations`（eklet）+ 容器镜像 | 让读者 5 分钟内理解并跑通注入机制 |
| **增强配置** | 最小基础上 + `updateStrategy`（滚动更新）+ `ds-inject-rate`（注入速率控制）+ 端口 | 生产环境推荐配置 |

## 验证

### 控制面（tccli）

```bash
# 验证超级节点池状态
tccli tke DescribeClusterVirtualNodePools --region <Region> \
    --ClusterId <ClusterId>
# expected: exit 0，NodePoolSet 非空，LifeState 为 normal

# 验证虚拟节点状态
tccli tke DescribeClusterVirtualNode --region <Region> \
    --ClusterId <ClusterId> \
    --NodePoolId <NodePoolId>
# expected: exit 0，Nodes 非空，Phase 为 Running
```

### 数据面（kubectl）

```bash
# 1. 验证 DaemonSet 状态
kubectl --kubeconfig ~/.kube/config-super get ds <DaemonSetName> -n <Namespace>
# expected: exit 0，DESIRED == CURRENT
```

```text
NAME              DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
<DaemonSetName>   3         3         3       3            3           <none>          2m
```

```bash
# 2. 验证 DaemonSet Pod 已注入到业务 Pod
kubectl --kubeconfig ~/.kube/config-super get pod -n <Namespace> -o wide
# expected: 业务 Pod 的 READY 列显示 2/2（业务容器 + DaemonSet 容器）
```

```text
NAME                     READY   STATUS    RESTARTS   AGE   NODE
<BusinessPod>            2/2     Running   0          5m    eklet-subnet-example
```

```bash
# 3. 验证注入的容器在 Pod 中可见
kubectl --kubeconfig ~/.kube/config-super describe pod <PodName> -n <Namespace>
# expected: Containers 列表中包含注入的 DaemonSet 容器名
```

```text
Containers:
  <BusinessContainer>:
    Container ID:   containerd://abc123
    Image:          nginx:1.25
    State:          Running
  <DaemonSetName>:
    Container ID:   containerd://def456
    Image:          nginx:1.25
    State:          Running
```

```bash
# 4. 登录 DaemonSet 容器（须指定 -c 参数）
kubectl --kubeconfig ~/.kube/config-super exec -it <PodName> \
    -n <Namespace> \
    -c <DaemonSetName> -- /bin/sh
# expected: 进入 DaemonSet 容器 shell

# 5. 查看 DaemonSet 容器日志（须指定 -c 参数）
kubectl --kubeconfig ~/.kube/config-super logs <PodName> \
    -n <Namespace> \
    -c <DaemonSetName>
# expected: 返回 DaemonSet 容器的日志输出
```

```text
2026-06-23 08:15:00 INFO  Starting daemonset container...
2026-06-23 08:15:01 INFO  Loading configuration...
2026-06-23 08:15:02 INFO  DaemonSet container ready.
```

> **注意**：`kubectl >= 1.18` 客户端会在发起 `logs` 请求前校验 Pod Spec 是否包含指定容器，因容器是注入的而非 Pod 原始定义的一部分，会报错 `container xxx is not valid for pod xxx`。解决方案：使用 `kubectl 1.16` 版本，详见 [排障](#排障)。

| 验证维度 | 命令 | 预期 |
|---------|------|------|
| 节点池就绪 | `DescribeClusterVirtualNodePools` | `LifeState: "normal"` |
| 虚拟节点运行 | `DescribeClusterVirtualNode` | `Phase: "Running"` |
| DaemonSet 创建 | `kubectl get ds <DaemonSetName>` | DESIRED > 0，状态正常 |
| 容器注入 | `kubectl describe pod <PodName>` | Containers 列表中含 DaemonSet 容器 |
| Pod 容器数 | `kubectl get pod -o wide` | READY 列显示业务容器数 + 注入的 DaemonSet 容器数 |

## 清理

> **警告**：删除 DaemonSet 会将 DaemonSet Pod 从**所有被注入的业务 Pod** 中移除（sidecar 容器消失），可能影响日志采集、监控、安全等功能。如果 DaemonSet 仅运行在超级节点上（排他性），删除后所有相关 sidecar 容器消失且无恢复手段。
>
> 清理前建议先备份 DaemonSet 配置：`kubectl get ds <DaemonSetName> -n <Namespace> -o yaml > ds-backup.yaml`

### 控制面（tccli）

此操作不涉及 tccli 控制面资源删除。

### 数据面（kubectl）

```bash
# 1. 清理前状态确认
kubectl --kubeconfig ~/.kube/config-super get ds <DaemonSetName> -n <Namespace>
# 确认是待删除的目标 DaemonSet
```

```text
NAME              DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
<DaemonSetName>   3         3         3       3            3           <none>          10m
```

```bash
# 2. 备份配置（推荐）
kubectl --kubeconfig ~/.kube/config-super get ds <DaemonSetName> \
    -n <Namespace> -o yaml > ds-backup.yaml
# 输出已写入 ds-backup.yaml（DaemonSet 完整 YAML 配置，含 spec/status/metadata，可用于 kubectl apply 恢复）

# 3. 删除 DaemonSet
kubectl --kubeconfig ~/.kube/config-super delete ds <DaemonSetName> -n <Namespace>
# expected: DaemonSet 已删除
```

```text
daemonset.apps "<DaemonSetName>" deleted
```

```bash
# 4. 验证已删除
kubectl --kubeconfig ~/.kube/config-super get ds <DaemonSetName> -n <Namespace>
# expected: DaemonSet 不存在
```

```text
Error from server (NotFound): daemonsets.apps "<DaemonSetName>" not found
```

```bash
# 5. 验证业务 Pod 中 sidecar 容器已移除
kubectl --kubeconfig ~/.kube/config-super get pod -n <Namespace> -o wide
# expected: 业务 Pod 的 READY 列恢复为仅业务容器数（如 1/1）
```

```text
NAME                     READY   STATUS    RESTARTS   AGE   NODE
<BusinessPod>            1/1     Running   0          15m   eklet-subnet-example
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `CreateClusterEndpoint` 返回 `InvalidParameter.Param` + `SecurityGroup can not be empty` | 检查请求是否传了 `SecurityGroup` 参数 | 创建公网端口时 SecurityGroup 不能为空 | 指定一个已有的安全组 ID：`tccli vpc DescribeSecurityGroups --region <Region>` |
| `CreateClusterEndpoint` 返回 `InvalidParameter.Param` + `CAM deny: tke:clusterExtranetEndpoint=true, strategyId:240463971` | 检查 CAM 策略限制：`tccli cam GetUserPermissionBoundary` 或登录控制台查看组织策略 | 组织级 CAM 策略 strategyId:240463971 以 `tke:clusterExtranetEndpoint=true` 条件硬拒绝公网端点创建。自建安全组无法绕过 | 使用内网端点作为替代方案：`DescribeClusterEndpoints` 获取 `ClusterIntranetEndpoint`，通过 IOA/VPN/专线/同 VPC CVM 接入集群执行 kubectl 命令 |
| `CreateClusterEndpoint`（内网）返回 `OperationDenied` + `same type task in execution` | 执行 `tccli tke DescribeClusterEndpointStatus --region <Region> --ClusterId <ClusterId> --IsExtranet false` 查看端点状态 | 内网端点已存在或正在创建中，无需重复创建 | 等待当前操作完成后重试。如端点已存在（`DescribeClusterEndpoints` 返回 `ClusterIntranetEndpoint`），直接使用 |
| `kubectl cluster-info` 返回 `Client.Timeout` 超时 | `tccli tke DescribeClusterEndpoints --region <Region> --ClusterId <ClusterId>` 查看端点。本地 `ping <端点IP>` 测试可达性 | 内网端点 IP（如 `10.0.0.1`）仅 VPC 内可达，本地开发机不可达 | 切换到同 VPC CVM 或通过 IOA/VPN/专线 接入后重试。参见 [连接集群](../../../../集群管理/连接集群/tccli%20操作.md) <!-- TODO: 多平台发布时需将相对链接转换为平台兼容格式（iWiki docid / 写写 nodeId） --> |
| `kubectl cluster-info` 返回 `DNSLookup: no such host` | `nslookup <ClusterDomain>` 检查 DNS 解析 | 无公网端点时，集群域名 `<ClusterId>.ccs.tencent-cloud.com` 无公网 DNS 解析 | 与"客户端超时"相同——需要 VPC 内可达环境 |
| `kubectl logs <Pod> -c <DaemonSetName>` 返回 `container xxx is not valid for pod xxx` | `kubectl version --client` 检查客户端版本 | kubectl >= 1.18 客户端在发起 logs 请求前会校验 Pod Spec 中是否包含指定容器名。注入的 sidecar 容器不在 Pod 原始 Spec 中，校验失败 | 使用 kubectl 1.16 版本：`curl -LO https://dl.k8s.io/release/v1.16.0/bin/<os>/amd64/kubectl && chmod +x kubectl && ./kubectl logs <Pod> -c <DaemonSetName>` |

### 创建成功但注入/调度异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| DaemonSet 创建成功但 Pod 未注入到业务 Pod | `kubectl describe ds <DaemonSetName> -n <Namespace>` 查看 Events；`kubectl get pod -n <Namespace> -o wide` 确认容器数 | 缺少 `eks.tke.cloud.tencent.com/ds-injection: "true"` 注解 | 在 DaemonSet `spec.template.metadata.annotations` 中添加 `eks.tke.cloud.tencent.com/ds-injection: "true"`，重新 apply |
| DaemonSet Pod 无法创建到超级节点，Events 显示 `untolerated taint` | `kubectl describe node <eklet-node>` 查看节点污点 | 缺少 eklet 污点容忍配置。超级节点默认打 `eks.tke.cloud.tencent.com/eklet`（NoSchedule）污点 | 在 DaemonSet `spec.template.spec.tolerations` 中添加：`[{key: eks.tke.cloud.tencent.com/eklet, operator: Exists, effect: NoSchedule}]` |
| kube-system 下的 DaemonSet 无 Pod 被注入 | `kubectl get ds -n kube-system` 检查 DaemonSet | `kube-system` 命名空间的 Pod 不会被注入 DaemonSet Pod（官方机制） | 将 DaemonSet 部署到其他命名空间，或评估是否需要在超级节点上运行 kube-system 级 DaemonSet |
| `kubectl exec` 无法登录 DaemonSet 容器 | `kubectl exec -it <Pod> -n <Namespace> -- /bin/sh` 不加 `-c`，观察进入的是哪个容器 | DaemonSet 容器名与业务 Pod 容器名相同时，exec 默认进入业务容器 | 必须使用 `-c <DaemonSetName>` 显式指定容器名。建议 DaemonSet 容器名与业务容器名区分 |
| 滚动更新状态异常 | `kubectl rollout status ds <DaemonSetName> -n <Namespace>` 查看更新进度 | 可能 updateStrategy 配置不当或注入速率过高导致资源冲突 | 调整 `updateStrategy.rollingUpdate.maxUnavailable` 降低并行度；或设置 `eks.tke.cloud.tencent.com/ds-inject-rate` 降低注入速率。回滚命令：`kubectl rollout undo ds <DaemonSetName> -n <Namespace>` |
| kubectl 命令全部不可达（本地环境） | 参见"命令返回错误"表中华 `kubectl cluster-info` 超时和 DNS 失败两项 | 保留 region、ClusterId → 登录 [TKE 控制台](https://console.cloud.tencent.com/tke2/cluster) 使用 Web Shell 执行 kubectl 命令 → 仍无法解决则 [提交工单](https://console.cloud.tencent.com/workorder) |

## 下一步

- [超级节点 Annotation 说明](https://cloud.tencent.com/document/product/457/44173) — 完整 Annotation 列表及字段定义
- [指定资源规格](https://cloud.tencent.com/document/product/457/44174) — 业务 Pod 的 CPU/内存/系统盘规格配置
- [新建超级节点](https://cloud.tencent.com/document/product/457/80229) — 创建超级节点池及虚拟节点
- [超级节点调度说明](https://cloud.tencent.com/document/product/457/53030) — Pod 调度策略与限制
- [采集超级节点上的 Pod 日志](https://cloud.tencent.com/document/product/457/60701) — 日志采集方案

## 控制台替代

[TKE 控制台 → 工作负载 → DaemonSet → 新建](https://console.cloud.tencent.com/tke2/cluster)：在 YAML 编辑模式下添加 `eks.tke.cloud.tencent.com/ds-injection: "true"` 注解和 eklet 污点容忍配置。
