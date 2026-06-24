# 超级节点上支持运行 DaemonSet

> 对照官方：[超级节点上支持运行 DaemonSet](https://cloud.tencent.com/document/product/457/98730) · page_id `98730`

## 概述

超级节点默认不支持 DaemonSet 调度。需要通过 Pod Annotation 为 DaemonSet 配置节点容忍（tolerations）和节点亲和性，使 DaemonSet Pod 可调度到超级节点上。

| 方式 | 配置位置 | 说明 |
|------|---------|------|
| tolerations | `DaemonSet.spec.template.spec` | 容忍超级节点的污点（taint），允许调度 |
| nodeSelector / affinity | 同上 | 指定调度到超级节点的标签（如 `node.kubernetes.io/instance-type: eklet`） |
| Annotation | `DaemonSet.spec.template.metadata.annotations` | 配置资源规格等（与普通 Pod 一致） |

**控制面验证**：通过 `DescribeClusterVirtualNodePools` 确认超级节点池已存在且状态正常。

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
# 4. 确认超级节点池已创建且状态正常
tccli tke DescribeClusterVirtualNodePools --region <Region> \
    --ClusterId CLUSTER_ID
# expected: NodePoolSet 非空，目标节点池 LifeState 为 normal

# 5. 获取 kubeconfig 并验证集群连通性
tccli tke DescribeClusterKubeconfig --region <Region> --ClusterId CLUSTER_ID \
    | jq -r '.Kubeconfig' > ~/.kube/config-super
kubectl --kubeconfig ~/.kube/config-super cluster-info
# expected: Kubernetes control plane 可达

# 6. 查看超级节点标签和污点（确认调度条件）
kubectl --kubeconfig ~/.kube/config-super get nodes -l node.kubernetes.io/instance-type=eklet --show-labels
# expected: 返回超级节点列表及标签
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
| 创建 DaemonSet | `kubectl apply -f` | 是 |
| 查看 DaemonSet 状态 | `kubectl get daemonset` | 是 |
| 查看超级节点标签 | `kubectl get nodes --show-labels` | 是 |

## 操作步骤

### 步骤 1：确认超级节点就绪并查看调度条件

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
```

```bash
kubectl --kubeconfig ~/.kube/config-super describe node NODE_NAME | grep -A5 Taints
# expected: 显示超级节点的污点（taint）
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `NODE_NAME` | 超级节点名称 | 以 `eklet` 开头 | `kubectl get nodes` |

### 步骤 2：创建支持超级节点的 DaemonSet

#### 选择依据

- **tolerations**：超级节点带有特定污点（taint），DaemonSet Pod 必须声明对应的容忍才能调度。
- **nodeAffinity**：通过节点标签 `node.kubernetes.io/instance-type: eklet` 限制 DaemonSet 仅调度到超级节点（或同时支持普通节点和超级节点）。
- **资源 Annotation**：超级节点上的 Pod 必须声明资源规格 Annotation（如 CPU/内存/系统盘），详见 [指定资源规格](../指定资源规格/tccli%20操作.md)。

#### 最小示例（仅调度到超级节点）

`daemonset-super-minimal.yaml`：

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: DAEMONSET_NAME
  namespace: NAMESPACE
spec:
  selector:
    matchLabels:
      app: APP_LABEL
  template:
    metadata:
      labels:
        app: APP_LABEL
      annotations:
        eks.tke.cloud.tencent.com/cpu: "1"
        eks.tke.cloud.tencent.com/mem: "2Gi"
    spec:
      tolerations:
      - key: "eks.tke.cloud.tencent.com/eklet"
        operator: "Exists"
        effect: "NoSchedule"
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: node.kubernetes.io/instance-type
                operator: In
                values:
                - eklet
      containers:
      - name: CONTAINER_NAME
        image: nginx:1.25
        resources:
          requests:
            cpu: "1"
            memory: "2Gi"
```

```bash
kubectl --kubeconfig ~/.kube/config-super apply -f daemonset-super-minimal.yaml
# expected: daemonset.apps/DAEMONSET_NAME created
```

#### 增强示例（同时支持普通节点和超级节点）

`daemonset-super-enhanced.yaml`：

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: DAEMONSET_NAME
  namespace: NAMESPACE
spec:
  selector:
    matchLabels:
      app: APP_LABEL
  template:
    metadata:
      labels:
        app: APP_LABEL
      annotations:
        eks.tke.cloud.tencent.com/cpu: "1"
        eks.tke.cloud.tencent.com/mem: "2Gi"
        eks.tke.cloud.tencent.com/root-cbs-size: "20"
    spec:
      tolerations:
      - key: "eks.tke.cloud.tencent.com/eklet"
        operator: "Exists"
        effect: "NoSchedule"
      containers:
      - name: CONTAINER_NAME
        image: nginx:1.25
        resources:
          requests:
            cpu: "1"
            memory: "2Gi"
          limits:
            cpu: "1"
            memory: "2Gi"
```

```bash
kubectl --kubeconfig ~/.kube/config-super apply -f daemonset-super-enhanced.yaml
# expected: daemonset.apps/DAEMONSET_NAME created
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `DAEMONSET_NAME` | DaemonSet 名称 | 须遵循 K8s 命名规范 | 自定义 |
| `NAMESPACE` | K8s 命名空间 | 如 `default` | `kubectl get ns` |
| `APP_LABEL` | 应用标签 | 须与 selector 一致 | 自定义 |
| `CONTAINER_NAME` | 容器名称 | 须遵循 K8s 命名规范 | 自定义 |

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
# 验证 DaemonSet 状态
kubectl --kubeconfig ~/.kube/config-super get daemonset DAEMONSET_NAME -n NAMESPACE
# expected: DESIRED == CURRENT == READY，节点数匹配

# 验证 Pod 调度到超级节点
kubectl --kubeconfig ~/.kube/config-super get pod -n NAMESPACE -o wide | grep DAEMONSET_NAME
# expected: NODE 列为虚拟节点名（以 eklet 开头），STATUS 为 Running

# 验证 Pod 详情
kubectl --kubeconfig ~/.kube/config-super describe pod -n NAMESPACE -l app=APP_LABEL
# expected: Annotations 包含资源规格，Node 为虚拟节点，Tolerations 包含 eklet
```

| 验证维度 | 命令 | 预期 |
|---------|------|------|
| 节点池就绪 | `DescribeClusterVirtualNodePools --ClusterId CLUSTER_ID` | `LifeState: "normal"` |
| DaemonSet 状态 | `kubectl get daemonset DAEMONSET_NAME` | DESIRED == READY |
| Pod 节点 | `kubectl get pod -l app=APP_LABEL -o wide` | NODE 以 `eklet` 开头 |
| Pod 状态 | 同上 | `STATUS: "Running"` |
| Annotation | `kubectl describe pod -l app=APP_LABEL` | 含资源规格 Annotation |

## 清理

> **警告**：删除 DaemonSet 将级联删除其管理的所有 Pod。如有业务数据需保留，先备份再清理。

```bash
# 删除 DaemonSet
kubectl --kubeconfig ~/.kube/config-super delete daemonset DAEMONSET_NAME -n NAMESPACE
# expected: daemonset.apps "DAEMONSET_NAME" deleted

# 验证 Pod 已清理
kubectl --kubeconfig ~/.kube/config-super get pod -n NAMESPACE -l app=APP_LABEL
# expected: No resources found in NAMESPACE namespace

# 验证 DaemonSet 已删除
kubectl --kubeconfig ~/.kube/config-super get daemonset DAEMONSET_NAME -n NAMESPACE
# expected: Error from server (NotFound): daemonsets.apps "DAEMONSET_NAME" not found
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl apply` 返回 `Error from server (Forbidden)` | `kubectl --kubeconfig ~/.kube/config-super auth can-i create daemonsets` | 当前 kubeconfig 对应子账号缺少 DaemonSet 创建权限 | 联系集群管理员授予 namespace 级别的 DaemonSet 创建权限 |
| `kubectl get nodes -l node.kubernetes.io/instance-type=eklet` 无节点 | `tccli tke DescribeClusterVirtualNodePools --region <Region> --ClusterId CLUSTER_ID` 确认节点池 | 超级节点池尚未创建或节点未注册 | 先执行 [新建超级节点](../新建超级节点/tccli%20操作.md) 创建节点池 |

### 创建成功但 Pod 未调度到超级节点

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| DaemonSet 创建成功但 Pod 未落在超级节点 | `kubectl describe daemonset DAEMONSET_NAME -n NAMESPACE \| tail -20` 查看 Events | 缺失 tolerations 或 nodeAffinity 配置 | 检查 DaemonSet YAML：1) 确认 `tolerations` 包含 eklet 污点容忍；2) 确认 `nodeAffinity`（如使用）匹配 `node.kubernetes.io/instance-type: eklet` |
| Pod Pending，Events 显示 `0/1 nodes are available: 1 node(s) had untolerated taint` | `kubectl describe node NODE_NAME \| grep Taints` 查看节点污点 | DaemonSet 未声明对超级节点污点的 tolerations | 在 DaemonSet Pod 模板中增加 eklet 相关的 tolerations 配置 |
| Pod Pending，Events 显示 `insufficient resources` | `kubectl describe pod -n NAMESPACE -l app=APP_LABEL` | Annotation 声明的资源规格超出超级节点可调度范围 | 参见 [超级节点可调度Pod说明](../../超级节点调度说明/超级节点可调度Pod说明/tccli%20操作.md) 调整规格 |
| Pod 调度到了普通节点而非超级节点 | `kubectl get pod -n NAMESPACE -l app=APP_LABEL -o wide` 查看 NODE | 未设置 nodeAffinity 或 matchExpressions 不正确 | 在 YAML 中添加 `requiredDuringSchedulingIgnoredDuringExecution` 限定 `node.kubernetes.io/instance-type: eklet` |

## 下一步

- [超级节点 Annotation 说明](https://cloud.tencent.com/document/product/457/44173) — 完整 Annotation 列表
- [指定资源规格](https://cloud.tencent.com/document/product/457/44174) — CPU/内存/系统盘规格配置
- [调度 Pod 至超级节点](https://cloud.tencent.com/document/product/457/74016) — 调度策略配置
- [采集超级节点上的 Pod 日志](https://cloud.tencent.com/document/product/457/60701) — 日志采集配置

## 控制台替代

[TKE 控制台 → 工作负载 → DaemonSet → 新建](https://console.cloud.tencent.com/tke2/cluster)：在高级设置中配置节点容忍和亲和性，勾选超级节点调度选项。
