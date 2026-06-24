# 服务升级指南

> 对照官方：[服务升级指南](https://cloud.tencent.com/document/product/457/104910) · page_id `104910`

## 概述

原生节点的服务组件（kubelet、containerd、节点管家等）需要定期升级以获取安全补丁和新特性。原生节点升级采用**滚动升级**策略：一次升级少量节点，验证无误后再继续，最大程度减少对业务的影响。

原生节点涉及的组件层级及升级方式：

| 组件 | 升级方式 | 对业务影响 | API 操作 |
|------|---------|:--:|------|
| kubelet | 滚动升级，节点逐台 Cordon → Drain → 升级 → Uncordon | 升级中节点上 Pod 被驱逐，需确保业务多副本 | `ModifyClusterNodePool` 修改版本参数 |
| containerd 运行时 | 同上，与 kubelet 升级可合并 | 同 kubelet | 同上 |
| 节点管家（Node Operator） | 后台自动升级，无感知 | 无 | 无需操作 |
| 操作系统内核 | 需重启节点 | 同 kubelet | 镜像更新后通过节点池滚动升级 |

**核心原则**：升级前确认业务多副本，选择业务低峰窗口执行，先升级一个节点验证，确认无问题后逐批推进。

## 前置条件

- [环境准备](../../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:DescribeClusterInstances, tke:DescribeClusterNodePoolDetail
#    tke:DescribeClusterNodePools, tke:ModifyClusterNodePool
# 验证：执行 DescribeClusterNodePools 确认权限
tccli tke DescribeClusterNodePools --region <Region> --ClusterId CLUSTER_ID
# expected: exit 0，返回节点池列表
```

```json
{
  "NodePoolSet": "<NodePoolSet>",
  "NodePoolId": "<NodePoolId>",
  "Name": "<Name>",
  "ClusterInstanceId": "<ClusterInstanceId>",
  "LifeState": "<LifeState>",
  "LaunchConfigurationId": "<LaunchConfigurationId>"
}
```

### 资源检查

```bash
# 4. 查看当前节点版本信息
tccli tke DescribeClusterInstances --region <Region> \
    --ClusterId CLUSTER_ID \
    --Filters '[{"Name":"instance-role","Values":["WORKER"]}]'
# expected: 返回节点列表含 InstanceId 和当前运行时/kubelet 版本

# 5. 确认 K8s 集群版本
tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterVersion 为当前版本，ClusterStatus 为 Running

# 6. 确认目标节点池为原生节点池
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId CLUSTER_ID \
    --NodePoolId NODE_POOL_ID \
    | jq '.NodePool.Type'
# expected: "Native"
```

```json
{
  "TotalCount": "<TotalCount>",
  "InstanceSet": "<InstanceSet>",
  "InstanceId": "<InstanceId>",
  "InstanceRole": "<InstanceRole>",
  "FailedReason": "<FailedReason>",
  "InstanceState": "<InstanceState>"
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看节点版本 | `DescribeClusterInstances` | 是 |
| 查看节点池配置 | `DescribeClusterNodePoolDetail` | 是 |
| 发起节点池升级（修改版本） | `ModifyClusterNodePool` | 否 |
| 查看集群 K8s 版本 | `DescribeClusters` | 是 |

## 操作步骤

### 步骤 1：查看当前节点组件的版本信息

#### 选择依据

升级前必须了解当前版本，才能确定目标版本。升级决策点：

- **kubelet 版本**：必须与集群控制面版本一致或次版本落后（如集群 1.30 控制面，kubelet 可选 1.30 或 1.29）。**不建议 kubelet 版本高于控制面**。
- **containerd 版本**：跟随 kubelet 版本一起升级，不单独升级。
- **操作系统内核**：通过更新节点镜像间接升级，需修改节点池 `ImageId` 参数。

```bash
tccli tke DescribeClusterInstances --region <Region> \
    --ClusterId CLUSTER_ID
# expected: 返回 InstanceSet，含各节点实例信息
```

**预期输出**（关键字段）：

```json
{
    "TotalCount": 3,
    "InstanceSet": [
        {
            "InstanceId": "ins-example-001",
            "InstanceRole": "WORKER",
            "InstanceState": "running",
            "InstanceAdvancedSettings": {
                "ExtraArgs": {
                    "Kubelet": ["--kubelet-arg=value"]
                }
            }
        },
        {
            "InstanceId": "ins-example-002",
            "InstanceRole": "WORKER",
            "InstanceState": "running",
            "InstanceAdvancedSettings": {
                "ExtraArgs": {
                    "Kubelet": ["--kubelet-arg=value"]
                }
            }
        }
    ],
    "RequestId": "..."
}
```

### 步骤 2：确认节点池滚动升级配置

```bash
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId CLUSTER_ID \
    --NodePoolId NODE_POOL_ID \
    | jq '{Name: .NodePool.Name, LifeState: .NodePool.LifeState, NodeCount: .NodePool.NodeCountSummary}'
# expected: LifeState 为 normal，NodeCountSummary 含节点数量统计
```

```json
{
  "NodePool": "<NodePool>",
  "NodePoolId": "<NodePoolId>",
  "Name": "<Name>",
  "ClusterInstanceId": "<ClusterInstanceId>",
  "LifeState": "<LifeState>",
  "LaunchConfigurationId": "<LaunchConfigurationId>"
}
```

### 步骤 3：升级前准备（业务影响评估）

升级前确认业务可容忍节点滚动：

```bash
# 查看业务 Pod 的分布情况（需 kubectl）
kubectl get pods -o wide --all-namespaces \
    --field-selector spec.nodeName=NODE_NAME
# expected: 列出该节点上所有 Pod，确认均为多副本或有容忍的 workload

# 查看 PodDisruptionBudget（确保滚动升级不违反 PDB）
kubectl get pdb --all-namespaces
# expected: 列出所有 PDB 策略，确认升级操作不会导致可用副本数低于 PDB 阈值
```

```text
NAME  STATUS  AGE
...
```

### 步骤 4：触发升级（通过修改节点池镜像 ID 和版本）

```bash
cat > upgrade-nodepool.json <<'EOF'
{
  "ClusterId": "CLUSTER_ID",
  "NodePoolId": "NODE_POOL_ID",
  "Name": "NODE_POOL_NAME",
  "Native": {
    "ImageId": "TARGET_IMAGE_ID",
    "InstanceTypes": ["CXM_INSTANCE_TYPE"],
    "SubnetIds": ["SUBNET_ID"]
  }
}
EOF

tccli tke ModifyClusterNodePool --region <Region> \
    --cli-input-json file://upgrade-nodepool.json
# expected: exit 0

# ⚠️ 此操作会触发节点池滚动升级，存量节点会逐台重建
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `CLUSTER_ID` | 目标集群 ID | 托管集群，状态 Running | `tccli tke DescribeClusters --region <Region>` |
| `NODE_POOL_ID` | 节点池 ID | 已存在的 Native 类型节点池 | `tccli tke DescribeClusterNodePools --region <Region> --ClusterId CLUSTER_ID` |
| `NODE_POOL_NAME` | 节点池名称 | 长度 1-60 | 自定义 |
| `TARGET_IMAGE_ID` | 目标操作系统镜像 ID | 原生节点兼容的镜像 | 控制台查看可用镜像或 `cvm DescribeImages` |
| `CXM_INSTANCE_TYPE` | CXM 机型 | 当前节点池使用的机型 | `DescribeClusterNodePoolDetail` → `Native.InstanceTypes` |
| `SUBNET_ID` | 子网 ID | 节点池当前子网 | `DescribeClusterNodePoolDetail` → `Native.SubnetIds` |

### 步骤 5：监控升级进度

```bash
# 持续轮询节点池状态直到完成
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId CLUSTER_ID \
    --NodePoolId NODE_POOL_ID \
    | jq '{LifeState: .NodePool.LifeState, Summary: .NodePool.NodeCountSummary.AutoscalingAdded}'
# expected: LifeState 从 updating 变为 normal；Running 数量逐步恢复，Starting 数量减少至 0
```

**预期输出**（升级中）：

```json
{
  "LifeState": "updating",
  "Summary": {
    "Total": 3,
    "Running": 1,
    "Failed": 0,
    "Starting": 2
  }
}
```

**预期输出**（升级完成）：

```json
{
  "LifeState": "normal",
  "Summary": {
    "Total": 3,
    "Running": 3,
    "Failed": 0,
    "Starting": 0
  }
}
```

## 验证

### 验证升级完成

```bash
# 确认节点池状态恢复为 normal
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId CLUSTER_ID \
    --NodePoolId NODE_POOL_ID \
    | jq '.NodePool.LifeState'
# expected: "normal"
```

```json
{
  "NodePool": "<NodePool>",
  "NodePoolId": "<NodePoolId>",
  "Name": "<Name>",
  "ClusterInstanceId": "<ClusterInstanceId>",
  "LifeState": "<LifeState>",
  "LaunchConfigurationId": "<LaunchConfigurationId>"
}
```

### 验证节点版本

```bash
# 检查节点上的 kubelet 版本（通过 kubectl）
kubectl get nodes -o wide
# expected: 各节点版本号与目标版本一致

# 或 SSH 登录节点验证
ssh root@NODE_IP kubelet --version
# expected: Kubernetes v1.30.x

ssh root@NODE_IP containerd --version
# expected: containerd x.x.x（目标版本）
```

```text
NAME  STATUS  AGE
...
```

### 验证业务恢复

```bash
# 确认所有 Pod 已恢复 Running
kubectl get pods --all-namespaces --field-selector status.phase!=Running
# expected: 输出为空（无异常 Pod）
```

```text
NAME  STATUS  AGE
...
```

| 验证维度 | 命令 | 预期 |
|---------|------|------|
| 节点池状态 | `DescribeClusterNodePoolDetail` → `LifeState` | `"normal"` |
| 节点数恢复 | 同上 → `NodeCountSummary.AutoscalingAdded.Running` | = Total |
| kubelet 版本 | `kubectl get nodes -o wide` | 所有节点版本一致 |
| Pod 状态 | `kubectl get pods --all-namespaces` | 全部 Running |

## 清理

本页为概念说明页，无资源需清理。

> **注意**：升级操作本身会触发节点重建，旧版本节点会被自动替换。如升级过程中遇到问题，可通过撤销 `ModifyClusterNodePool` 参数回滚到旧镜像 ID 或配置。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `ModifyClusterNodePool` 返回 `InvalidParameter.ImageId` | `tccli cvm DescribeImages --region <Region> --ImageIds '["IMAGE_ID"]'` 检查镜像是否存在 | 填写的 `ImageId` 不存在或不属于当前地域 | 用正确镜像 ID 重新调用；`tccli cvm DescribeImages --region <Region>` 查询可用镜像 |
| `ModifyClusterNodePool` 返回 `InvalidParameter.SubnetId` | 检查 JSON 中 `SubnetIds` 数组 | 子网 ID 已删除或不属于集群 VPC | `tccli vpc DescribeSubnets --region <Region>` 查询可用子网 |
| `ModifyClusterNodePool` 返回 `FailedOperation.NodePoolUpdating` | `tccli tke DescribeClusterNodePoolDetail --region <Region> --ClusterId CLUSTER_ID --NodePoolId NODE_POOL_ID` 查看当前 LifeState | 节点池正在执行其他变更操作（如扩容、缩容） | 等待当前操作完成（LifeState 变为 normal）后重试 |

### 升级卡住或异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 节点池 `LifeState=updating` 超过 30 分钟未完成 | `tccli tke DescribeClusterNodePoolDetail --region <Region> --ClusterId CLUSTER_ID --NodePoolId NODE_POOL_ID` 查看 NodeCountSummary 中各状态数量 | 新节点创建失败、镜像兼容性问题、子网 IP 不足 | 检查 `Failed` 节点数量 → 查看节点实例 `FailedReason` → 如果是 IP 不足，释放子网 IP 或换子网；如果是镜像问题，换用已验证的镜像 ID |
| 升级后新节点 NotReady | `kubectl describe node NODE_NAME` 查看 Conditions + `kubectl get events` | kubelet/containerd 版本与新镜像不兼容，或内核参数不匹配 | 回滚 ImageId 到旧版本；如为 KubeletArgs 问题，同步更新 Management 参数 |
| 业务 Pod 无法调度到新节点 | `kubectl describe pod POD_NAME` 查看 Events | 新节点缺少 Pod 所需的 nodeSelector 标签或污点/容忍不匹配 | 检查新节点的 labels 和 taints 与旧节点是否一致；不一致需用 `ModifyClusterNodePool` 补齐配置 |

## 下一步

- [Management 参数介绍](https://cloud.tencent.com/document/product/457/79698) — 升级后可调整 KubeletArgs 等运维参数
- [镜像和内核参数说明](https://cloud.tencent.com/document/product/457/XXXXX) — 原生节点支持的镜像和内核版本
- [原生节点生命周期](https://cloud.tencent.com/document/product/457/78200) — 节点从创建到删除的完整生命周期
- [故障自愈规则](https://cloud.tencent.com/document/product/457/XXXXX) — 升级异常时的自动恢复策略
- [升级集群](https://cloud.tencent.com/document/product/457/XXXXX) — K8s 控制面版本升级指南

## 控制台替代

[TKE 控制台 → 集群 → 节点管理 → 节点池详情](https://console.cloud.tencent.com/tke2/cluster)：在节点池详情页的「更多操作」中选择「升级」，按指引选择目标镜像版本并执行滚动升级。
