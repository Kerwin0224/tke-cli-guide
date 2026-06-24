# Pod 原地升降配（tccli + kubectl）

> 对照官方：[Pod 原地升降配](https://cloud.tencent.com/document/product/457/79697) · page_id `79697`

## 概述

在**原生节点**上支持 Pod **原地**（InPlace）调整 CPU/内存资源限制，无需驱逐和重建 Pod。该特性依赖原生节点池开启原地升降配能力（Annotations 声明）。控制面通过 `ModifyClusterNodePool` 配置节点池级 Annotations 开启能力；数据面通过 `kubectl annotate` 或 `kubectl edit` 修改 Pod 资源 limits，实现原地升降配。

原地升降配不改变 Pod 所在节点，Pod IP 保持不变，适用于有状态服务和需要快速调整资源规格的场景。

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
#    tke:DescribeClusterNodePools, tke:DescribeClusterNodePoolDetail
#    tke:ModifyClusterNodePool
tccli tke DescribeClusterNodePools --region <Region> --ClusterId CLUSTER_ID
# expected: exit 0，返回节点池列表

# 4. 检查 kubectl 可用
kubectl version --client
# expected: Client Version 显示版本号

# 5. 确认 kubeconfig 可用
tccli tke DescribeClusterKubeconfig --region <Region> --ClusterId CLUSTER_ID
# expected: 返回 Kubeconfig 内容
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
# 6. 确认原生节点池存在并支持原地升降配
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId CLUSTER_ID \
    --NodePoolId NODE_POOL_ID
# expected: Type 为 "Native"

# 7. 确认节点已就绪
kubectl --kubeconfig KUBECONFIG_PATH get nodes
# expected: 原生节点 STATUS 为 Ready
```

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 查看节点池 | `tccli tke DescribeClusterNodePoolDetail` | 是 |
| 开启原地升降配能力 | `tccli tke ModifyClusterNodePool --Annotations` | 是 |
| 修改 Pod 资源限制 | `kubectl annotate` / `kubectl edit` | 否（每次修改增量） |
| 确认 Pod 资源变更 | `kubectl describe pod` | 是 |

### 关键 Annotation

| Annotation Key | 值 | 位置 | 说明 |
|----------------|-----|------|------|
| `tke.cloud.tencent.com/enable-inplace-update` | `"true"` | 节点池 | 开启节点池原地升降配能力 |
| `tke.cloud.tencent.com/inplace-update` | `"true"` | Pod | 标记 Pod 允许原地升级 |

## 操作步骤

### 步骤1：控制面 — 开启节点池原地升降配

`enable-inplace-nodepool.json`：

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
    --cli-input-json file://enable-inplace-nodepool.json
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

> **注意**：Annotation 仅对新增节点生效。存量节点如需开启，需通过数据面直接在节点上打 Annotation。

### 步骤2：数据面 — 给存量原生节点打 Annotation

对存量原生节点手动添加 Annotation，使其支持原地升降配：

```bash
# 查看原生节点名称
kubectl --kubeconfig KUBECONFIG_PATH get nodes -l node.tke.cloud.tencent.com/type=native
# expected: 返回原生节点列表

# 给节点打 Annotation
kubectl --kubeconfig KUBECONFIG_PATH annotate node NODE_NAME \
    tke.cloud.tencent.com/enable-inplace-update=true
# expected: node/NODE_NAME annotated
```

### 步骤3：数据面 — 标记 Pod 允许原地升级

```bash
# 标记 Pod 允许原地升降配
kubectl --kubeconfig KUBECONFIG_PATH annotate pod POD_NAME \
    -n NAMESPACE \
    tke.cloud.tencent.com/inplace-update=true
# expected: pod/POD_NAME annotated
```

### 步骤4：数据面 — 原地调整 Pod 资源

通过 `kubectl edit` 修改 Pod 的 CPU/内存资源 limits：

```bash
kubectl --kubeconfig KUBECONFIG_PATH edit pod POD_NAME -n NAMESPACE
```

在编辑器中修改 `resources.limits`：

```yaml
spec:
  containers:
  - name: CONTAINER_NAME
    resources:
      limits:
        cpu: "2"      # 从 "1" 调整到 "2"
        memory: "2Gi"  # 从 "1Gi" 调整到 "2Gi"
      requests:
        cpu: "1"
        memory: "1Gi"
```

保存后 Pod 不会重建，原地生效。可通过以下方式确认：

```bash
kubectl --kubeconfig KUBECONFIG_PATH get pod POD_NAME -n NAMESPACE -o yaml
# expected: resources.limits 已更新，但 Pod UID 和 IP 不变
```

### 步骤5：数据面 — 通过 Annotation 声明式调整

也可通过 Annotation 声明目标资源值实现原地升降配：

```bash
kubectl --kubeconfig KUBECONFIG_PATH annotate pod POD_NAME -n NAMESPACE \
    tke.cloud.tencent.com/desired-cpu="2" \
    tke.cloud.tencent.com/desired-memory="2Gi" \
    --overwrite
# expected: pod/POD_NAME annotated

# 确认变更已生效
kubectl --kubeconfig KUBECONFIG_PATH describe pod POD_NAME -n NAMESPACE
# expected: Annotations 中包含 desired-cpu 和 desired-memory，Limits 已更新
```

## 验证

### 控制面（tccli）

| 维度 | 检查内容 | 预期 |
|------|---------|------|
| 节点池 Annotation | `DescribeClusterNodePoolDetail` → `Annotations` | 包含 `tke.cloud.tencent.com/enable-inplace-update: "true"` |

### 数据面（kubectl）

| 维度 | 检查内容 | 预期 |
|------|---------|------|
| 节点 Annotation | `kubectl get node NODE_NAME -o jsonpath='{.metadata.annotations}'` | 包含 `tke.cloud.tencent.com/enable-inplace-update: "true"` |
| Pod Annotation | `kubectl describe pod POD_NAME -n NAMESPACE` | 包含 `tke.cloud.tencent.com/inplace-update: "true"` |
| Pod 资源变更 | `kubectl get pod POD_NAME -n NAMESPACE -o yaml` | `resources.limits` 已更新为调整值 |
| Pod 未重启 | `kubectl get pod POD_NAME -n NAMESPACE` → `RESTARTS` | 不变（证明未重建） |
| 容器内生效 | `kubectl exec POD_NAME -n NAMESPACE -- cat /sys/fs/cgroup/cpu/cpu.cfs_quota_us` | 反映新 CPU 限制 |

```bash
# 确认 Pod 原地调整后未重建（UID 不变）
kubectl --kubeconfig KUBECONFIG_PATH get pod POD_NAME -n NAMESPACE -o jsonpath='{.metadata.uid}'
# 原地升降配前后对比此值
```

## 清理

### 控制面（tccli）

无需额外清理。取消原地升降配能力只需移除节点池对应 Annotation。

### 数据面（kubectl）

```bash
# 移除 Pod 的原地升级标记
kubectl --kubeconfig KUBECONFIG_PATH annotate pod POD_NAME -n NAMESPACE \
    tke.cloud.tencent.com/inplace-update-

# 移除节点的原地升级标记
kubectl --kubeconfig KUBECONFIG_PATH annotate node NODE_NAME \
    tke.cloud.tencent.com/enable-inplace-update-
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `ModifyClusterNodePool` 返回 `InvalidParameterValue.Annotation` | 检查 Annotation 的 `Name` 是否以 `tke.cloud.tencent.com/` 开头 | Annotation Key 格式不符合 TKE 命名规范 | 使用 `tke.cloud.tencent.com/` 前缀 |
| `kubectl annotate` 返回 `forbidden` | `kubectl auth can-i update pods --as USER --namespace NAMESPACE` | RBAC 权限不足 | 绑定 pod/update 权限的 ClusterRole |

### 配置成功但原地升级不生效

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 修改 Pod 资源后 Pod 重建（UID 变化） | `kubectl get pod POD_NAME -n NAMESPACE -w` 观察 Pod 变化 | 节点池或 Pod 未开启原地升级 Annotation | 检查节点池和 Pod 的 Annotation 均已添加 |
| 资源变更后 Pod 内无效 | `kubectl exec POD_NAME -n NAMESPACE -- cat /proc/1/cgroup` 确认 cgroup | 容器运行时未就地生效 cgroup 变更 | 确保使用 containerd 运行时，且节点为原生节点 |
| 节点 Annotation 不存在 | `kubectl get node NODE_NAME -o jsonpath='{.metadata.annotations}'` | 节点非原生节点或 Annotation 未添加 | 确认节点为原生节点，手动添加 Annotation |

## 下一步

- [声明式操作实践](../声明式操作实践/tccli%20操作.md) — page_id `78649`
- [修改原生节点](../修改原生节点/tccli%20操作.md) — page_id `103599`
- [原生节点功能支持说明](../原生节点功能支持说明/tccli%20操作.md)

## 控制台替代

[容器服务控制台 → 集群 → 节点管理 → 原生节点 → Pod 原地升降配](https://console.cloud.tencent.com/tke2/cluster)
