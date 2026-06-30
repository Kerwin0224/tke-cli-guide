# 创建注册集群（tccli）

> 对照官方：[创建注册集群](https://cloud.tencent.com/document/product/457/63218) · page_id `63218`

## 概述

将外部 Kubernetes 集群注册到腾讯云 TKE，纳管为**注册集群**。注册集群基于开源 [Clusternet](https://github.com/clusternet/clusternet) 多集群治理框架，以 Hub Cluster + Child Cluster 架构运行：Hub 集群部署在腾讯云广州/北京/新加坡，Child 集群（你的外部集群）通过 `clusternet-agent` 连接至 Hub。

本指南覆盖完整 CLI 路径：创建 Hub 集群、创建注册集群、获取并执行注册命令。

## 前置条件

- 已完成 [环境准备](../../../../../环境准备.md)：`tccli` 已安装、凭证与 `region` 已配置。
- 外部 Kubernetes 集群已就绪，版本范围为 **1.18.x ~ 1.24.5**。
- 外部集群的控制面可通过 kubectl 访问，且需具备 cluster-admin 权限（注册 agent 需要创建 `clusternet-system` 命名空间及 RBAC 资源）。
- Hub 集群开通地域仅支持：`ap-guangzhou`（广州）、`ap-beijing`（北京）、`ap-singapore`（新加坡）。
- Hub 集群需 VPC 与子网，用于部署 Hub 侧代理网卡（proxy ENI）。若你尚无 VPC，请先创建。

## 控制台与 CLI 参数映射

| 控制台 | tccli / JSON 字段 | 幂等 |
|--------|------------------|------|
| Hub 集群名称 | `ClusterBasicSettings.ClusterName` | 否 |
| Hub 开通地域 | `--region`（仅广州/北京/新加坡） | 否 |
| Hub VPC / 子网 | `ClusterBasicSettings.VpcId` / `SubnetId` | 否 |
| 注册集群名称（≤60 字符） | `ClusterBasicSettings.ClusterName` | 否 |
| 注册集群接入地域 | `--region` | 否 |
| 腾讯云标签 | `ClusterBasicSettings.TagSpecification` | 是 |
| 集群描述 | `ClusterBasicSettings.Description` | 是 |
| 获取注册命令 | `DescribeClusterRegistrationCommand`（API 待验证） | 是 |
| 查看注册集群列表 | `DescribeClusters` + `ClusterType` 过滤 | 是 |

## 操作步骤

### 1. 网络前置 — 确认 Hub 集群 VPC 与子网

Hub 集群需在本步骤创建前已有 VPC。确认可用 VPC：

```bash
tccli vpc DescribeVpcs --region ap-guangzhou --output json \
  --filter "VpcSet[0:3].{VpcId:VpcId,VpcName:VpcName,CidrBlock:CidrBlock}"
```

```json
[
  {
    "VpcId": "vpc-hub-gz-001",
    "VpcName": "tke-hub-vpc-gz",
    "CidrBlock": "10.0.0.0/16"
  },
  {
    "VpcId": "vpc-hub-gz-002",
    "VpcName": "tke-hub-vpc-gz-dr",
    "CidrBlock": "10.1.0.0/16"
  }
]
```

确认子网可用：

```bash
tccli vpc DescribeSubnets --region ap-guangzhou \
  --Filters "[{\"Name\":\"vpc-id\",\"Values\":[\"vpc-hub-gz-001\"]}]" --output json \
  --filter "SubnetSet[0:3].{SubnetId:SubnetId,SubnetName:SubnetName,AvailableIpAddressCount:AvailableIpAddressCount}"
```

```json
[
  {
    "SubnetId": "subnet-hub-gz-01",
    "SubnetName": "tke-hub-subnet-az1",
    "AvailableIpAddressCount": 248
  }
]
```

### 2. 创建 Hub 集群（TDCC Hub）

Hub 集群是注册集群的管控面。如果你已有 Hub 集群，跳过本步直接进入步骤 3。

使用 JSON 桥接创建 Hub 集群（含 `ClusterType` 指定注册集群 Hub 用途及代理网卡配置）：

```bash
cat > /tmp/create-hub-cluster.json << 'EOF'
{
  "ClusterBasicSettings": {
    "ClusterName": "hub-cls-example",
    "ClusterVersion": "1.24.4",
    "VpcId": "vpc-hub-gz-001",
    "SubnetId": "subnet-hub-gz-01",
    "Description": "TDCC Hub cluster for registered child clusters",
    "TagSpecification": [
      {
        "ResourceType": "cluster",
        "Tags": [
          {"Key": "env", "Value": "production"},
          {"Key": "managed-by", "Value": "tccli"}
        ]
      }
    ]
  },
  "ClusterType": "MANAGED_CLUSTER",
  "ClusterAdvancedSettings": {
    "ContainerRuntime": "containerd",
    "RuntimeVersion": "1.6.9"
  }
}
EOF

tccli tke CreateCluster \
  --cli-input-json file:///tmp/create-hub-cluster.json \
  --region ap-guangzhou \
  --output json
```

```json
{
  "ClusterId": "hub-cls-example",
  "Response": {
    "RequestId": "ba76f8c8-6f05-47b6-93f4-19cf7fa3a1e4"
  }
}
```

等待 Hub 集群就绪：

```bash
tccli tke DescribeClusters --region ap-guangzhou --output json \
  --filter "Clusters[?ClusterId=='hub-cls-example'].{ClusterId:ClusterId,ClusterStatus:ClusterStatus}"
```

```json
{
  "TotalCount": 0,
  "Clusters": [],
  "ClusterId": "<ClusterId>",
  "ClusterName": "<ClusterName>",
  "ClusterDescription": "<ClusterDescription>",
  "ClusterVersion": "<ClusterVersion>",
  "ClusterOs": "<ClusterOs>",
  "ClusterType": "<ClusterType>"
}
```

重复执行直至 `ClusterStatus` 为 `Running`。

### 3. 创建注册集群

注册集群代表你的外部 Child 集群在 TKE 控制台的逻辑对象。创建时指定名称、接入地域及标签。

```bash
cat > /tmp/create-registered-cluster.json << 'EOF'
{
  "ClusterBasicSettings": {
    "ClusterName": "prod-uswest-registered",
    "ClusterVersion": "1.23.6",
    "Description": "Production registered cluster in US West - on-prem Kubernetes",
    "TagSpecification": [
      {
        "ResourceType": "cluster",
        "Tags": [
          {"Key": "env", "Value": "production"},
          {"Key": "location", "Value": "us-west-onprem"},
          {"Key": "managed-by", "Value": "tccli"}
        ]
      }
    ]
  },
  "ClusterType": "REGISTERED_CLUSTER",
  "ClusterCIDRSettings": {
    "IgnoreClusterCIDRConflict": true
  }
}
EOF

tccli tke CreateCluster \
  --cli-input-json file:///tmp/create-registered-cluster.json \
  --region ap-guangzhou \
  --output json
```

```json
{
  "ClusterId": "cls-registered-example",
  "Response": {
    "RequestId": "7a3e1f2d-8c45-492b-a1e3-b5d9f0c2e8a1"
  }
}
```

记录返回的 `ClusterId`（示例中为 `cls-registered-example`），后续步骤均需此 ID。

### 4. 获取并执行注册命令

创建注册集群后，系统生成一条有效期 **24 小时**的注册命令。此命令会在外部集群中部署 `clusternet-agent`。

```bash
tccli tke DescribeClusterRegistrationCommand \
  --ClusterId cls-registered-example \
  --region ap-guangzhou \
  --output json
```

```json
{
  "RegistrationCommand": "curl -fsSL https://tke-register.tencentcloud.com/install.sh | bash -s -- --token eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9... --hub-server https://hub.ap-guangzhou.tke.tencentcs.com",
  "TokenExpireTime": "2025-01-15T12:00:00+08:00",
  "RequestId": "f1a2b3c4-5678-90ab-cdef-1234567890ab"
}
```

在**外部集群**的 kubectl 上下文中执行注册命令：

```bash
# 在外部集群中执行
kubectl get nodes
# 确认你正在正确的集群上操作

# 执行注册命令（复制 DescribeClusterRegistrationCommand 返回的命令）
curl -fsSL https://tke-register.tencentcloud.com/install.sh | bash -s -- \
  --token <your-token> \
  --hub-server https://hub.ap-guangzhou.tke.tencentcs.com
```

注册脚本执行的操作：
- 创建 `clusternet-system` 命名空间
- 部署 `clusternet-agent` DaemonSet/Deployment
- 配置 RBAC（ServiceAccount、ClusterRole、ClusterRoleBinding）
- 注入 Hub Server 地址与注册 Token

若注册命令过期（超过 24 小时），需在控制台或通过 

```json
{
  "TotalCount": 0,
  "Clusters": [],
  "ClusterId": "<ClusterId>",
  "ClusterName": "<ClusterName>",
  "ClusterDescription": "<ClusterDescription>",
  "ClusterVersion": "<ClusterVersion>",
  "ClusterOs": "<ClusterOs>",
  "ClusterType": "<ClusterType>"
}
```API 重新生成。

## 验证

### 注册状态（tccli）

```bash
tccli tke DescribeClusters --region ap-guangzhou --output json \
  --filter "Clusters[?ClusterId=='cls-registered-example'].{ClusterId:ClusterId,ClusterName:ClusterName,ClusterStatus:ClusterStatus,ClusterType:ClusterType}"
```

```json
[
  {
    "ClusterId": "cls-registered-example",
    "ClusterName": "prod-uswest-registered",
    "ClusterStatus": "Running",
    "ClusterType": "REGISTERED_CLUSTER"
  }
]
```

### clusternet-agent Pod 状态（在外部集群中）

```bash
kubectl get pod -n clusternet-system
```

预期输出：

```
NAME                                  READY   STATUS    RESTARTS   AGE
clusternet-agent-7d8f9c6b5b-8xq2k   1/1     Running   0          2m
clusternet-agent-7d8f9c6b5b-p4v7n   1/1     Running   0          2m
```

### clusternet-scheduler（在 Hub 集群中）

```bash
kubectl --context hub-cls-example get pod -n clusternet-system
```

确认 `clusternet-controller-manager` 和 `clusternet-scheduler` 处于 `Running` 状态。

## 清理

### 解除注册（保留外部集群）

解除注册会删除 TKE 侧的注册集群记录及 agent 软件，**外部集群的业务 Pod 与资源不受影响**。

```bash
tccli tke DeleteCluster \
  --ClusterId cls-registered-example \
  --InstanceDeleteMode terminate \
  --region ap-guangzhou \
  --output json
```

```json
{
  "Response": {
    "RequestId": "e04b8f1a-7d6c-49e5-b3f2-a8c9d0e1f2a3"
  }
}
```

### 清理 Hub 集群（如不再需要）

```bash
tccli tke DeleteCluster \
  --ClusterId hub-cls-example \
  --InstanceDeleteMode terminate \
  --region ap-guangzhou \
  --output json
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `ClusterType` 不支持 `REGISTERED_CLUSTER` | `tccli --version` 检查 CLI 版本 | tccli 版本过旧，不识别新增 `ClusterType` 枚举 | 升级 tccli 至最新版：`pip install tccli --upgrade`；或使用 `--cli-input-json file://...` JSON 桥接方式 |
| Hub 集群 `CreateCluster` 报 CIDR 冲突 | `tccli tke DescribeClusters --region <Region>` 对比已有集群 CIDR | Hub 集群的 ClusterCIDR 或 ServiceCIDR 与已有集群冲突 | 设置 `IgnoreClusterCIDRConflict: true` 或更换不冲突的 CIDR 网段 |
| 注册命令过期（24 小时后） | — | 注册命令 Token 有效期为 24 小时 | 重新调用 `tccli tke DescribeClusterRegistrationCommand --ClusterId <ClusterId> --region <Region>` 获取新命令 |

### 注册已提交但状态异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `clusternet-agent` CrashLoopBackOff | `kubectl logs -n clusternet-system deploy/clusternet-agent` 查看日志 | 外部集群与 Hub 间网络不通（TLS 双向认证需放通 443 端口） | 检查防火墙/安全组规则放通 443 端口；验证 Hub 集群 API Server 公网/内网端点可达 |
| Hub 集群与 Child 集群 API Server 版本不兼容 | `kubectl version` 对比双方 K8s 版本 | Hub 集群 K8s 版本必须 ≥ 外部集群版本 | 升级 Hub 集群或降级外部集群版本 |
| 注册后 TKE 控制台看不到节点 | `kubectl --context hub-cls-example get managedcluster` 检查心跳状态 | 节点同步有 1-2 分钟延迟 | 等待 1-2 分钟后刷新控制台；若持续异常，检查 `clusternet-agent` 运行状态 |

## 下一步

- [连接注册集群](../连接注册集群/tccli%20操作.md) — 获取 kubeconfig 并访问注册集群
- [解除注册集群](../解除注册集群/tccli%20操作.md) — 从 TKE 移除集群纳管
- [日志采集](../../运维指南/日志采集/tccli%20操作.md) — 为注册集群配置 CLS 日志采集
- [集群审计](../../运维指南/集群审计/tccli%20操作.md) — 配置注册集群审计日志

## 控制台替代

[控制台 → 集群 → 注册集群 → 新建](https://console.cloud.tencent.com/tke2/cluster?action=CreateRegisteredCluster) — 控制台提供可视化向导：选择 Hub 集群地域、填写集群信息、生成注册命令。
