# IDC 集群添加超级节点用于弹性扩容（tccli）

> 对照官方：[IDC 集群添加超级节点用于弹性扩容](https://cloud.tencent.com/document/product/457/62028) · page_id `62028`

## 概述

超级节点（Virtual Node）是一种 Serverless 节点类型，可将 IDC（自建数据中心）集群的弹性负载直接调度到腾讯云上运行，无需事先购买和管理 CVM 节点。本文档介绍如何使用 `tccli` 命令行工具在已注册到腾讯云的 IDC 集群中添加超级节点，实现混合云场景下的弹性扩缩容。

> **注意**：本文档中涉及 `kubectl` 的操作均需在已通过 VPN、专线或 IOA 等方案与集群控制面网络打通的客户端上执行，`tccli` 无法替代 `kubectl` 对 Kubernetes API 的直接访问。

### 前置条件

- 已有一个注册到腾讯云 TKE 的 IDC 集群（外部集群或注册集群），且集群状态为 **Running**。
- IDC 集群已开启外部节点支持（`EnableExternalNodeSupport`）。
- 已准备好超级节点池所需的 VPC、子网和安全组，且该 VPC 已与 IDC 网络打通（VPN/CCN/专线）。
- 已安装并配置 `tccli`（`tccli configure` 已完成）。
- 已安装 `kubectl` 并配置可访问目标集群（数据面操作需要）。
- 已安装 `jq`（用于解析 JSON 输出）。

### 控制台与 CLI 参数映射

| 控制台参数 | CLI 参数 | 说明 | 幂等性 |
|---|---|---|---|
| 集群 ID | `--ClusterId` | 目标 IDC 集群 ID | 必填，不可变 |
| 地域 | `--region` | 腾讯云地域 | 必填，不可变 |
| 节点池名称 | `--Name` / `Name` (JSON) | 虚拟节点池名称 | 可更新 |
| VPC ID | `--VpcId` / `VpcId` (JSON) | 虚拟节点池所属 VPC | 创建时必填，不可变 |
| 子网 ID 列表 | `--SubnetIds` / `SubnetIds` (JSON) | 虚拟节点池关联子网 | 可更新 |
| 安全组 ID 列表 | `--SecurityGroupIds` / `SecurityGroupIds` (JSON) | 虚拟节点关联安全组 | 可更新 |
| 节点数量 | `--VirtualNodes.N` (JSON) | 要创建的虚拟节点列表 | 增量（每次新增） |
| 虚拟节点名称 | `Name` (JSON) | 虚拟节点名称 | 不可变 |
| 节点标签 | `Labels` (JSON) | 虚拟节点 Kubernetes 标签 | 可更新 |
| 污点 | `Taints` (JSON) | 虚拟节点 Kubernetes 污点 | 可更新 |
| 节点池 ID | `--ClusterVirtualNodePoolId` | 查询/操作目标节点池 | 必填，不可变 |

## 操作步骤

### 步骤 1：确认 IDC 集群状态

#### 选择依据

在操作之前需要确认目标 IDC 集群是否存在且处于正常运行状态，同时检查集群的外部节点能力是否已启用。

#### 最小配置

```bash
tccli tke DescribeClusters \
  --region <Region> \
  --ClusterIds '["CLUSTER_ID"]'
```

**预期输出（关键字段截取）**：

```json
{
  "TotalCount": 1,
  "Clusters": [
    {
      "ClusterId": "cls-xxxxxxxx",
      "ClusterName": "my-idc-cluster",
      "ClusterStatus": "Running",
      "ClusterType": "MANAGED_CLUSTER",
      "ClusterNetworkSettings": {
        "VpcId": "vpc-xxxxxxxx",
        "ClusterCIDR": "10.0.0.0/16"
      }
    }
  ],
  "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

通过 `jq` 提取关键信息：

```bash
tccli tke DescribeClusters \
  --region <Region> \
  --ClusterIds '["CLUSTER_ID"]' \
  | jq '{ClusterId: .Clusters[0].ClusterId, Status: .Clusters[0].ClusterStatus, VpcId: .Clusters[0].ClusterNetworkSettings.VpcId}'
```

**输出示例**：

```json
{
  "ClusterId": "CLUSTER_ID",
  "Status": "Running",
  "VpcId": "vpc-xxxxxxxx"
}
```

确认 `Status` 为 `Running` 后方可继续。若 `TotalCount` 为 0，请检查 `CLUSTER_ID` 和 `REGION` 是否正确。

### 步骤 2：开启集群虚拟节点支持

#### 选择依据

外部节点支持和虚拟节点功能需要在集群级别启用。如果集群来自注册场景（非 TKE 原生创建），必须显式开启。若已开启，此步骤可跳过（幂等操作）。

#### 最小配置

先检查当前外部节点支持状态：

```bash
tccli tke DescribeExternalNodeSupportConfig \
  --region <Region> \
  --ClusterId CLUSTER_ID
```

**预期输出**：

```json
{
  "EnableExternalNode": true,
  "ClusterId": "CLUSTER_ID",
  "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

若 `EnableExternalNode` 为 `false`，执行开启：

```bash
tccli tke EnableExternalNodeSupport \
  --region <Region> \
  --ClusterId CLUSTER_ID \
  --ClusterExternalConfig '{"Enabled": true}'
```

**预期输出**：

```json
{
  "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

> **幂等说明**：重复调用 `EnableExternalNodeSupport` 不会产生副作用，API 返回成功即表示功能已开启。

### 步骤 3：创建虚拟节点池

#### 选择依据

虚拟节点池（Virtual Node Pool）是虚拟节点的管理单元，定义子网、安全组和节点模板。必须先创建节点池，再在池内创建虚拟节点。参数较多（>=4 个），使用 `--cli-input-json` 方式。

#### 最小配置

创建 JSON 输入文件 `create-virtual-nodepool.json`：

```json
{
  "ClusterId": "CLUSTER_ID",
  "Name": "idc-burst-nodepool",
  "VpcId": "VPC_ID",
  "SubnetIds": [
    "subnet-xxxxxxxx"
  ],
  "SecurityGroupIds": [
    "sg-xxxxxxxx"
  ],
  "VirtualNodes": [
    {
      "DisplayName": "virtual-node-01",
      "SubnetId": "subnet-xxxxxxxx",
      "Labels": [
        {
          "Name": "node.kubernetes.io/instance-type",
          "Value": "eklet"
        },
        {
          "Name": "tke.cloud.tencent.com/burst-to-cloud",
          "Value": "true"
        }
      ]
    }
  ]
}
```

执行创建：

```bash
tccli tke CreateClusterVirtualNodePool \
  --cli-input-json file://create-virtual-nodepool.json \
  --region <Region>
```

**预期输出**：

```json
{
  "NodePoolId": "np-xxxxxxxx",
  "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

记录返回的 `NodePoolId`，后续操作需要。

#### 增强配置

带污点和多虚拟节点的 `create-virtual-nodepool.json`：

```json
{
  "ClusterId": "CLUSTER_ID",
  "Name": "idc-burst-nodepool",
  "VpcId": "VPC_ID",
  "SubnetIds": [
    "subnet-xxxxxxxx",
    "subnet-yyyyyyyy"
  ],
  "SecurityGroupIds": [
    "sg-xxxxxxxx"
  ],
  "DeletionProtection": false,
  "VirtualNodes": [
    {
      "DisplayName": "virtual-node-01",
      "SubnetId": "subnet-xxxxxxxx",
      "Labels": [
        {
          "Name": "node.kubernetes.io/instance-type",
          "Value": "eklet"
        },
        {
          "Name": "tke.cloud.tencent.com/burst-to-cloud",
          "Value": "true"
        },
        {
          "Name": "workload-type",
          "Value": "batch"
        }
      ],
      "Taints": [
        {
          "Key": "tke.cloud.tencent.com/burst-to-cloud",
          "Value": "true",
          "Effect": "NoSchedule"
        }
      ]
    },
    {
      "DisplayName": "virtual-node-02",
      "SubnetId": "subnet-yyyyyyyy",
      "Labels": [
        {
          "Name": "node.kubernetes.io/instance-type",
          "Value": "eklet"
        },
        {
          "Name": "tke.cloud.tencent.com/burst-to-cloud",
          "Value": "true"
        },
        {
          "Name": "workload-type",
          "Value": "service"
        }
      ]
    }
  ]
}
```

### 步骤 4：确认节点池创建完成

```bash
tccli tke DescribeClusterVirtualNodePools \
  --region <Region> \
  --ClusterId CLUSTER_ID
```

**预期输出**：

```json
{
  "TotalCount": 1,
  "NodePoolSet": [
    {
      "NodePoolId": "np-xxxxxxxx",
      "Name": "idc-burst-nodepool",
      "ClusterId": "CLUSTER_ID",
      "LifeState": "normal",
      "SubnetIds": ["subnet-xxxxxxxx"],
      "VirtualNodes": [
        {
          "DisplayName": "virtual-node-01",
          "SubnetId": "subnet-xxxxxxxx",
          "State": "running"
        }
      ]
    }
  ],
  "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

确认 `LifeState` 为 `normal` 且虚拟节点 `State` 为 `running`。

### 步骤 5：确认网络连通性（VPC 侧）

#### 选择依据

超级节点运行在指定的 VPC 子网中，必须确保该 VPC 与 IDC 网络的互通已建立。本步骤验证 VPC 配置，实际网络打通（VPN/CCN/专线）需提前完成。

```bash
tccli vpc DescribeVpcs \
  --region <Region> \
  --VpcIds '["VPC_ID"]'
```

**预期输出**：

```json
{
  "VpcSet": [
    {
      "VpcId": "VPC_ID",
      "VpcName": "tke-idc-interconnect",
      "CidrBlock": "10.0.0.0/16",
      "IsDefault": false,
      "EnableMulticast": false,
      "CreatedTime": "2024-01-01T00:00:00Z"
    }
  ],
  "TotalCount": 1,
  "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

如需查看子网：

```bash
tccli vpc DescribeSubnets \
  --region <Region> \
  --SubnetIds '["subnet-xxxxxxxx"]'
```

**预期输出**：

```json
{
  "SubnetSet": [
    {
      "SubnetId": "subnet-xxxxxxxx",
      "SubnetName": "tke-supernode-subnet",
      "VpcId": "VPC_ID",
      "CidrBlock": "10.0.1.0/24",
      "AvailableIpAddressCount": 200,
      "Zone": "ap-guangzhou-6"
    }
  ],
  "TotalCount": 1,
  "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

确认 `AvailableIpAddressCount` 充足。

> **注意**：IDC 与腾讯云 VPC 间的网络打通（通过 VPN 网关、云联网 CCN 或专线接入）是前置条件，本文档不展开。若 `kubectl` 可正常访问集群 API Server，说明网络已通。

### 步骤 6：创建额外虚拟节点

#### 选择依据

若节点池创建时未包含所需数量的虚拟节点，或需要按需新增虚拟节点，使用 `CreateClusterVirtualNode` 接口。

参数较多（>=4 个），使用 `--cli-input-json` 方式。

#### 最小配置

创建 JSON 输入文件 `create-virtual-node.json`：

```json
{
  "ClusterId": "CLUSTER_ID",
  "NodePoolId": "np-xxxxxxxx",
  "VirtualNodes": [
    {
      "DisplayName": "virtual-node-02",
      "SubnetId": "subnet-xxxxxxxx",
      "Labels": [
        {
          "Name": "node.kubernetes.io/instance-type",
          "Value": "eklet"
        }
      ]
    }
  ]
}
```

执行创建：

```bash
tccli tke CreateClusterVirtualNode \
  --cli-input-json file://create-virtual-node.json \
  --region <Region>
```

**预期输出**：

```json
{
  "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

### 步骤 7：验证虚拟节点就绪（数据面）

以下操作需要在已连通集群网络的客户端上执行 `kubectl`（`tccli` 无法替代）：

```bash
kubectl get nodes -l node.kubernetes.io/instance-type=eklet
```

**预期输出**：

```
NAME              STATUS   ROLES    AGE   VERSION
virtual-node-01   Ready    agent    5m    v1.28.3-tke.x
virtual-node-02   Ready    agent    1m    v1.28.3-tke.x
```

查看虚拟节点详情：

```bash
kubectl describe node virtual-node-01
```

**预期输出（关键字段截取）**：

```
Name:               virtual-node-01
Roles:              agent
Labels:             node.kubernetes.io/instance-type=eklet
                    tke.cloud.tencent.com/burst-to-cloud=true
                    tke.cloud.tencent.com/nodepool-id=np-xxxxxxxx
Annotations:        node.alpha.kubernetes.io/ttl=0
Taints:             tke.cloud.tencent.com/burst-to-cloud=true:NoSchedule
Conditions:
  Type             Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
  ----             ------  -----------------                 ------------------                ------                       -------
  MemoryPressure   False   Wed, 01 Jan 2025 12:00:00 +0800   Wed, 01 Jan 2025 12:00:00 +0800   KubeletHasSufficientMemory   kubelet has sufficient memory available
  DiskPressure     False   Wed, 01 Jan 2025 12:00:00 +0800   Wed, 01 Jan 2025 12:00:00 +0800   KubeletHasNoDiskPressure     kubelet has no disk pressure
  PIDPressure      False   Wed, 01 Jan 2025 12:00:00 +0800   Wed, 01 Jan 2025 12:00:00 +0800   KubeletHasSufficientPID      kubelet has sufficient PID available
  Ready            True    Wed, 01 Jan 2025 12:00:00 +0800   Wed, 01 Jan 2025 12:00:00 +0800   KubeletReady                 kubelet is posting ready status
```

确认所有条件为 `True` 状态和 `Ready` 状态。

### 步骤 8：部署测试 Workload 到虚拟节点

#### 选择依据

验证 Pod 是否能成功调度到虚拟节点并正常运行，以确认端到端弹性伸缩能力。

#### 最小配置

创建部署清单 `test-burst-deploy.yaml`，通过 `nodeSelector` 将 Pod 调度到超级节点：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: DEPLOYMENT_NAME
  namespace: NAMESPACE
spec:
  replicas: 2
  selector:
    matchLabels:
      app: burst-test
  template:
    metadata:
      labels:
        app: burst-test
    spec:
      nodeSelector:
        node.kubernetes.io/instance-type: eklet
      tolerations:
        - key: "tke.cloud.tencent.com/burst-to-cloud"
          operator: "Exists"
          effect: "NoSchedule"
      containers:
        - name: nginx
          image: nginx:alpine
          resources:
            requests:
              cpu: "250m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
```

执行部署（数据面操作，需要 `kubectl`）：

```bash
kubectl apply -f test-burst-deploy.yaml
```

**预期输出**：

```
deployment.apps/DEPLOYMENT_NAME created
```

### 步骤 9：验证 Workload 调度与运行

#### 最小配置

```bash
kubectl get pods -n NAMESPACE -l app=burst-test -o wide
```

**预期输出**：

```
NAME                              READY   STATUS    RESTARTS   AGE   IP          NODE              NOMINATED NODE   READINESS GATES
DEPLOYMENT_NAME-xxxxxxxxxx-xxxxx  1/1     Running   0          30s   10.0.1.x    virtual-node-01   <none>           <none>
DEPLOYMENT_NAME-xxxxxxxxxx-yyyyy  1/1     Running   0          30s   10.0.2.x    virtual-node-02   <none>           <none>
```

确认 Pod 的 `NODE` 列显示为虚拟节点名称（`virtual-node-01`、`virtual-node-02`）。

#### 增强配置

验证 Pod 间、Pod 与 IDC 内服务间的网络连通性：

```bash
# 从 IDC 内节点访问超级节点上的 Pod（需网络已打通）
kubectl exec -n NAMESPACE DEPLOYMENT_NAME-xxxxxxxxxx-xxxxx -- ping -c 2 <IDC_INTERNAL_SERVICE_IP>

# 从超级节点上的 Pod 访问 IDC 内服务
kubectl exec -n NAMESPACE DEPLOYMENT_NAME-xxxxxxxxxx-xxxxx -- curl -s -o /dev/null -w "%{http_code}" http://<IDC_INTERNAL_SERVICE_IP>:<PORT>
```

查看 Deployment 状态：

```bash
kubectl get deployment DEPLOYMENT_NAME -n NAMESPACE
```

**预期输出**：

```
NAME              READY   UP-TO-DATE   AVAILABLE   AGE
DEPLOYMENT_NAME   2/2     2            2           5m
```

## 验证

### 控制面验证

验证节点池状态：

```bash
tccli tke DescribeClusterVirtualNodePools \
  --region <Region> \
  --ClusterId CLUSTER_ID \
  | jq '.NodePoolSet[] | {PoolId: .NodePoolId, Name: .Name, LifeState: .LifeState, NodeCount: (.VirtualNodes | length)}'
```

**预期输出**：

```json
{
  "PoolId": "np-xxxxxxxx",
  "Name": "idc-burst-nodepool",
  "LifeState": "normal",
  "NodeCount": 2
}
```

查看节点池内的虚拟节点列表：

```bash
tccli tke DescribeClusterVirtualNodePools \
  --region <Region> \
  --ClusterId CLUSTER_ID \
  | jq '.NodePoolSet[0].VirtualNodes[] | {Name: .DisplayName, SubnetId: .SubnetId, State: .State}'
```

**预期输出**：

```json
{"Name": "virtual-node-01", "SubnetId": "subnet-xxxxxxxx", "State": "running"}
{"Name": "virtual-node-02", "SubnetId": "subnet-yyyyyyyy", "State": "running"}
```

### 数据面验证

```bash
kubectl get nodes -l node.kubernetes.io/instance-type=eklet -o custom-columns=NAME:.metadata.name,STATUS:.status.conditions[-1].type,READY:.status.conditions[-1].status,VERSION:.status.nodeInfo.kubeletVersion
```

**预期输出**：

```
NAME              STATUS   READY   VERSION
virtual-node-01   Ready    True    v1.28.3-tke.x
virtual-node-02   Ready    True    v1.28.3-tke.x
```

验证 Pod 分布：

```bash
kubectl get pods -n NAMESPACE -l app=burst-test -o custom-columns=NAME:.metadata.name,NODE:.spec.nodeName,STATUS:.status.phase,IP:.status.podIP
```

**预期输出**：

```
NAME                              NODE              STATUS    IP
DEPLOYMENT_NAME-xxxxxxxxxx-xxxxx  virtual-node-01   Running   10.0.1.x
DEPLOYMENT_NAME-xxxxxxxxxx-yyyyy  virtual-node-02   Running   10.0.2.x
```

## 清理

> **计费提醒**：超级节点按量计费（vCPU 和内存按实际使用时长计费）。虚拟节点本身不直接产生费用，但调度到虚拟节点上的 Pod 会根据实际资源配置产生费用。请及时清理不再需要的资源。

### 步骤 1：删除测试 Workload

```bash
kubectl delete deployment DEPLOYMENT_NAME -n NAMESPACE
```

**预期输出**：

```
deployment.apps "DEPLOYMENT_NAME" deleted
```

确认 Pod 已清理：

```bash
kubectl get pods -n NAMESPACE -l app=burst-test
```

**预期输出**：

```
No resources found in NAMESPACE namespace.
```

### 步骤 2：删除虚拟节点

```bash
tccli tke DeleteClusterVirtualNode \
  --region <Region> \
  --ClusterId CLUSTER_ID \
  --NodePoolId np-xxxxxxxx \
  --VirtualNodeNames '["virtual-node-01", "virtual-node-02"]'
```

**预期输出**：

```json
{
  "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

### 步骤 3：删除虚拟节点池

```bash
tccli tke DeleteClusterVirtualNodePool \
  --region <Region> \
  --ClusterId CLUSTER_ID \
  --NodePoolId np-xxxxxxxx
```

**预期输出**：

```json
{
  "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

### 步骤 4：验证清理完成

验证节点池已删除：

```bash
tccli tke DescribeClusterVirtualNodePools \
  --region <Region> \
  --ClusterId CLUSTER_ID \
  | jq '.TotalCount'
```

**预期输出**：

```
0
```

验证节点已删除：

```bash
kubectl get nodes -l node.kubernetes.io/instance-type=eklet
```

**预期输出**：

```
No resources found.
```

## 排障

| 问题现象 | 可能原因 | 排查方法 | 解决方案 |
|---|---|---|---|
| 虚拟节点长时间处于 `NotReady` 状态 | 虚拟节点与集群控制面网络不通，或 VPC 子网配置有误 | `kubectl describe node VIRTUAL_NODE_NAME` 查看 Conditions 和 Events；检查 VPC 子网路由表和 ACL 规则 | 确认 VPC 与 IDC 网络已打通（VPN/CCN/专线）；检查子网是否有可用 IP；检查安全组出站规则是否允许访问集群 API Server |
| Pod 无法调度到虚拟节点（Pending 状态） | `nodeSelector` 或 `tolerations` 不匹配，或虚拟节点资源不足 | `kubectl describe pod POD_NAME` 查看 Events；`kubectl get nodes -l node.kubernetes.io/instance-type=eklet` 确认节点 Ready | 检查 Pod 的 `nodeSelector` 与节点标签是否匹配；确认 `tolerations` 覆盖了节点的所有污点；检查虚拟节点是否有足够的可分配资源 |
| Pod 与 IDC 内服务网络不通 | VPC 与 IDC 网络未完全打通，或安全组规则限制了流量 | `kubectl exec POD_NAME -- ping IDC_INTERNAL_IP`；检查安全组入站/出站规则；检查 VPC 路由表 | 确认 VPN/CCN/专线配置正确；放通安全组中必要的端口和协议；检查 VPC 路由表中是否有指向 IDC 网段的路由 |
| 虚拟节点池创建失败 | 子网或安全组不存在，或子网无可用 IP | 查看 CreateClusterVirtualNodePool API 返回的错误信息；`tccli vpc DescribeSubnets` 检查子网状态和可用 IP | 确认子网 ID 和安全组 ID 正确存在；确认子网 `AvailableIpAddressCount` 大于 0；确认安全组未达到数量上限 |
| Pod 卡在 `ContainerCreating` 状态 | 镜像拉取失败，或存储卷挂载失败 | `kubectl describe pod POD_NAME` 查看 Events；检查镜像仓库访问权限 | 确认镜像地址正确且可访问；若镜像在 IDC 私有仓库，确认超级节点所在 VPC 可访问该仓库；检查镜像拉取密钥配置是否正确；确认存储卷后端（如 CBS）在指定可用区可用 |

## 控制台替代

[TKE 控制台](https://console.cloud.tencent.com/tke2)

## 下一步

- [环境准备](../../环境准备.md)
