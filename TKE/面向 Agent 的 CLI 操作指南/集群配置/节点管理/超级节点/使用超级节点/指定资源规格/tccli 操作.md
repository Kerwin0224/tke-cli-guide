# 指定资源规格

> 对照官方：[指定资源规格](https://cloud.tencent.com/document/product/457/44174) · page_id `44174`

## 概述

在超级节点上为 Pod **指定 CPU、内存、GPU 和系统盘规格**。可通过两种方式配置：创建超级节点池时通过 `VirtualNodes[].Quota` 字段声明默认配额，或在 Pod Annotation 中覆盖单个 Pod 的规格。Pod Annotation 优先级高于节点池 Quota。

| 配置方式 | 作用范围 | 配置位置 | 可修改性 |
|---------|---------|---------|:--:|
| `VirtualNodes[].Quota` | 超级节点池内所有 Pod 默认值 | `CreateClusterVirtualNodePool` / `ModifyClusterVirtualNodePool` | 修改后新 Pod 生效 |
| Pod Annotation | 单个 Pod | `.spec.template.metadata.annotations` | 每个 Pod 独立声明 |

**控制面验证**：通过 `DescribeClusterVirtualNodePools` 查看节点池 Quota 配置；通过 `kubectl describe pod` 查看 Pod 级 Annotation。

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
#    tke:ModifyClusterVirtualNodePool
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

# 5. 获取 kubeconfig
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
| 查看节点池 Quota | `DescribeClusterVirtualNodePools` | 是 |
| 修改节点池 Quota | `ModifyClusterVirtualNodePool` | 是 |
| 声明 Pod Annotation | `kubectl apply -f` | 是 |
| 查看 Pod 规格 | `kubectl describe pod` | 是 |

## 关键字段说明

### 节点池 Quota（`VirtualNodes[].Quota`）

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `Quota.Cpu` | Integer | 否 | CPU 核数：包年包月 0.25~8，按量 0.25~64。非标准规格向上取整 | 超出范围 → Pod 调度失败 |
| `Quota.Memory` | Integer | 否 | 内存 GiB，与 CPU 配合：包年包月比 < 1:4，按量比 ≤ 1:8 | CPU:内存比超标 → Pod 调度失败 |
| `Quota.Gpu` | Integer | 否 | GPU 卡数，仅按量计费超级节点支持 | 包年包月节点池声明 GPU → 配置被忽略 |

### Pod Annotation（覆盖级别）

| Annotation Key | 类型 | 取值示例 | 说明 |
|------|------|------|------|
| `eks.tke.cloud.tencent.com/cpu` | String | `"2"` | 为单个 Pod 覆盖 CPU 核数 |
| `eks.tke.cloud.tencent.com/mem` | String | `"4Gi"` | 为单个 Pod 覆盖内存大小 |
| `eks.tke.cloud.tencent.com/root-cbs-size` | String | `"20"` | 系统盘大小（GiB） |
| `eks.tke.cloud.tencent.com/root-cbs-type` | String | `"CLOUD_PREMIUM"` | 系统盘类型 |

## 操作步骤

### 步骤 1：查看节点池当前 Quota

```bash
tccli tke DescribeClusterVirtualNodePools --region <Region> \
    --ClusterId CLUSTER_ID
# expected: exit 0，查看 VirtualNodes[].Quota 字段
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
            "VirtualNodes": [
                {
                    "SubnetId": "SUBNET_ID",
                    "Quota": {
                        "Cpu": 2,
                        "Memory": 4
                    }
                }
            ]
        }
    ],
    "RequestId": "..."
}
```

### 步骤 2：通过 Pod Annotation 指定资源规格

#### 选择依据

- **Pod 级 Annotation** 优先级高于节点池 Quota。推荐为每个 Pod 单独声明规格，避免全局设置影响其他工作负载。
- **CPU:内存比**：按量超级节点最大 1:8，包年包月最大 1:4。非标准规格向上取整。
- **系统盘**：默认 20GiB，可通过 `root-cbs-size` 和 `root-cbs-type` Annotation 调整。

#### 最小示例（仅 CPU 和内存）

`pod-resource-minimal.yaml`：

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
kubectl --kubeconfig ~/.kube/config-super apply -f pod-resource-minimal.yaml
# expected: pod/POD_NAME created
```

#### 增强示例（含系统盘配置）

`pod-resource-enhanced.yaml`：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: POD_NAME
  annotations:
    eks.tke.cloud.tencent.com/cpu: "4"
    eks.tke.cloud.tencent.com/mem: "8Gi"
    eks.tke.cloud.tencent.com/root-cbs-size: "50"
    eks.tke.cloud.tencent.com/root-cbs-type: "CLOUD_SSD"
spec:
  containers:
  - name: CONTAINER_NAME
    image: nginx:1.25
    resources:
      requests:
        cpu: "4"
        memory: "8Gi"
      limits:
        cpu: "4"
        memory: "8Gi"
```

```bash
kubectl --kubeconfig ~/.kube/config-super apply -f pod-resource-enhanced.yaml
# expected: pod/POD_NAME created
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `POD_NAME` | Pod 名称 | 须遵循 K8s 命名规范 | 自定义 |
| `CONTAINER_NAME` | 容器名称 | 须遵循 K8s 命名规范 | 自定义 |

### 步骤 3：验证 Pod 规格生效

```bash
kubectl --kubeconfig ~/.kube/config-super describe pod POD_NAME
# expected: Annotations 段包含 cpu/mem/root-cbs 等规格声明
```

```bash
kubectl --kubeconfig ~/.kube/config-super get pod POD_NAME -o json \
    | jq '.spec.containers[0].resources'
# expected: requests/limits 与 Annotation 声明一致
```

## 验证

### 控制面（tccli）

```bash
tccli tke DescribeClusterVirtualNodePools --region <Region> \
    --ClusterId CLUSTER_ID
# expected: NodePoolSet 非空，VirtualNodes[].Quota 字段可见
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
# 验证 Pod 状态和调度
kubectl --kubeconfig ~/.kube/config-super get pod POD_NAME -o wide
# expected: STATUS 为 Running，NODE 为虚拟节点

# 验证资源规格
kubectl --kubeconfig ~/.kube/config-super describe pod POD_NAME | grep -A5 "Annotations"
# expected: 包含 eks.tke.cloud.tencent.com/cpu、mem 等键
```

| 验证维度 | 命令 | 预期 |
|---------|------|------|
| 节点池 Quota | `DescribeClusterVirtualNodePools --ClusterId CLUSTER_ID` | VirtualNodes[].Quota 字段可见 |
| Pod 就绪 | `kubectl get pod POD_NAME` | `STATUS: "Running"` |
| 资源规格生效 | `kubectl describe pod POD_NAME` | Annotations 含声明的资源规格 |
| 调度到超级节点 | `kubectl get pod POD_NAME -o wide` | NODE 以 `eklet` 开头 |
| 容器资源匹配 | `kubectl get pod POD_NAME -o json \| jq '.spec.containers[0].resources'` | requests/limits 与 Annotation 一致 |

## 清理

> **警告**：删除 Pod 将清除该 Pod 的运行时数据（包括系统盘上的数据）。测试完成后应及时清理避免产生不必要的按量计费。

```bash
# 删除测试 Pod
kubectl --kubeconfig ~/.kube/config-super delete pod POD_NAME
# expected: pod "POD_NAME" deleted

# 验证删除
kubectl --kubeconfig ~/.kube/config-super get pod POD_NAME
# expected: Error from server (NotFound): pods "POD_NAME" not found
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Pod 创建失败，Events 显示 `FailedScheduling` | `kubectl describe pod POD_NAME \| tail -20` 查看 Events | Annotation 声明的 CPU/内存超出超级节点可调度范围 | 参见 [超级节点可调度Pod说明](../../超级节点调度说明/超级节点可调度Pod说明/tccli%20操作.md) 调整规格。包年包月：CPU 0.25~8C，内存比 < 1:4；按量：CPU 0.25~64C，内存比 ≤ 1:8 |
| `kubectl describe pod` 中 Annotations 缺少声明的键 | `kubectl get pod POD_NAME -o json \| jq '.metadata.annotations'` 检查实际 Annotation | Annotation Key 拼写错误 | 核对 Key 格式：前缀 `eks.tke.cloud.tencent.com/`，Key 全小写，连字符分隔 |
| Pod CPU/内存规格不符合预期 | `kubectl get pod POD_NAME -o json \| jq '.spec.containers[0].resources'` 检查 resources 字段 | `resources.requests` 与 Annotation 不一致 | 确保 `.spec.containers[].resources.requests` 与 Annotation 声明值一致 |

### 创建成功但规格不生效

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Pod Running 但实际分配资源远小于声明值 | `kubectl describe pod POD_NAME` 查看节点资源分配 | Annotation 声明的非标准规格被向上取整，或资源已超过可调度上限 | 检查 [可调度 Pod 说明](../../超级节点调度说明/超级节点可调度Pod说明/tccli%20操作.md) 确认规格是否在允许范围内 |
| 系统盘大小未按 Annotation 生效 | `kubectl exec POD_NAME -- df -h` 查看磁盘 | `root-cbs-size` 值格式错误（应为纯数字字符串，如 `"50"`） | 修正为纯数字字符串格式，不加单位 |
| GPU Annotation 不生效 | `tccli tke DescribeClusterVirtualNodePools --region <Region> --ClusterId CLUSTER_ID` 确认计费模式 | 包年包月超级节点池不支持 GPU 规格 | 改用按量计费超级节点池 |

## 下一步

- [超级节点 Annotation 说明](https://cloud.tencent.com/document/product/457/44173) — 完整 Annotation 列表
- [超级节点可调度 Pod 说明](https://cloud.tencent.com/document/product/457/74015) — 可调度规格范围
- [镜像缓存](https://cloud.tencent.com/document/product/457/65908) — 镜像缓存加速 Pod 启动
- [调度 Pod 至超级节点](https://cloud.tencent.com/document/product/457/74016) — 调度策略配置

## 控制台替代

[TKE 控制台 → 工作负载 → 新建 → 高级设置 → 超级节点资源规格](https://console.cloud.tencent.com/tke2/cluster)：在控制台表单中选择 CPU、内存和系统盘规格。
