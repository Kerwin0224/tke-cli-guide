# 超级节点常见问题

> 对照官方：[超级节点常见问题](https://cloud.tencent.com/document/product/457/60411) · page_id `60411`

## 概述

汇总超级节点使用中的**常见问题**和解决方案，覆盖创建、调度、计费、网络、镜像等方面。本页以 tccli 诊断命令为主入口，结合排障路径定位问题根因。

**诊断入口**：

- 控制面：`DescribeClusterVirtualNodePools` 查看节点池状态和配置
- 控制面：`DescribeClusterVirtualNodes` 查看超级节点详情
- 数据面：`kubectl describe pod` 查看 Pod Events 和调度状态

## 前置条件

- [环境准备](../../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查 kubectl 已安装且可连接集群
kubectl version --client
# expected: 显示 Client Version

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:DescribeClusterVirtualNodePools, tke:DescribeClusterVirtualNodes
#    tke:DescribeClusterKubeconfig
# 验证：
tccli tke DescribeClusterVirtualNodePools --region <Region> --ClusterId CLUSTER_ID
# expected: exit 0，返回节点池列表
```

```json
{
  "TotalCount": "<TotalCount>",
  "NodePoolSet": "<NodePoolSet>",
  "NodePoolId": "<NodePoolId>",
  "SubnetIds": "<SubnetIds>",
  "Name": "<Name>",
  "LifeState": "<LifeState>"
}
```

### 资源检查

```bash
# 4. 确认集群和超级节点池状态
tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterStatus 为 Running

tccli tke DescribeClusterVirtualNodePools --region <Region> \
    --ClusterId CLUSTER_ID
# expected: NodePoolSet 非空

# 5. 获取 kubeconfig 用于数据面诊断
tccli tke DescribeClusterKubeconfig --region <Region> --ClusterId CLUSTER_ID \
    | jq -r '.Kubeconfig' > ~/.kube/config-super
kubectl --kubeconfig ~/.kube/config-super cluster-info
# expected: Kubernetes control plane 可达
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

| 控制台操作 | tccli / kubectl | 幂等 |
|-----------|-----------|:--:|
| 查看超级节点池 | `DescribeClusterVirtualNodePools` | 是 |
| 查看超级节点详情 | `DescribeClusterVirtualNodes` | 是 |
| 查看 Pod 事件 | `kubectl describe pod` | 是 |
| 查看节点信息 | `kubectl describe node` | 是 |
| 获取 kubeconfig | `DescribeClusterKubeconfig` | 是 |
| 查询集群信息 | `DescribeClusters` | 是 |

## 操作步骤

### 通用诊断流程

当超级节点出现问题时，按以下顺序排查：

```bash
# 1. 确认节点池状态
tccli tke DescribeClusterVirtualNodePools --region <Region> \
    --ClusterId CLUSTER_ID
# expected: LifeState 为 normal

# 2. 确认超级节点存在且状态正常
tccli tke DescribeClusterVirtualNodes --region <Region> \
    --NodePoolId NODE_POOL_ID
# expected: VirtualNodeSet 非空

# 3. 查看问题 Pod 状态和 Events
kubectl --kubeconfig ~/.kube/config-super describe pod POD_NAME
# expected: Events 显示最近的操作和错误信息

# 4. 查看问题 Pod 所在节点详情
kubectl --kubeconfig ~/.kube/config-super describe node NODE_NAME
# expected: 节点信息、容量、Conditions
```

**预期输出**（DescribeClusterVirtualNodePools）：

```json
{
    "TotalCount": 1,
    "NodePoolSet": [
        {
            "NodePoolId": "np-example",
            "Name": "NODE_POOL_NAME",
            "LifeState": "normal",
            "SubnetIds": ["SUBNET_ID"],
            "NodeCountSummary": {
                "ManuallyAdded": {"Description": "1"},
                "AutoscalingAdded": {"Description": "0"}
            }
        }
    ],
    "RequestId": "..."
}
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `POD_NAME` | 有问题的 Pod 名称 | — | `kubectl get pods` |
| `NODE_NAME` | Pod 调度的节点名称 | — | `kubectl get pod POD_NAME -o wide` |
| `NODE_POOL_ID` | 超级节点池 ID | 格式 `np-xxxxxxxx` | `DescribeClusterVirtualNodePools` |

## 验证

### 确认诊断结果

```bash
# 确认诊断命令可执行
tccli tke DescribeClusterVirtualNodePools --region <Region> \
    --ClusterId CLUSTER_ID
# expected: exit 0，RequestId 非空

# 确认 kubectl 可达
kubectl --kubeconfig ~/.kube/config-super get ns
# expected: 返回 default 等系统命名空间
```

```json
{
  "TotalCount": "<TotalCount>",
  "NodePoolSet": "<NodePoolSet>",
  "NodePoolId": "<NodePoolId>",
  "SubnetIds": "<SubnetIds>",
  "Name": "<Name>",
  "LifeState": "<LifeState>"
}
```

## 清理

本页为常见问题诊断页，仅执行只读操作，无资源需清理。

## 排障

### 创建超级节点池失败

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `CreateClusterVirtualNodePool` 返回 `InvalidParameter.SubnetId` | `tccli vpc DescribeSubnets --region <Region> --SubnetIds '["SUBNET_ID"]'` 确认子网存在 | 子网 ID 不存在或子网不属于目标 VPC | 用 `tccli vpc DescribeSubnets --region <Region> --Filters '[{"Name":"vpc-id","Values":["VPC_ID"]}]'` 获取 VPC 下的可用子网 ID |
| `CreateClusterVirtualNodePool` 返回 `InvalidParameter.SecurityGroupId` | `tccli vpc DescribeSecurityGroups --region <Region> --SecurityGroupIds '["SECURITY_GROUP_ID"]'` | 安全组 ID 不存在 | 用 `tccli vpc DescribeSecurityGroups --region <Region>` 获取可用安全组 |
| `CreateClusterVirtualNodePool` 返回 `LimitExceeded` | `tccli tke DescribeClusterVirtualNodePools --region <Region> --ClusterId CLUSTER_ID` 查看已有节点池数量 | 超级节点池数量达到配额上限（此为环境限制，非命令错误） | 删除不再使用的超级节点池后重试；如需提升配额，提交工单 |
| 包年包月选项不可用 | `tccli tke DescribeClusterVirtualNodePools --region <Region> --ClusterId CLUSTER_ID` | 当前地域不支持包年包月超级节点 | 切换至支持的地域，或使用按量计费模式 |

### Pod 无法调度到超级节点

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Pod Pending，Events 显示 `0/1 nodes are available` | `kubectl describe pod POD_NAME \| tail -20` | CPU 或内存超出超级节点可调度规格 | 参见 [超级节点可调度Pod说明](../超级节点调度说明/超级节点可调度Pod说明/tccli%20操作.md) 调整规格 |
| Pod Pending，Events 显示 `node(s) didn't match node selector` | `kubectl get nodes --show-labels \| grep instance-type` | nodeSelector 标签键名不匹配 | 确认使用 `node.kubernetes.io/instance-type: eklet` |
| Pod Pending，Events 显示 `node(s) had untolerated taint` | `kubectl describe node NODE_NAME \| grep -A5 Taints` | Pod 未声明对超级节点 taint 的 tolerations | 在 Pod spec 中增加 tolerations 配置 |
| Pod 调度到普通节点而非超级节点 | `kubectl get pod POD_NAME -o wide` 查看 NODE | 未设置 `nodeSelector: {node.kubernetes.io/instance-type: eklet}` 或自动调度条件不满足 | 添加 nodeSelector 显式指定；或确认按量超级节点池的自动调度条件（普通节点资源不足） |

### 超级节点上 Pod 运行异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Pod 运行中突然被驱逐 | `kubectl describe pod POD_NAME \| tail -30` 查看 Events | 系统资源不足或超级节点维护 | 保留 Pod 名称、命名空间、时间戳 → 查看 Events 确认驱逐原因（Evicted/Preempted）。增加 Pod 优先级或使用包年包月超级节点 |
| Pod 中无法访问外网 | `kubectl exec POD_NAME -- curl -sI https://cloud.tencent.com --connect-timeout 5` | 超级节点池绑定的子网未配置 NAT 网关或公网 IP | 为超级节点池子网配置 NAT 网关或使用绑定了 EIP Annotation 的 Pod |
| Pod 启动慢、镜像拉取超时 | `kubectl describe pod POD_NAME \| grep -A5 "Pulling image"` | 镜像较大且未使用镜像缓存 | 参见 [镜像缓存](../使用超级节点/镜像缓存/tccli%20操作.md) 添加 `cbs-reuse-key` Annotation |
| Pod 日志未出现在 CLS | `kubectl describe pod POD_NAME \| grep -i cls` 查看 Annotation | CLS 日志集/主题 ID 配置错误或拼写错误 | 参见 [采集超级节点上的Pod日志](../使用超级节点/采集超级节点上的Pod日志/tccli%20操作.md) 检查 Annotation 配置 |
| DaemonSet 未调度到超级节点 | `kubectl describe ds DAEMONSET_NAME \| tail -20` | 缺少 taint tolerations | 参见 [超级节点上支持运行DaemonSet](../使用超级节点/超级节点上支持运行DaemonSet/tccli%20操作.md) 配置 tolerations |

### 计费相关疑问

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 按量超级节点池删除后仍收到账单 | 账单周期以 UTC+8 自然日为单位 | 删除前已产生的用量计入当前账单周期 | 无。此为正常计费行为。删除后不再产生新费用 |
| 包年包月超级节点池到期前能否退款 | — | 包年包月资源到期前一般不退还剩余费用 | 购买前确认需求，建议先创建按量计费超级节点池验证后再购买包年包月 |

### 操作成功但结果不符合预期

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DescribeClusterVirtualNodePools` 返回空列表但控制台显示有节点池 | `tccli configure list` 确认 region | tccli 配置的 region 与控制台当前 region 不一致 | 用 `tccli configure set region <Region>` 调整至与控制台一致的地域 |
| 修改超级节点池配置后未生效 | `tccli tke DescribeClusterVirtualNodePools --region <Region> --ClusterId CLUSTER_ID --NodePoolIds '["NODE_POOL_ID"]'` 查看当前配置 | 修改操作未成功或修改结果有延迟 | 重新执行 ModifyClusterVirtualNodePool 命令；等待 10-30 秒后重试 DescribeClusterVirtualNodePools |
| Annotation 修改后 Pod 行为不变 | `kubectl get pod POD_NAME -o yaml` 查看当前 Annotation | 修改已运行 Pod 的 Annotation 不触发重新调度 | 删除 Pod（由 Deployment 自动重建) 或滚动更新 Deployment |

## 下一步

- [新建超级节点](https://cloud.tencent.com/document/product/457/78328) — 创建超级节点池和虚拟节点
- [超级节点概述](https://cloud.tencent.com/document/product/457/74014) — 超级节点 vs 普通节点对比
- [调度 Pod 至超级节点](https://cloud.tencent.com/document/product/457/74016) — 调度策略配置
- [超级节点可调度 Pod 说明](https://cloud.tencent.com/document/product/457/74015) — 可调度规格和限制
- [超级节点 Annotation 说明](https://cloud.tencent.com/document/product/457/44173) — 完整 Annotation 列表

## 控制台替代

[TKE 控制台 → 工单/FAQ](https://console.cloud.tencent.com/tke2/cluster)：控制台内提交工单或查阅官方 FAQ。
