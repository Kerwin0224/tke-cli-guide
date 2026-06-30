# TKE Serverless 集群概述

> 对照官方：[TKE Serverless 集群概述](https://cloud.tencent.com/document/product/457/39804) · page_id `39804`

## 概述

TKE Serverless 集群（弹性容器实例 EKS）是腾讯云容器服务提供的无节点 Serverless Kubernetes 服务。与标准 TKE 托管/独立集群不同，Serverless 集群无需管理任何底层节点（CVM），用户只需提交工作负载，平台按 Pod 实际使用的 vCPU、内存和 GPU 资源按秒计费。

核心特性：
- **免节点运维**：无需购买、管理或扩缩容 CVM 节点，集群控制面和调度均由腾讯云托管。
- **按需付费**：按 Pod 实际使用的 vCPU/内存/GPU 规格 x 运行时长计费，无闲置资源成本。
- **弹性伸缩**：Pod 按请求自动调度到底层资源池，秒级启动，无需预置节点容量。
- **VPC-CNI 网络**：仅支持 VPC-CNI 网络模式，每个 Pod 直接从 VPC 子网分配独立 IP，与 VPC 内其他资源（CVM、CLB、TCR 等）原生互通。
- **兼容 K8s API**：支持标准 Kubernetes API，兼容 kubectl、Helm、CI/CD 工具链。

与标准 TKE 集群的关键区别：

| 维度 | 标准 TKE 集群（托管/独立） | TKE Serverless 集群 |
|------|--------------------------|---------------------|
| 节点管理 | 需管理 CVM 节点池 | 无节点概念，无需管理 |
| 计费模型 | 集群管理费 + CVM 实例费 | 仅按 Pod 规格和时长计费 |
| 网络模式 | GlobalRouter / VPC-CNI | 仅 VPC-CNI |
| 资源隔离 | Pod 共享节点内核 | Pod 级安全沙箱隔离 |
| 适用场景 | 长期稳定运行、自建集群迁移 | 突发弹性、CI/CD、事件驱动 |

## 前置条件

- [环境准备](../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看 Serverless 集群列表 | `DescribeEKSClusters` | 是 |
| 查看单个集群详情 | `DescribeEKSClusters --ClusterIds` | 是 |
| 获取集群访问凭证 | `DescribeEKSClusterCredential` | 是 |
| 查看集群 Kubeconfig | `DescribeEKSClusterCredential` | 是 |
| 创建 Serverless 集群 | `CreateEKSCluster`（详见[创建集群](../TKE%20Serverless%20集群管理/创建集群/tccli%20操作.md)） | 否 |
| 删除 Serverless 集群 | `DeleteEKSCluster` | 否 |

## 操作步骤

### 步骤 1：查询所有 Serverless 集群

```bash
tccli tke DescribeEKSClusters --region <Region>
# expected: exit 0，返回当前地域所有 Serverless 集群列表
```

**预期输出**：

```json
{
    "TotalCount": 2,
    "Clusters": [
        {
            "ClusterId": "cls-xxxxxxxx",
            "ClusterName": "my-serverless-cluster",
            "ClusterDesc": "生产环境 Serverless 集群",
            "K8SVersion": "1.30.0",
            "Status": "Running",
            "CreatedTime": "2025-01-15T10:00:00Z",
            "VpcId": "vpc-xxxxxxxx",
            "SubnetIds": ["subnet-xxxxxxxx"],
            "ClusterType": "SERVERLESS_CLUSTER"
        }
    ],
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

记录目标集群的 `ClusterId`，后续操作使用。

### 步骤 2：查看特定集群详情

```bash
tccli tke DescribeEKSClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]'
# expected: exit 0，返回指定集群的完整信息
```

**预期输出**：

```json
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "CLUSTER_ID",
            "ClusterName": "my-serverless-cluster",
            "K8SVersion": "1.30.0",
            "Status": "Running",
            "ClusterType": "SERVERLESS_CLUSTER",
            "VpcId": "vpc-xxxxxxxx",
            "SubnetIds": ["subnet-xxxxxxxx"],
            "ClusterNetworkSettings": {
                "VpcId": "vpc-xxxxxxxx",
                "ServiceCIDR": "10.0.0.0/16",
                "MaxClusterServiceNum": 256
            },
            "CreatedTime": "2025-01-15T10:00:00Z"
        }
    ],
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

关键字段说明：

| 字段 | 说明 |
|------|------|
| `Status` | 集群状态：`Creating`、`Running`、`Deleting`、`Abnormal` |
| `VpcId` | 集群所在 VPC，Pod IP 从此 VPC 子网分配 |
| `SubnetIds` | 集群关联的子网列表，Pod 调度时从此列表选子网分配 IP |
| `K8SVersion` | K8s 版本，与标准 TKE 可用的版本列表一致 |
| `ClusterNetworkSettings.ServiceCIDR` | 集群 Service CIDR，不可与 VPC CIDR 重叠 |

### 步骤 3：获取集群访问凭证

```bash
tccli tke DescribeEKSClusterCredential --region <Region> \
    --ClusterId CLUSTER_ID | jq -r '.Kubeconfig' > ~/.kube/config
# expected: 生成 ~/.kube/config 文件
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

> 集群需处于 `Running` 状态才能获取有效的 kubeconfig。若状态非 `Running`，等待集群就绪后重试。

### 步骤 4：验证集群连通性

```bash
kubectl cluster-info
# expected: Kubernetes control plane is running at https://...

kubectl get ns
# expected: 返回 default 等系统命名空间
```

```text
NAME  STATUS  AGE
...
```

## 验证

### 控制面（tccli）

```bash
# 验证集群存在且状态正常
tccli tke DescribeEKSClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]' \
    | jq '.Clusters[0].Status'
# expected: "Running"

# 验证集群类型为 SERVERLESS
tccli tke DescribeEKSClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]' \
    | jq '.Clusters[0].ClusterType'
# expected: "SERVERLESS_CLUSTER"
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

### 数据面

> **需 kubectl 可达环境**

```bash
# 验证集群连通性
kubectl cluster-info
# expected: Kubernetes control plane is running at ...

# 验证无节点（Serverless 集群特性）
kubectl get nodes
# expected: No resources found（或仅显示虚拟节点）
```

```text
NAME  STATUS  AGE
...
```

## 清理

本页面为只读操作，无需清理。

> 如创建了验证用集群且不再需要，请前往 [创建集群](../TKE%20Serverless%20集群管理/创建集群/tccli%20操作.md) 或使用 `DeleteEKSCluster` 删除，避免持续计费。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DescribeEKSClusters` 返回 `AuthFailure.SignatureFailure` | `tccli configure list` 检查 secretId/secretKey | 凭据错误或过期 | 重新获取 API 密钥并执行 `tccli configure set` |
| `DescribeEKSClusters` 返回 `UnauthorizedOperation` | `tccli configure list` 检查当前凭据 | 子账号缺少 `tke:DescribeEKSClusters` 权限 | 联系主账号授予 `QcloudTKEFullAccess` 或最小权限 `tke:DescribeEKSClusters` |
| `DescribeEKSClusters` 返回空列表 | `tccli configure get region` 确认当前地域 | 当前地域无 Serverless 集群，或地域选择错误 | 换地域重试或前往其他有集群的地域查询 |
| `DescribeEKSClusterCredential` 返回 `FailedOperation.ClusterNotFound` | `tccli tke DescribeEKSClusters` 确认集群存在 | 集群 ID 错误或集群所属地域不匹配 | 检查 `ClusterId` 是否正确，确保 `--region` 与集群所在地域一致 |
| `DescribeEKSClusterCredential` 返回 `FailedOperation.ClusterState` | `tccli tke DescribeEKSClusters --ClusterIds` 检查状态 | 集群非 `Running` 状态，可能为 `Creating` 或 `Deleting` | 等待集群进入 `Running` 状态后重试 |

### 操作成功但异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubeconfig` 生成成功但 kubectl 无法连接 | `cat ~/.kube/config` 检查 server 地址；`curl -k SERVER_ADDRESS` 测试连通性 | 集群 API Server 未开启公网访问或开启了内网访问但当前网络不可达 | 前往 [TKE 控制台](https://console.cloud.tencent.com/tke2/ecluster) 开启公网访问，或从 VPC 内网环境执行 |
| 集群状态长期为 `Creating` | `tccli tke DescribeEKSClusters --ClusterIds '["CLUSTER_ID"]'` 轮询状态 | 后端创建中，通常 1-2 分钟 | 每 30 秒轮询一次；超过 5 分钟则保留 region、ClusterId → 登录控制台查看 |

## 下一步

- [创建 Serverless 集群](../TKE%20Serverless%20集群管理/创建集群/tccli%20操作.md) — 通过 CLI 创建第一个 Serverless 集群
- [连接集群](../TKE%20Serverless%20集群管理/连接集群/tccli%20操作.md) — 获取 kubeconfig 并配置访问
- [集群生命周期](../TKE%20Serverless%20集群管理/集群生命周期/tccli%20操作.md) — 了解集群状态变化
- [计费概述](../购买%20TKE%20Serverless%20集群/计费概述/tccli%20操作.md) — 了解 Serverless 计费模型
- [TKE API 概览](https://cloud.tencent.com/document/product/457/31853) — 完整 API 列表

## 控制台替代

[TKE 控制台 → Serverless 集群](https://console.cloud.tencent.com/tke2/ecluster)：进入控制台后查看所有 Serverless 集群列表，点击集群名可查看详情、获取 kubeconfig，或创建新集群。
