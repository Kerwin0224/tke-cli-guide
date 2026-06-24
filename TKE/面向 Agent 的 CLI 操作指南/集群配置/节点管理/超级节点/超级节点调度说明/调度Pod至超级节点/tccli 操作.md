# 调度 Pod 至超级节点

> 对照官方：[调度 Pod 至超级节点](https://cloud.tencent.com/document/product/457/74016) · page_id `74016`

## 概述

将 Pod 调度到超级节点上有两种方式：**自动调度**（集群资源不足时按量超级节点自动承接）和**手动调度**（通过 `nodeSelector`、`tolerations` 或 TKE 调度 Annotation 显式指定）。

| 调度方式 | 机制 | 适用场景 |
|---------|------|---------|
| 自动调度 | 按量超级节点池在普通节点资源不足时自动承接 Pod | 弹性场景，无需修改工作负载配置 |
| 手动调度 — nodeSelector | `spec.nodeSelector: {node.kubernetes.io/instance-type: eklet}` | 明确指定 Pod 必须调度到超级节点 |
| 手动调度 — tolerations | 配合超级节点污点，允许 Pod 容忍并调度 | 超级节点配置了 taint 时的必要配置 |
| 手动调度 — affinity | `nodeAffinity` 或 `podAntiAffinity` | 复杂调度策略（跨可用区、互斥等） |

**控制面验证**：通过 `DescribeClusterVirtualNodePools` 确认超级节点池状态；数据面通过 `kubectl` 验证 Pod 调度结果。

## 前置条件

- [环境准备](../../../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查 kubectl 已安装且可连接集群
kubectl version --client
# expected: 显示 Client Version

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:DescribeClusterVirtualNodePools, tke:DescribeClusterKubeconfig
# 验证：
tccli tke DescribeClusterVirtualNodePools --region <Region> --ClusterId CLUSTER_ID
# expected: exit 0
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
# 4. 确认超级节点池已创建且状态正常
tccli tke DescribeClusterVirtualNodePools --region <Region> \
    --ClusterId CLUSTER_ID
# expected: NodePoolSet 非空，目标节点池 LifeState 为 normal

# 5. 获取 kubeconfig 并验证连通性
tccli tke DescribeClusterKubeconfig --region <Region> --ClusterId CLUSTER_ID \
    | jq -r '.Kubeconfig' > ~/.kube/config-super
kubectl --kubeconfig ~/.kube/config-super cluster-info
# expected: Kubernetes control plane 可达

# 6. 查看超级节点标签（确认调度条件）
kubectl --kubeconfig ~/.kube/config-super get nodes -l node.kubernetes.io/instance-type=eklet --show-labels
# expected: 返回超级节点列表，确认可用标签名称
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

## 控制台与 CLI 参数映射

| 控制台操作 | tccli / kubectl | 幂等 |
|-----------|-----------|:--:|
| 查看超级节点池 | `DescribeClusterVirtualNodePools` | 是 |
| 获取 kubeconfig | `DescribeClusterKubeconfig` | 是 |
| 查看超级节点 | `kubectl get nodes -l node.kubernetes.io/instance-type=eklet` | 是 |
| 创建带节点选择的 Pod | `kubectl apply -f` | 是 |
| 查看 Pod 调度结果 | `kubectl get pod -o wide` | 是 |

## 操作步骤

### 步骤 1：确认超级节点就绪和标签

```bash
tccli tke DescribeClusterVirtualNodePools --region <Region> \
    --ClusterId CLUSTER_ID
# expected: exit 0，NodePoolSet 中目标节点池 LifeState 为 normal
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

```bash
kubectl --kubeconfig ~/.kube/config-super get nodes -l node.kubernetes.io/instance-type=eklet
# expected: 返回至少 1 个超级节点

kubectl --kubeconfig ~/.kube/config-super get nodes NODE_NAME --show-labels
# expected: 显示节点完整标签列表
```

### 步骤 2：通过 nodeSelector 调度 Pod 到超级节点

#### 选择依据

- **nodeSelector**：最简单的手动调度方式，适合"指定 Pod 只跑在超级节点"的场景。使用 `node.kubernetes.io/instance-type: eklet` 标签。
- **tolerations**：仅当超级节点配置了特殊 taint 时才需要（新建的超级节点池通常无额外 taint）。
- **自动调度**：如果按量超级节点池已创建且 Pod 未指定 nodeSelector，普通节点资源不足时自动调度到超级节点。无需修改 YAML。

#### 最小示例（nodeSelector）

`pod-schedule-minimal.yaml`：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: POD_NAME
  annotations:
    eks.tke.cloud.tencent.com/cpu: "1"
    eks.tke.cloud.tencent.com/mem: "2Gi"
spec:
  nodeSelector:
    node.kubernetes.io/instance-type: eklet
  containers:
  - name: CONTAINER_NAME
    image: nginx:1.25
    resources:
      requests:
        cpu: "1"
        memory: "2Gi"
```

```bash
kubectl --kubeconfig ~/.kube/config-super apply -f pod-schedule-minimal.yaml
# expected: pod/POD_NAME created
```

#### 增强示例（nodeSelector + affinity + tolerations）

`pod-schedule-enhanced.yaml`：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: POD_NAME
  annotations:
    eks.tke.cloud.tencent.com/cpu: "2"
    eks.tke.cloud.tencent.com/mem: "4Gi"
spec:
  nodeSelector:
    node.kubernetes.io/instance-type: eklet
  tolerations:
  - key: "eks.tke.cloud.tencent.com/eklet"
    operator: "Exists"
    effect: "NoSchedule"
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: app
              operator: In
              values:
              - APP_LABEL
          topologyKey: kubernetes.io/hostname
  containers:
  - name: CONTAINER_NAME
    image: nginx:1.25
    resources:
      requests:
        cpu: "2"
        memory: "4Gi"
      limits:
        cpu: "2"
        memory: "4Gi"
```

```bash
kubectl --kubeconfig ~/.kube/config-super apply -f pod-schedule-enhanced.yaml
# expected: pod/POD_NAME created
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `POD_NAME` | Pod 名称 | 须遵循 K8s 命名规范 | 自定义 |
| `CONTAINER_NAME` | 容器名称 | 须遵循 K8s 命名规范 | 自定义 |
| `APP_LABEL` | 应用标签 | 用于 podAntiAffinity | 自定义 |

## 验证

### 控制面（tccli）

```bash
tccli tke DescribeClusterVirtualNodePools --region <Region> \
    --ClusterId CLUSTER_ID
# expected: NodePoolSet 非空，LifeState 为 normal
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

### 数据面（kubectl）

```bash
# 验证 Pod 调度到超级节点
kubectl --kubeconfig ~/.kube/config-super get pod POD_NAME -o wide
# expected: NODE 列为虚拟节点名（以 eklet 开头），STATUS 为 Running

# 验证 Pod 详情 — nodeSelector 生效
kubectl --kubeconfig ~/.kube/config-super describe pod POD_NAME | grep "Node-Selectors"
# expected: Node-Selectors: node.kubernetes.io/instance-type=eklet

# 验证 Pod 未调度到普通节点
kubectl --kubeconfig ~/.kube/config-super get pod POD_NAME -o json \
    | jq '.spec.nodeName'
# expected: 返回超级节点名称（以 eklet 开头）
```

| 验证维度 | 命令 | 预期 |
|---------|------|------|
| 节点池就绪 | `DescribeClusterVirtualNodePools --ClusterId CLUSTER_ID` | `LifeState: "normal"` |
| Pod 状态 | `kubectl get pod POD_NAME` | `STATUS: "Running"` |
| Pod 节点 | `kubectl get pod POD_NAME -o wide` | NODE 以 `eklet` 开头 |
| nodeSelector | `kubectl describe pod POD_NAME \| grep "Node-Selectors"` | 含 `node.kubernetes.io/instance-type=eklet` |
| 调度策略 | `kubectl get pod POD_NAME -o json \| jq '.spec.nodeSelector'` | 声明的 nodeSelector 均匹配 |

## 清理

> **警告**：删除 Pod 将清除该 Pod 的运行时数据。如 Pod 为 Deployment 管理，删除 Pod 后 Deployment 可能自动重建，需先删除 Deployment 或缩容至 0。

```bash
# 删除测试 Pod
kubectl --kubeconfig ~/.kube/config-super delete pod POD_NAME
# expected: pod "POD_NAME" deleted

# 验证 Pod 已删除
kubectl --kubeconfig ~/.kube/config-super get pod POD_NAME
# expected: Error from server (NotFound): pods "POD_NAME" not found
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl apply` 返回 `Error from server (Forbidden)` | `kubectl --kubeconfig ~/.kube/config-super auth can-i create pods` | 子账号缺少 Pod 创建权限 | 联系集群管理员授予 namespace 级别的 Pod 创建权限 |
| `kubectl get nodes -l node.kubernetes.io/instance-type=eklet` 无节点 | `tccli tke DescribeClusterVirtualNodePools --region <Region> --ClusterId CLUSTER_ID` | 超级节点池未创建或节点未注册 | 先执行 [新建超级节点](../../使用超级节点/新建超级节点/tccli%20操作.md) |
| `DescribeClusterKubeconfig` 返回 `InvalidParameter.ClusterId` | `tccli tke DescribeClusters --region <Region>` 列出集群 | 集群 ID 错误或不存在 | 用 `tccli tke DescribeClusters --region <Region>` 获取正确 ClusterId |

### 创建成功但 Pod 未调度到超级节点

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Pod 调度到了普通节点 | `kubectl get pod POD_NAME -o wide` 查看 NODE | 未设置 `nodeSelector` 或 `nodeSelector` 键名不匹配 | 检查 YAML 中 `nodeSelector` 的键名是否为 `node.kubernetes.io/instance-type`，值为 `eklet`。验证标签名称：`kubectl get nodes --show-labels \| grep instance-type` |
| Pod Pending，Events 显示 `0/1 nodes are available: 1 node(s) didn't match node selector` | `kubectl describe pod POD_NAME \| tail -20` | nodeSelector 标签不存在于任何节点上 | 检查超级节点实际标签：`kubectl get nodes -l node.kubernetes.io/instance-type=eklet` 确认标签名是否正确 |
| Pod Pending，Events 显示 `node(s) had untolerated taint` | `kubectl describe node NODE_NAME \| grep Taints` | 超级节点带有额外 taint，Pod 未声明对应 tolerations | 在 Pod spec 中增加对超级节点 taint 的 tolerations 配置 |
| 自动调度未触发（普通节点资源充足时 Pod 未调度到按量超级节点） | `kubectl describe pod POD_NAME \| tail -20` | 此为目标行为：自动调度仅在普通节点资源不足时触发 | 如需强制调度到超级节点，添加 `nodeSelector: {node.kubernetes.io/instance-type: eklet}` |
| 按量超级节点上 Pod 被优先缩容 | `kubectl get pod -o wide` 查看 Pod 分布 | 此为设计行为：资源充足时按量超级节点上的 Pod 优先被缩容 | 如不希望 Pod 被优先缩容，考虑使用包年包月超级节点或添加 Pod Disruption Budget |

## 下一步

- [超级节点 Annotation 说明](https://cloud.tencent.com/document/product/457/44173) — 完整 Annotation 列表
- [超级节点可调度 Pod 说明](https://cloud.tencent.com/document/product/457/74015) — 可调度规格范围
- [指定资源规格](https://cloud.tencent.com/document/product/457/44174) — CPU/内存/GPU 规格配置
- [超级节点常见问题](https://cloud.tencent.com/document/product/457/60411) — 常见 FAQ
- [超级节点上支持运行 DaemonSet](https://cloud.tencent.com/document/product/457/98730) — DaemonSet 调度配置

## 控制台替代

[TKE 控制台 → 工作负载 → 新建/编辑 → 调度策略](https://console.cloud.tencent.com/tke2/cluster)：在调度策略中选择「调度到超级节点」。
