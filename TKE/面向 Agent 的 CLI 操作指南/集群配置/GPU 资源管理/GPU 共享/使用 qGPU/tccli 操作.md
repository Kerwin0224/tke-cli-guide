# 使用 qGPU

> 对照官方：[使用 qGPU](https://cloud.tencent.com/document/product/457/65734) · page_id `65734` · tccli ≥3.1.107 · API 2018-05-25

## 概述

qGPU 是 TKE 针对原生节点推出的 GPU 容器虚拟化技术，将一张物理 GPU 虚拟化为多张 vGPU，多个 Pod 可共享同一张 GPU 并实现算力与显存精细隔离。qGPU 仅针对 TKE 原生节点开放，支持 Volta（V100）、Turing（T4）、Ampere（A100/A10）等架构。

使用 qGPU 分两步：

1. **安装 qgpu 组件**：`InstallAddon --AddonName qgpu`，部署 qgpu-manager（DaemonSet）和 qgpu-scheduler（Deployment）。
2. **Pod 声明 vGPU 资源**：通过 `resources.limits` 中 `tke.cloud.tencent.com/qgpu-core` 和 `tke.cloud.tencent.com/qgpu-memory` 申请算力与显存。

> **关于 QGPUShareEnable**：官方文档建议创建集群时在 `ClusterAdvancedSettings` 中设 `QGPUShareEnable: true`。但在实际验证中发现，`InstallAddon qgpu` 在未启用 QGPUShareEnable 的集群上也安装成功（Phase: `Succeeded`），该开关并非安装 qgpu 组件的硬性前提。不过，QGPU 开关仅创建集群时可设置，若后续需要使用 QGPU 特性的完整能力（如节点自动注入 qGPU 资源），建议创建集群时开启。存量集群未启用可直接尝试 InstallAddon。

| 方案 | 适用场景 | 算力隔离 | 显存隔离 |
|------|---------|---------|---------|
| qGPU 共享 | 单卡多 Pod 共享，需要算力+显存隔离 | 支持（百分比切分） | 支持（GB 切分） |
| 整卡独占 | 单 Pod 独占一卡 | 不需切分 | 独占 |

**本文覆盖路径**：tccli（控制面：安装/卸载组件）+ kubectl（数据面：部署使用 vGPU 的 Pod）。

## 前置条件

- [环境准备](../../../../环境准备.md) <!-- 发布时需将相对 .md 链接转换为平台链接：iWiki `/p/<docid>` / 写写 `/document/<nodeId>` -->

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置，region 为目标地域

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:DescribeClusters, tke:DescribeClusterEndpoints, tke:InstallAddon
#    tke:DescribeAddon, tke:DescribeAddonValues, tke:DeleteAddon
#    tke:DescribeClusterKubeconfig, tke:DescribeClusterNodePools
# 验证：执行 DescribeClusters 确认 TKE 权限
tccli tke DescribeClusters --region <Region>
# expected: exit 0，返回集群列表（可为空）
```

预期输出：

```json
{
    "Clusters": [
        {
            "ClusterId": "cls-example",
            "ClusterName": "cluster-example",
            "ClusterStatus": "Running",
            "ClusterType": "MANAGED_CLUSTER",
            "ClusterVersion": "1.32.2",
            "ClusterNetworkSettings": {
                "VpcId": "vpc-example",
                "ClusterCIDR": "10.0.0.0/16",
                "ServiceCIDR": "10.1.0.0/20"
            },
            "ClusterAdvancedSettings": {}
        }
    ],
    "TotalCount": 1
}
```

### 资源检查

```bash
# 4. 查询目标集群状态
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["<ClusterId>"]'
# expected: TotalCount >= 1, ClusterStatus: "Running"
```

预期输出：

```json
{
    "Clusters": [
        {
            "ClusterId": "cls-example",
            "ClusterName": "cluster-example",
            "ClusterStatus": "Running",
            "ClusterType": "MANAGED_CLUSTER",
            "ClusterVersion": "1.32.2",
            "ClusterNetworkSettings": {
                "VpcId": "vpc-example",
                "ClusterCIDR": "10.0.0.0/16",
                "ServiceCIDR": "10.1.0.0/20"
            },
            "ClusterAdvancedSettings": {}
        }
    ],
    "TotalCount": 1
}
```

```bash
# 5. 查看集群端点状态（后续 kubectl 操作需要端点可达）
tccli tke DescribeClusterEndpoints --region <Region> \
    --ClusterId <ClusterId>
# expected: ClusterDomain 非空，ClusterExternalEndpoint 或 ClusterIntranetEndpoint 至少一个非空
```

预期输出：

```json
{
    "CertificationAuthority": "-----BEGIN CERTIFICATE-----\n...",
    "ClusterExternalEndpoint": "",
    "ClusterIntranetEndpoint": "<INTERNAL_IP>",
    "ClusterDomain": "cls-example.ccs.tencent-cloud.com",
    "ClusterIntranetSubnetId": "subnet-example",
    "SecurityGroup": ""
}
```

> **注意**：内网端点仅 VPC 内可达。非 VPC 环境需公网端点（`IsExtranet=true`）或通过 IOA/VPN/专线/同 VPC CVM 跳板连接。

```bash
# 6. 查询集群节点池（确认是否有 GPU 节点）
tccli tke DescribeClusterNodePools --region <Region> \
    --ClusterId <ClusterId>
# expected: 返回节点池列表；若空列表，集群暂无 GPU 节点，qGPU 组件仍可安装（组件部署后等待 GPU 节点加入即可）
```

预期输出：

```json
{
    "NodePoolSet": [],
    "TotalCount": 0
}
```

### 版本与规格选择

- K8s 版本：qGPU 要求 >= 1.14.x。当前集群版本 `1.32.2` 满足。
- 节点要求：原生节点，GPU 架构为 V100 / T4 / A100 / A10，驱动版本 450 / 470 / 515 / 525 / 535 / 550 / 570 系列。
- 运行时：`containerd`（托管集群强制）。

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看集群列表及状态 | `DescribeClusters` | 是 |
| 查看集群端点 | `DescribeClusterEndpoints` | 是 |
| 查看组件管理 | `DescribeAddon --AddonName qgpu` | 是 |
| 安装 qgpu 组件 | `InstallAddon --AddonName qgpu` | 否 |
| 卸载 qgpu 组件 | `DeleteAddon --AddonName qgpu` | 是 |
| 部署使用 vGPU 的 Pod | `kubectl apply -f` | 否 |

## 关键字段说明

`InstallAddon` 的主要参数：

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `ClusterId` | String | 是 | 已存在的集群 ID，格式 `cls-xxxxxxxx`。`DescribeClusters` 获取 | 集群不存在 → `InvalidParameter.ClusterId` |
| `AddonName` | String | 是 | 固定值 `qgpu`（全小写）。部署 qgpu-manager（DaemonSet）和 qgpu-scheduler（Deployment） | 填 `QGPU`、`QGPUShare` 等变体 → `InvalidParameter.AddonName` |
| `AddonVersion` | String | 否 | 组件版本号，如 `1.1.3`。不传默认安装最新版本。`DescribeAddonValues` 查询可用版本。**注意**：版本号以实际 `DescribeAddonValues` 返回为准，本文示例版本可能已过期 | 版本不存在 → addon version not found |
| `RawValues` | String | 否 | Base64 编码的配置 JSON，覆盖默认组件配置。`DescribeAddonValues` 获取默认配置 | 配置格式错误 → 组件安装失败 |

Pod 资源声明关键字段：

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `tke.cloud.tencent.com/qgpu-core` | String | 是（Pod limits） | 算力百分比，整数，5-100（单卡），>100 为多卡（如 200 = 2 卡），必须是 5 的倍数 | 非 5 的倍数 → 调度失败；单卡超 100 → 调度失败 |
| `tke.cloud.tencent.com/qgpu-memory` | String | 是（Pod limits） | 显存 GB，整数，最小 1 | 非整数或超节点显存 → 调度失败 |

## 操作步骤

### 步骤 1：验证集群状态

```bash
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["<ClusterId>"]'
# expected: exit 0, ClusterStatus: "Running"
```

预期输出：

```json
{
    "Clusters": [
        {
            "ClusterId": "cls-example",
            "ClusterName": "cluster-example",
            "ClusterStatus": "Running",
            "ClusterType": "MANAGED_CLUSTER",
            "ClusterVersion": "1.32.2",
            "ClusterNetworkSettings": {
                "VpcId": "vpc-example",
                "ClusterCIDR": "10.0.0.0/16",
                "ServiceCIDR": "10.1.0.0/20"
            },
            "ClusterAdvancedSettings": {}
        }
    ],
    "TotalCount": 1
}
```

| 字段 | 值 | 含义 |
|------|-----|------|
| `ClusterStatus` | `Running` | 集群就绪，可执行安装操作 |
| `ClusterType` | `MANAGED_CLUSTER` | 托管集群，qGPU 支持 |
| `ClusterAdvancedSettings` | `{}`（空） | 未配置 QGPUShareEnable 等高级设置 |

### 步骤 2：安装 qGPU 组件

#### 2.1 查询当前组件状态

```bash
tccli tke DescribeAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName qgpu
# expected: 未安装时返回错误（ResourceNotFound），已安装时返回 Phase: "Succeeded"
```

**未安装时**的预期输出：

```json
{
    "Error": {
        "Code": "ResourceNotFound",
        "Message": "get addon failed"
    },
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

**已安装时**的预期输出：

```json
{
    "Addons": [
        {
            "AddonName": "qgpu",
            "AddonVersion": "1.1.3",
            "RawValues": "eyJnbG9iYWwiOnsiY2x1c3RlciI6eyJoaWdoQXZhaWxhYmlsaXR5Ijp0cnVlfX19",
            "Phase": "Succeeded",
            "Reason": "",
            "CreateTime": "2026-06-23T08:16:09Z"
        }
    ]
}
```

#### 2.2 查询组件可用配置

```bash
tccli tke DescribeAddonValues --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName qgpu
# expected: 返回组件默认配置和可用参数
```

预期输出：

```json
{
    "Values": "{\"config\":{\"priority\":\"binpack\"},\"global\":{\"cluster\":{\"id\":\"cls-example\",\"kubeversion\":\"1.32.2-tke.11\"},\"image\":{\"host\":\"ccr.ccs.tencentyun.com\"}}}",
    "DefaultValues": "{\"config\":{\"priority\":\"binpack\"},\"global\":{\"cluster\":{\"id\":\"cls-example\",\"kubeversion\":\"1.32.2-tke.11\"},\"image\":{\"host\":\"ccr.ccs.tencentyun.com\"}}}"
}
```

> **注意**：API 返回的 `Values` 和 `DefaultValues` 为 JSON 编码的**字符串**（非展开的 JSON 对象）。如需解析，使用 `jq '.Values | fromjson'` 或等价方式。

| 配置项 | 值 | 说明 |
|--------|-----|------|
| `config.priority` | `binpack` | qGPU 默认调度策略为装箱（binpack），优先将 vGPU 调度到已有负载的 GPU 节点上 |
| `global.cluster.highAvailability` | `true` | 高可用模式 |
| `global.image.host` | `ccr.ccs.tencentyun.com` | 组件镜像仓库地址 |

#### 2.3 选择依据

- **AddonName**：固定为 `qgpu`（全小写）。qgpu 组件部署 qgpu-manager（DaemonSet，每个 GPU 节点自动运行一个 Pod）和 qgpu-scheduler（Deployment，单副本）。
- **AddonVersion**：不传时默认安装最新版本。如需指定版本，从 `DescribeAddonValues` 输出中确认可用版本号。
- **安装方式选择**：以下 2.4（最简安装）和 2.5（指定版本安装）为**二选一**关系，请根据需要择一执行。若 qGPU 组件已安装（`DescribeAddon` 返回 Phase: `Succeeded`），重复执行 `InstallAddon` 会报 `UnknownParameter: check addon is exist`，需先 `DeleteAddon` 卸载后再重新安装。
- **GPU 节点**：qGPU 组件可在没有 GPU 节点的集群上安装成功（Phase: `Succeeded`）。组件本身不依赖 GPU 硬件，GPU 节点加入后 DaemonSet 自动在新节点上运行。
- **QGPUShareEnable**：官方文档建议创建集群时启用，但实测在 `ClusterAdvancedSettings` 为空的集群上 `InstallAddon qgpu` 仍安装成功。如果仅是实验性安装组件，可直接 InstallAddon。
- **安装超时**：首次安装可能超时（尤其是集群无 GPU 节点时），Phase 变为 `InstallFailed`，Reason 为 `timed out waiting for the condition`。这是正常行为——使用 `DeleteAddon` 删除失败的组件后重新 `InstallAddon` 即可成功。

#### 2.4 安装命令（最简方式 — 不指定版本）

```bash
tccli tke InstallAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName qgpu
# expected: exit 0，返回 RequestId
```

预期输出：

```json
{
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

#### 2.5 安装命令（指定版本）

```bash
tccli tke InstallAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName qgpu \
    --AddonVersion 1.1.3
# expected: exit 0，返回 RequestId
```

预期输出：

```json
{
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<ClusterId>` | 集群 ID | 格式 `cls-xxxxxxxx` | `tccli tke DescribeClusters --region <Region>` |
| `<Region>` | 地域 | 如 `ap-guangzhou` | `tccli configure list` |

#### 2.6 轮询组件安装状态

安装是异步操作。轮询直到组件就绪：

```bash
tccli tke DescribeAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName qgpu
# expected: Phase: "Succeeded"
```

安装过程中 Phase 变化：`Installing` → `Succeeded`（或 `InstallFailed`）。

**安装成功**时的预期输出：

```json
{
    "Addons": [
        {
            "AddonName": "qgpu",
            "AddonVersion": "1.1.3",
            "RawValues": "eyJnbG9iYWwiOnsiY2x1c3RlciI6eyJoaWdoQXZhaWxhYmlsaXR5Ijp0cnVlfX19",
            "Phase": "Succeeded",
            "Reason": "",
            "CreateTime": "2026-06-23T08:16:09Z"
        }
    ]
}
```

> **注意**：API 返回的 Phase 值为 `"Succeeded"`，不是 `"running"`。文档常误写为 `running`，正确值以 API 实际返回为准。

### 步骤 3：部署使用 vGPU 的 Pod

> **注意**：以下 kubectl 命令需在集群端点可达的环境下执行。内网端点仅 VPC 内可达；公网端点需 CAM 策略允许。非 VPC 环境需通过 IOA/VPN/专线或同 VPC CVM 跳板连接。如 kubectl 不可达，详见 [排障](#排障) 中的连接问题排查。

#### 3.1 获取 kubeconfig

```bash
tccli tke DescribeClusterKubeconfig --region <Region> \
    --ClusterId <ClusterId> \
    --IsExtranet false \
    | jq -r '.Kubeconfig' > /tmp/kubeconfig
# expected: exit 0，Kubeconfig 字段非空

export KUBECONFIG=/tmp/kubeconfig
```

预期输出：

```json
{
    "Kubeconfig": "apiVersion: v1\nclusters:\n- cluster:\n    certificate-authority-data: LS0t...\n    server: https://<INTERNAL_IP>\n..."
}
```

**备选方案**（获取集群安全信息，含用户名/密码认证）：

```bash
tccli tke DescribeClusterSecurity --region <Region> \
    --ClusterId <ClusterId>
# expected: 返回 UserName、Password、Domain、PgwEndpoint 和 Kubeconfig
```

预期输出：

```json
{
    "UserName": "admin",
    "Password": "<PASSWORD>",
    "Domain": "cls-example.ccs.tencent-cloud.com",
    "PgwEndpoint": "<INTERNAL_IP>",
    "Kubeconfig": "apiVersion: v1\n..."
}
```

#### 3.2 验证 kubectl 连通性

```bash
kubectl cluster-info
# expected: Kubernetes control plane is running at https://...
```

预期输出：

```text
Kubernetes control plane is running at https://<INTERNAL_IP>
CoreDNS is running at https://<INTERNAL_IP>/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
```

```bash
kubectl get ns
# expected: 返回 default 等系统命名空间
```

预期输出：

```text
NAME              STATUS   AGE
default           Active   7d
kube-node-lease   Active   7d
kube-public       Active   7d
kube-system       Active   7d
```

#### 3.3 部署 vGPU Pod

**选择依据**：

- **qgpu-core**：算力百分比，单卡范围 5-100（步长 5，即 5、10、15...100）。`100` 表示独占单卡全部算力。`>100` 表示跨卡（如 `200` = 2 张卡），需集群有多卡节点。必须是 5 的倍数。
- **qgpu-memory**：显存 GB，整数，最小 1。分配粒度为 1GB。
- **requests 与 limits 一致**：资源声明时 requests 必须与 limits 一致。
- **节点标签**：Pod 必须调度到带 qGPU 标签的 GPU 节点。原生节点需设置 `tke.cloud.tencent.com/qgpu-schedule-policy` 标签开启共享。
- **单卡最多 16 个 qGPU 设备**：即一张 GPU 卡最多供 16 个 Pod 共享。

`qgpu-pod-minimal.yaml`（最简配置 — 仅必填字段）：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: qgpu-demo
  namespace: default
spec:
  containers:
  - name: cuda-app
    image: nvidia/cuda:11.0-base
    resources:
      limits:
        tke.cloud.tencent.com/qgpu-core: "30"
        tke.cloud.tencent.com/qgpu-memory: "5"
      requests:
        tke.cloud.tencent.com/qgpu-core: "30"
        tke.cloud.tencent.com/qgpu-memory: "5"
```

`qgpu-pod-enhanced.yaml`（增强配置 — 含节点亲和性、容忍和多卡声明）：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: qgpu-demo-enhanced
  namespace: default
spec:
  nodeSelector:
    tke.cloud.tencent.com/qgpu-schedule-policy: "true"
  tolerations:
  - key: tke.cloud.tencent.com/qgpu
    operator: Exists
    effect: NoSchedule
  containers:
  - name: cuda-app
    image: nvidia/cuda:11.0-base
    command: ["nvidia-smi", "-L"]
    resources:
      limits:
        tke.cloud.tencent.com/qgpu-core: "200"
        tke.cloud.tencent.com/qgpu-memory: "10"
      requests:
        tke.cloud.tencent.com/qgpu-core: "200"
        tke.cloud.tencent.com/qgpu-memory: "10"
```

| 文件 | 用途 | 适用场景 |
|------|------|---------|
| `qgpu-pod-minimal.yaml` | 最简配置，仅声明 qgpu-core 和 qgpu-memory | 快速验证 qGPU 功能是否可用 |
| `qgpu-pod-enhanced.yaml` | 含 nodeSelector、tolerations 的增强配置，声明多卡资源（200 = 2 卡） | 生产环境，确保 Pod 调度到正确的 GPU 节点 |

```bash
# 最简安装
kubectl apply -f qgpu-pod-minimal.yaml
# expected: pod/qgpu-demo created

# 增强配置（含节点亲和性、容忍、多卡）
kubectl apply -f qgpu-pod-enhanced.yaml
# expected: pod/qgpu-demo-enhanced created
```

```bash
kubectl get pod qgpu-demo
# expected: STATUS: Running
```

预期输出：

```text
NAME        READY   STATUS    RESTARTS   AGE
qgpu-demo   1/1     Running   0          30s
```

## 验证

### 控制面（tccli）

```bash
# 验证 qgpu 组件已安装并运行
tccli tke DescribeAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName qgpu
# expected: Phase: "Succeeded"
```

预期输出：

```json
{
    "Addons": [
        {
            "AddonName": "qgpu",
            "AddonVersion": "1.1.3",
            "RawValues": "eyJnbG9iYWwiOnsiY2x1c3RlciI6eyJoaWdoQXZhaWxhYmlsaXR5Ijp0cnVlfX19",
            "Phase": "Succeeded",
            "Reason": "",
            "CreateTime": "2026-06-23T08:16:09Z"
        }
    ]
}
```

### 数据面

> **注意**：以下 kubectl 命令需在集群端点可达的环境下执行。

```bash
# 获取 GPU 节点名称
kubectl get nodes -l nvidia.com/gpu.present=true -o name | head -1 | cut -d/ -f2
# expected: 返回 GPU 节点名称，如 gpu-node-xxxxxxxx
# 若返回空，说明当前集群无 GPU 节点，跳过后续节点验证步骤
```

```bash
# 确认 GPU 节点已注册 qGPU 扩展资源
kubectl describe node <NodeName> | grep qgpu
# expected: 返回 tke.cloud.tencent.com/qgpu-core 和 tke.cloud.tencent.com/qgpu-memory 的容量
```

预期输出：

```text
  tke.cloud.tencent.com/qgpu-core:      100
  tke.cloud.tencent.com/qgpu-memory:    80
```

```bash
# 确认 qgpu-manager 和 qgpu-scheduler 运行
kubectl get pods -n kube-system | grep qgpu
# expected: qgpu-manager-* 和 qgpu-scheduler-* 均为 Running
```

预期输出：

```text
qgpu-manager-xxxxx     1/1     Running   0          5m
qgpu-scheduler-xxxxx   1/1     Running   0          5m
```

```bash
# 确认 Pod 已分配 vGPU
kubectl describe pod qgpu-demo | grep -A5 "Limits"
# expected: 显示 tke.cloud.tencent.com/qgpu-core: 30 和 qgpu-memory: 5
```

预期输出：

```text
    Limits:
      tke.cloud.tencent.com/qgpu-core: 30
      tke.cloud.tencent.com/qgpu-memory: 5
```

```bash
# 进入 Pod 验证 GPU 可见性
kubectl exec qgpu-demo -- nvidia-smi
# expected: 显示分配的 GPU 设备及其算力/显存限制
```

预期输出：

```text
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 550.127.08   Driver Version: 550.127.08   CUDA Version: 12.4     |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|===============================+======================+======================|
|   0  Tesla T4            On   | 00000000:00:08.0 Off |                    0 |
| N/A   35C    P8     9W /  70W |   1024MiB /  5120MiB |      0%      Default |
+-------------------------------+----------------------+----------------------+

+-----------------------------------------------------------------------------+
| Processes:                                                                  |
|  GPU   GI   CI        PID   Type   Process name                  GPU Memory |
|        ID   ID                                                   Usage      |
|=============================================================================|
+-----------------------------------------------------------------------------+
```

> **注意**：qGPU 隔离在容器层生效，`nvidia-smi` 在容器内执行时，显示的显存容量为容器的 vGPU 配额（上例 5120MiB = 5GB），而非物理 GPU 的全部显存。如果容器内 `nvidia-smi` 显示物理卡全量显存，以 `kubectl describe pod` 的 `Limits` 字段为准，qGPU 隔离在调度层生效。

### 验证维度总览

| 维度 | 命令 | 预期 |
|------|------|------|
| 组件状态 | `DescribeAddon --AddonName qgpu → .Addons[0].Phase` | `Succeeded` |
| 节点资源 | `kubectl describe node <NodeName> \| grep qgpu` | 显示 qgpu-core、qgpu-memory 扩展资源 |
| 组件 Pod | `kubectl get pods -n kube-system \| grep qgpu` | qgpu-manager、qgpu-scheduler 均 Running |
| Pod 资源分配 | `kubectl describe pod qgpu-demo \| grep -A5 Limits` | qgpu-core、qgpu-memory 与 YAML 声明一致 |
| GPU 可见性 | `kubectl exec qgpu-demo -- nvidia-smi` | 显示分配的 GPU 设备 |

## 清理

> **警告**：`DeleteAddon` 卸载 qgpu 组件后，所有使用 qGPU 资源的 Pod 将因设备插件移除而进入异常状态（ContainerCreating 或 Error）。卸载前请先删除或迁移依赖 qGPU 的工作负载。卸载后组件进入 Terminating 阶段约 30 秒，此期间无法重新安装（报 UnknownParameter: check addon is exist）。

### 数据面

> **注意**：以下 kubectl 命令需在集群端点可达的环境下执行。

```bash
# 1. 清理前状态检查 — 确认是待删除的目标 Pod
kubectl get pod qgpu-demo
# expected: STATUS: Running
```

预期输出：

```text
NAME        READY   STATUS    RESTARTS   AGE
qgpu-demo   1/1     Running   0          5m
```

```bash
# 2. 删除使用 vGPU 的 Pod
kubectl delete pod qgpu-demo
# expected: pod "qgpu-demo" deleted
```

预期输出：

```text
pod "qgpu-demo" deleted
```

```bash
# 3. 验证 Pod 已删除
kubectl get pod qgpu-demo 2>&1
# expected: Error from server (NotFound)
```

预期输出：

```text
Error from server (NotFound): pods "qgpu-demo" not found
```

### 控制面（tccli）

```bash
# 4. 清理前状态检查 — 确认 qgpu 组件已安装
tccli tke DescribeAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName qgpu
# 确认 AddonName: "qgpu", Phase: "Succeeded"
```

预期输出：

```json
{
    "Addons": [
        {
            "AddonName": "qgpu",
            "AddonVersion": "1.1.3",
            "Phase": "Succeeded",
            "Reason": ""
        }
    ]
}
```

```bash
# 5. 卸载 qgpu 组件
tccli tke DeleteAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName qgpu
# expected: exit 0，返回 RequestId
```

预期输出：

```json
{
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

```bash
# 6. 验证组件已卸载
tccli tke DescribeAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName qgpu
# expected: ResourceNotFound 或 Addons 列表中不含 qgpu
```

**卸载中**（Phase: `Terminating`，约持续 30 秒）的预期输出：

```json
{
    "Addons": [
        {
            "AddonName": "qgpu",
            "AddonVersion": "1.1.3",
            "Phase": "Terminating",
            "Reason": ""
        }
    ]
}
```

**卸载完成**后的预期输出：

```json
{
    "Error": {
        "Code": "ResourceNotFound",
        "Message": "get addon failed"
    },
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

> **注意**：卸载后组件进入 Terminating 约 30 秒。Terminating 期间 `DescribeAddon` 返回 `Phase: "Terminating"`，此时 `InstallAddon` 会报 `UnknownParameter: check addon is exist`。等待 Phase 为空（返回 `ResourceNotFound`）后再重新安装。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `InstallAddon` 返回 `InvalidParameter.AddonName` | 检查 `--AddonName` 值 | 填了 `QGPU`、`QGPUShare` 等变体，正确值为全小写 `qgpu` | 使用 `--AddonName qgpu`（全小写） |
| `InstallAddon` 返回 "addon version not found" | `tccli tke DescribeAddonValues --region <Region> --ClusterId <ClusterId> --AddonName qgpu` 查看可用版本 | 指定的 `AddonVersion` 不存在 | 从 `DescribeAddonValues` 输出取正确版本号，或不传 `AddonVersion` 使用最新版 |
| `InstallAddon` 返回 `ResourceNotFound` | `tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'` | 集群不存在或不在指定地域 | 确认 ClusterId 和 Region 正确 |
| `DeleteAddon` 后立即 `InstallAddon` 返回 `UnknownParameter` | `tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName qgpu` 查看 Phase | `check addon is exist` — 组件处于 Terminating 阶段，尚未完全清理 | 等待约 30 秒直到 `DescribeAddon` 返回空（Terminating 完成），再执行 `InstallAddon` |
| `DescribeAddon` 返回 `ResourceNotFound` | `tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName qgpu` | qgpu 组件未安装或已卸载 | 执行 `InstallAddon --AddonName qgpu` 安装 |
| kubectl 返回 `http: server gave HTTP response to HTTPS client` | `curl -k -s http://<INTERNAL_IP>/api` 检查端点 | 内网端点被代理/堡垒机拦截，HTTPS 连接被降级为 HTTP | 内网端点仅 VPC 内可达。非 VPC 环境需通过以下方式之一获得端点可达性：(1) 使用公网端点（如 CAM 策略允许）；(2) 通过 IOA/VPN/专线连接内网端点；(3) 在同 VPC CVM 上执行 kubectl 命令；(4) 使用 `DescribeClusterSecurity` 获取用户名/密码通过其他方式认证 |
| `CreateClusterEndpoint`（公网端点）返回 `InvalidParameter.Param` | 检查 CAM 策略限制 | CAM 策略 `strategyId:240463971` 以 `tke:clusterExtranetEndpoint=true` 条件硬拒绝公网端点创建 | 使用内网端点 `IsExtranet=false` 作为替代。此为组织级 CAM 策略限制，自建安全组也无法绕过。如需公网端点，联系 CAM 管理员解除策略限制 |
| `kubectl apply` 报 `Insufficient nvidia.com/gpu` | `kubectl describe pod <POD_NAME>` 查看 Events | GPU 节点未注册 qGPU 扩展资源，或节点未开启 qGPU 共享 | 确认节点 Label：`kubectl get node -l tke.cloud.tencent.com/qgpu-schedule-policy`；确认 qgpu-manager DaemonSet 已在节点运行：`kubectl get pods -n kube-system \| grep qgpu-manager` |
| `kubectl apply` 报 `requested device tke.cloud.tencent.com/qgpu-core not found` | `kubectl describe node <NodeName> \| grep qgpu` | qgpu 组件未安装或未就绪 | 回到步骤 2 安装 qgpu 组件，确认 `DescribeAddon` 返回 `Phase: "Succeeded"` |

### 操作成功但异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `InstallAddon` 返回 RequestId 但组件安装超时 | `tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName qgpu` 查看 Phase | `Phase: "InstallFailed"`, `Reason: "timed out waiting for the condition"`。首次安装可能超时，尤其在集群无 GPU 节点时，组件 DaemonSet 等待节点但超时 | 使用 `DeleteAddon` 删除失败的组件，等待 Terminating 完成后重新 `InstallAddon`。第二次安装通常成功（组件以 Succeeded 状态完成部署，DaemonSet 和 Scheduler 已部署，等待 GPU 节点加入）。保留 RequestId 以备进一步排查 |
| qgpu 组件安装成功但 Pod 一直 ContainerCreating | `kubectl describe pod <POD_NAME>` 查看 Events；`kubectl get pods -n kube-system \| grep qgpu` 确认组件 Pod 状态 | GPU 节点未就绪、驱动未安装、或节点池未设置 qGPU 共享 Label | 确认节点为原生节点且 GPU 驱动已安装（`kubectl describe node <NodeName> \| grep nvidia`）；确认节点 Label `tke.cloud.tencent.com/qgpu-schedule-policy` 已设置 |
| 组件 Phase 为 `InstallFailed` | `tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName qgpu` 查看 Reason | Reason 含 `timed out waiting for the condition` — 通常是集群无 GPU 节点导致 DaemonSet 无法就绪 | `DeleteAddon` → 等待 Terminating 完成 → 重新 `InstallAddon`。第二次安装时组件以 `Succeeded` 状态完成部署 |
| Pod 调度成功但算力/显存不正确 | `kubectl describe pod <POD_NAME> \| grep -A5 Limits` 检查资源声明 | `qgpu-core` 非 5 的倍数，或 `qgpu-memory` 非整数 | `qgpu-core` 必须为 5 的倍数（5-100 单卡，>100 多卡），`qgpu-memory` 为整数（>=1） |
| `nvidia-smi` 显示的显存与声明不符 | `kubectl exec <POD_NAME> -- nvidia-smi` 查看 GPU 信息 | qGPU 隔离在容器层，`nvidia-smi` 可能显示物理卡全量显存（已知行为） | 以 `kubectl describe pod` 的 `Limits` 为准，qGPU 隔离在调度层生效，`nvidia-smi` 显示仅供参考 |
| `InstallAddon` 返回 RequestId 但集群 `ClusterAdvancedSettings` 为空 | `tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]' \| jq '.Clusters[0].ClusterAdvancedSettings'` | QGPUShareEnable 并非安装 qgpu 组件的硬性前提。组件可在未启用 QGPU 的集群上安装成功（Phase: Succeeded），但完整功能需 GPU 节点配合 | 如果仅是实验性安装组件，可直接 `InstallAddon`。生产使用仍建议创建集群时开启 QGPUShareEnable，且需要 GPU 节点支持 |

## 下一步

- [qGPU 概述](https://cloud.tencent.com/document/product/457/61448) — qGPU 架构、隔离策略与资源声明规则
- [qGPU 多卡互联](https://cloud.tencent.com/document/product/457/127667) — Pod 跨多张 GPU 使用 vGPU 资源
- [qGPU 离在线混部](https://cloud.tencent.com/document/product/457/103982) — 在离线混部提升 GPU 利用率
- [GPU 监控指标获取](https://cloud.tencent.com/document/product/457/102005) — GPU 使用率、显存等监控指标
- [TKE API 概览](https://cloud.tencent.com/document/product/457/31853) — 完整 API 列表

## 控制台替代

[TKE 控制台 → 集群](https://console.cloud.tencent.com/tke2/cluster)：集群详情 → 组件管理 → 安装「qgpu」组件；部署使用 vGPU 的 Pod 需通过 kubectl 或控制台 YAML 编辑器。
