# qGPU 多卡互联

> 对照官方：[qGPU 多卡互联](https://cloud.tencent.com/document/product/457/127667) · page_id `127667` · tccli ≥3.1.x · API 2018-05-25

## 概述

qGPU 多卡互联允许单个 Pod 同时使用多张物理 GPU 的 vGPU 资源，突破单卡算力上限。通过在 Pod `resources.limits` 中将 `tke.cloud.tencent.com/qgpu-core` 设置为大于 100 的值，调度器自动将 Pod 绑定到多张 GPU，并按比例分配显存。

**触发规则**：`qgpu-core` 值 1-100 为单卡模式（Pod 使用单张 GPU 的 vGPU）；`qgpu-core` 值 > 100 为多卡模式（每 100 算力对应一张 GPU，显存按 GPU 数量均分）。例如 `qgpu-core: "200"` + `qgpu-memory: "20"` 表示跨 2 张 GPU，每张分配 10GiB 显存。

**拓扑依赖**：多卡互联性能取决于物理 GPU 间互联拓扑（NVLink > PCIe）。部署前应确认节点存在多张 GPU 且互联拓扑满足需求。

| 模式 | qgpu-core 取值 | GPU 数量 | 显存分配 | 适用场景 |
|------|---------------|---------|---------|---------|
| 单卡 | 1-100 | 1 张 | 整块分配给该 Pod | 算力需求在单卡范围内的任务 |
| 多卡 | > 100（如 200） | 2 张（每 100 对应 1 张） | 按 GPU 数量均分 | 需要跨卡聚合算力的训练/推理任务 |

## 前置条件

- [环境准备](../../../../环境准备.md)
- [使用 qGPU](../使用%20qGPU/tccli%20操作.md) — 集群已启用 QGPU 并安装 qgpu 组件

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置，region 为集群所在地域

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:DescribeClusters
# 验证：执行 DescribeClusters 确认权限
tccli tke DescribeClusters --region <Region>
# expected: exit 0，返回集群列表（可为空）

# 4. 检查 kubectl 版本
kubectl version --client
# expected: Client Version: v1.28 或更高

# 5. 获取并验证 kubeconfig 连通性
tccli tke DescribeClusterKubeconfig --region <Region> \
    --ClusterId <ClusterId> | jq -r '.Kubeconfig' > ~/.kube/config
# expected: exit 0，kubeconfig 写入 ~/.kube/config

kubectl cluster-info
# expected: Kubernetes control plane 运行中
```

`tccli configure list` 输出示例：

```text
credential:
  secretId:    AKID****************************
  secretKey:   ********************************
  region:      ap-guangzhou
  ...
```

`kubectl version --client` 输出示例：

```text
Client Version: v1.28.0
Kustomize Version: v5.0.4-0.20230601165947-6ce0bf390ce3
```

`kubectl cluster-info` 输出示例：

```text
Kubernetes control plane is running at https://cls-example.ccs.tencent-cloud.com
CoreDNS is running at https://cls-example.ccs.tencent-cloud.com/api/v1/.../proxy
```

### 资源检查

```bash
# 6. 确认目标集群存在且 QGPU 已启用
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["<ClusterId>"]'
# expected: TotalCount >= 1，ClusterStatus 为 Running，QGPUShareEnable 为 true
```

预期输出（节选关键字段）：

```json
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "cls-example",
            "ClusterName": "CLUSTER_NAME",
            "ClusterStatus": "Running",
            "ClusterType": "MANAGED_CLUSTER",
            "QGPUShareEnable": true
        }
    ],
    "RequestId": "..."
}
```

> **注意**：若 `QGPUShareEnable` 为 `false`，集群节点不会注册 `tke.cloud.tencent.com/qgpu-core` 和 `tke.cloud.tencent.com/qgpu-memory` 扩展资源，qGPU 多卡互联无法工作。需先在控制台或通过 `ModifyClusterAttribute` API 启用 QGPU 功能，参考 [使用 qGPU](../使用%20qGPU/tccli%20操作.md)。

```bash
# 7. 确认节点有多张 GPU（多卡互联的必要条件）
kubectl describe node <GpuNode> | grep -A 5 nvidia.com/gpu
# expected: nvidia.com/gpu 的 Capacity 和 Allocatable 行，数量 >= 2

# 8. 确认 qgpu 资源已注册到节点
kubectl describe node <GpuNode> | grep -E 'qgpu-core|qgpu-memory'
# expected: 出现 tke.cloud.tencent.com/qgpu-core 和 tke.cloud.tencent.com/qgpu-memory 资源行

# 9. 确认节点 GPU 互联拓扑（NVLink 或 PCIe）
kubectl exec -n kube-system <QgpuDsPod> -- nvidia-smi topo -m
# expected: 输出 GPU 间拓扑矩阵，NVLink 列显示 NV 表示 NVLink 互联
```

预期输出 — GPU 资源确认（`kubectl describe node <GpuNode>`，2 张以上 GPU 节点）：

```text
 nvidia.com/gpu:     4
 nvidia.com/gpu:     4
```

预期输出 — qgpu 扩展资源注册：

```text
 tke.cloud.tencent.com/qgpu-core:     400
 tke.cloud.tencent.com/qgpu-core:     400
 tke.cloud.tencent.com/qgpu-memory:   160
 tke.cloud.tencent.com/qgpu-memory:   160
```

预期输出（`nvidia-smi topo -m`，2 张 GPU 示例）：

```text
        GPU0    GPU1    NIC0    CPU Affinity    NUMA Affinity
GPU0     X      NV      SYS     0-15            N/A
GPU1     NV      X      SYS     0-15            N/A
Legend:

  X    = Self
  SYS  = Connection traversing PCIe as well as the SMP interconnect between NUMA nodes
  NV   = Connection traversing a bonded set of NVLinks
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<Region>` | 集群所在地域 | 如 `ap-guangzhou` | `tccli configure list` 确认当前 region |
| `<ClusterId>` | 集群 ID | 格式 `cls-xxxxxxxx` | `tccli tke DescribeClusters --region <Region>` |
| `<GpuNode>` | GPU 节点名 | 节点须有多张 GPU（`nvidia.com/gpu >= 2`） | `kubectl get nodes -l accelerator=nvidia` |
| `<QgpuDsPod>` | qgpu 组件 DaemonSet Pod | 位于 `kube-system` 命名空间 | `kubectl get pods -n kube-system -l app=qgpu` |

## 控制台与 CLI 参数映射

| 控制台操作 | tccli / kubectl 命令 | 幂等 |
|-----------|----------------------|:--:|
| 查看集群 QGPU 状态 | `DescribeClusters`（检查 `QGPUShareEnable`） | 是 |
| 查看节点 GPU 资源 | `kubectl describe node` | 是 |
| 查看 GPU 互联拓扑 | `kubectl exec -- nvidia-smi topo -m` | 是 |
| 部署多卡互联 Pod | `kubectl apply -f`（YAML 中 `qgpu-core > 100`） | 否 |
| 验证多卡生效 | `kubectl exec POD -- nvidia-smi` | 是 |

## 关键字段说明

qGPU 多卡互联通过 Pod 的 `resources.limits` 注解触发，核心字段如下：

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `tke.cloud.tencent.com/qgpu-core` | String | 是 | GPU 算力百分比。1-100 为单卡模式；> 100（如 200）为多卡模式，每 100 对应 1 张 GPU。最大值取决于节点 GPU 数 × 100 | 值 > 节点可用算力 → Pod 一直 Pending |
| `tke.cloud.tencent.com/qgpu-memory` | String | 是 | GPU 显存 GiB。多卡模式下按 GPU 数量均分（如 20GiB + 2 张 GPU → 每张 10GiB） | 值 > 节点可用显存 → Pod 一直 Pending |
| `qgpu-core` 与 GPU 数的关系 | — | — | 多卡 GPU 数 = `ceil(qgpu-core / 100)`。如 `qgpu-core: "200"` → 2 张，`qgpu-core: "350"` → 4 张 | `qgpu-core: "250"` 但节点仅 2 张 GPU → Pod 一直 Pending |
| 互联拓扑 | — | 否 | NVLink 互联性能远优于 PCIe。通过 `nvidia-smi topo -m` 确认 | PCIe 互联下多卡性能提升有限 |
| `image` | String | 是 | 示例使用 `nvidia/cuda:11.0-base`。镜像的 CUDA 版本需与节点 GPU 驱动兼容（`nvidia-smi` 顶部显示 CUDA Version），不匹配可能导致 CUDA 库加载失败 | 镜像 CUDA 版本 > 驱动版本 → Pod 内 `nvidia-smi` 报 `CUDA driver version is insufficient` |
| `restartPolicy` | String | 否 | 示例设为 `Never`（Pod 退出后不自动重启，GPU 资源不释放）。长期运行应设为 `Always` | 设为 `Never` 且 `command` 到期 → Pod 进入 `Completed` 状态，需手动删除重建 |
| `nodeName` | String | 否 | 示例设为 `<GpuNode>`（指定 GPU 节点）。副作用：若该节点 GPU 不足，Pod 无法调度到其他节点，需配合 `tolerations` 使用 | 指定节点 GPU 不足 → Pod 一直 Pending，不会尝试其他节点 |
| `command` | String | 否 | 示例设为 `["sleep", "3600"]`（1 小时后自动退出），仅适用于临时验证。生产应用需替换为实际业务命令 | 3600 秒到期 → Pod 退出，无法持续服务 |

## 操作步骤

### 步骤 1：确认节点 GPU 数量满足多卡需求

#### 选择依据

多卡互联要求单个节点上有多张可用 GPU。需先确认目标节点的 GPU 数量，再据此设定 `qgpu-core` 值。例如节点有 2 张 GPU，则 `qgpu-core` 最大可设 200；有 4 张则最大 400。

```bash
# 列出带 GPU 标签的节点
kubectl get nodes -l accelerator=nvidia
# expected: 返回 GPU 节点列表

# 查看目标节点的 GPU 资源
kubectl describe node <GpuNode> | grep -A 10 -E 'Capacity|Allocatable'
# expected: nvidia.com/gpu 的 Capacity >= 2
```

`kubectl get nodes -l accelerator=nvidia` 输出示例：

```text
NAME                STATUS   ROLES    AGE   VERSION
gpu-node-01         Ready    <none>   3d    v1.28.0
gpu-node-02         Ready    <none>   3d    v1.28.0
```

预期输出（`kubectl describe node` 节选）：

```text
Capacity:
  cpu:                64
  memory:             256Gi
  nvidia.com/gpu:     4
  tke.cloud.tencent.com/qgpu-core:  400
  tke.cloud.tencent.com/qgpu-memory: 160
Allocatable:
  cpu:                62
  memory:             250Gi
  nvidia.com/gpu:     4
  tke.cloud.tencent.com/qgpu-core:  400
  tke.cloud.tencent.com/qgpu-memory: 160
```

> 上例中节点有 4 张 GPU，`qgpu-core` 最大可设 400，`qgpu-memory` 最大 160GiB。多卡互联 Pod 的 `qgpu-core` 不可超过节点的 Allocatable 值。

### 步骤 2：查看 GPU 互联拓扑

#### 选择依据

多卡互联性能受物理拓扑影响。NVLink 互联的 GPU 间带宽远高于 PCIe。部署前确认拓扑可预估多卡加速效果。

```bash
# 找到 qgpu 组件 Pod
kubectl get pods -n kube-system -l app=qgpu
# expected: 返回 qgpu DaemonSet Pod 列表，选择 <GpuNode> 上的 Pod

# 查看拓扑矩阵
kubectl exec -n kube-system <QgpuDsPod> -- nvidia-smi topo -m
# expected: 输出 GPU 间拓扑矩阵，NV 列为 NV 表示 NVLink 互联
```

`kubectl get pods -n kube-system -l app=qgpu` 输出示例：

```text
NAME              READY   STATUS    RESTARTS   AGE   NODE
qgpu-manager-xxx  1/1     Running   0          2d    gpu-node-01
qgpu-manager-yyy  1/1     Running   0          2d    gpu-node-02
qgpu-scheduler-zzz 1/1    Running   0          2d    master-node
```

### 步骤 3：部署多卡互联 Pod

#### 选择依据

- **qgpu-core 值**：设为 `200` 表示跨 2 张 GPU（每 100 算力对应 1 张）。需确保节点 GPU 数 >= 2。
- **qgpu-memory 值**：设为 `20` 表示总显存 20GiB，多卡模式下自动均分（2 张 GPU 各 10GiB）。
- **镜像选择**：使用 `nvidia/cuda:11.0-base` 作为验证镜像。实际业务替换为含 CUDA 工具链的应用镜像。
- **资源 request 等于 limit**：避免超卖导致调度不稳定。

#### 最小创建（单容器，跨 2 张 GPU）

`multi-gpu-pod-minimal.yaml`：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-gpu-pod
spec:
  containers:
  - name: cuda-app
    image: nvidia/cuda:11.0-base
    command: ["sleep", "3600"]
    resources:
      limits:
        tke.cloud.tencent.com/qgpu-core: "200"
        tke.cloud.tencent.com/qgpu-memory: "20"
  restartPolicy: Never
```

```bash
kubectl apply -f multi-gpu-pod-minimal.yaml
# expected: pod/multi-gpu-pod created
```

```bash
kubectl get pod multi-gpu-pod
# expected: STATUS 为 Running
```

预期输出：

```text
NAME             READY   STATUS    RESTARTS   AGE
multi-gpu-pod    1/1     Running   0          30s
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `qgpu-core` | GPU 算力百分比 | > 100 触发多卡模式。`200` → 2 张 GPU | 节点 Allocatable 的 `qgpu-core` 值 |
| `qgpu-memory` | GPU 显存 GiB | 多卡模式下自动均分。`20` + 2 张 GPU → 每张 10GiB | 节点 Allocatable 的 `qgpu-memory` 值 |

#### 增强配置（指定节点 + 容忍 + 标签）

`multi-gpu-pod-enhanced.yaml`：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-gpu-pod
  labels:
    app: multi-gpu-demo
spec:
  nodeName: <GpuNode>
  containers:
  - name: cuda-app
    image: nvidia/cuda:11.0-base
    command: ["sleep", "3600"]
    resources:
      limits:
        tke.cloud.tencent.com/qgpu-core: "200"
        tke.cloud.tencent.com/qgpu-memory: "20"
  tolerations:
  - key: "nvidia.com/gpu"
    operator: "Exists"
    effect: "NoSchedule"
  restartPolicy: Never
```

```bash
kubectl apply -f multi-gpu-pod-enhanced.yaml
# expected: pod/multi-gpu-pod created
```

| 层级 | 包含字段 | 目的 |
|------|---------|------|
| **最小创建** | 只含 Pod 基本字段 + qgpu 资源声明 | 验证多卡调度是否生效 |
| **增强配置** | 增加 `nodeName`（指定 GPU 节点）、`tolerations`（容忍 GPU 污点）、`labels` | 生产环境推荐，避免调度到非 GPU 节点 |

### 步骤 4：验证多卡生效

```bash
# 确认 Pod 已调度到 GPU 节点且 Running
kubectl get pod multi-gpu-pod -o wide
# expected: STATUS 为 Running，NODE 为 <GpuNode>

# 进入 Pod 查看 GPU 使用情况
kubectl exec multi-gpu-pod -- nvidia-smi
# expected: 显示 2 张 GPU（GPU 0 和 GPU 1），每张显存约 10GiB
```

预期输出（`nvidia-smi`，多卡互联 Pod）：

```text
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 470.182.03   Driver Version: 470.182.03   CUDA Version: 11.4     |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC|
|   0  Tesla V100-SXM2...  On   | 00000000:06:00.0 Off |                    0|
|   1  Tesla V100-SXM2...  On   | 00000000:07:00.0 Off |                    0|
|-------------------------------+----------------------+----------------------+
| GPU  qGPU-Util  Memory-Usage      |
|   0      0%      10MiB/10240MiB   |
|   1      0%      10MiB/10240MiB   |
+-----------------------------------------------------------------------------+
```

> 多卡互联生效的标志：`nvidia-smi` 输出显示 2 张 GPU（GPU 0 和 GPU 1），每张显存约为 `qgpu-memory / GPU数量`（本例 20/2 = 10GiB）。

## 验证

### 控制面（tccli）

```bash
# 确认集群 QGPU 仍启用
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["<ClusterId>"]'
# expected: QGPUShareEnable 为 true，ClusterStatus 为 Running
```

```json
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "cls-example",
            "ClusterName": "<ClusterName>",
            "ClusterStatus": "Running",
            "ClusterType": "MANAGED_CLUSTER",
            "QGPUShareEnable": true
        }
    ],
    "RequestId": "..."
}
```

### 数据面

```bash
# 1. 确认节点 GPU 数量（多卡前提）
kubectl describe node <GpuNode> | grep nvidia.com/gpu
# expected: Capacity 和 Allocatable 的 nvidia.com/gpu >= 2

# 2. 确认 Pod 已调度到 GPU 节点
kubectl get pod multi-gpu-pod -o wide
# expected: STATUS 为 Running，NODE 为 GPU 节点

# 3. 验证多卡可见 — nvidia-smi 显示多张 GPU
kubectl exec multi-gpu-pod -- nvidia-smi -L
# expected: 输出多行 GPU 信息（如 GPU 0: Tesla V100 ... 和 GPU 1: Tesla V100 ...）

# 4. 验证显存分配 — 每张 GPU 显存为 qgpu-memory / GPU数量
kubectl exec multi-gpu-pod -- nvidia-smi --query-gpu=index,memory.total --format=csv
# expected: 每张 GPU 的 memory.total 约为 qgpu-memory / GPU数量（如 20/2=10GiB → 10240 MiB）

# 5. 验证互联拓扑 — 确认 NVLink 可用
kubectl exec multi-gpu-pod -- nvidia-smi topo -m
# expected: GPU 间拓扑矩阵，NV 列显示 NV 表示 NVLink 互联

# 6. 查看 Pod 日志确认应用正常运行
kubectl logs multi-gpu-pod
# expected: 无异常错误日志
```

`kubectl get pod multi-gpu-pod -o wide` 输出示例：

```text
NAME             READY   STATUS    RESTARTS   AGE   IP            NODE          NOMINATED NODE
multi-gpu-pod    1/1     Running   0          5m    10.0.16.10    gpu-node-01   <none>
```

`kubectl exec multi-gpu-pod -- nvidia-smi -L` 输出示例：

```text
GPU 0: Tesla V100-SXM2-32GB (UUID: GPU-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx)
GPU 1: Tesla V100-SXM2-32GB (UUID: GPU-yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy)
```

`kubectl exec multi-gpu-pod -- nvidia-smi --query-gpu=index,memory.total --format=csv` 输出示例：

```text
index, memory.total [MiB]
0, 10240 MiB
1, 10240 MiB
```

`kubectl exec multi-gpu-pod -- nvidia-smi topo -m` 输出示例（NVLink 互联）：

```text
        GPU0    GPU1    NIC0    CPU Affinity    NUMA Affinity
GPU0     X      NV      SYS     0-15            N/A
GPU1     NV      X      SYS     0-15            N/A
```

验证维度：

| 维度 | 命令 | 预期 |
|------|------|------|
| 集群 QGPU 状态 | `DescribeClusters`（`QGPUShareEnable`） | `true` |
| 节点 GPU 数量 | `kubectl describe node <GpuNode> \| grep nvidia.com/gpu` | `>= 2` |
| Pod 调度状态 | `kubectl get pod -o wide` | `Running`，NODE 为 GPU 节点 |
| 多卡可见性 | `kubectl exec -- nvidia-smi -L` | 输出多行 GPU |
| 显存分配 | `kubectl exec -- nvidia-smi --query-gpu=index,memory.total` | 每张约 `qgpu-memory / GPU数` |
| 互联拓扑 | `kubectl exec -- nvidia-smi topo -m` | NV 列为 NV（NVLink） |
| 应用日志 | `kubectl logs multi-gpu-pod` | 无异常 |

#### kubectl 不可达时的替代验证

当 kubectl 因公网端点被 CAM 策略禁止创建或内网端点不可达而无法连接集群时，可使用以下 tccli 命令验证 qGPU 组件状态：

```bash
# 检查 qgpu 组件状态
tccli tke DescribeAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName qgpu
# expected: Phase 为 Succeeded，AddonVersion >= 1.0.0
```

预期输出：

```json
{
    "AddonName": "qgpu",
    "AddonVersion": "1.0.0",
    "Phase": "Succeeded"
}
```

> 此替代方案仅验证 qGPU 组件是否就绪，无法验证 Pod 调度和 GPU 拓扑。完整的 GPU 多卡验证仍需 kubectl 接入集群数据面。kubectl 不可达的排查见 [排障](#排障) 中"命令返回错误"表格的 DNS 解析失败行。

## 清理

> **⚠️ 警告**：删除 Pod 会终止运行中的 GPU 任务。确认 Pod 内无未保存的计算结果后再执行删除。
>
> **💰 计费提示**：删除 Pod 不影响 GPU 节点计费。节点 CVM 持续计费，如需停止计费需删除或缩容节点池。

### 数据面

#### 1. 清理前状态检查

```bash
# 确认待删除的 Pod 名称和所在节点
kubectl get pod multi-gpu-pod -o wide
# expected: 确认 POD_NAME 和 NODE，确认是测试 Pod
```

`kubectl get pod multi-gpu-pod -o wide` 输出示例：

```text
NAME             READY   STATUS    RESTARTS   AGE   IP            NODE          NOMINATED NODE
multi-gpu-pod    1/1     Running   0          10m   10.0.16.10    gpu-node-01   <none>
```

#### 2. 删除多卡 Pod

```bash
kubectl delete pod multi-gpu-pod
# expected: pod "multi-gpu-pod" deleted
```

预期输出：

```text
pod "multi-gpu-pod" deleted
```

#### 3. 验证 Pod 已删除

```bash
kubectl get pod multi-gpu-pod
# expected: Error from server (NotFound) 或 No resources found
```

预期输出：

```text
Error from server (NotFound): pods "multi-gpu-pod" not found
```

### 控制面（tccli）

多卡互联页面无控制面资源需清理（集群和 QGPU 配置由"使用 qGPU"页面管理，不在此页面创建或销毁）。如需关闭集群 QGPU 功能，参考 [使用 qGPU](../使用%20qGPU/tccli%20操作.md)。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl apply` 后 Pod 一直 Pending | `kubectl describe pod multi-gpu-pod` 查看 Events，关注 `FailedScheduling` 原因 | 节点可用 `qgpu-core` 或 `qgpu-memory` 不足，或节点 GPU 数量 < 所需张数（如 `qgpu-core: 200` 需 2 张 GPU，但节点仅 1 张） | 降低 `qgpu-core` 值至节点可承载范围；或 `kubectl get nodes -l accelerator=nvidia` 找到 GPU 数量更多的节点并通过 `nodeName` 指定 |
| `kubectl describe node` 无 `nvidia.com/gpu` 或 `qgpu-core` 资源 | `tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'` 检查 `QGPUShareEnable` | 集群未启用 QGPU 或 qgpu 组件未安装/未就绪 | 参考 [使用 qGPU](../使用%20qGPU/tccli%20操作.md) 启用 QGPU 并安装 qgpu 组件；安装后等待 DaemonSet 就绪 |
| `kubectl exec -- nvidia-smi` 返回 `command not found` | `kubectl exec multi-gpu-pod -- ls /usr/local/nvidia/bin` | Pod 镜像不含 NVIDIA 驱动工具（如使用非 CUDA 基础镜像） | 使用含 CUDA 工具链的镜像（如 `nvidia/cuda:11.0-base`） |
| `nvidia-smi` 只显示 1 张 GPU | `kubectl describe pod multi-gpu-pod` 确认 `qgpu-core` 值；`kubectl describe node <GpuNode> \| grep nvidia.com/gpu` 确认节点 GPU 数 | `qgpu-core` 值 <= 100（单卡模式），或节点实际只有 1 张 GPU 但 qgpu-core > 100 导致仅分配了可用部分 | 确保 `qgpu-core > 100` 且节点 GPU 数 >= 2。如节点仅 1 张 GPU，无法使用多卡互联 |
| `CreateClusterEndpoint --IsExtranet true` 返回 `InvalidParameter.Param` | 检查错误信息中的 condition `tke:clusterExtranetEndpoint=true` 和 `strategyId:240463971`；`tccli tke DescribeClusterEndpointStatus --region <Region> --ClusterId <ClusterId> --IsExtranet true` 查看当前端点状态（预期 `Status: "Deleted"`） | CAM 策略 `strategyId:240463971` 以 `effect: deny` + condition `tke:clusterExtranetEndpoint=true` 硬拒绝公网端点创建。此为组织级安全管控策略，非资源配额或权限配置问题。RequestId: `b5763edf-85d3-40d0-a1e7-f3074eb4c5a1` | 联系 CAM 管理员在策略 `strategyId:240463971` 中移除或修改 `tke:clusterExtranetEndpoint=true` 的 deny 规则；或改用内网端点（`--IsExtranet false`）通过 IOA/VPN/专线接入集群 VPC |
| `kubectl describe node` 无 `qgpu-core`/`qgpu-memory` 资源，Pod 一直 Pending 提示 `Insufficient tke.cloud.tencent.com/qgpu-core` | `tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'` 检查 `QGPUShareEnable` 字段 | `QGPUShareEnable` 为 `false`，集群未启用 QGPU。节点不会注册 `tke.cloud.tencent.com/qgpu-core` 和 `tke.cloud.tencent.com/qgpu-memory` 扩展资源 | 在控制台或通过 `ModifyClusterAttribute` API 启用 QGPU 功能，确保 qgpu 组件已安装且 Phase 为 Succeeded。参考 [使用 qGPU](../使用%20qGPU/tccli%20操作.md) |
| `kubectl cluster-info` 返回 `dial tcp: lookup <ClusterDomain>: no such host` | `tccli tke DescribeClusterEndpoints --region <Region> --ClusterId <ClusterId>` 确认 `ClusterExternalEndpoint`（预期为空）和 `ClusterIntranetEndpoint`；`tccli tke DescribeClusterEndpointStatus --region <Region> --ClusterId <ClusterId> --IsExtranet true` 确认公网端点状态（预期 `Status: "Deleted"`） | 集群仅内网端点（如 `172.24.0.12`），公网端点被 CAM 策略禁止创建或已删除。集群域名在当前网络环境下无法解析，内网端点不可达 | (1) 通过 IOA/VPN/专线接入集群 VPC 内网；(2) 在同 VPC 内创建 CVM 跳板机执行 kubectl 命令；(3) 联系 CAM 管理员修改策略 `strategyId:240463971` 的 deny 规则以允许创建公网端点 |

### Pod 运行异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Pod Running 但 `nvidia-smi` 显存与预期不符 | `kubectl exec multi-gpu-pod -- nvidia-smi --query-gpu=index,memory.total --format=csv` 查看每张 GPU 显存 | `qgpu-memory` 设置后按 GPU 数均分。如设 20GiB + 2 张 GPU → 每张 10GiB，非每张 20GiB | 确认理解多卡显存分配规则：总显存均分到每张 GPU。如需每张 20GiB，设 `qgpu-memory: "40"`（2 张 × 20GiB） |
| 多卡性能未达预期 | `kubectl exec multi-gpu-pod -- nvidia-smi topo -m` 查看 GPU 间拓扑 | GPU 间为 PCIe 互联而非 NVLink，跨卡通信带宽受限 | 检查节点硬件规格，选择 NVLink 互联的 GPU 机型；PCIe 互联场景下多卡加速效果有限 |
| Pod 日志报 CUDA error | `kubectl logs multi-gpu-pod` 查看错误详情 | 应用初始化时检测到的 GPU 数量或显存不足，或 CUDA 版本与驱动不匹配 | 确认 `nvidia-smi` 显示的 CUDA Version 与应用要求的版本一致；调整 `qgpu-core` 和 `qgpu-memory` 满足应用最低需求 |

## 下一步

- [使用 qGPU](https://cloud.tencent.com/document/product/457/65734) — qGPU 基础概念与单卡模式使用
- [GPU 监控指标获取](https://cloud.tencent.com/document/product/457/90912) — GPU 使用率、显存、温度等监控指标
- [GPU 故障检测与自愈](https://cloud.tencent.com/document/product/457/127668) — GPU 故障自动检测与隔离
- [TKE 集群管理](https://cloud.tencent.com/document/product/457/31013) — 集群类型与管理模式
- [TKE API 概览](https://cloud.tencent.com/document/product/457/31853) — 完整 API 列表

## 控制台替代

[TKE 控制台 → 集群 → 工作负载 → Deployment/Pod](https://console.cloud.tencent.com/tke2/workload)：在 Pod 模板的资源限制中填写 `tke.cloud.tencent.com/qgpu-core`（> 100 触发多卡）和 `tke.cloud.tencent.com/qgpu-memory`。
