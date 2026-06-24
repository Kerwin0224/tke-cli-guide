# 超级节点 Annotation 说明

> 对照官方：[超级节点 Annotation 说明](https://cloud.tencent.com/document/product/457/44173) · page_id `44173`

## 概述

超级节点通过 Kubernetes **Pod Annotation** 控制资源规格、镜像缓存、调度策略、日志采集等行为。超级节点没有统一的「写 Annotation」tccli API——Annotation 在 workload YAML 的 `.spec.template.metadata.annotations` 字段中声明，由超级节点后端（eklet）解析和执行。

| Annotation 类别 | 控制内容 | 详细文档 |
|------|------|------|
| 资源规格 | CPU、内存、GPU、系统盘大小/类型 | [指定资源规格](../指定资源规格/tccli%20操作.md) |
| 镜像缓存 | 镜像缓存复用键、忽略证书校验 | [镜像缓存](../镜像缓存/tccli%20操作.md) |
| 调度策略 | DaemonSet 调度、节点容忍 | [超级节点上支持运行DaemonSet](../超级节点上支持运行DaemonSet/tccli%20操作.md) |
| 日志采集 | CLS 日志集/主题、Kafka 投递 | [采集超级节点上的Pod日志](../采集超级节点上的Pod日志/tccli%20操作.md) |
| 网络 | 固定 IP、绑定 EIP、DNS 配置 | 官方文档 |
| 安全 | 安全组覆盖、IAM 角色绑定 | 官方文档 |

**控制面验证**：通过 `DescribeClusterVirtualNodePools` 确认超级节点池已创建且状态正常，确保 Annotation 可以生效。

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
#    tke:DescribeClusterVirtualNodePools
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
# 4. 确认超级节点池已创建且状态正常
tccli tke DescribeClusterVirtualNodePools --region <Region> \
    --ClusterId CLUSTER_ID
# expected: NodePoolSet 非空，目标节点池 LifeState 为 normal

# 5. 获取 kubeconfig 并验证集群连通性
tccli tke DescribeClusterKubeconfig --region <Region> --ClusterId CLUSTER_ID \
    | jq -r '.Kubeconfig' > ~/.kube/config-super
kubectl --kubeconfig ~/.kube/config-super cluster-info
# expected: Kubernetes control plane 可达
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
| 查看超级节点池状态 | `DescribeClusterVirtualNodePools` | 是 |
| 获取 kubeconfig | `DescribeClusterKubeconfig` | 是 |
| 声明 Pod Annotation | `kubectl apply -f` | 是 |
| 查看 Pod 已挂载 Annotation | `kubectl describe pod` | 是 |

## 关键字段说明

超级节点 Annotation 的通用字段说明：

| 字段 | 格式 | 作用层级 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| Annotation Key | `eks.tke.cloud.tencent.com/<key>` | Pod | 须严格匹配官方键名，区分大小写 | 拼写错误 → Annotation 被忽略，不生效 |
| Annotation Value | String | Pod | 由各 Annotation 类型决定，如 `"2"` 表示 2 核 CPU | 格式错误 → Pod 创建失败或规格不符 |

> **注意**：Annotation 在 Pod 创建时生效，修改已运行 Pod 的 Annotation 不会触发资源调整。需重建 Pod 使新 Annotation 生效。

## 操作步骤

### 步骤 1：确认超级节点就绪

```bash
tccli tke DescribeClusterVirtualNodePools --region <Region> \
    --ClusterId CLUSTER_ID
# expected: exit 0，NodePoolSet 中目标节点池 LifeState 为 normal
```

**预期输出**：

```json
{
    "TotalCount": 1,
    "NodePoolSet": [
        {
            "NodePoolId": "np-example",
            "Name": "NODE_POOL_NAME",
            "LifeState": "normal",
            "SubnetIds": ["SUBNET_ID"]
        }
    ],
    "RequestId": "..."
}
```

### 步骤 2：在工作负载中声明 Annotation

#### 选择依据

- **资源规格类** Annotation（CPU/内存/GPU）：在 Pod 模板中与 `resources.requests` 配合使用。优先用 Annotation 声明精确规格。
- **镜像缓存类** Annotation：对镜像较大、启动频繁的 Pod 加 `cbs-reuse-key` 以复用磁盘数据。
- **日志采集类** Annotation：需先开通 CLS 并在 Annotation 中指定日志集/主题 ID。

#### 最小示例（资源规格）

`pod-annotation-minimal.yaml`：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: POD_NAME
  annotations:
    eks.tke.cloud.tencent.com/cpu: "1"
    eks.tke.cloud.tencent.com/mem: "2Gi"
spec:
  containers:
  - name: CONTAINER_NAME
    image: nginx:1.25
    resources:
      requests:
        cpu: "1"
        memory: "2Gi"
```

```bash
kubectl --kubeconfig ~/.kube/config-super apply -f pod-annotation-minimal.yaml
# expected: pod/POD_NAME created
```

#### 增强示例（含多 Annotation）

`pod-annotation-enhanced.yaml`：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: POD_NAME
  annotations:
    eks.tke.cloud.tencent.com/cpu: "2"
    eks.tke.cloud.tencent.com/mem: "4Gi"
    eks.tke.cloud.tencent.com/root-cbs-size: "20"
    eks.tke.cloud.tencent.com/cbs-reuse-key: "CACHE_KEY"
spec:
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
kubectl --kubeconfig ~/.kube/config-super apply -f pod-annotation-enhanced.yaml
# expected: pod/POD_NAME created
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `POD_NAME` | Pod 名称 | 须遵循 K8s 命名规范 | 自定义 |
| `CONTAINER_NAME` | 容器名称 | 须遵循 K8s 命名规范 | 自定义 |
| `CACHE_KEY` | 镜像缓存复用键 | 相同键的 Pod 可复用镜像数据 | 自定义 |

### 步骤 3：查看 Pod 已挂载的 Annotation

```bash
kubectl --kubeconfig ~/.kube/config-super describe pod POD_NAME
# expected: Annotations 段包含声明的 Annotation
```

```bash
kubectl --kubeconfig ~/.kube/config-super get pod POD_NAME -o json \
    | jq '.metadata.annotations'
# expected: 返回已声明的 Annotation 键值对
```

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
# 验证 Pod Annotation 生效
kubectl --kubeconfig ~/.kube/config-super describe pod POD_NAME
# expected: Annotations 段包含声明的 Annotation 键值对

# 验证 Pod 调度到超级节点
kubectl --kubeconfig ~/.kube/config-super get pod POD_NAME -o wide
# expected: NODE 列为虚拟节点名（以 eklet 开头），STATUS 为 Running
```

| 验证维度 | 命令 | 预期 |
|---------|------|------|
| 节点池就绪 | `DescribeClusterVirtualNodePools --ClusterId CLUSTER_ID` | `LifeState: "normal"` |
| Annotation 声明 | `kubectl describe pod POD_NAME` | Annotations 段含目标 Annotation |
| Pod 状态 | `kubectl get pod POD_NAME` | `STATUS: "Running"` |
| 节点类型 | `kubectl get pod POD_NAME -o wide` | NODE 以 `eklet` 开头 |

## 清理

> **警告**：删除 Pod 将清除该 Pod 上的 Annotation 配置和运行时数据。如使用包年包月超级节点池，空节点池仍可能产生费用，需同时删除节点池。

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
| `kubectl apply` 返回 `Error from server (Forbidden)` | `kubectl --kubeconfig ~/.kube/config-super auth can-i create pods` | 当前 kubeconfig 对应子账号缺少 Pod 创建权限 | 联系集群管理员授予 namespace 级别的 Pod 创建权限 |
| `kubectl get nodes` 无超级节点 | `tccli tke DescribeClusterVirtualNodePools --region <Region> --ClusterId CLUSTER_ID` 确认节点池状态 | 超级节点池未创建或状态异常 | 先执行 [新建超级节点](../新建超级节点/tccli%20操作.md) 创建节点池 |
| `DescribeClusterVirtualNodePools` 返回 `InvalidParameter.ClusterId` | `tccli tke DescribeClusters --region <Region>` 列出集群 | 集群 ID 错误或不属于当前地域 | 用 `tccli tke DescribeClusters --region <Region>` 获取正确 ClusterId |

### 创建成功但 Annotation 不生效

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Pod 创建成功但 Annotation 挂载不完整 | `kubectl get pod POD_NAME -o json \| jq '.metadata.annotations'` 对比声明值 | Annotation Key 拼写错误（大小写、前缀不匹配） | 严格按官方文档键名修正 Annotation Key。常见错误：`tke` 写成 `TKE`，`eks` 写成 `EKS` |
| Pod 状态 Pending，Events 显示 `insufficient resources` | `kubectl describe pod POD_NAME \| tail -20` 查看 Events | Annotation 声明的规格超出超级节点可调度范围 | 参见 [超级节点可调度Pod说明](../../超级节点调度说明/超级节点可调度Pod说明/tccli%20操作.md) 调整规格 |
| 修改 Annotation 后 Pod 行为不变 | `kubectl get pod POD_NAME -o yaml` 查看当前 Annotation | 修改已运行 Pod 的 Annotation 不会触发重新调度 | 删除 Pod 并重新创建（或滚动更新 Deployment） |

## 下一步

- [指定资源规格](https://cloud.tencent.com/document/product/457/44174) — CPU/内存/GPU/系统盘 Annotation 详解
- [镜像缓存](https://cloud.tencent.com/document/product/457/65908) — 镜像缓存 Annotation 加速 Pod 启动
- [超级节点上支持运行 DaemonSet](https://cloud.tencent.com/document/product/457/98730) — DaemonSet 调度 Annotation
- [采集超级节点上的 Pod 日志](https://cloud.tencent.com/document/product/457/60701) — 日志采集 Annotation
- [调度 Pod 至超级节点](https://cloud.tencent.com/document/product/457/74016) — 调度策略配置

## 控制台替代

[TKE 控制台 → 工作负载 → 新建 → YAML 编辑器](https://console.cloud.tencent.com/tke2/cluster)：在 Pod 模板的 `metadata.annotations` 中添加 Annotation。
