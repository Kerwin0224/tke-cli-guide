# 使用固定 IP 模式

> 对照官方：[固定 IP 使用方法](https://cloud.tencent.com/document/product/457/34994) · page_id `34994` · tccli ≥3.1.107 · API 2018-05-25

## 概述

VPC-CNI 固定 IP 模式：Pod 使用 VPC 子网弹性网卡（ENI）IP，Pod 删除重建后 IP 保持不变。适用于依赖固定 IP 的有状态服务（如数据库、消息中间件）。通过 `EnableVpcCniNetworkType` 接口加上 `EnableStaticIp=true` 启用，要求 `VpcCniType` 为 `tke-route-eni`（共享网卡多 IP 模式）。

### 固定 IP vs 非固定 IP

| 维度 | 固定 IP | 非固定 IP |
|------|:--:|:--:|
| Pod IP 跨重建保持 | 是 | 否 |
| 适用场景 | 有状态服务（DB、MQ） | 无状态多副本服务 |
| 子网 IP 规划 | 需预留足够 IP | 按需分配，弹性更高 |
| Pod 驱逐/迁移 | IP 保持，新节点须在同一子网 | IP 重新分配 |
| 子网要求 | Pod 只能调度到同一子网中的节点 | 无限制 |

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
#    tke:EnableVpcCniNetworkType, tke:DescribeClusters, tke:DescribeEnableVpcCniProgress
#    tke:DescribeIPAMD, tke:DescribeVpcCniPodLimits, tke:AddVpcCniSubnets
#    tke:DisableVpcCniNetworkType
#    tke:DescribeClusterEndpoints, tke:DescribeClusterEndpointStatus
#    tke:DescribeClusterKubeconfig, tke:DescribePostNodeResources
#    vpc:DescribeSubnets, vpc:CreateSecurityGroup, vpc:DeleteSecurityGroup
# 验证
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
  ]
}
```

### 资源检查

```bash
# 4. 查询目标集群，确认当前为 GR 模式且 VPC-CNI 未开启（或已开启需确认 EnableStaticIp 状态）
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["<ClusterId>"]' \
    | jq '.Clusters[0].ClusterNetworkSettings | {Cni, VpcId}'
# expected: VpcId 有效。Cni=false 表示 GR 模式未开 VPC-CNI；Cni=true 需确认 Property.VpcCniType 和 EnableStaticIp

**预期输出**：

```json
{
  "Cni": false,
  "VpcId": "vpc-example"
}
```

> Cni=true 表示 VPC-CNI 已开启；Cni=false 表示 GR 模式未开启 VPC-CNI。

# 5. 查询子网 IP 可用量
tccli vpc DescribeSubnets --region <Region> \
    --Filters '[{"Name":"vpc-id","Values":["<VpcId>"]}]' \
    | jq '.SubnetSet[] | {SubnetId, AvailableIpAddressCount, CidrBlock}'
# expected: AvailableIpAddressCount 满足预期 Pod 数量（固定 IP 模式下每 Pod 占 1 个 IP）
```

**预期输出**：

```json
{
  "SubnetId": "subnet-example",
  "AvailableIpAddressCount": 248,
  "CidrBlock": "172.16.0.0/24"
}
```

```bash
# 6. 查询 VPC-CNI Pod 数量限制（评估单节点最大 Pod 数）
tccli tke DescribeVpcCniPodLimits --region <Region> \
    --Zone <Zone> \
    --InstanceFamily <InstanceFamily>
# expected: 返回该可用区和实例族的单节点 Pod 上限
```

**预期输出**：

```json
{
  "TotalCount": 34,
  "PodLimitsInstanceSet": [
    {
      "Zone": "ap-guangzhou-3",
      "InstanceFamily": "S5",
      "InstanceType": "S5.MEDIUM4",
      "PodLimits": {
        "TKERouteENINonStaticIP": 27,
        "TKERouteENIStaticIP": 27,
        "TKEDirectENI": 9,
        "TKESubENI": 100
      }
    }
  ]
}
```

### 版本与规格选择

- 集群网络模式：必须是 GR（GlobalRouter）模式，VPC-CNI 固定 IP 在 GR 集群上叠加开启。**VPC-CNI 原生集群不支持 `EnableVpcCniNetworkType`**
- VPC-CNI 类型：`tke-route-eni`（共享网卡多 IP）。`tke-direct-eni`（独占网卡）**也支持固定 IP**，但资源利用率较低，单节点 Pod 上限更少
- K8s 版本：无特殊要求，推荐 >= 1.28
- `EnableStaticIp` 在 `EnableVpcCniNetworkType` 发起时一次性指定，**启用后不可关闭**——如需切换需 Disable 后重新 Enable

## 控制台与 CLI 参数映射

| 控制台操作 | tccli / kubectl 命令 | 幂等 |
|-----------|---------------------|:--:|
| 查看集群网络模式 | `tccli tke DescribeClusters`（`ClusterNetworkSettings.Cni`） | 是 |
| 查看 VPC-CNI 开启进度 | `tccli tke DescribeEnableVpcCniProgress` | 是 |
| 查看 IPAMD 状态 | `tccli tke DescribeIPAMD` | 是 |
| 查询 Pod 数量限制 | `tccli tke DescribeVpcCniPodLimits` | 是 |
| 开启 VPC-CNI（固定 IP） | `tccli tke EnableVpcCniNetworkType` | 部分（重复调用无副作用） |
| 添加 VPC-CNI 子网 | `tccli tke AddVpcCniSubnets` | 是（重复添加已存在子网无影响） |
| 查看集群端点 | `tccli tke DescribeClusterEndpoints` | 是 |
| 获取 kubeconfig | `tccli tke DescribeClusterKubeconfig` | 是 |
| 部署固定 IP Pod | `kubectl apply -f`（含 `tke.cloud.tencent.com/vpc-ip-claim-delete-policy: Never`） | 是 |
| 查看 Pod IP | `kubectl get pods -o wide` | 是 |

## 操作步骤

### 步骤 1：确认集群 VPC-CNI 状态

```bash
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["<ClusterId>"]'
# expected: exit 0，返回集群网络配置
```

**预期输出**：

```json
{
  "TotalCount": 1,
  "Clusters": [
    {
      "ClusterId": "cls-example",
      "ClusterStatus": "Running",
      "ClusterNetworkSettings": {
        "ClusterCIDR": "10.0.0.0/16",
        "VpcId": "vpc-example",
        "Cni": true,
        "ServiceCIDR": "10.1.0.0/20",
        "SubnetId": "subnet-example",
        "Subnets": ["subnet-example-01", "subnet-example-02"]
      },
      "Property": {
        "NetworkType": "GR",
        "VpcCniType": "tke-route-eni",
        "IsSupportMultiENI": true
      }
    }
  ]
}
```

关键字段：
- `Cni`: `true` 表示 VPC-CNI 已启用
- `NetworkType`: `"GR"` 为基础网络模式，VPC-CNI 叠加其上。非 GR 模式集群（如 VPC-CNI 原生集群）**不支持 `EnableVpcCniNetworkType`**
- `VpcCniType`: `"tke-route-eni"` 为共享网卡多 IP 模式

### 步骤 2：启用 VPC-CNI 固定 IP 模式

#### 选择依据

*以下决策依据来自真实执行环境：*

- **VpcCniType 选 `tke-route-eni`**：共享网卡模式，Pod 共享节点主网卡的辅助 IP。与 `tke-direct-eni`（独占网卡）相比，资源利用率更高，单节点可支持更多 Pod（S5.MEDIUM4 可达 27 个固定 IP Pod，而 `tke-direct-eni` 仅 9 个）。固定 IP 功能两种模式均支持，但推荐共享网卡模式以获得更高 Pod 密度。

- **EnableStaticIp 选 `true`**：启用后 Pod 删除重建保留原 VPC IP。适用于需要固定 Pod IP 的场景（如数据库、有状态服务）。启用后不可关闭——这是 VPC-CNI 开通时一次性指定的特性。若无此需求，`EnableStaticIp=false` 即可（Pod IP 可能变化）。

- **Subnets 选择**：VPC-CNI 子网应选择 IP 资源充足的子网，**建议与集群主节点子网分离**（避免 CIDR 冲突和管理混乱）。固定 IP 模式下 Pod 调度到这些子网中的节点，回收/迁移时 IP 在同一子网内保持。子网必须与集群在同一 VPC。可用 IP 数应大于预期固定 IP Pod 数量。查询方式：
  ```bash
  tccli vpc DescribeSubnets --region <Region> \
      --Filters '[{"Name":"vpc-id","Values":["<VpcId>"]}]' \
      | jq '.SubnetSet[] | {SubnetId, AvailableIpAddressCount, CidrBlock}'
  ```

- **Pod 数量上限**：固定 IP 模式下每 Pod 占 1 个 ENI IP，单节点最大 Pod 数 = min(机型 ENI 上限, 子网可用 IP)。使用 `DescribeVpcCniPodLimits` 根据机型和可用区查询上限：
  ```bash
  tccli tke DescribeVpcCniPodLimits --region <Region> \
      --Zone <Zone> \
      --InstanceFamily <InstanceFamily>
  ```
  `tke-route-eni` 模式下 S5.MEDIUM4 单节点最大 27 个固定 IP Pod（`TKERouteENIStaticIP`），`tke-direct-eni` 模式下仅 9 个（`TKEDirectENI`）。

#### 关键字段说明

以下说明 `EnableVpcCniNetworkType` 的主要参数。完整定义见 `tccli tke EnableVpcCniNetworkType --generate-cli-skeleton`。

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `ClusterId` | String | 是 | 目标集群 ID，格式 `cls-xxxxxxxx`。集群必须为 GR 模式（`Property.NetworkType: "GR"`） | 集群不存在 → `InvalidParameter.ClusterId`；非 GR 模式 → `FailedOperation.EnableVPCCNIFailed: 'cluster is not vpc-cni cluster'` |
| `VpcCniType` | String | 是 | `tke-route-eni`（共享网卡）或 `tke-direct-eni`（独占网卡）。推荐 `tke-route-eni`：Pod 密度更高 | 填 `tke-direct-eni` → 单节点 Pod 上限大幅降低（S5 系列仅 9 个 vs 27 个） |
| `EnableStaticIp` | Boolean | 是 | `true`（启用固定 IP）；`false`（非固定 IP）。一次性指定，启用后不可关闭 | 忘开 → Pod IP 不固定，需 Disable 后重新 Enable（中断所有 Pod 网络） |
| `Subnets` | Array\<String\> | 是 | 子网 ID 列表，Pod IP 从这些子网分配。子网必须与集群同 VPC，建议独立于集群主子网 | 子网 IP 不足 → 新 Pod 无法获取 IP（Pending）；子网不在集群 VPC → `InvalidParameter.Subnet` |
| `ExpiredSeconds` | Integer | 否 | IP 回收等待时间（秒）。Pod 删除后 IP 保留此时长再回收 | 设置过短 → Pod 重建时 IP 可能已被回收；默认值视集群情况而定 |
| `SkipAddingNonMasqueradeCIDRs` | Boolean | 否 | 是否跳过自动添加 VPC-CNI 子网 CIDR 到 NonMasqueradeCIDRs | 非默认行为，一般不设置 |

#### 最小启用（仅固定 IP，只含必填字段）

`enable-static-ip-minimal.json`：

```json
{
  "ClusterId": "<ClusterId>",
  "VpcCniType": "tke-route-eni",
  "EnableStaticIp": true,
  "Subnets": ["<SubnetId>"]
}
```

```bash
tccli tke EnableVpcCniNetworkType --region <Region> --cli-input-json file://enable-static-ip-minimal.json
# expected: exit 0，返回 RequestId
```

**预期输出**：

```json
{
  "RequestId": "4a565829-594e-408b-9668-eb57722b1b5e"
}
```

#### 增强配置（含过期时间）

`enable-static-ip-enhanced.json`：

```json
{
  "ClusterId": "<ClusterId>",
  "VpcCniType": "tke-route-eni",
  "EnableStaticIp": true,
  "Subnets": ["<SubnetId>"],
  "ExpiredSeconds": 300
}
```

```bash
tccli tke EnableVpcCniNetworkType --region <Region> --cli-input-json file://enable-static-ip-enhanced.json
# expected: exit 0，返回 RequestId
```

`ExpiredSeconds` 设为 300（5 分钟）意味着 Pod 删除后 IP 保留 5 分钟再回收，在此期间重建的 Pod 可以拿回原 IP。

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<ClusterId>` | 集群 ID | 格式 `cls-xxxxxxxx`，必须是 GR 模式、状态 Running | `tccli tke DescribeClusters --region <Region>` |
| `<SubnetId>` | VPC-CNI 子网 ID | 须与集群在同一 VPC，IP 充足 | `tccli vpc DescribeSubnets --region <Region> --Filters '[{"Name":"vpc-id","Values":["<VpcId>"]}]'` |

### 步骤 3：添加 VPC-CNI 子网

如需更多子网供 Pod IP 分配，或原始子网 IP 不足，可追加子网。已存在的子网重复添加无副作用。

```bash
tccli tke AddVpcCniSubnets --region <Region> \
    --ClusterId <ClusterId> \
    --SubnetIds '["<SubnetId>"]' \
    --VpcId <VpcId>
# expected: exit 0，返回 RequestId
```

**预期输出**：

```json
{
  "RequestId": "095e606e-56e4-4278-bfba-c65efd658fe0"
}
```

### 步骤 4：轮询确认 VPC-CNI 已启用

启用 VPC-CNI 是异步操作，需轮询确认完成。

```bash
# 4.1 查看启用进度
tccli tke DescribeEnableVpcCniProgress --region <Region> \
    --ClusterId <ClusterId>
# expected: Status: "Running"，ErrorMessage: ""
```

**预期输出**：

```json
{
  "Status": "Running",
  "ErrorMessage": ""
}
```

```bash
# 4.2 查看 IPAMD 组件状态
tccli tke DescribeIPAMD --region <Region> \
    --ClusterId <ClusterId>
# expected: EnableIPAMD: true，Phase 非空
```

**预期输出**：

```json
{
  "EnableIPAMD": true,
  "EnableCustomizedPodCidr": false,
  "DisableVpcCniMode": false,
  "Phase": "upgrading",
  "SubnetIds": ["subnet-example-01", "subnet-example-02"],
  "ClaimExpiredDuration": "0s",
  "EnableTrunkingENI": false
}
```

> **注意**：`Phase` 字段显示为 `"upgrading"` 是 VPC-CNI 集群的正常运行状态（无节点时 IPAMD 持续等待节点注册），非故障状态。确认 `EnableIPAMD: true` 且 `DescribeClusters` 返回 `Status: Running` 即表示 VPC-CNI 功能已可用。

```bash
# 4.3 查看集群端点配置（确认连通方式）
tccli tke DescribeClusterEndpoints --region <Region> \
    --ClusterId <ClusterId>
# expected: 返回内网端点 IP，外网端点可能为空
```

**预期输出**：

```json
{
  "ClusterExternalEndpoint": "",
  "ClusterIntranetEndpoint": "10.0.0.50",
  "ClusterDomain": "cls-example.ccs.tencent-cloud.com",
  "IntranetSubnetId": "subnet-example"
}
```

> **注意**：`ClusterExternalEndpoint` 为空说明未开启公网端点。内网端点仅在同 VPC 或通过 VPN/专线/IOA 环境下可达。公网端点可能被组织级 CAM 策略拒绝（常见错误：`CAM deny: tke:clusterExtranetEndpoint=true, strategyId:240463971`）。如公网端点不可用，需通过 IOA/VPN/专线或同 VPC CVM 执行后续 kubectl 命令。

```bash
# 4.4 确认端点创建状态
tccli tke DescribeClusterEndpointStatus --region <Region> \
    --ClusterId <ClusterId> \
    --IsExtranet false
# expected: Status: "Created"
```

**预期输出**：

```json
{
  "Status": "Created",
  "ErrorMsg": "Created"
}
```

```bash
# 4.5 获取 kubeconfig（内网端点）
tccli tke DescribeClusterKubeconfig --region <Region> \
    --ClusterId <ClusterId> \
    --IsExtranet false | jq -r '.Kubeconfig' > ~/.kube/config
# expected: kubeconfig 写入成功，server 指向内网 IP
```

**预期输出**（kubeconfig 概览）：

```json
{
  "server": "https://10.0.0.50",
  "note": "certificate-authority-data、client-certificate-data、client-key-data 均已包含"
}
```

```bash
# 4.6 验证 kubectl 连通性（须在 VPC 内执行）
kubectl cluster-info
# expected: Kubernetes control plane running at https://<IntranetIP>
```

> **kubectl 连通性说明**：内网端点 IP 仅在同 VPC 或通过 VPN/专线/IOA 环境下可达。本地工作站直连会返回 `http: server gave HTTP response to HTTPS client`（IP 端口被 HTTP 代理/网关占用）或连接超时。内网域名 `*.ccs.tencent-cloud.com` 仅在腾讯云内网 DNS 可解析（本地返回 `no such host`）。如需从本地工作站执行 kubectl 命令，需先开启公网端点（`CreateClusterEndpoint --IsExtranet true`）——但公网端点可能被组织级 CAM 策略拒绝（`CAM deny: tke:clusterExtranetEndpoint=true, strategyId:240463971`，此为环境限制，非命令错误）。如公网端点不可用，请在 VPC 内 CVM 上执行后续 kubectl 命令。

```bash
# 4.7 查看 PostNode 资源（确认超级节点/虚拟节点状态）
tccli tke DescribePostNodeResources --region <Region> \
    --ClusterId <ClusterId>
# expected: 返回关联的超级节点信息
```

**预期输出**：

```json
{
  "PodSet": [
    {
      "NodeName": "eklet-subnet-xxxxxxxx",
      "Num": 4,
      "Cpu": 11,
      "Memory": 22,
      "Gpu": 0,
      "QuotaType": "...",
      "ChargeType": "...",
      "ResourceType": "...",
      "DisasterRecoverGroupId": "",
      "PriceType": "..."
    }
  ],
  "ReservedInstanceSet": [
    {
      "ReservedInstanceId": "...",
      "ReservedInstanceName": "...",
      "InstanceType": "...",
      "TotalCount": 0,
      "ActiveCount": 0,
      "Status": "..."
    }
  ]
}
```

### 步骤 5：部署固定 IP 工作负载（数据面）

> **前置**：以下 kubectl 命令须在 VPC 内（同 VPC CVM 或 IOA/VPN/专线）执行。若本地工作站无法直连内网端点，参见 [排障](#排障) 中的 kubectl 连接排查。

固定 IP Pod 需在 Pod 注解中声明 `tke.cloud.tencent.com/vpc-ip-claim-delete-policy: "Never"`（IP 保留策略），并在资源请求中声明 `tke.cloud.tencent.com/eni-ip`。

`fixed-ip-pod.yaml`：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: fixed-ip-demo
  annotations:
    tke.cloud.tencent.com/vpc-ip-claim-delete-policy: "Never"
    tke.cloud.tencent.com/networks: "tke-route-eni"
spec:
  containers:
  - name: nginx
    image: nginx:latest
    resources:
      requests:
        tke.cloud.tencent.com/eni-ip: "1"
      limits:
        tke.cloud.tencent.com/eni-ip: "1"
```

```bash
kubectl apply -f fixed-ip-pod.yaml
# expected: pod/fixed-ip-demo created
```

**预期输出**：

```text
pod/fixed-ip-demo created
```

部署 StatefulSet 版本（适用于管理多副本固定 IP 工作负载）：

`fixed-ip-sts.yaml`：

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: fixed-ip-app
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: fixed-ip-app
  serviceName: ""
  template:
    metadata:
      annotations:
        tke.cloud.tencent.com/vpc-ip-claim-delete-policy: "Never"
        tke.cloud.tencent.com/networks: "tke-route-eni"
      labels:
        app: fixed-ip-app
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        resources:
          limits:
            tke.cloud.tencent.com/eni-ip: "1"
          requests:
            tke.cloud.tencent.com/eni-ip: "1"
```

```bash
kubectl apply -f fixed-ip-sts.yaml
# expected: statefulset.apps/fixed-ip-app created
```

**预期输出**：

```text
statefulset.apps/fixed-ip-app created
```

## 验证

### 控制面（tccli）

```bash
# 1. 确认 VPC-CNI 固定 IP 已开启
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["<ClusterId>"]'
# expected: ClusterNetworkSettings.Cni=true, Property.VpcCniType="tke-route-eni"
```

**预期输出**（关键字段）：

```json
{
  "ClusterNetworkSettings": {
    "Cni": true,
    "VpcId": "vpc-example"
  },
  "Property": {
    "NetworkType": "GR",
    "VpcCniType": "tke-route-eni"
  }
}
```

```bash
# 2. 确认 IPAMD 组件运行
tccli tke DescribeIPAMD --region <Region> \
    --ClusterId <ClusterId>
# expected: EnableIPAMD=true
```

**预期输出**：

```json
{
  "EnableIPAMD": true,
  "EnableCustomizedPodCidr": false,
  "Phase": "upgrading",
  "SubnetIds": ["subnet-example-01", "subnet-example-02"]
}
```

```bash
# 3. 确认子网配置与开启参数一致
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["<ClusterId>"]' \
    | jq '.Clusters[0].ClusterNetworkSettings.Subnets'
# expected: 返回开启时指定的子网列表
```

**预期输出**：

```json
[
  "subnet-example-01",
  "subnet-example-02"
]
```

```bash
# 4. 确认端点状态
tccli tke DescribeClusterEndpointStatus --region <Region> \
    --ClusterId <ClusterId> \
    --IsExtranet false
# expected: Status: "Created"
```

**预期输出**：

```json
{
  "Status": "Created",
  "ErrorMsg": "Created"
}
```

### 数据面（kubectl）

> **前置**：kubectl 命令须在 VPC 内（同 VPC CVM 或 IOA/VPN/专线）执行。若本地工作站无法直连内网端点，参见 [排障](#排障) 中的 kubectl 连接排查。

```bash
# 5. 确认 Pod 分配到 VPC 子网 IP
kubectl get pods -o wide
# expected: Pod Running，IP 字段为 VPC 子网 IP（非容器 CIDR IP）
```

**预期输出**：

```text
NAME            READY   STATUS    RESTARTS   AGE   IP            NODE
fixed-ip-demo   1/1     Running   0          30s   172.16.0.10   node-example
```

```bash
# 6. 验证 Pod 删除重建后 IP 保持（固定 IP 核心功能）
kubectl delete pod fixed-ip-demo
kubectl get pods -o wide
# expected: 新建 Pod 的 IP 与删除前相同
```

**预期输出**（重建后 IP 不变）：

```text
NAME            READY   STATUS    RESTARTS   AGE   IP            NODE
fixed-ip-demo   1/1     Running   0          5s    172.16.0.10   node-example
```

### 验证维度

| 维度 | 命令 | 预期 |
|------|------|------|
| VPC-CNI 状态 | `DescribeClusters` -> `Cni` | `true` |
| 网络类型 | `DescribeClusters` -> `Property.VpcCniType` | `"tke-route-eni"` |
| IPAMD 状态 | `DescribeIPAMD` -> `EnableIPAMD` | `true` |
| 子网配置 | `DescribeClusters` -> `Subnets` | 与开启时指定子网一致 |
| 端点状态 | `DescribeClusterEndpointStatus --IsExtranet false` | `Status: "Created"` |
| Pod IP 来源 | `kubectl get pods -o wide` | IP 属 VPC 子网网段 |
| IP 重建保持 | `kubectl delete pod` + `kubectl get pods` | 重建后 IP 不变 |

## 清理

> **警告**：
> - `DisableVpcCniNetworkType` 会清除所有 VPC-CNI 相关的子网配置，Pod IP 将从 ENI 释放回 VPC 子网。这是**不可逆操作**——重新启用后 Pod IP 段可能与之前不同。
> - 禁用前需确认所有使用 VPC-CNI 网络的 Pod 已被删除，否则 Pod 网络将中断。
> - VPC-CNI 子网在 DisableVpcCniNetworkType 后**仍存在于 VPC 中**，不会自动删除。如不再需要，需手动在 VPC 控制台或通过 `tccli vpc DeleteSubnet` 删除以停止计费。
> - 安全组（如为本 session 创建的 `sg-example`）需手动通过 `tccli vpc DeleteSecurityGroup` 删除。

### 数据面（kubectl）

先删除所有固定 IP Pod，释放 ENI IP：

```bash
# 1. 删除测试 StatefulSet 和 Pod
kubectl delete sts fixed-ip-app -n default
# expected: statefulset.apps "fixed-ip-app" deleted

kubectl delete pod fixed-ip-demo
# expected: pod "fixed-ip-demo" deleted
```

```bash
# 2. 确认所有 Pod 已清理
kubectl get pods -A | grep -E 'fixed-ip'
# expected: 无输出（所有固定 IP Pod 已删除）
```

### 控制面（tccli）

```bash
# 3. 清理前状态检查
tccli tke DescribeIPAMD --region <Region> \
    --ClusterId <ClusterId>
# expected: 确认 EnableIPAMD=true，记录 SubnetIds 以备后续手动清理
```

**预期输出**：

```json
{
  "EnableIPAMD": true,
  "EnableCustomizedPodCidr": false,
  "Phase": "upgrading",
  "SubnetIds": ["subnet-example-01", "subnet-example-02"]
}
```

> 记录 `SubnetIds`，以备后续手动在 VPC 控制台删除子网。

```bash
# 4. 禁用 VPC-CNI 网络（⚠️ 不可逆）
tccli tke DisableVpcCniNetworkType --region <Region> \
    --ClusterId <ClusterId>
# expected: exit 0，返回 RequestId
```

> **注意**：`DisableVpcCniNetworkType` 不需要指定 `VpcCniType` 参数，该接口会检测当前集群的 VPC-CNI 类型并执行禁用。禁用后 DescribeIPAMD 将返回 `EnableIPAMD: false`。

```bash
# 5. 验证 VPC-CNI 已禁用
tccli tke DescribeIPAMD --region <Region> \
    --ClusterId <ClusterId>
# expected: 返回错误 ResourceNotFound.VpcCniMode 或 EnableIPAMD=false
```

**预期输出**（禁用成功时）：

```json
{
  "Error": {
    "Code": "ResourceNotFound",
    "Message": "The specified VpcCniMode is not found."
  },
  "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

> 若返回 `ResourceNotFound` 错误，表示 VPC-CNI 模式已不可用，禁用成功。部分环境可能返回 `EnableIPAMD: false`。

```bash
# 6. 删除安全组（如为本 session 创建）
tccli vpc DeleteSecurityGroup --region <Region> \
    --SecurityGroupId <SecurityGroupId>
# expected: exit 0
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `EnableVpcCniNetworkType` 返回 `FailedOperation.EnableVPCCNIFailed: 'cluster is not vpc-cni cluster'` | `tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]' \| jq '.Clusters[0].Property.NetworkType'` | 集群非 GR 模式，可能是 VPC-CNI 原生集群或已启用 VPC-CNI | 仅 GR 模式集群支持 EnableVpcCniNetworkType。先用 `DescribeClusters` 确认 `Property.NetworkType: "GR"`。VPC-CNI 原生集群需通过控制台配置固定 IP |
| `EnableVpcCniNetworkType` 返回 `FailedOperation.EnableVPCCNIFailed: 'cluster has been vpc-cni cluster'` | `tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]' \| jq '.Clusters[0].ClusterNetworkSettings.Cni'` | 集群已是 VPC-CNI 模式，无需再启用 | 检查 `ClusterNetworkSettings.Cni` 是否为 `true`。如已启用但需固定 IP：先 `DisableVpcCniNetworkType`（⚠️ 不可逆，中断所有 Pod 网络）再重新 `EnableVpcCniNetworkType --EnableStaticIp true` |
| `EnableVpcCniNetworkType` 返回 `EnableStaticIp` 未指定 | `tccli tke EnableVpcCniNetworkType --generate-cli-skeleton` 确认 `EnableStaticIp` 为 Required 字段 | 忘记指定 `--EnableStaticIp true`，创建后发现固定 IP 不可用 | 创建时同时指定 `--EnableStaticIp true` 和 `--VpcCniType tke-route-eni`。已错过则需 Disable 后重新 Enable（中断所有 Pod 网络） |
| `EnableVpcCniNetworkType` 返回 `InvalidParameter.Subnet` | `tccli vpc DescribeSubnets --region <Region> --SubnetIds '["<SubnetId>"]'` 检查子网是否存在及所属 VPC | 子网 ID 不存在或不在集群 VPC 内 | 用 `tccli vpc DescribeSubnets --region <Region> --Filters '[{"Name":"vpc-id","Values":["<VpcId>"]}]'` 获取正确子网列表 |
| `EnableVpcCniNetworkType` 返回 `InvalidParameter.ClusterState` | `tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'` 查看状态 | 集群非 Running 状态 | 等待集群状态为 Running 后重试 |
| `EnableVpcCniNetworkType` 返回 `CidrMaskSizeOutOfRange` | 检查 VPC-CNI 子网 CIDR 和集群 ServiceCIDR | 子网 CIDR 掩码与 ServiceCIDR 掩码范围冲突 | 确保 VPC-CNI 子网 CIDR 不与 ServiceCIDR、ClusterCIDR 重叠。ServiceCIDR 掩码必须 17-27 |
| Pod 处于 Pending | `kubectl describe pod <PodName>` 查看 Events，关注 `failed to allocate IP address` | 子网 IP 地址耗尽 | `tccli tke AddVpcCniSubnets` 添加新子网。同时 `tccli vpc DescribeSubnets` 检查 `AvailableIpAddressCount` |
| kubectl 返回 `http: server gave HTTP response to HTTPS client` | `tccli tke DescribeClusterEndpoints --region <Region> --ClusterId <ClusterId>` 确认端点 IP | 连接到内网端点 IP `443` 端口，但该端口实际运行的是 HTTP 代理/网关而非 K8s API Server（常见于本地工作站直连 VPC 内网 IP 的场景） | 确认 `DescribeClusterEndpointStatus` 返回 `Status: "Created"`。在同 VPC CVM 上执行 kubectl 命令（内网端点仅 VPC 内可达）。排查流程：(1) 确认 `Status: Created`；(2) 在同 VPC CVM 上执行 `curl -k https://<IntranetIP>/api` |
| kubectl 返回 `dial tcp: lookup cls-example.ccs.tencent-cloud.com: no such host` | 使用域名 kubeconfig 时本地 DNS 无法解析内网域名 | 内网域名 `*.ccs.tencent-cloud.com` 仅在腾讯云内网 DNS 可解析 | 改用 IP 地址的 kubeconfig：`tccli tke DescribeClusterKubeconfig --IsExtranet false \| jq -r '.Kubeconfig'`（输出中含 `server: https://<IntranetIP>`）。或在同 VPC CVM 上执行 |
| `DescribeIPAMD` 返回错误 | `tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]' \| jq '.Clusters[0].ClusterNetworkSettings.Cni'` | 集群未开启 VPC-CNI 模式 | 执行 `EnableVpcCniNetworkType` 开启 VPC-CNI |

### 创建成功但状态异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| IPAMD Phase 为 `upgrading` | `tccli tke DescribeIPAMD --region <Region> --ClusterId <ClusterId>` 查看 Phase 和 EnableIPAMD | 集群无 VPC-CNI 节点时 IPAMD 持续等待节点注册，此为正常运行状态（非故障） | 确认 `EnableIPAMD: true` 且 `DescribeClusters` 返回 `Status: Running`。添加节点后 Phase 会自动恢复正常 |
| Pod IP 重建后变化 | `kubectl get pod <PodName> -o yaml \| grep annotations` 检查 `vpc-ip-claim-delete-policy` | Pod 缺少注解 `tke.cloud.tencent.com/vpc-ip-claim-delete-policy: "Never"` | 添加注解并重建 Pod。StatefulSet 需在 `spec.template.metadata.annotations` 中添加此注解 |
| 固定 IP Pod 调度到非目标子网节点 | `kubectl get pod <PodName> -o yaml \| grep nodeName`，然后 `kubectl get node <NodeName> -o yaml` 查节点子网 | 未配置 Pod nodeSelector/affinity 限制调度 | 在 Pod/StatefulSet spec 中添加 `nodeSelector` 或 `nodeAffinity`，匹配 VPC-CNI 子网所在可用区的节点 |
| 公网端点开启失败（`CreateClusterEndpoint --IsExtranet true`） | `tccli tke DescribeClusterEndpoints --region <Region> --ClusterId <ClusterId>` 检查 `ClusterExternalEndpoint` 为空；执行 `CreateClusterEndpoint --IsExtranet true` 返回 `InvalidParameter.Param` | CAM 组织策略硬拒绝：`CAM deny: tke:clusterExtranetEndpoint=true, strategyId:240463971`。condition: `[{'key': 'tke:clusterExtranetEndpoint', 'value': 'true', 'ope': 'string_equal'}], effect: deny`。**此为环境限制（不可抗力），非命令错误**。自建安全组无法绕过 CAM 层权限检查 | 使用内网端点（`--IsExtranet false`）。内网端点需通过 IOA/VPN/专线或同 VPC CVM 才能 kubectl 可达。排障确认：(1) `DescribeClusterEndpointStatus --IsExtranet false` → Status: "Created"；(2) 同 VPC CVM 上测试连通性。如需公网端点，联系管理员检查 CAM 策略 `strategyId:240463971` 的条件限制 |
| kubectl 连接超时（内网端点） | `tccli tke DescribeClusterEndpointStatus --region <Region> --ClusterId <ClusterId> --IsExtranet false` 确认 Status: "Created"；`curl -k https://<IntranetIP>/api` 测试连通性 | 内网端点 IP 仅在同 VPC 或通过 VPN/专线可达。本地工作站不在 VPC 网络内时连接超时/失败是正常现象。已尝试：(1) IP kubeconfig → `http: server gave HTTP response to HTTPS client`；(2) 域名 kubeconfig → `no such host`；(3) curl → 超时 | 在同 VPC CVM 上执行 kubectl 命令；或通过 IOA/VPN/专线接入 VPC 网络。排查流程：(1) 确认端点 Status=Created；(2) 在同 VPC CVM 上执行 `curl -k https://<IntranetIP>/api` 验证可达；(3) 获取 kubeconfig：`tccli tke DescribeClusterKubeconfig --IsExtranet false \| jq -r '.Kubeconfig'`；(4) 检查 `server` 字段使用的是内网 IP 而非域名 |

> **如问题未解决**：保留 `region`、`ClusterId`、`RequestId` 和完整的创建 JSON 文件（`enable-static-ip-minimal.json` / `enable-static-ip-enhanced.json`），以便登录 [TKE 控制台](https://console.cloud.tencent.com/tke2/cluster) 查看详细状态或 [提交工单](https://console.cloud.tencent.com/workorder)。

## 下一步

- [VPC-CNI 模式 Pod 数量限制](../VPC-CNI%20模式%20Pod%20数量限制/tccli%20操作.md)
- [非固定 IP 模式使用说明](../非固定%20IP%20模式使用说明/tccli%20操作.md) -- page_id `64940`
- [固定 IP 相关特性](../固定%20IP%20相关特性/tccli%20操作.md) -- page_id `50358`
- [VPC-CNI 模式介绍](../VPC-CNI%20模式介绍/tccli%20操作.md) -- page_id `45709`
- 容器网络概述（页面待创建，参见 [TKE 容器网络概述](https://cloud.tencent.com/document/product/457/50354)）

> **注**：以上相对路径链接（`../*.md`）为文件系统内链，在本地和 Git 仓库中有效。发布至 iWiki/写写等平台时，需由同步脚本将相对路径转换为平台链接（iWiki: `/p/<docid>`，写写: `/document/<nodeId>`）。

## 控制台替代

[容器服务控制台 -> 集群 -> 基本信息 -> 网络信息 -> 开启 VPC-CNI](https://console.cloud.tencent.com/tke2/cluster) 并勾选「固定 Pod IP」。
