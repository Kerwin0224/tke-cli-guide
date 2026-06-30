# GPU 监控指标获取

> 对照官方：[GPU 监控指标获取](https://cloud.tencent.com/document/product/457/90912) · page_id `90912`

## 概述

通过安装 `nvidia-gpu` 组件，TKE 自动采集 GPU 节点的 DCGM（Data Center GPU Manager）监控指标，包括 GPU 使用率、显存、温度、功耗、XID 错误等。

> **重要说明**：`nvidia-gpu` 组件（v1.0.5）在 K8s 1.32.2 中已内置 DCGM exporter（v1.0.21），安装 `nvidia-gpu` 后即可通过 Prometheus 查询标准 GPU 指标，**无需额外安装 `tke-monitor-agent`**（该组件在 K8s 1.32.2 下不可用）。对于更早的 K8s 版本（1.30 及以下），可能需要单独安装 `tke-monitor-agent`。

| 采集方式 | 适用场景 | 需要组件 | 数据格式 |
|---------|---------|---------|---------|
| DCGM（nvidia-gpu 内置） | 集群级监控、告警、Prometheus 查询 | `nvidia-gpu` | Prometheus metrics |
| `kubectl exec ... nvidia-smi` | 实时排查、单卡验证 | `nvidia-gpu` | 命令行文本 |

## 前置条件

- [环境准备](../../../环境准备.md)
- 已有 GPU 节点

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:InstallAddon, tke:DescribeAddon, tke:DeleteAddon
#    tke:DescribeClusters, tke:DescribeAddonValues
# 验证：
tccli tke DescribeClusters --region <Region>
# expected: exit 0，返回集群列表（可为空）

# 4. 检查 kubectl 连通性（集群无公网端点时需内网可达）
#    以下 kubectl 命令需在集群端点可达的环境下执行（内网端点需 IOA/VPN/专线或同 VPC CVM 跳板机）
tccli tke DescribeClusterEndpoints --region <Region> \
    --ClusterId <ClusterId>
# expected: ClusterExternalEndpoint 或 ClusterIntranetEndpoint 非空
# 若 ClusterExternalEndpoint 为空，使用内网端点（172.24.0.x），需 VPC 可达
```

### 资源检查

```bash
# 5. 确认 GPU 节点存在
kubectl get nodes -l accelerator=nvidia
# expected: 至少 1 个 GPU 节点

# 6. 检查 nvidia-gpu 组件是否已安装
tccli tke DescribeAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName nvidia-gpu
# expected: Phase = Succeeded 或 ResourceNotFound（未安装）
```

**预期输出**（已安装时）：

```json
{
  "Addons": [
    {
      "AddonName": "nvidia-gpu",
      "AddonVersion": "1.0.5",
      "Phase": "Succeeded",
      "Reason": "",
      "CreateTime": "2026-06-23T00:00:00Z"
    }
  ],
  "RequestId": "..."
}
```

**预期输出**（未安装时）：

```json
{
  "Error": {
    "Code": "ResourceNotFound",
    "Message": "get addon failed"
  }
}
```

```bash
# 7. 检查 nvidia-gpu 是否内置 DCGM exporter（K8s 1.32.2+）
tccli tke DescribeAddonValues --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName nvidia-gpu | jq .Values | grep exporter
# expected: 输出含 "exporter" 字段及版本号（如 "tag":"v1.0.21"），表明 DCGM 采集能力已内置
```

> 如果 exporter 字段不存在，可能是较早版本的 `nvidia-gpu`，建议升级组件版本。

| 占位符 | 说明 | 获取方式 |
|--------|------|---------|
| `<Region>` | 地域 | `tccli configure list` 或 `tccli tke DescribeRegions` |
| `<ClusterId>` | 集群 ID | `tccli tke DescribeClusters --region <Region>` |
| `<GPU_NODE>` | GPU 节点名 | `kubectl get nodes -l accelerator=nvidia` |

## 控制台与 CLI 参数映射

| 控制台操作 | tccli / kubectl 命令 | 幂等 |
|-----------|---------------------|:--:|
| 安装 GPU 驱动组件 | `InstallAddon --AddonName nvidia-gpu` | 否 |
| 查看组件状态 | `DescribeAddon` | 是 |
| 查看组件配置 | `DescribeAddonValues` | 是 |
| 查看 GPU 实时状态 | `kubectl exec GPU_POD -- nvidia-smi` | — |
| 查询 Prometheus 指标 | Prometheus 查询接口 | — |

## 操作步骤

### 步骤 1：安装 nvidia-gpu 组件

#### 选择依据

- **为什么只装 nvidia-gpu**：`nvidia-gpu` 组件 v1.0.5 已内置 DCGM exporter v1.0.21，安装后自动部署 GPU 驱动、设备插件和 DCGM 监控采集器，可直接通过 Prometheus 查询标准 GPU 指标（DCGM_FI_DEV_GPU_UTIL、DCGM_FI_DEV_FB_USED 等）。`tke-monitor-agent` 组件在 K8s 1.32.2 下不可用（chart 未发布），且功能已被 nvidia-gpu 内置，无需单独安装。
- **如何确认**：执行 `tccli tke DescribeAddonValues --region <Region> --ClusterId <ClusterId> --AddonName nvidia-gpu | jq .Values | grep exporter`，若输出含 `"exporter"` 字段，表明 DCGM 监控能力已内置。
- **什么情况需要 tke-monitor-agent**：仅在较早 K8s 版本（1.30 及以下）且 nvidia-gpu 组件版本较旧（不含内置 exporter）时，才需要单独安装 `tke-monitor-agent`。

#### 最小安装

```bash
tccli tke InstallAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName nvidia-gpu
# expected: exit 0，返回 RequestId，组件开始异步安装
```

**预期输出**：

```json
{
    "RequestId": "..."
}
```

#### 轮询确认安装完成

> 安装为异步操作，从 `Upgrading` 到 `Succeeded` 约需 20-40 秒。

```bash
tccli tke DescribeAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName nvidia-gpu
# expected: Phase = Succeeded
```

**预期输出**：

```json
{
    "Addons": [
        {
            "AddonName": "nvidia-gpu",
            "AddonVersion": "1.0.5",
            "Phase": "Succeeded",
            "Reason": "",
            "CreateTime": "2026-06-23T00:00:00Z"
        }
    ],
    "RequestId": "..."
}
```

### 步骤 2：确认 DCGM 监控能力已内置

安装 `nvidia-gpu` 后，验证 DCGM exporter 已随组件自动部署：

```bash
tccli tke DescribeAddonValues --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName nvidia-gpu | jq '.Values' | grep exporter
# expected: 输出含 exporter 配置，tag 为 v1.0.21
```

**预期输出**：

```json
{
    "exporter": {
        "tag": "v1.0.21"
    },
    "plugin": {
        "tag": "v0.18.2-tke.1"
    }
}
```

输出中包含 `exporter.tag = v1.0.21` 即表明 DCGM 监控采集器已随 nvidia-gpu 自动部署，无需额外安装任何组件。

### 步骤 3：确认 DCGM exporter Pod 运行

> 以下 kubectl 命令需在集群端点可达的环境下执行（内网端点需 IOA/VPN/专线或同 VPC CVM 跳板机）。

```bash
# 检查 DCGM exporter Pod 状态
kubectl get pods -n kube-system -l app=nvidia-dcgm-exporter
# expected: 至少 1 个 Pod STATUS = Running
```

**预期输出**：

```text
NAME                          READY   STATUS    RESTARTS   AGE
nvidia-dcgm-exporter-abc123   1/1     Running   0          2m
nvidia-dcgm-exporter-def456   1/1     Running   0          2m
```

### 步骤 4：查看 GPU 实时指标

> 以下 kubectl 命令需在集群端点可达的环境下执行（内网端点需 IOA/VPN/专线或同 VPC CVM 跳板机）。

#### 方法一：kubectl exec nvidia-smi

```bash
# 确定一个 GPU Pod
kubectl get pods -l accelerator=nvidia
# expected: 返回 GPU Pod 列表

kubectl exec GPU_POD -- nvidia-smi
# expected: 显示 GPU 使用率、显存占用、温度、功耗等
```

**预期输出**：

```text
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 525.60.13    Driver Version: 525.60.13    CUDA Version: 12.0     |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|===============================+======================+======================|
|   0  Tesla T4            Off  | 00000000:00:1E.0 Off |                    0 |
| N/A   45C    P0    25W /  70W |   2048MiB / 15360MiB |     35%      Default |
+-------------------------------+----------------------+----------------------+
```

#### 方法二：Prometheus 查询 DCGM 指标

nvidia-gpu 内置的 DCGM exporter 以 Prometheus 格式暴露以下常用 GPU 指标：

| 指标名称 | 说明 | 单位 | 告警阈值建议 |
|---------|------|------|------------|
| `DCGM_FI_DEV_GPU_UTIL` | GPU 核心使用率 | % | > 90% 持续 5m |
| `DCGM_FI_DEV_FB_USED` | 显存已用量 | MiB | > 90% 总显存 |
| `DCGM_FI_DEV_FB_FREE` | 显存空闲量 | MiB | < 10% 总显存 |
| `DCGM_FI_DEV_GPU_TEMP` | GPU 核心温度 | 摄氏度 | > 80 度 |
| `DCGM_FI_DEV_POWER_USAGE` | 当前功耗 | W | 接近功率上限 |
| `DCGM_FI_DEV_SM_CLOCK` | SM 核心时钟频率 | MHz | — |
| `DCGM_FI_DEV_MEM_CLOCK` | 显存时钟频率 | MHz | — |
| `DCGM_FI_DEV_PCIE_TX_THROUGHPUT` | PCIe 发送吞吐量 | KB/s | — |
| `DCGM_FI_DEV_PCIE_RX_THROUGHPUT` | PCIe 接收吞吐量 | KB/s | — |
| `DCGM_FI_DEV_XID_ERRORS` | XID 致命错误计数 | 次 | > 0 |

## 验证

### 控制面（tccli）

```bash
# 1. 确认 nvidia-gpu 组件正常
tccli tke DescribeAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName nvidia-gpu
# expected: Phase = Succeeded, AddonVersion = 1.0.5
```

**预期输出**：

```json
{
    "Addons": [
        {
            "AddonName": "nvidia-gpu",
            "AddonVersion": "1.0.5",
            "Phase": "Succeeded",
            "Reason": "",
            "CreateTime": "2026-06-23T00:00:00Z"
        }
    ],
    "RequestId": "..."
}
```

```bash
# 2. 确认 DCGM exporter 已内置
tccli tke DescribeAddonValues --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName nvidia-gpu | jq '.Values' | grep exporter
# expected: 输出含 "exporter" 字段及版本号
```

**预期输出**：

```json
{
    "exporter": {
        "tag": "v1.0.21"
    }
}
```

### 数据面（kubectl）

> 以下 kubectl 命令需在集群端点可达的环境下执行（内网端点需 IOA/VPN/专线或同 VPC CVM 跳板机）。

```bash
# 3. 确认 DCGM exporter Pod 运行
kubectl get pods -n kube-system -l app=nvidia-dcgm-exporter
# expected: 所有 Pod STATUS = Running

# 4. 确认 GPU 资源可见
kubectl describe node GPU_NODE | grep nvidia.com/gpu
# expected: nvidia.com/gpu 在 Capacity/Allocatable 中，值 > 0

# 5. 确认 nvidia-smi 可查询
kubectl exec GPU_POD -- nvidia-smi --query-gpu=utilization.gpu,temperature.gpu,memory.used --format=csv
# expected: 返回 GPU 使用率、温度、显存数值
```

**预期输出**（kubectl describe node）：

```text
  nvidia.com/gpu:     1
  nvidia.com/gpu:     1
```

**预期输出**（nvidia-smi query）：

```text
utilization.gpu [%], temperature.gpu, memory.used [MiB]
35 %, 45, 2048 MiB
```

| 验证维度 | 命令 | 预期 |
|---------|------|------|
| nvidia-gpu 组件 | `DescribeAddon --AddonName nvidia-gpu` | Phase = Succeeded, AddonVersion = 1.0.5 |
| DCGM exporter 内置 | `DescribeAddonValues --AddonName nvidia-gpu | grep exporter` | 输出含 exporter.tag = v1.0.21 |
| DCGM exporter Pod | `kubectl get pods -n kube-system -l app=nvidia-dcgm-exporter` | Running |
| GPU 资源 | `kubectl describe node GPU_NODE | grep nvidia.com/gpu` | nvidia.com/gpu > 0 |
| GPU 可查询 | `kubectl exec GPU_POD -- nvidia-smi` | 正常输出 GPU 状态 |

## 清理

### 数据面（kubectl）

> 以下 kubectl 命令需在集群端点可达的环境下执行（内网端点需 IOA/VPN/专线或同 VPC CVM 跳板机）。

```bash
# 1. 清理前确认 DCGM exporter Pod
kubectl get pods -n kube-system -l app=nvidia-dcgm-exporter
# 确认列出的是待清理的监控 Pod
```

**预期输出**：

```text
NAME                          READY   STATUS    RESTARTS   AGE
nvidia-dcgm-exporter-abc123   1/1     Running   0          10m
```

### 控制面（tccli）

> **警告**：卸载 `nvidia-gpu` 组件会导致所有 GPU Pod 无法调度，节点丢失 `nvidia.com/gpu` 资源。生产环境执行前务必确认无业务依赖 GPU。
>
> 卸载后 DCGM 监控指标不再采集，Prometheus 中的 GPU 指标将中断。
>
> 如集群使用 VPC-CNI 网络模式，卸载 nvidia-gpu 后 GPU 设备插件 DaemonSet 会级联删除，需重新安装才能恢复 GPU 能力。

```bash
# 2. 卸载 nvidia-gpu（停止 GPU 驱动、设备插件和 DCGM 监控采集）
tccli tke DeleteAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName nvidia-gpu
# expected: exit 0，返回 RequestId
```

**预期输出**：

```json
{
    "RequestId": "..."
}
```

```bash
# 3. 验证已卸载
tccli tke DescribeAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName nvidia-gpu
# expected: ResourceNotFound
```

**预期输出**：

```json
{
    "Error": {
        "Code": "ResourceNotFound",
        "Message": "get addon failed"
    }
}
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `InstallAddon nvidia-gpu` 返回 `FailedOperation.AddonInstallFailed` | `tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName nvidia-gpu` 查看 `Reason` | 集群无 GPU 节点，GPU 驱动安装无目标节点 | 先创建 GPU 节点池，再安装 nvidia-gpu 组件 |
| `InstallAddon tke-monitor-agent` 返回 `UnknownParameter: get addon version failed` | `tccli tke DescribeAddonValues --region <Region> --ClusterId <ClusterId> --AddonName nvidia-gpu \| jq '.Values' \| grep exporter` 确认 nvidia-gpu 已内置 exporter | 在 K8s 1.32.2 集群上，`tke-monitor-agent` 组件的 Helm Chart 尚未发布（`DescribeAddonValues` 返回 `ResourceUnavailable: not found chart`）。nvidia-gpu v1.0.5 已内置 DCGM exporter v1.0.21，无需单独安装。 | **不需要安装 `tke-monitor-agent`**。仅安装 `nvidia-gpu` 即可获得 GPU 监控能力。执行 `DescribeAddonValues nvidia-gpu` 确认 exporter 标签存在即为就绪。 |
| `DescribeAddon` 返回 `ResourceNotFound` | `tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'` 确认集群存在 | 集群 ID 错误或组件确实未安装 | 确认集群 ID；如组件未安装，执行 `InstallAddon --AddonName nvidia-gpu` |
| `kubectl exec -- nvidia-smi` 返回 `command not found` | `kubectl exec GPU_POD -- which nvidia-smi` 检查容器内 nvidia-smi | 容器镜像不含 nvidia-smi | 使用含 CUDA 的镜像（如 `nvidia/cuda:11.0-base`） |
| `kubectl exec -- nvidia-smi` 返回 `no devices found` | `kubectl describe node GPU_NODE \| grep nvidia.com/gpu` 检查节点 GPU 资源 | nvidia-gpu 组件未安装或 Pod 无 GPU 资源请求 | 确认 Pod 含 GPU 资源请求（`nvidia.com/gpu` 或 `tke.cloud.tencent.com/qgpu-core`）且 `nvidia-gpu` 组件 `Phase=Succeeded` |
| kubectl 连接超时（`dial tcp: lookup ... : no such host` / `i/o timeout`） | `tccli tke DescribeClusterEndpoints --region <Region> --ClusterId <ClusterId>` 查看端点信息 | 集群无外网端点。公网端点被组织级 CAM 策略 `strategyId:240463971` 以 `tke:clusterExtranetEndpoint=true` 条件硬拒绝创建（返回 `InvalidParameter.Param: CAM deny`）。自建安全组（已放行 TCP 443 from 0.0.0.0/0）也无法绕过。 | 使用内网端点（`ClusterIntranetEndpoint`，如 `172.24.0.x`）并通过 IOA/VPN/专线或同 VPC 内的 CVM 跳板机执行 kubectl 命令。详见 [连接集群](https://cloud.tencent.com/document/product/457/32191)。 |

### 组件安装成功但无监控数据

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| DCGM exporter Pod Running 但 Prometheus 无 GPU 指标 | `kubectl logs -n kube-system DCGM_EXPORTER_POD` 查看采集日志 | DCGM 服务未启动或 GPU 驱动版本不兼容 | 检查 nvidia-gpu 组件版本与 GPU 型号兼容性；保留 region、ClusterId、RequestId → [提交工单](https://console.cloud.tencent.com/workorder) |
| `nvidia-smi` 显示的 GPU-Util 始终为 0% | `kubectl exec GPU_POD -- nvidia-smi -q -d UTILIZATION` 查看详情 | Pod 内无 GPU 计算负载（正常） | 运行 GPU 计算任务后重试 |
| DCGM exporter Pod CrashLoopBackOff | `kubectl describe pod -n kube-system DCGM_EXPORTER_POD \| grep Events` 查看事件 | 节点资源不足或镜像拉取失败 | 检查节点资源配额；检查镜像拉取策略；保留 region、ClusterId、RequestId 备查 |

### 常见错误

以下错误是读者在实际操作中容易遇到的，提前了解可避免走弯路：

| 错误现象 | 原因 | 正确做法 |
|---------|------|---------|
| 在 K8s 1.32 集群上尝试安装 `tke-monitor-agent`，返回 `UnknownParameter: get addon version failed` | `tke-monitor-agent` 在 K8s 1.32.2 下无可用版本 | 检查 `DescribeAddonValues nvidia-gpu` 输出 → 确认 exporter 存在 → 仅安装 `nvidia-gpu` 即可 |
| 以为需要同时安装 `nvidia-gpu` 和 `tke-monitor-agent` 两个组件 | 不必要的额外操作 | `nvidia-gpu` 已集成 DCGM exporter，K8s 1.32+ 只需安装 `nvidia-gpu` |
| 公网 kubectl 连接超时 | 集群无外网端点（组织级 CAM 硬策略拒绝） | 确认集群端点：`tccli tke DescribeClusterEndpoints --region <Region> --ClusterId <ClusterId> \| jq .ClusterExternalEndpoint`。若为空，使用内网端点 + IOA/VPN/专线 |

## 下一步

- [GPU 故障检测与自愈](https://cloud.tencent.com/document/product/457/127503) — 基于 GPU 指标的自动故障检测和修复
- [使用 qGPU](https://cloud.tencent.com/document/product/457/65734) — GPU 共享和算力隔离
- [qGPU 多卡互联](https://cloud.tencent.com/document/product/457/127667) — 跨多张物理 GPU
- [使用 qGPU 离在线混部](https://cloud.tencent.com/document/product/457/81034) — 在线/离线 GPU 混部监控
- [TKE 监控概述](https://cloud.tencent.com/document/product/457/34180) — TKE 整体监控体系

## 控制台替代

[控制台 → 集群 → 组件管理](https://console.cloud.tencent.com/tke2/cluster)：安装 nvidia-gpu 组件后，在集群监控页面查看 GPU 监控面板。
