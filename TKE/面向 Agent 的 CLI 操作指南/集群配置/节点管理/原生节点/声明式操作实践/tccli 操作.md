# 声明式操作实践（tccli + kubectl）

> 对照官方：[声明式操作实践](https://cloud.tencent.com/document/product/457/78649) · page_id `78649`

## 概述

原生节点池支持通过**声明式方式**管理节点配置：控制面通过 `ModifyClusterNodePool` 的 `Annotations` 参数声明节点池级期望配置；数据面通过 `kubectl annotate node` 或 `kubectl label node` 在节点上声明式管理标签和注解。

声明式操作的核心思想是"期望状态描述"——用户声明期望的节点属性，系统自动将实际状态收敛到期望状态，而非执行命令式步骤。这种方式适合 GitOps 工作流和 Infrastructure as Code 实践。

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
#    tke:ModifyClusterNodePool, tke:DescribeClusterNodePoolDetail
#    tke:DescribeClusterNodePools, tke:DescribeClusterKubeconfig
tccli tke DescribeClusterNodePools --region <Region> --ClusterId CLUSTER_ID
# expected: exit 0，返回节点池列表

# 4. 检查 kubectl 可用
kubectl version --client
# expected: Client Version 显示版本号
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
# 5. 确认节点池存在
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId CLUSTER_ID \
    --NodePoolId NODE_POOL_ID
# expected: Type 为 "Native"

# 6. 获取 kubeconfig
tccli tke DescribeClusterKubeconfig --region <Region> --ClusterId CLUSTER_ID
# expected: 返回 Kubeconfig 内容

# 7. 确认集群可达
kubectl --kubeconfig KUBECONFIG_PATH cluster-info
# expected: Kubernetes control plane is running
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

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 查看节点池 Annotations | `tccli tke DescribeClusterNodePoolDetail` | 是 |
| 声明式修改节点池 | `tccli tke ModifyClusterNodePool --Annotations` | 否 |
| 获取 kubeconfig | `tccli tke DescribeClusterKubeconfig` | 是 |
| 查看节点标签/注解 | `kubectl get node -o wide` / `kubectl describe node` | 是 |
| 声明式管理节点标签 | `kubectl label node` | 是（同名标签覆盖） |
| 声明式管理节点注解 | `kubectl annotate node` | 是（同名注解覆盖） |
| 查看节点池内节点 | `kubectl get nodes` / `tccli tke DescribeClusterInstances` | 是 |

## 操作步骤

### 步骤1：控制面 — 获取当前节点池配置

```bash
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId CLUSTER_ID \
    --NodePoolId NODE_POOL_ID
# expected: 返回包含 Annotations, Labels, Taints 的完整节点池配置
```

**预期输出**：

```json
{
    "Response": {
        "NodePool": {
            "NodePoolId": "np-example",
            "Name": "native-pool-example",
            "ClusterId": "cls-example",
            "LifeState": "normal",
            "Type": "Native",
            "Annotations": [],
            "Labels": [],
            "Taints": [],
            "NodeCountSummary": {
                "ManuallyAdded": {"Total": 2}
            }
        },
        "RequestId": "xxxx-xxxx-xxxx-xxxx"
    }
}
```

### 步骤2：控制面 — 声明式修改节点池 Annotations

通过 Annotations 声明节点池的期望行为。常见声明式操作场景：

#### 场景 A：开启原地升降配

`declare-inplace-update.json`：

```json
{
  "ClusterId": "CLUSTER_ID",
  "NodePoolId": "NODE_POOL_ID",
  "Annotations": [
    {
      "Name": "tke.cloud.tencent.com/enable-inplace-update",
      "Value": "true"
    }
  ]
}
```

```bash
tccli tke ModifyClusterNodePool --region <Region> \
    --cli-input-json file://declare-inplace-update.json
# expected: exit 0，返回 RequestId
```

**预期输出**：

```json
{
    "Response": {
        "RequestId": "xxxx-xxxx-xxxx-xxxx"
    }
}
```

#### 场景 B：声明节点调优参数

`declare-tuning.json`：

```json
{
  "ClusterId": "CLUSTER_ID",
  "NodePoolId": "NODE_POOL_ID",
  "Annotations": [
    {
      "Name": "tke.cloud.tencent.com/kube-reserved",
      "Value": "cpu=200m,memory=500Mi"
    },
    {
      "Name": "tke.cloud.tencent.com/system-reserved",
      "Value": "cpu=200m,memory=500Mi"
    },
    {
      "Name": "tke.cloud.tencent.com/eviction-hard",
      "Value": "memory.available<500Mi,nodefs.available<10%"
    }
  ]
}
```

```bash
tccli tke ModifyClusterNodePool --region <Region> \
    --cli-input-json file://declare-tuning.json
# expected: exit 0，返回 RequestId
```

**预期输出**：

```json
{
    "Response": {
        "RequestId": "xxxx-xxxx-xxxx-xxxx"
    }
}
```

### 步骤3：控制面 — 验证节点池注解已更新

```bash
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId CLUSTER_ID \
    --NodePoolId NODE_POOL_ID
# expected: Annotations 列表包含新增的声明
```

**预期输出**：

```json
{
    "Response": {
        "NodePool": {
            "NodePoolId": "np-example",
            "Annotations": [
                {
                    "Name": "tke.cloud.tencent.com/enable-inplace-update",
                    "Value": "true"
                },
                {
                    "Name": "tke.cloud.tencent.com/kube-reserved",
                    "Value": "cpu=200m,memory=500Mi"
                },
                {
                    "Name": "tke.cloud.tencent.com/system-reserved",
                    "Value": "cpu=200m,memory=500Mi"
                },
                {
                    "Name": "tke.cloud.tencent.com/eviction-hard",
                    "Value": "memory.available<500Mi,nodefs.available<10%"
                }
            ]
        },
        "RequestId": "xxxx-xxxx-xxxx-xxxx"
    }
}
```

### 步骤4：数据面 — 获取节点信息

```bash
# 列出集群原生节点
kubectl --kubeconfig KUBECONFIG_PATH get nodes -o wide
# expected: 返回节点列表，包含名称、状态、版本、IP 等

# 查看节点详细注解和标签
kubectl --kubeconfig KUBECONFIG_PATH describe node NODE_NAME
# expected: 返回 Labels 和 Annotations 区块
```

### 步骤5：数据面 — 声明式管理节点标签

#### 添加标签

```bash
# 给节点添加标签
kubectl --kubeconfig KUBECONFIG_PATH label node NODE_NAME \
    env=production \
    team=backend \
    workload-type=stateful
# expected: node/NODE_NAME labeled

# 确认标签已添加
kubectl --kubeconfig KUBECONFIG_PATH get node NODE_NAME --show-labels
# expected: LABELS 列包含 env=production,team=backend,workload-type=stateful
```

#### 修改标签

```bash
# 覆盖已有标签值
kubectl --kubeconfig KUBECONFIG_PATH label node NODE_NAME \
    env=staging --overwrite
# expected: node/NODE_NAME labeled

# 删除标签
kubectl --kubeconfig KUBECONFIG_PATH label node NODE_NAME env-
# expected: node/NODE_NAME unlabeled
```

### 步骤6：数据面 — 声明式管理节点注解

#### 添加注解

```bash
# 给节点添加注解
kubectl --kubeconfig KUBECONFIG_PATH annotate node NODE_NAME \
    tke.cloud.tencent.com/maintenance-window="02:00-04:00" \
    tke.cloud.tencent.com/owner="platform-team"
# expected: node/NODE_NAME annotated

# 确认注解已添加
kubectl --kubeconfig KUBECONFIG_PATH get node NODE_NAME -o jsonpath='{.metadata.annotations}'
# expected: 输出中包含 maintenance-window 和 owner
```

#### 修改和删除注解

```bash
# 覆盖已有注解值
kubectl --kubeconfig KUBECONFIG_PATH annotate node NODE_NAME \
    tke.cloud.tencent.com/maintenance-window="03:00-05:00" \
    --overwrite
# expected: node/NODE_NAME annotated

# 删除注解（后缀减号）
kubectl --kubeconfig KUBECONFIG_PATH annotate node NODE_NAME \
    tke.cloud.tencent.com/owner-
# expected: node/NODE_NAME annotated
```

### 步骤7：数据面 — 声明式批量操作（使用 kubectl patch）

`node-patch-labels.json`：

```json
{
  "metadata": {
    "labels": {
      "env": "production",
      "tier": "backend",
      "region": "gz"
    }
  }
}
```

```bash
kubectl --kubeconfig KUBECONFIG_PATH patch node NODE_NAME \
    --type merge \
    --patch-file node-patch-labels.json
# expected: node/NODE_NAME patched
```

### 步骤8：常见声明式操作场景汇总

| 场景 | 声明方式 | 命令 |
|------|---------|------|
| 开启原地升降配 | 节点池 Annotation | `tccli tke ModifyClusterNodePool --Annotations [{"Name":"tke.cloud.tencent.com/enable-inplace-update","Value":"true"}]` |
| 设置资源预留 | 节点池 Annotation | `tccli tke ModifyClusterNodePool --Annotations [{"Name":"tke.cloud.tencent.com/kube-reserved","Value":"cpu=200m,memory=500Mi"}]` |
| 驱逐阈值 | 节点池 Annotation | `tccli tke ModifyClusterNodePool --Annotations [{"Name":"tke.cloud.tencent.com/eviction-hard","Value":"memory.available<500Mi"}]` |
| 节点分组 | 节点 Label | `kubectl label node NODE_NAME env=production` |
| 维护窗口 | 节点 Annotation | `kubectl annotate node NODE_NAME maintenance-window="02:00-04:00"` |
| 污点管理 | 节点 Taint | `kubectl taint node NODE_NAME dedicated=gpu:NoSchedule` |
| 批量标签更新 | kubectl patch | `kubectl patch node NODE_NAME --type merge --patch '{"metadata":{"labels":{"key":"value"}}}'` |

## 验证

### 控制面（tccli）

| 维度 | 检查内容 | 预期 |
|------|---------|------|
| 节点池注解 | `DescribeClusterNodePoolDetail` → `Annotations` | 包含声明的 Key-Value 对 |
| 修改状态 | `ModifyClusterNodePool` 返回 `RequestId` | 请求提交成功 |

### 数据面（kubectl）

| 维度 | 检查内容 | 预期 |
|------|---------|------|
| 节点标签 | `kubectl get node NODE_NAME --show-labels` | 包含声明的 labels |
| 节点注解 | `kubectl describe node NODE_NAME` → `Annotations` | 包含声明的 annotation Key-Value 对 |
| 节点污点 | `kubectl describe node NODE_NAME` → `Taints` | 包含声明的 taint |
| 声明一致性 | 对比节点池 Annotations 和节点 Annotations | 新增节点应继承节点池 Annotations |

```bash
# 查询带特定标签的节点
kubectl --kubeconfig KUBECONFIG_PATH get nodes -l env=production
# expected: 只返回包含 env=production 标签的节点

# 导出节点完整 YAML 用于审计
kubectl --kubeconfig KUBECONFIG_PATH get node NODE_NAME -o yaml > node-audit.yaml
```

## 清理

### 控制面（tccli）

```bash
# 移除节点池声明
tccli tke ModifyClusterNodePool --region <Region> \
    --ClusterId CLUSTER_ID \
    --NodePoolId NODE_POOL_ID \
    --Annotations '[]'
# expected: exit 0
```

### 数据面（kubectl）

```bash
# 移除节点标签
kubectl --kubeconfig KUBECONFIG_PATH label node NODE_NAME env- team- workload-type-

# 移除节点注解
kubectl --kubeconfig KUBECONFIG_PATH annotate node NODE_NAME \
    tke.cloud.tencent.com/maintenance-window- \
    tke.cloud.tencent.com/owner-

# 移除节点污点
kubectl --kubeconfig KUBECONFIG_PATH taint node NODE_NAME dedicated=gpu:NoSchedule-
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `ModifyClusterNodePool` 返回 `InvalidParameterValue.Annotation` | 检查 Annotation Name 是否以 `tke.cloud.tencent.com/` 开头 | Annotation Name 不符合 TKE 规范 | 使用 `tke.cloud.tencent.com/` 前缀 |
| `kubectl annotate` 返回 `the object has been modified` | 并发修改冲突 | 节点被其他操作同时修改 | 使用 `--overwrite` 强制覆盖或重试 |
| `kubectl label` 返回 `forbidden` | `kubectl auth can-i update nodes --as USER` | RBAC 权限不足 | 绑定 node/update 的 ClusterRole |
| APIServer 连接超时 | `kubectl --kubeconfig KUBECONFIG_PATH cluster-info` | kubeconfig 内网地址在外部网络不可达 | 使用内网 kubeconfig 或配置跳板机 |

### 声明生效但结果不符合预期

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 节点池 Annotation 已更新但存量节点未继承 | `kubectl describe node NODE_NAME` 查看节点 Annotation | 节点池级 Annotation 仅对新节点生效 | 手动在存量节点上 `kubectl annotate node`，或扩容触发新节点创建 |
| 标签删除后 Pod 调度异常 | `kubectl get pods -A -o wide` 查看调度情况 | Pod 的 `nodeSelector` 引用了已删除的标签 | 更新 Pod/Deployment 的 `nodeSelector` 或恢复标签 |
| `kubectl taint` 导致 Pod 被驱逐 | `kubectl get pods -A -o wide` 查看 Pod 调度 | 添加的 taint 与现有 Pod 的 toleration 不匹配 | 为 Pod 添加对应的 toleration 后再 taint 节点 |
| Annotation 值包含特殊字符导致解析错误 | 检查 Annotation 值是否包含 `/`、`=`、换行等特殊字符 | Annotation Value 需为合法字符串 | 对特殊字符进行转义或使用 base64 编码 |

## 下一步

- [修改原生节点](../修改原生节点/tccli%20操作.md) — page_id `103599`
- [Pod 原地升降配](../Pod%20原地升降配/tccli%20操作.md) — page_id `79697`
- [原生节点扩缩容](../原生节点扩缩容/tccli%20操作.md) — page_id `78648`
- [故障自愈规则](../故障自愈规则/tccli%20操作.md) — page_id `78650`

## 控制台替代

[容器服务控制台 → 集群 → 节点管理 → 原生节点 → 详情](https://console.cloud.tencent.com/tke2/cluster)
