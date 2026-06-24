# TKE Serverless 集群概述（tccli）

> 对照官方：TKE Serverless 集群概述(https://cloud.tencent.com/document/product/457/39804) · page_id `39804`

## 概述

TKE Serverless 集群是腾讯云容器服务提供的无服务器 Kubernetes 服务。你无需购买和管理节点（CVM/BM），只需提供容器镜像并指定资源需求即可运行容器化应用，由腾讯云底层虚拟机资源池按需调度、按实际使用量计费。Serverless 集群托管控制面，提供原生 Kubernetes API 兼容性，支持启动速度快（通常数秒）、秒级扩缩容。

核心价值：把集群运维（节点打补丁、内核升级、污点容忍、容量规划）的负担从你身上转移到云平台。你只管 Pod。

> **注意**：TKE Serverless 集群的新建入口已于 2025 年关闭，现有集群使用不受影响。新用户建议直接使用 [TKE 标准集群](https://cloud.tencent.com/document/product/457/6759) 中的超级节点（Serverless 节点）。

## 前置条件

- [环境准备](../../../../环境准备.md)
- 理解 [Kubernetes 核心概念](https://kubernetes.io/docs/concepts/)（Pod、Service、Deployment、Namespace）
- 了解 [腾讯云容器镜像服务 TCR](https://cloud.tencent.com/document/product/1141)

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 查看 Serverless 集群列表 | `tccli tke DescribeEKSClusters --region <Region>` | 是 |
| 查看集群详情 | `tccli tke DescribeEKSCluster --ClusterId <ClusterId>` | 是 |
| 按状态筛选集群 | `tccli tke DescribeEKSClusters --filter "ClusterStatus=running"` | 是 |
| 查看集群 kubeconfig | `tccli tke DescribeClusterKubeconfig --ClusterId <ClusterId>` | 是 |
| 查看集群安全配置 | `tccli tke DescribeClusterSecurity --ClusterId <ClusterId>` | 是 |

## 操作步骤

### 1. 列出所有 TKE Serverless 集群

```bash
tccli tke DescribeEKSClusters --region ap-guangzhou
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

```output
{
    "TotalCount": 3,
    "Clusters": [
        {
            "ClusterId": "cls-eks-a1b2c3d4",
            "ClusterName": "eks-prod-microservices",
            "ClusterDescription": "Production microservices cluster",
            "ClusterVersion": "1.28.3",
            "ClusterOs": "tlinux3.1",
            "ClusterType": "SERVERLESS_CLUSTER",
            "ClusterStatus": "Running",
            "VpcId": "vpc-9x8y7z6w",
            "ClusterNetworkSettings": {
                "ClusterCIDR": "172.16.0.0/16",
                "ServiceCIDR": "10.96.0.0/12",
                "SubnetId": "subnet-k1l2m3n4"
            },
            "TagSpecification": [
                {"ResourceType": "cluster", "Tags": [
                    {"Key": "env", "Value": "production"},
                    {"Key": "team", "Value": "platform"}
                ]}
            ],
            "CreatedTime": "2024-08-15T10:30:00Z"
        },
        {
            "ClusterId": "cls-eks-e5f6g7h8",
            "ClusterName": "eks-staging-inference",
            "ClusterDescription": "Online inference staging",
            "ClusterVersion": "1.28.3",
            "ClusterOs": "tlinux3.1",
            "ClusterType": "SERVERLESS_CLUSTER",
            "ClusterStatus": "Running",
            "VpcId": "vpc-9x8y7z6w",
            "ClusterNetworkSettings": {
                "ClusterCIDR": "172.20.0.0/16",
                "ServiceCIDR": "10.96.0.0/12",
                "SubnetId": "subnet-p5q6r7s8"
            },
            "TagSpecification": [
                {"ResourceType": "cluster", "Tags": [
                    {"Key": "env", "Value": "staging"}
                ]}
            ],
            "CreatedTime": "2024-11-20T08:00:00Z"
        },
        {
            "ClusterId": "cls-eks-i9j0k1l2",
            "ClusterName": "eks-dev-offline",
            "ClusterDescription": "Offline batch computing dev",
            "ClusterVersion": "1.26.1",
            "ClusterOs": "tlinux3.1",
            "ClusterType": "SERVERLESS_CLUSTER",
            "ClusterStatus": "Idling",
            "VpcId": "vpc-a1b2c3d4",
            "TagSpecification": [
                {"ResourceType": "cluster", "Tags": [
                    {"Key": "env", "Value": "development"}
                ]}
            ],
            "CreatedTime": "2025-02-01T14:00:00Z"
        }

```json
{
  "TotalCount": 0,
  "Clusters": [],
  "ClusterId": "<ClusterId>",
  "ClusterName": "<ClusterName>",
  "VpcId": "<VpcId

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
```>",
  "SubnetIds": [],
  "K8SVersion": "<K8SVersion>",
  "Status": "<Status>"
}
```
    ],
    "RequestId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
}
```

### 2. 按状态筛选 Serverless 集群

```bash
# 只看 Running 状态的集群
tccli tke DescribeEKSClusters --region ap-guangzhou --filter "ClusterStatus=Running"
```

```json
{
  "RequestId": "..."
}
```

```bash
# 查看非 Running 状态的集群（需关注）
tccli tke DescribeEKSClusters --region ap-guangzhou --filter "ClusterStatus=Idling,Abnormal"
```

```json
{
  "RequestId": "..."
}
```

### 3. 查看单个集群详情

```bash
tccli tke DescribeEKSCluster --region ap-guangzhou --ClusterId cls-eks-a1b2c3d4
```

```json
{
  "RequestId": "..."
}
```

```output
{
    "ClusterId": "cls-eks-a1b2c3d4",
    "ClusterName": "eks-prod-microservices",
    "ClusterVersion": "1.28.3",
    "ClusterStatus": "Run

```json
{
  "Kubeconfig": "<Kubeconfig>",
  "RequestId": "<RequestId>"
}
```ning",
    "VpcId": "vpc-9x8y7z6w",
    "ClusterNetworkSettings": {
        "ClusterCIDR": "172.16.0.0/16",
        "ServiceCIDR": "10.96.0.0/12",
        "MaxClusterServiceNum": 4096,
        "MaxNodePodNum": 256,
        "Subnets": ["subnet-k1l2m3n4"]
    },
    "ClusterNodeNum": 0,
    "ProjectId": 0,
    "Property": "{\"NodeNameType\":\"lan-ip\",\"NetworkType\":\"VPC-CNI\"}",
    "RequestId": "b2c3d4e5-f6a7-8901-bcde-f12345678901"
}
```

> Serverless 集群的 `ClusterNodeNum` 始终为 0 — 没有传统 Node，Pod 由超级节点资源池直接承载。

### 4. 获取 kubeconfig 连接集群

```bash
tccli tke DescribeClusterKubeconfig --region ap-guangzhou --ClusterId cls-eks-a1b2c3d4
```

```output
{
    "Kubeconfig": "apiVersion: v1\nclusters:\n- cluster:\n    certificate-authority-data: LS0t...\n    server: https://cls-eks-a1b2c3d4.ccs.tencent-cloud.com\n  name: cls-eks-a1b2c3d4\n...",
    "RequestId": "c3d4e5f6-a7b8-9012-cdef-234567890123"
}
```

保存到文件后即可使用 kubectl：

```bash
tccli tke DescribeClusterKubeconfig --region ap-guangzhou --ClusterId cls-eks-a1b2c3d4 \
  --filter "Kubeconfig" | jq -r '.Kubeconfig' > ~/.kube/eks-prod-config
kubectl --kubeconfig ~/.kube/eks-prod-config get pods -A
```

### 5. 理解产品概念

#### 什么是 TKE Serverless 集群

TKE Serverless 集群是**完全托管**的 Kubernetes 集群 — 你无需购买、管理云服务器节点，直接通过 Kubernetes API 创建 Pod，底层由腾讯云弹性容器实例（EKS）在超级节点资源池中调度并运行。Pod 之间通过 VPC 网络互通，具备与 CVM 同级别的网络隔离。

#### 相关概念

| 概念 | 说明 | tccli 参考 |
|------|------|-----------|
| 容器和镜像 | 容器是应用运行的标准单元，镜像包含应用及依赖 | `tccli tcr DescribeImages` |
| Kubernetes | 容器编排平台，管理应用部署、伸缩、服务发现 | — |
| 超级节点 | TKE Serverless 底层，一个虚拟可无限扩容的调度单元 | — |
| Pod | Kubernetes 最小调度单元，在 Serverless 集群中直接由 EKS 调度 | `kubectl get pods` |

#### 产品优势

| 优势 | 说明 |
|------|------|
| 原生支持 | 完整兼容 Kubernetes API，标准 kubectl 和 API 调用方式 |
| 无服务器 | 零节点管理，无需规划容量、安装 Agent、处理节点故障 |
| 安全可靠 | Pod 基于独立虚拟机运行，计算和网络隔离；集群内跨可用区高可用 |
| 秒级伸缩 | 从 0 到 N 个 Pod 秒级弹性，无节点预热等待 |
| 降低成本 | 按 Pod 实际运行资源（CPU/内存/GPU）计费，无闲置资源成本 |
| 服务集成 | 与 CLB、VPC、TCR、CLS、Prometheus 监控等云服务原生集成 |

#### 使用限制

| 限制项 | 说明 |
|--------|------|
| 资源规格 | 单 Pod 最大 32 vCPU / 128 GB 内存（含 GPU 型号限制） |
| 存储 | 仅支持 CBS 块存储、CFS 文件存储、COS 对象存储 |
| 网络模式 | 仅支持 VPC-CNI 模式，Pod 直接从 VPC 子网获取 IP |
| 子网 IP | Pod 占用 VPC 子网 IP，需规划充足 IP 地址空间 |
| 地域 | 支持大部分腾讯云地域，具体以可用地区列表为准 |
| 新建集群 | 已关闭新建入口，仅存量用户可继续使用 |

#### 产品定价

按 Pod 实际使用的 vCPU、内存、GPU 规格和运行时长计费（秒级），无最小购买时长。不使用的时段不产生费用。计费由创建 Pod 时声明的 `requests` 决定，而非实际使用量。详见 [购买 TKE Serverless 集群](../购买 TKE Serverless 集群/地域和可用区/tccli 操作.md)。

#### 与容器服务的对比

| 维度 | TKE 标准集群 | TKE Serverless 集群 |
|------|-------------|---------------------|
| 节点管理 | 用户购买和管理 CVM/BM 节点 | 无需管理，平台调度 |
| 容量规划 | 需要预规划节点规格和数量 | 按需弹性，无需预规划 |
| 计费方式 | 包年包月/按量 CVM + 集群管理费 | 按 Pod 实际使用量（vCPU/内存/GPU）计费 |
| 网络模式 | GlobalRouter / VPC-CNI | 仅 VPC-CNI |
| 安全组 | 节点级 + Pod 级 | Pod 级（绑定到弹性网卡） |
| 适合场景 | 稳定负载、需要节点级控制 | 弹性伸缩、间歇性负载、批处理 |

#### 应用场景

| 场景 | 说明 | 示例 |
|------|------|------|
| 微服务 | 无状态服务，按请求量弹性伸缩 | API 网关、Web 后端 |
| 离线计算 | 大批量一次性任务，完成后资源自动回收 | 数据 ETL、AI 训练 |
| 在线推理 | GPU 推理服务，按请求量弹性扩缩 | 模型推理、视频转码 |

#### 相关服务

| 服务 | 用途 | tccli |
|------|------|-------|
| CBS 云硬盘 | Pod 持久化块存储 | `tccli cbs DescribeDisks` |
| CFS 文件存储 | Pod 间共享文件存储 | `tccli cfs DescribeCfsFileSystems` |
| COS 对象存储 | 静态资源、数据湖 | `tccli cos GetBucket` |
| TencentDB | 托管数据库服务 | `tccli cdb DescribeDBInstances` |
| VPC 私有网络 | Pod 网络隔离 | `tccli vpc DescribeVpcs` |
| CLB 负载均衡 | 将流量分发到 Pod | `tccli clb DescribeLoadBalancers` |

## 验证

```

```text
NAME  STATUS  AGE
...
```bash
# 确认集群存在且状态正常
tccli tke DescribeEKSClusters --region ap-guangzhou --filter "ClusterId=cls-eks-a1b2c3d4" --filter "ClusterStatus=Running"

# 确认 kubeconfig 可用
kubectl --kubeconfig ~/.kube/eks-prod-config cluster-info

# 确认 Pod 可正常调度
kubectl --kubeconfig ~/.kube/eks-prod-config run test-pod --image=nginx:alpine --restart=Never --rm -it -- echo "ok"
```

## 清理

无需清理。本页为概念概述，不涉及资源创建。

## 排障

| 现象 | 可能原因 | 处理 |
|------|---------|------|
| `DescribeEKSClusters` 返回空结果 | 地域选择错误，或该地域无 Serverless 集群 | 检查 `--region` 参数，尝试其他地域 |
| 集群状态为 Abnormal | 集群控制面异常 | 提交工单或联系售后排查 |
| 集群状态为 Idling | 集群内无运行中 Pod，进入闲置状态 | 部署工作负载后自动恢复 Running |
| kubectl 连接超时 | 集群 API Server 域名不可达 | 确认 VPC 内访问或已开启公网访问，检查安全组 |
| 新建集群失败 | 新建入口已关闭 | 迁移至 TKE 标准集群 + 超级节点方案 |

## 下一步

- [购买 TKE Serverless 集群（存量用户）](../购买 TKE Serverless 集群/地域和可用区/tccli 操作.md)
- [TKE Serverless 集群管理](../TKE Serverless 集群管理/注意事项/tccli 操作.md)
- [Kubernetes 对象管理（Serverless）](../Kubernetes 对象管理/工作负载管理/tccli 操作.md)
- [常见问题](../常见问题/tccli%20操作.md)

## 控制台替代

控制台：容器服务 → 集群 → 选择 Serverless 集群 → 基本信息 / 节点管理 / 工作负载。
