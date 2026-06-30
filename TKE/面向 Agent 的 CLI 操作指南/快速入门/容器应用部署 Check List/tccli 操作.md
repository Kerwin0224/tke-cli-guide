# 容器应用部署 Check List

> 对照官方：[容器应用部署 Check List](https://cloud.tencent.com/document/product/457/41497) · page_id `41497`

## 概述

在 TKE 集群中部署容器应用前，需逐项验证集群就绪度。本文提供 tccli + kubectl 双通道检查清单，覆盖集群状态、节点可用性、存储、网络、镜像访问、配额六大维度。全部通过后，方可开始部署。

| 检查维度 | CLI 方式 | 说明 |
|---------|---------|------|
| 集群状态 | `DescribeClusters` | 集群必须为 `Running` |
| kubeconfig 可用 | `DescribeClusterKubeconfig` | 获取并验证 kubeconfig |
| 节点可用性 | `DescribeClusterNodePools` + `kubectl get nodes` | 节点数 > 0，状态 Ready |
| 存储 | `kubectl get sc` | 至少一个默认 StorageClass |
| 网络 | `kubectl get svc` | 确认 CoreDNS 等组件运行 |
| 镜像访问 | `kubectl run --image=... --dry-run=client` | 镜像仓库可达 |
| 配额 | `kubectl describe limits` | 确认 Namespace 资源限制 |

## 前置条件

- [环境准备](../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置，region 为目标地域

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:DescribeClusters, tke:DescribeClusterKubeconfig
#    tke:DescribeClusterNodePools
# 验证：执行 DescribeClusters 确认权限
tccli tke DescribeClusters --region <Region>
# expected: exit 0，返回集群列表
```

```json
{
  "TotalCount": "<TotalCount>",
  "Clusters": "<Clusters>",
  "ClusterId": "<ClusterId>",
  "ClusterName": "<ClusterName>",
  "ClusterDescription": "<ClusterDescription>",
  "ClusterVersion": "<ClusterVersion>"
}
```

### 资源检查

```bash
# 4. 确认目标集群存在且状态正常
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]'
# expected: TotalCount >= 1，ClusterStatus 为 Running

# 5. 检查 VPC 配额（如涉及 CLB 或跨 VPC 通信）
tccli vpc DescribeVpcLimits --region <Region> \
    --LimitTypes '["appid-max-vpcs"]'
# expected: 已用量 < 总量
```

```json
{
  "TotalCount": "<TotalCount>",
  "Clusters": "<Clusters>",
  "ClusterId": "<ClusterId>",
  "ClusterName": "<ClusterName>",
  "ClusterDescription": "<ClusterDescription>",
  "ClusterVersion": "<ClusterVersion>"
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看集群状态 | `DescribeClusters` | 是 |
| 获取集群凭证 | `DescribeClusterKubeconfig` | 是 |
| 查看节点池列表 | `DescribeClusterNodePools` | 是 |
| 查看节点列表 | `kubectl get nodes` | 是 |
| 查看存储类 | `kubectl get sc` | 是 |
| 查看资源配额 | `kubectl describe limits` | 是 |

## 操作步骤

### 步骤 1：确认集群状态

```bash
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]'
# expected: exit 0，ClusterStatus 为 Running
```

**预期输出**（以真实集群 `cls-xxxxxxxx` 为例）：

```json
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "cls-xxxxxxxx",
            "ClusterName": "my-tke-cluster",
            "ClusterType": "MANAGED_CLUSTER",
            "ClusterStatus": "Running",
            "ClusterVersion": "1.30.0",
            "ClusterNodeNum": 3,
            "DeletionProtection": true,
            "ClusterDescription": "TKE 托管集群",
            "CreatedTime": "2024-01-01T00:00:00Z"
        }
    ],
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

### 步骤 2：获取并配置 kubeconfig

```bash
tccli tke DescribeClusterKubeconfig --region <Region> \
    --ClusterId CLUSTER_ID | jq -r '.Kubeconfig' > ~/.kube/config
# expected: 生成 ~/.kube/config 文件
```

```json
{
  "Kubeconfig": "<Kubeconfig>",
  "RequestId": "<RequestId>"
}
```

### 步骤 3：检查节点可用性（控制面）

```bash
tccli tke DescribeClusterNodePools --region <Region> \
    --ClusterId CLUSTER_ID
# expected: exit 0，至少 1 个节点池，DesiredNodesNum > 0
```

**预期输出**（以集群 `cls-xxxxxxxx` 为例）：

```json
{
    "TotalCount": 1,
    "NodePoolSet": [
        {
            "NodePoolId": "np-xxxxxxxx",
            "Name": "default-pool",
            "ClusterId": "cls-xxxxxxxx",
            "LifeState": "normal",
            "NodeCountSummary": {
                "ManuallyAdded": {"Total": 3},
                "AutoscalingAdded": {"Total": 0}
            },
            "AutoscalingGroupPara": "{}"
        }
    ],
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

### 步骤 4：检查节点可用性（数据面）

> **kubectl 数据面不可达**：CAM 拒绝公网端点（strategyId:240463971），内网需 VPN/IOA。以下 kubectl 命令需在内网/VPN 环境下执行。

```bash
# 确认节点全部 Ready
kubectl get nodes
# expected: 所有节点 STATUS 为 Ready，且 ROLE 包含 worker 角色
```

**预期输出**：

```
NAME            STATUS   ROLES    AGE   VERSION
10.0.0.2        Ready    <none>   2d    v1.30.0
10.0.0.3        Ready    <none>   2d    v1.30.0
10.0.0.4        Ready    <none>   2d    v1.30.0
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `CLUSTER_ID` | 集群 ID | 格式 `cls-xxxxxxxx`，必须属于当前账号/地域 | `tccli tke DescribeClusters --region <Region>` |

### 步骤 5：检查存储准备

> **kubectl 数据面不可达**：CAM 拒绝公网端点（strategyId:240463971），内网需 VPN/IOA。以下命令需在内网/VPN 环境下执行。

```bash
# 确认至少有一个 StorageClass
kubectl get sc
# expected: 至少 1 个 StorageClass，默认 SC 标注 (default)
```

**预期输出**：

```
NAME                  PROVISIONER                    RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
cbs (default)         com.tencent.cloud.csi.cbs      Delete          Immediate           true                   2d
cfs                   com.tencent.cloud.csi.cfs      Retain          Immediate           false                  2d
```

若无默认 StorageClass，部署 PVC 的应用将处于 Pending 状态。

### 步骤 6：检查网络组件

> **kubectl 数据面不可达**：CAM 拒绝公网端点（strategyId:240463971），内网需 VPN/IOA。以下命令需在内网/VPN 环境下执行。

```bash
# 确认 CoreDNS 等关键组件运行正常
kubectl get pods -n kube-system -l k8s-app=kube-dns
# expected: 所有 Pod STATUS 为 Running

# 确认 kube-proxy 运行正常
kubectl get pods -n kube-system -l k8s-app=kube-proxy
# expected: 所有 Pod STATUS 为 Running
```

**预期输出**：

```
NAME                       READY   STATUS    RESTARTS   AGE
coredns-xxx-xxx            1/1     Running   0          2d
coredns-xxx-yyy            1/1     Running   0          2d
```

### 步骤 7：检查镜像访问

> **kubectl 数据面不可达**：CAM 拒绝公网端点（strategyId:240463971），内网需 VPN/IOA。以下命令需在内网/VPN 环境下执行。

```bash
# 测试是否能拉取公共镜像
kubectl run image-test --image=registry.k8s.io/pause:3.9 --restart=Never --dry-run=client -o yaml
# expected: 返回 Pod YAML 内容（dry-run 仅验证参数，不实际创建）
```

如需使用 TCR 或私有镜像仓库，还需验证镜像仓库登录凭据（`kubectl get secrets` 确认 `docker-registry` 类型 Secret 存在）。

### 步骤 8：检查 Namespace 资源配额

> **kubectl 数据面不可达**：CAM 拒绝公网端点（strategyId:240463971），内网需 VPN/IOA。以下命令需在内网/VPN 环境下执行。

```bash
# 查看目标 Namespace 的资源限制
kubectl describe limits -n NAMESPACE
# expected: 如已设置 ResourceQuota，显示 CPU/内存限制
```

```text
NAME  STATUS  AGE
...
```

若无输出，说明未设置 ResourceQuota，应用可无限制使用集群资源。

## 验证

### 控制面（tccli）

```bash
# 验证集群状态
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]' \
    | jq '{ClusterStatus: .Clusters[0].ClusterStatus, ClusterNodeNum: .Clusters[0].ClusterNodeNum}'
# expected: ClusterStatus 为 "Running"，ClusterNodeNum > 0

# 验证节点池状态
tccli tke DescribeClusterNodePools --region <Region> \
    --ClusterId CLUSTER_ID \
    | jq '.NodePoolSet[].LifeState'
# expected: "normal"

# 验证 kubeconfig 可用
tccli tke DescribeClusterKubeconfig --region <Region> \
    --ClusterId CLUSTER_ID \
    | jq '.Kubeconfig' | head -c 100
# expected: 返回 base64 编码的 kubeconfig（非空）
```

```json
{
  "TotalCount": "<TotalCount>",
  "Clusters": "<Clusters>",
  "ClusterId": "<ClusterId>",
  "ClusterName": "<ClusterName>",
  "ClusterDescription": "<ClusterDescription>",
  "ClusterVersion": "<ClusterVersion>"
}
```

### 数据面

> **kubectl 数据面不可达**：CAM 拒绝公网端点（strategyId:240463971），内网需 VPN/IOA。以下命令需在内网/VPN 环境下执行。

```bash
# 验证节点就绪
kubectl get nodes --no-headers | grep -v Ready
# expected: 无输出（所有节点 Ready）

# 验证系统组件健康
kubectl get componentstatuses 2>/dev/null || echo "componentstatuses 不可用（托管集群正常）"
# expected: componentstatuses 不可用（托管集群正常）

# 验证 CoreDNS
kubectl get deployment coredns -n kube-system
# expected: READY 列显示期望副本数全部就绪
```

**预期输出**：

```
NAME      READY   UP-TO-DATE   AVAILABLE   AGE
coredns   2/2     2            2           2d
```

## 清理

本页为检查清单，不创建持久化资源。如执行了 `kubectl run --dry-run` 测试，不会产生实际 Pod，无需清理。

> 测试时创建的临时文件（如 `~/.kube/config` 副本）可按需清理。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DescribeClusters` 返回 `InvalidParameter.ClusterId` | 检查 `--ClusterIds` 中的集群 ID 格式 | 集群 ID 格式错误或不属于当前账号/地域 | `tccli tke DescribeClusters --region <Region>` 列出全部集群确认 ID |
| `DescribeClusterKubeconfig` 返回 `FailedOperation.ClusterState` | `tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'` 检查集群状态 | 集群非 Running 状态，无法生成有效 kubeconfig | 等待集群状态变为 Running 后重试 |
| `kubectl get nodes` 返回 `Unable to connect to the server` | `cat ~/.kube/config` 检查 server 地址；`curl -k SERVER_ADDRESS` 测试连通性 | kubeconfig 中的 API Server 地址不可达（公网未开或网络不通） | `tccli tke DescribeClusterEndpoints --region <Region> --ClusterId CLUSTER_ID` 检查端点；若未开启外网访问，前往控制台开启或使用内网访问 |
| `kubectl get nodes` 返回 `error: You must be logged in` | `kubectl config view` 检查当前上下文 | kubeconfig 过期或凭据无效 | 重新执行 `DescribeClusterKubeconfig` 获取新凭据 |

### 检查通过但部署异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 节点数 = 0 | `tccli tke DescribeClusterNodePools --region <Region> --ClusterId CLUSTER_ID` 查看 `NodeCountSummary` | 集群为空（无工作节点） | [新增游离普通节点](../../集群配置/节点管理/普通节点/新增游离普通节点/tccli%20操作.md) 或新建节点池 |
| 节点有 Ready 但 Pod 一直 Pending | `kubectl describe pod POD_NAME` 查看 Events | 资源不足（CPU/内存）或 PVC 无匹配 StorageClass | `kubectl top nodes` 检查节点资源；`kubectl get sc` 检查存储类 |
| 镜像拉取失败（`ImagePullBackOff`） | `kubectl describe pod POD_NAME` 查看 Events | 镜像地址错误、仓库不可达、或未配置拉取凭据 | 检查镜像地址拼写；`kubectl get secrets` 确认 `docker-registry` 类型 Secret；若使用 TCR 私有仓库，参考 [TCR 快速入门](https://cloud.tencent.com/document/product/1141/39287) |
| `kubectl describe limits` 显示配额已满 | `kubectl describe quota -n NAMESPACE` 查看当前用量 | ResourceQuota 已耗尽 | 扩容配额或清理不再使用的资源 |

## 下一步

- [创建简单的 Nginx 服务](https://cloud.tencent.com/document/product/457/7851) — 部署第一个工作负载
- [手动搭建 Hello World 服务](https://cloud.tencent.com/document/product/457/7204) — 手动部署验证
- [构建简单 Web 应用](https://cloud.tencent.com/document/product/457/6996) — 完整 Web 应用部署
- [新增节点](https://cloud.tencent.com/document/product/457/32203) — 为集群添加工作节点
- [连接集群](https://cloud.tencent.com/document/product/457/32191) — 多种方式连接集群

## 控制台替代

[TKE 控制台 → 集群详情](https://console.cloud.tencent.com/tke2/cluster)：集群详情页内「节点管理」「存储」「网络」「监控」等 Tab 覆盖本文所有检查维度。
