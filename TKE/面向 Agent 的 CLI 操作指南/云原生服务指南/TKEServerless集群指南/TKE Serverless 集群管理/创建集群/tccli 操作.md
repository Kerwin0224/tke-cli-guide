# 创建集群

> 对照官方：[创建集群](https://cloud.tencent.com/document/product/457/39813) · page_id `39813`

## 概述

通过 `CreateEKSCluster` 在指定地域创建一个 TKE Serverless (EKS) 集群。TKE Serverless 集群无需管理节点，Pod 直接运行在弹性容器实例上，按 Pod 实际使用的 vCPU 和内存计费。集群创建完成后需等待状态变为 `Running`，随后通过 `DescribeEKSClusterCredential` 获取 kubeconfig 即可使用 kubectl 连接。

**重要**：TKE Serverless 集群仅支持 VPC-CNI 网络模式，创建时必须指定 VPC 和子网。

## 前置条件

- [环境准备](../../../../环境准备.md)

### 环境检查

```bash
tccli --version
# expected: tccli version >= 1.0.0

tccli configure list
# expected: secretId, secretKey, region 均已配置

tccli tke DescribeEKSClusters --region <Region>
# expected: exit 0
```

```json
{
  "TotalCount": "<TotalCount>",
  "Clusters": "<Clusters>",
  "ClusterId": "<ClusterId>",
  "ClusterName": "<ClusterName>",
  "VpcId": "<VpcId>",
  "SubnetIds": "<SubnetIds>"
}
```

### 资源检查

```bash
# 查询可用 VPC
tccli vpc DescribeVpcs --region <Region>
# expected: 至少返回 1 个可用 VPC

# 查询 VPC 下的子网
tccli vpc DescribeSubnets --region <Region> \
    --Filters '[{"Name":"vpc-id","Values":["VPC_ID"]}]'
# expected: 至少返回 1 个子网

# 查询已有 EKS 集群，确认是否需要新建
tccli tke DescribeEKSClusters --region <Region>
# expected: 确认当前集群数量和命名
```

### 版本与规格选择

- **Kubernetes 版本**：支持 1.28、1.30 等版本，建议选择最新的稳定版本
- **网络模式**：仅支持 VPC-CNI，Pod IP 直接从 VPC 子网分配
- **Service CIDR**：需指定 Service 网段，不能与 VPC CIDR 重叠
- **集群规格**：L5（入门）、L20（基础）、L50（标准）、L100（高级），影响最大 Pod 数量

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 创建集群 | `tccli tke CreateEKSCluster --region <Region> --cli-input-json file://input.json` | 否 |
| 查看集群列表 | `tccli tke DescribeEKSClusters --region <Region>` | 是 |
| 查看集群详情 | `tccli tke DescribeEKSClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'` | 是 |
| 删除集群 | `tccli tke DeleteEKSCluster --region <Region> --ClusterId CLUSTER_ID` | 是 |

## 操作步骤

### 步骤 1: 准备输入 JSON

**选择依据**：
- **K8sVersion**：选择 `1.30` 以获得最新的 Kubernetes 特性支持
- **VpcId / SubnetId**：选择与后续工作负载在同一 VPC 的子网，确保网络连通性
- **ServiceSubnetId**：需与 Pod 子网分离，建议使用专门的 Service 子网段
- **ClusterLevel**：测试环境选 `L5`，生产环境按预期 Pod 数量选择
- **PublicLB**：是否需要公网 CLB 暴露 Service，按需开启

`eks-create.json`：

```json
{
    "ClusterName": "CLUSTER_NAME",
    "K8sVersion": "1.30",
    "VpcId": "VPC_ID",
    "SubnetId": "SUBNET_ID",
    "ServiceSubnetId": "SERVICE_SUBNET_ID",
    "ClusterDesc": "CLUSTER_DESCRIPTION",
    "PublicLB": {
        "Enabled": true,
        "SecurityPolicy": "3.0.0"
    },
    "InternalLB": {
        "Enabled": false
    },
    "DnsServers": [
        {
            "Domain": "CLUSTER_DOMAIN",
            "DnsServers": ["DNS_IP_1", "DNS_IP_2"]
        }
    ],
    "TagSpecification": [
        {
            "ResourceType": "cluster",
            "Tags": [
                {"Key": "env", "Value": "production"},
                {"Key": "team", "Value": "ops"}
            ]
        }
    ]
}
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `CLUSTER_NAME` | 集群名称 | 1-50 字符 | 自定义 |
| `VPC_ID` | VPC 实例 ID | 须为有效 VPC | `tccli vpc DescribeVpcs --region <Region>` |
| `SUBNET_ID` | Pod 子网 ID | 须属于 VPC_ID | `tccli vpc DescribeSubnets --region <Region>` |
| `SERVICE_SUBNET_ID` | Service 子网 ID | 须属于 VPC_ID，建议与 Pod 子网不同 | `tccli vpc DescribeSubnets --region <Region>` |
| `CLUSTER_DESCRIPTION` | 集群描述 | 可选 | 自定义 |
| `CLUSTER_DOMAIN` | 集群域名 | 可选，默认为 `cluster.local` | 自定义 |
| `DNS_IP_1/2` | 自定义 DNS 服务器 | 可选 | 自定义 |
| `REGION` | 地域 | 如 `ap-guangzhou` | `tccli configure list` |

### 步骤 2: 创建集群

```bash
tccli tke CreateEKSCluster --region <Region> \
    --cli-input-json file://eks-create.json
# expected: exit 0，返回 ClusterId
```

预期输出：

```json
{
    "ClusterId": "cls-xxxxxxxx",
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

### 步骤 3: 轮询等待集群就绪

集群创建通常需要 2-5 分钟，使用轮询等待状态变为 `Running`：

```bash
# 查询集群状态
tccli tke DescribeEKSClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterStatus: Running
```

预期输出：

```json
{
    "Clusters": [
        {
            "ClusterId": "cls-xxxxxxxx",
            "ClusterName": "my-eks-cluster",
            "ClusterStatus": "Running",
            "ClusterVersion": "1.30",
            "VpcId": "vpc-xxxxxxxx",
            "SubnetId": "subnet-xxxxxxxx",
            "CreatedTime": "2025-01-01T00:00:00Z"
        }
    ],
    "TotalCount": 1,
    "RequestId": "..."
}
```

| 状态 | 说明 |
|------|------|
| `Creating` | 创建中 |
| `Running` | 正常运行 |
| `Deleting` | 删除中 |
| `Abnormal` | 异常状态 |

### 步骤 4: 获取集群凭证

```bash
tccli tke DescribeEKSClusterCredential --region <Region> \
    --ClusterId CLUSTER_ID
# expected: exit 0，返回 kubeconfig 内容
```

```json
{
  "Addresses": "<Addresses>",
  "Type": "<Type>",
  "Ip": "<Ip>",
  "Port": "<Port>",
  "Credential": "<Credential>",
  "CACert": "<CACert>"
}
```

## 验证

| 维度 | 命令 | 预期 |
|------|------|------|
| 状态 | `tccli tke DescribeEKSClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'` | `ClusterStatus: Running` |
| 版本 | 同上，检查 `ClusterVersion` | 与创建参数 `K8sVersion` 一致 |
| 网络 | 同上，检查 `VpcId`、`SubnetId` | 与创建参数一致 |
| 连通性 | `tccli tke DescribeEKSClusters --region <Region> --ClusterIds '["CLUSTER_ID"]' \| jq '.Clusters[0].ClusterStatus'` | `"Running"` |

```bash
# 综合验证
tccli tke DescribeEKSClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]' | jq '.Clusters[0] | {ClusterId, ClusterName, ClusterStatus, ClusterVersion, VpcId}'
# expected: ClusterStatus: Running, VpcId/ClusterVersion 均与创建参数一致
```

```json
{
  "TotalCount": "<TotalCount>",
  "Clusters": "<Clusters>",
  "ClusterId": "<ClusterId>",
  "ClusterName": "<ClusterName>",
  "VpcId": "<VpcId>",
  "SubnetIds": "<SubnetIds>"
}
```

## 清理

> **警告**：`DeleteEKSCluster` 会**永久删除**集群及其中所有工作负载、Service、配置等资源，不可恢复。执行前确认集群内无重要业务。

### 1. 清理前状态检查

```bash
tccli tke DescribeEKSClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]'
# 确认是待删除的目标集群，记录 ClusterId、ClusterName
```

```json
{
  "TotalCount": 0,
  "Clusters": [],
  "ClusterId": "<ClusterId>",
  "ClusterName": "<ClusterName>",
  "VpcId": "<VpcId>",
  "SubnetIds": [],
  "K8SVersion": "<K8SVersion>",
  "Status": "<Status>"
}
```

### 2. 删除集群

```bash
tccli tke DeleteEKSCluster --region <Region> \
    --ClusterId CLUSTER_ID
# expected: exit 0，返回 RequestId
```

### 3. 验证已删除

```bash
tccli tke DescribeEKSClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]'
# expected: ResourceNotFound 或 Clusters 列表不含该集群
```

```json
{
  "TotalCount": "<TotalCount>",
  "Clusters": "<Clusters>",
  "ClusterId": "<ClusterId>",
  "ClusterName": "<ClusterName>",
  "VpcId": "<VpcId>",
  "SubnetIds": "<SubnetIds>"
}
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `CreateEKSCluster` 返回 `InvalidParameter` | `python3 -m json.tool eks-create.json` 检查 JSON 格式 | JSON 格式错误或参数值不合法 | 修正 JSON 格式或参数值，参考上方参数表 |
| `CreateEKSCluster` 返回 `InvalidParameter.VpcId` | `tccli vpc DescribeVpcs --region <Region> --VpcIds '["VPC_ID"]'` 验证 VPC 存在 | VPC ID 不存在或地域不匹配 | 确认 VpcId 与 region 一致 |
| `CreateEKSCluster` 返回 `InvalidParameter.SubnetId` | `tccli vpc DescribeSubnets --region <Region> --SubnetIds '["SUBNET_ID"]'` 验证子网 | 子网 ID 不存在或与 VPC 不匹配 | 确认 SubnetId 属于指定的 VpcId |
| `CreateEKSCluster` 返回 `FailedOperation.VpcCidrConflict` | 检查 VPC CIDR 和 ServiceSubnetId CIDR | Service 子网 CIDR 与 VPC CIDR 冲突（此为环境配置问题） | 选择与 VPC CIDR 不重叠的 Service 子网段 |
| `CreateEKSCluster` 返回 `LimitExceeded` | `tccli tke DescribeEKSClusters --region <Region>` 查集群数 | 集群数量达到地域上限 | 删除不再使用的集群或联系管理员提升配额 |
| `CreateEKSCluster` 返回 `UnauthorizedOperation.CamNoAuth` | 检查 CAM 策略 | 缺少 `tke:CreateEKSCluster` 权限 | 联系管理员授权 |

### 操作成功但异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 返回 ClusterId 但 5 分钟后状态仍为 `Creating` | `tccli tke DescribeEKSClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'` 轮询状态 | 后端创建缓慢或资源不足 | 继续等待，超过 10 分钟则登录控制台查看详细状态 |
| 集群 Running 但 DescribeEKSClusterCredential 返回空 | 检查 `tccli tke DescribeEKSClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'` 确认状态 | 凭证尚未生成（罕见） | 等待 1-2 分钟重试 |
| 集群状态变为 `Abnormal` | `tccli tke DescribeEKSClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'` 查看详情 | 创建过程中底层资源异常 | 删除该集群重新创建，保留 region、ClusterId、RequestId 联系技术支持 |

## 下一步

- [连接集群](../连接集群/tccli%20操作.md) — 获取 kubeconfig 并通过 kubectl 连接集群
- [工作负载管理](../../Kubernetes%20对象管理/工作负载管理/tccli%20操作.md) — 在 Serverless 集群上部署工作负载
- [服务管理](../../Kubernetes%20对象管理/服务管理/tccli%20操作.md) — 创建 Service 暴露服务

## 控制台替代

控制台：容器服务控制台 → 集群 → 新建 → Serverless 集群。
