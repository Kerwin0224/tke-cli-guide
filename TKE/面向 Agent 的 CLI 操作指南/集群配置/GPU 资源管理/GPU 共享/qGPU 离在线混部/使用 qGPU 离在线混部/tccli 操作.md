# 使用 qGPU 离在线混部

> 对照官方：[使用 qGPU 离在线混部](https://cloud.tencent.com/document/product/457/81034) · page_id `81034` · tccli ≥3.1.x · API 2018-05-25

## 概述

qGPU 离在线混部允许同一 GPU 节点上同时运行在线（高优）和离线（低优）业务。在线业务独享显存保障，离线业务可抢占在线业务空闲时的 GPU 算力，提升 GPU 整体利用率。

前置条件：已完成 [使用 qGPU](../../使用%20qGPU/tccli%20操作.md) 的步骤 1-3（集群启用 QGPU、安装 qgpu 组件、GPU 原生节点池就绪）。本文在此基础上**重建 `qgpu-manager` / `qgpu-scheduler` Pod** 使混部配置生效，再用 annotation `tke.cloud.tencent.com/app-class` 区分三种 Pod。

<!-- L2 链接说明：本文中的相对链接（如 ../../使用qGPU/tccli 操作.md）在本地文件系统可正常解析，发布到 iWiki/写写平台后需通过 sync 脚本转换为平台链接格式 -->

| 维度 | 在线 Pod（Online） | 离线 Pod（Offline） | 普通 Pod |
|------|-------------------|---------------------|---------|
| Annotation | `app-class: online` | `app-class: offline` | 无 `app-class` |
| 算力资源 | 无需声明（仅显存） | `qgpu-core-greedy`（贪婪，1-100） | `qgpu-core`（1-100 或 >100） |
| 显存资源 | `qgpu-memory`（静态切分，不会被抢占） | `qgpu-memory` | `qgpu-memory` |
| 多卡支持 | 不支持 | 不支持（greedy <= 100） | 支持（core > 100） |
| 算力优先级 | 高优（不受离线 Pod 影响） | 低优（抢占在线空闲算力） | 无优先级区分 |
| 适用场景 | 实时推理、在线服务 | 批量处理、模型训练 | 开发测试、独占 GPU |

**本文覆盖路径**：tccli（控制面：确认组件状态）+ kubectl（数据面：重建组件 Pod、部署三种 Pod、验证混部效果）。

## 前置条件

- [环境准备](../../../../../环境准备.md)
- 已按 [使用 qGPU](../../使用%20qGPU/tccli%20操作.md) 完成 qgpu 组件安装和 GPU 节点池准备
- kubectl 已配置且可访问集群

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置，region 为集群所在地域

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:DescribeClusters, tke:DescribeAddon, tke:InstallAddon, tke:DeleteAddon, tke:DescribeAddonValues
# 验证：执行 DescribeClusters 确认 TKE 权限
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
# expected: Kubernetes control plane is running
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
            "ClusterAdvancedSettings": {
                "QGPUShareEnable": true
            }
        }
    ],
    "RequestId": "..."
}
```

```bash
# 7. 确认 qgpu 组件已安装且状态正常
tccli tke DescribeAddon --ClusterId <ClusterId> --AddonName qgpu --region <Region> --output json
# expected: Phase 为 Succeeded
```

预期输出：

```json
{
    "Addons": [
        {
            "AddonName": "qgpu",
            "AddonVersion": "<QgpuVersion>",
            "Phase": "Succeeded",
            "Reason": "",
            "CreateTime": "<CreateTime>"
        }
    ],
    "RequestId": "<RequestId>"
}
```

```bash
# 8. 确认 GPU 节点存在并已注册 qGPU 资源
kubectl get nodes -l accelerator=nvidia
# expected: 至少 1 个 GPU 节点

kubectl describe node <GpuNodeName> | grep -E "qgpu-core|qgpu-memory"
# expected: 节点 Allocatable 含 tke.cloud.tencent.com/qgpu-core 和 qgpu-memory，值 > 0
```

```text
NAME  STATUS  AGE
...
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<Region>` | 集群所在地域 | 如 `ap-guangzhou` | `tccli configure list` 确认当前 region |
| `<ClusterId>` | 集群 ID | 格式 `cls-xxxxxxxx` | `tccli tke DescribeClusters --region <Region>` |
| `<GpuNodeName>` | GPU 节点名 | 原生节点，已安装 GPU 驱动 | `kubectl get nodes -l accelerator=nvidia` |
| `<QgpuManagerPodName>` | qgpu-manager Pod 名称 | qgpu-manager DaemonSet Pod | `kubectl get pods -n kube-system \| grep qgpu-manager \| awk '{print $1}'` |
| `<QgpuVersion>` | qGPU addon 版本号 | 如 `1.1.3` | `tccli tke DescribeAddonValues --AddonName qgpu --region <Region> --ClusterId <ClusterId>`，从返回的 AddonVersion 字段获取 |

## 控制台与 CLI 参数映射

| 控制台操作 | tccli / kubectl 命令 | 幂等 |
|-----------|---------------------|:--:|
| 检查 qGPU addon 安装状态（安装前） | `DescribeAddon --AddonName qgpu` | 是 |
| 获取 qGPU addon 可配置参数 | `DescribeAddonValues --AddonName qgpu` | 是 |
| 安装 qGPU addon | `InstallAddon --AddonName qgpu --AddonVersion <QgpuVersion>` | 否（异步安装，约 60 秒） |
| 验证 qGPU addon 安装成功 | `DescribeAddon --AddonName qgpu` → Phase: Succeeded | 是 |
| 重建 qgpu-manager Pod | `kubectl delete pod -n kube-system <QgpuManagerPodName>` | 否（Deployment 自动重建） |
| 重建 qgpu-scheduler Pod | `kubectl delete pod -n kube-system -l app=qgpu-scheduler` | 否（Deployment 自动重建） |
| 查看低优算力资源 | `kubectl describe node <GpuNodeName>` | 是 |
| 部署在线 Pod | `kubectl apply -f`（含 `app-class: online`） | 否 |
| 部署离线 Pod | `kubectl apply -f`（含 `app-class: offline` + `qgpu-core-greedy`） | 否 |
| 部署普通 Pod | `kubectl apply -f`（无 `app-class`） | 否 |
| 删除 qGPU addon（清理） | `DeleteAddon --AddonName qgpu` | 否（异步删除，约 90 秒） |

## 关键字段说明

离在线混部通过 Pod annotation 和资源声明触发，核心字段如下：

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `tke.cloud.tencent.com/app-class` | Annotation | 否 | `online`（在线高优）或 `offline`（离线低优）。不设为普通 Pod | 填错误值 → Pod 调度后行为与预期不符，无混部优先级 |
| `tke.cloud.tencent.com/qgpu-core-greedy` | 资源 | 是（离线） | 1-100，贪婪算力百分比。仅离线 Pod 使用，不支持多卡 | 在线 Pod 误用 greedy → 高优保障失效；值 > 100 → 调度失败 |
| `tke.cloud.tencent.com/qgpu-core` | 资源 | 是（普通） | 1-100 单卡，>100 多卡。普通 Pod 使用 | 离线 Pod 用 core 而非 greedy → 无法抢占空闲算力 |
| `tke.cloud.tencent.com/qgpu-memory` | 资源 | 是 | GiB 显存，整数，最小 1。在线 Pod 静态切分不被抢占，离线/普通按量分配 | 超过节点总显存 → Pod 无法调度 |

## 操作步骤

### 步骤 1：确认 qgpu 组件已安装

qGPU 离在线混部依赖 qgpu 组件。如果尚未安装，参见 [使用 qGPU](../../使用%20qGPU/tccli%20操作.md) 的步骤 1-2，或按本节下方的安装流程操作。

#### 检查安装状态

```bash
# 安装前检查 qGPU addon 是否存在（预期：未安装时返回 ResourceNotFound）
tccli tke DescribeAddon --ClusterId <ClusterId> --AddonName qgpu --region <Region> --output json
```

未安装时的预期输出：

```json
{
    "Response": {
        "Error": {
            "Code": "ResourceNotFound",
            "Message": "get addon failed"
        },
        "RequestId": "<RequestId>"
    }
}
```

#### 获取可配置参数

在安装前，可通过 `DescribeAddonValues` 查看 qGPU addon 的默认配置参数：

```bash
tccli tke DescribeAddonValues --AddonName qgpu --ClusterId <ClusterId> --region <Region> --output json
```

预期输出（节选关键字段，`Values` 和 `DefaultValues` 为 JSON 编码的字符串）：

```json
{
    "Values": "{\"config\":{\"nodepriority\":\"spread\",\"priority\":\"binpack\",\"rootdir\":\"/var/lib/kubelet\"},\"manager\":{\"tag\":\"v2.0.2\"},\"scheduler\":{\"tag\":\"v2.0.2\"}}",
    "DefaultValues": "{\"config\":{\"nodepriority\":\"spread\",\"priority\":\"binpack\",\"rootdir\":\"/var/lib/kubelet\"},\"manager\":{\"tag\":\"v2.0.2\"},\"scheduler\":{\"tag\":\"v2.0.2\"}}",
    "RequestId": "<RequestId>"
}
```

> 以下为 `Values` 字符串解析后的内容（可用 `jq '.Values | fromjson'` 查看）：

```json
{
    "config": {
        "nodepriority": "spread",
        "priority": "binpack",
        "rootdir": "/var/lib/kubelet"
    },
    "manager": {
        "tag": "v2.0.2"
    },
    "scheduler": {
        "tag": "v2.0.2"
    }
}
```

#### 安装 qGPU addon（如未安装）

> **决策依据**：qGPU addon 是 TKE GPU 共享和离在线混部的基础组件。安装后会在集群中部署 qgpu-manager、qgpu-scheduler、qgpu-exporter 等组件。安装需要约 60 秒，期间可能短暂显示 InstallFailed 后自愈（组件部署过程中的重试）。
> **版本选择**：通过 `tccli tke DescribeAddonValues --AddonName qgpu --region <Region> --ClusterId <ClusterId>` 查看当前可用版本号，替换下述命令中的 `--AddonVersion <QgpuVersion>`。
> **常见错误**：安装后立即查询状态看到 InstallFailed → 等待 10-20 秒后自愈为 Succeeded；安装时忘记传 AddonVersion → 需显式指定版本号（通过 `DescribeAddonValues` 获取最新版）。

```bash
tccli tke InstallAddon --ClusterId <ClusterId> --AddonName qgpu --AddonVersion <QgpuVersion> --region <Region>
```

预期输出：

```json
{
    "RequestId": "<RequestId>"
}
```

> **注意**：`InstallAddon` 是异步操作，返回 RequestId 不代表安装完成。必须轮询 `DescribeAddon` 确认 Phase 变为 `Succeeded`（约 60 秒）。中间可能短暂出现 `Installing` 或 `InstallFailed` 状态，等待 10-30 秒后重查即可。

#### 验证安装成功

```bash
tccli tke DescribeAddon --ClusterId <ClusterId> --AddonName qgpu --region <Region> --output json
```

预期输出（安装成功后）：

```json
{
    "Addons": [
        {
            "AddonName": "qgpu",
            "AddonVersion": "<QgpuVersion>",
            "Phase": "Succeeded",
            "Reason": "",
            "CreateTime": "<CreateTime>"
        }
    ],
    "RequestId": "<RequestId>"
}
```

> 如 `Phase` 非 `Succeeded`，参见 [使用 qGPU](../../使用%20qGPU/tccli%20操作.md) 排障章节处理组件安装问题。

### 步骤 2：重建 qgpu 组件 Pod

#### 选择依据

- **为何需要重建**：qgpu-manager 和 qgpu-scheduler 需要重建才能识别节点的离在线混部配置。仅安装 qgpu 组件不会自动启用混部能力。
- **重建时机**：需在业务 Pod 调度到 GPU 节点**之前**完成重建。如节点上已有 Pod，建议先封锁节点（`kubectl cordon <GpuNodeName>`），重建完成后再解封（`kubectl uncordon <GpuNodeName>`）。
- **重建方式**：直接 `kubectl delete pod`，Deployment/DaemonSet 控制器会自动拉起新 Pod。无需 tccli 操作。

#### 执行重建

```bash
# 1. 查看当前 qgpu-manager Pod（DaemonSet，每 GPU 节点 1 个）
kubectl get pods -n kube-system | grep qgpu-manager
# expected: 返回 qgpu-manager-xxxxx，状态 Running
```

预期输出：

```text
NAME                READY   STATUS    RESTARTS   AGE
qgpu-manager-abc12  1/1     Running   0          10m
```

从步骤 1 的输出中复制 qgpu-manager Pod 的名称（NAME 列），替换 `<QgpuManagerPodName>`：

```bash
# 2. 删除目标 GPU 节点上的 qgpu-manager Pod（DaemonSet 自动重建）
kubectl delete pod -n kube-system <QgpuManagerPodName>
# expected: pod "<QgpuManagerPodName>" deleted
```

```bash
# 3. 删除所有 qgpu-scheduler Pod（Deployment 自动重建）
kubectl delete pod -n kube-system -l app=qgpu-scheduler
# expected: 返回已删除的 Pod 名称
```

```bash
# 4. 确认 Pod 已重建并 Running
kubectl get pods -n kube-system | grep qgpu
# expected: qgpu-manager 和 qgpu-scheduler Pod 状态均为 Running
```

预期输出（重建后确认）：

```text
NAME                 READY   STATUS    RESTARTS   AGE
qgpu-manager-abc12   1/1     Running   0          30s
qgpu-scheduler-def34 1/1     Running   0          25s
```

#### 验证低优算力资源

重建后，节点应出现 `qgpu-core-greedy` 扩展资源（离线 Pod 专用）：

```bash
kubectl describe node <GpuNodeName> | grep -A5 "Allocatable:"
# expected: 包含 tke.cloud.tencent.com/qgpu-core-greedy 等低优算力扩展资源
```

预期输出（`kubectl describe node` 节选）：

```text
Allocatable:
  cpu:                                     62
  memory:                                  250Gi
  nvidia.com/gpu:                          4
  tke.cloud.tencent.com/qgpu-core:         400
  tke.cloud.tencent.com/qgpu-core-greedy:  0
  tke.cloud.tencent.com/qgpu-memory:       160
```

> `qgpu-core-greedy` 显示为 `0` 是正常现象——该资源由离线 Pod 实际声明的总量动态占据，不影响调度。

### 步骤 3：部署三种业务 Pod

#### 选择依据

- **在线 Pod**：仅声明 `qgpu-memory`，算力由节点高优分配，不受离线业务干扰。适合实时推理、在线服务。
- **离线 Pod**：用 `qgpu-core-greedy` 贪婪抢占空闲算力，在线业务忙时自动降速。不支持多卡（greedy <= 100）。适合批量处理、模型训练。
- **普通 Pod**：无 `app-class`，标准 `qgpu-core` + `qgpu-memory`，不参与优先级抢占。支持多卡（core > 100）。适合开发测试、独占 GPU。

#### 在线 Pod（高优，仅显存）

`online-pod.yaml`：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: online-inference
  annotations:
    tke.cloud.tencent.com/app-class: "online"
spec:
  containers:
  - name: online-container
    image: nvidia/cuda:11.0-base
    command: ["sleep", "3600"]
    resources:
      requests:
        tke.cloud.tencent.com/qgpu-memory: "5"
      limits:
        tke.cloud.tencent.com/qgpu-memory: "5"
  restartPolicy: Never
```

```bash
kubectl apply -f online-pod.yaml
# expected: pod/online-inference created
```

```bash
kubectl get pod online-inference
# expected: STATUS 为 Running
```

| 资源声明 | 说明 | 示例 |
|---------|------|------|
| `tke.cloud.tencent.com/app-class` | `online` 标识在线高优 | `"online"` |
| `tke.cloud.tencent.com/qgpu-memory` | 显存 GiB，静态切分不被抢占 | `"5"` 表示 5GiB |

#### 离线 Pod（低优，贪婪算力）

`offline-pod.yaml`：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: offline-batch
  annotations:
    tke.cloud.tencent.com/app-class: "offline"
spec:
  containers:
  - name: offline-container
    image: nvidia/cuda:11.0-base
    command: ["sleep", "3600"]
    resources:
      requests:
        tke.cloud.tencent.com/qgpu-core-greedy: "30"
        tke.cloud.tencent.com/qgpu-memory: "8"
      limits:
        tke.cloud.tencent.com/qgpu-core-greedy: "30"
        tke.cloud.tencent.com/qgpu-memory: "8"
  restartPolicy: Never
```

```bash
kubectl apply -f offline-pod.yaml
# expected: pod/offline-batch created
```

```bash
kubectl get pod offline-batch
# expected: STATUS 为 Running
```

| 资源声明 | 说明 | 示例 |
|---------|------|------|
| `tke.cloud.tencent.com/app-class` | `offline` 标识离线低优 | `"offline"` |
| `tke.cloud.tencent.com/qgpu-core-greedy` | 贪婪算力百分比，1-100，不支持多卡 | `"30"` 表示 30% 算力 |
| `tke.cloud.tencent.com/qgpu-memory` | 显存 GiB | `"8"` 表示 8GiB |

#### 普通 Pod（无优先级区分）

`common-pod.yaml`：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: common-dev
spec:
  containers:
  - name: common-container
    image: nvidia/cuda:11.0-base
    command: ["sleep", "3600"]
    resources:
      requests:
        tke.cloud.tencent.com/qgpu-core: "25"
        tke.cloud.tencent.com/qgpu-memory: "6"
      limits:
        tke.cloud.tencent.com/qgpu-core: "25"
        tke.cloud.tencent.com/qgpu-memory: "6"
  restartPolicy: Never
```

```bash
kubectl apply -f common-pod.yaml
# expected: pod/common-dev created
```

```bash
kubectl get pod common-dev
# expected: STATUS 为 Running
```

```text
NAME  STATUS  AGE
...
```

| Pod 类型 | Annotation | 算力资源 | 显存资源 | 多卡 | 优先级 |
|---------|-----------|---------|---------|:--:|:--:|
| 在线 | `app-class: online` | 无需声明 | `qgpu-memory: 5` | 否 | 高优 |
| 离线 | `app-class: offline` | `qgpu-core-greedy: 30` | `qgpu-memory: 8` | 否 | 低优 |
| 普通 | 无 | `qgpu-core: 25` | `qgpu-memory: 6` | 是 | 无优先级 |

## 验证

### 控制面（tccli）

```bash
# 确认 qgpu 组件状态正常（重建不影响组件状态）
tccli tke DescribeAddon --ClusterId <ClusterId> --AddonName qgpu --region <Region> --output json
# expected: Phase 为 Succeeded
```

预期输出：

```json
{
    "Addons": [
        {
            "AddonName": "qgpu",
            "AddonVersion": "<QgpuVersion>",
            "Phase": "Succeeded",
            "Reason": "",
            "CreateTime": "<CreateTime>"
        }
    ],
    "RequestId": "<RequestId>"
}
```

### 数据面

```bash
# 1. 确认重建后 qgpu-manager 和 qgpu-scheduler 均 Running
kubectl get pods -n kube-system | grep qgpu
# expected: qgpu-manager 和 qgpu-scheduler Pod 状态均为 Running

# 2. 确认节点已注册低优算力资源
kubectl describe node <GpuNodeName> | grep qgpu-core-greedy
# expected: 返回 tke.cloud.tencent.com/qgpu-core-greedy 资源行

# 3. 确认三种 Pod 均已 Running
kubectl get pods -o wide
# expected: online-inference、offline-batch、common-dev 状态均为 Running，调度到同一 GPU 节点
```

预期输出：

```text
NAME                READY   STATUS    RESTARTS   AGE   IP            NODE
common-dev          1/1     Running   0          60s   10.244.1.10   gpu-node-01
offline-batch       1/1     Running   0          55s   10.244.1.11   gpu-node-01
online-inference    1/1     Running   0          65s   10.244.1.9    gpu-node-01
```

```bash
# 4. 确认离线 Pod 使用低优算力
kubectl describe pod offline-batch | grep qgpu-core-greedy
# expected: 显示 tke.cloud.tencent.com/qgpu-core-greedy: 30

# 5. 确认在线 Pod 仅申请显存（无 qgpu-core/qgpu-core-greedy）
kubectl describe pod online-inference | grep qgpu
# expected: 仅显示 tke.cloud.tencent.com/qgpu-memory: 5，无 qgpu-core 或 qgpu-core-greedy

# 6. 确认普通 Pod 使用标准 qgpu-core
kubectl describe pod common-dev | grep qgpu
# expected: 显示 tke.cloud.tencent.com/qgpu-core: 25 和 qgpu-memory: 6

# 7. 验证 GPU 实际共享
kubectl exec online-inference -- nvidia-smi
# expected: GPU 已被多 Pod 共享使用，显存占用分别为各 Pod 声明的 qgpu-memory
```

```text
NAME  STATUS  AGE
...
```

验证维度：

| 维度 | 命令 | 预期 |
|------|------|------|
| 组件状态 | `DescribeAddon --AddonName qgpu` | Phase 为 Succeeded |
| 组件 Pod 重建 | `kubectl get pods -n kube-system \| grep qgpu` | qgpu-manager、qgpu-scheduler 均 Running |
| 低优算力资源 | `kubectl describe node <GpuNodeName> \| grep qgpu-core-greedy` | 节点含 `qgpu-core-greedy` 扩展资源 |
| 在线 Pod | `kubectl describe online-inference \| grep qgpu` | 仅含 `qgpu-memory` |
| 离线 Pod | `kubectl describe offline-batch \| grep qgpu` | 含 `qgpu-core-greedy` + `qgpu-memory` |
| 普通 Pod | `kubectl describe common-dev \| grep qgpu` | 含 `qgpu-core` + `qgpu-memory` |
| 实际 GPU 共享 | `kubectl exec online-inference -- nvidia-smi` | 多 Pod 共享 GPU，显存按声明分配 |

## 清理

> **注意**：删除测试 Pod 不会影响 qgpu 组件。如需彻底卸载 qGPU 能力，参见 [使用 qGPU](../../使用%20qGPU/tccli%20操作.md) 的清理章节。

### 数据面

```bash
# 1. 清理前确认目标 Pod
kubectl get pods
# 确认列出的是本页创建的测试 Pod：online-inference、offline-batch、common-dev
```

```text
NAME  STATUS  AGE
...
```

```bash
# 2. 删除测试 Pod
kubectl delete pod online-inference offline-batch common-dev
# expected: 三个 Pod 均已删除
```

```bash
# 3. 验证 Pod 已删除
kubectl get pods | grep -E "online-inference|offline-batch|common-dev"
# expected: 无输出（Pod 已删除）
```

```text
NAME  STATUS  AGE
...
```

### 控制面（tccli）— 卸载 qGPU addon

> **清理警告**：
> - 删除 qGPU addon 会移除 qgpu-manager/qgpu-scheduler/qgpu-exporter 等组件 Pod，已部署的 qGPU 工作负载将因无法识别 qGPU 资源而出错
> - 删除 addon 前确认无业务负载依赖 qGPU 资源分配
> - 组件删除是异步的，DeleteAddon 返回成功不代表组件已完全清理，需轮询 DescribeAddon 确认 ResourceNotFound
> - 重新安装 qGPU addon 不会自动恢复已有工作负载的 qGPU 配置——需重新部署

```bash
# 删除 qGPU addon
tccli tke DeleteAddon --ClusterId <ClusterId> --AddonName qgpu --region <Region>
```

预期输出：

```json
{
    "RequestId": "<RequestId>"
}
```

```bash
# 轮询确认组件完全删除（约 90 秒后返回 ResourceNotFound）
tccli tke DescribeAddon --ClusterId <ClusterId> --AddonName qgpu --region <Region> --output json
```

中间状态（删除进行中，Phase 为 Terminating）：

```json
{
    "Addons": [
        {
            "AddonName": "qgpu",
            "AddonVersion": "<QgpuVersion>",
            "Phase": "Terminating",
            "Reason": "",
            "CreateTime": "<CreateTime>"
        }
    ],
    "RequestId": "<RequestId>"
}
```

最终状态（完全删除后）：

```json
{
    "Response": {
        "Error": {
            "Code": "ResourceNotFound",
            "Message": "get addon failed"
        },
        "RequestId": "<RequestId>"
    }
}
```

> **注意**：务必等待 `DescribeAddon` 返回 `ResourceNotFound` 后再进行后续操作。未等待完全删除就重新安装可能导致组件冲突或状态异常。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DescribeAddon --AddonName qgpu` 返回空列表或 `ResourceNotFound` | `tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId>`（不传 AddonName）查看全部组件 | qgpu 组件未安装，或 `AddonName` 拼写错误 | 用 `qgpu`（全小写）作为 AddonName。如未安装，参见 [使用 qGPU](../../使用%20qGPU/tccli%20操作.md) 安装 |
| `InstallAddon` 返回 `MissingParameter` | 检查命令是否包含 `--AddonVersion` 参数 | 安装时忘记传 AddonVersion | 通过 `DescribeAddonValues` 查看当前可用版本号，添加 `--AddonVersion <QgpuVersion>` |
| `DescribeAddon` 返回 `Phase: InstallFailed` | 等待 10-30 秒后重新查询 | 组件部署过程中的自愈重试可能导致短暂 InstallFailed，最终会自愈为 Succeeded | 等待 10-30 秒后重新执行 `DescribeAddon`，确认 Phase 变为 Succeeded |
| `CreateClusterEndpoint` (extranet) 返回 `InvalidParameter.Param` + CAM deny | 查看错误详情中的 strategyId 和 condition | 组织级 CAM 策略（strategyId: 240463971）以 `tke:clusterExtranetEndpoint=true` 条件硬拒绝公网端点创建。即使使用自建安全组也无法绕过 | 使用内网端点（`--IsExtranet false`），但内网端点需通过 IOA/VPN/专线或同 VPC CVM 才能 kubectl 可达 |
| kubectl 返回 `http: server gave HTTP response to HTTPS client` | 内网端点直连 IP 时出现 | API Server 返回 HTTP 而非 HTTPS，kubectl 默认期望 HTTPS | 通过 IOA/VPN/专线接入腾讯云内网，或从同 VPC 的 CVM 执行 kubectl 命令 |
| kubectl 返回 `dial tcp: lookup cls-xxx.ccs.tencent-cloud.com: no such host` | 检查 DNS 解析 | 内网域名（ccs.tencent-cloud.com）仅 VPC 内可解析，外网不可达 | 使用 VPC 内 DNS，或通过 IOA/VPN/专线连接腾讯云内网 |
| `kubectl apply` 离线 Pod 返回 `Insufficient tke.cloud.tencent.com/qgpu-core-greedy` | `kubectl describe node <GpuNodeName> \| grep qgpu-core-greedy` 检查节点资源 | 未重建 qgpu-manager/scheduler Pod，节点未注册 greedy 资源 | 执行步骤 2 重建 qgpu-manager 和 qgpu-scheduler Pod，重建后重新 `kubectl describe node <GpuNodeName>` 确认 `qgpu-core-greedy` 出现 |
| `kubectl apply` 返回 `Insufficient tke.cloud.tencent.com/qgpu-memory` | `kubectl describe node <GpuNodeName> \| grep qgpu-memory` 查看节点可用显存 | 节点显存已被其他 Pod 占满 | 降低 `qgpu-memory` 值，或 `kubectl delete pod` 删除部分 Pod 释放显存 |
| `kubectl delete pod -l app=qgpu-scheduler` 无输出 | `kubectl get pods -n kube-system -l app=qgpu-scheduler` 确认 Pod 是否存在 | qgpu-scheduler 标签非 `app=qgpu-scheduler`，或 Pod 已被删除 | `kubectl get pods -n kube-system \| grep scheduler` 找到正确 Pod 名，按名称删除 |
| GPU 节点不可用（集群节点数为 0，或无 GPU 实例） | `tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'` 检查 ClusterNodeNum；`kubectl get nodes -l accelerator=nvidia` 检查 GPU 节点 | 集群无节点或 GPU 原生节点池未创建 | 创建 GPU 原生节点池（`tccli tke CreateClusterNodePool`，机型选 GN10Xp/GN7 等），确认节点 Ready 后再部署 qGPU 工作负载 |
| Pod 部署后 Pending（无可用 GPU 节点调度） | `kubectl describe pod <pod-name> \| grep Events` 查看调度事件 | GPU 节点池未就绪时部署 qGPU 负载，无可用 GPU 节点 | 先创建 GPU 原生节点池（`tccli tke CreateClusterNodePool`，机型选 GN10Xp/GN7 等），确认节点 Ready 后再部署 |

### 操作成功但 Pod 行为异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 在线 Pod 被离线业务影响，推理延迟增大 | `kubectl describe pod online-inference \| grep -A3 Annotations` 检查 `app-class` | 在线 Pod 错误使用了 `app-class: offline` 或未设 `app-class`，导致算力无高优保障 | 确认 annotation 为 `tke.cloud.tencent.com/app-class: "online"`，仅声明 `qgpu-memory`，不声明 `qgpu-core-greedy` |
| 离线 Pod 无法抢占更多算力 | `kubectl exec online-inference -- nvidia-smi` 检查在线 Pod GPU 使用率 | 在线 Pod 持续高负载，离线 Pod 无法抢占空闲算力（此为正常行为，非错误） | 降低在线 Pod 的计算负载，或调整离线 Pod 的 `qgpu-core-greedy` 目标值 |
| 离线 Pod 调度失败 | `kubectl describe pod offline-batch \| grep -E "Events\|Warning"` 查看调度事件 | 离线 Pod 不支持多卡，`qgpu-core-greedy` 值 > 100 无效 | `qgpu-core-greedy` 控制在 1-100 范围内。如需多卡，改用普通 Pod（`qgpu-core > 100`，无 `app-class`） |
| 重建 qgpu Pod 后节点仍无 `qgpu-core-greedy` 资源 | `kubectl logs -n kube-system <QgpuManagerPodName>` 查看新 Pod 日志 | qgpu-manager 启动失败或版本不支持混部 | 检查 qgpu 组件版本：`tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName qgpu`。保留 region、ClusterId、RequestId → [提交工单](https://console.cloud.tencent.com/workorder) |
| 普通 Pod 混用了优先级 | `kubectl describe pod common-dev \| grep -A3 Annotations` 检查是否误设 `app-class` | 普通 Pod 不应设 `app-class` annotation | 删除 `tke.cloud.tencent.com/app-class` annotation，仅保留 `qgpu-core` + `qgpu-memory` |

## 下一步

- [使用 qGPU](https://cloud.tencent.com/document/product/457/65734) — qGPU 基础安装和 Pod 部署
- [qGPU 多卡互联](https://cloud.tencent.com/document/product/457/127667) — Pod 跨多张物理 GPU 使用 vGPU 资源
- [qGPU 概述](https://cloud.tencent.com/document/product/457/61448) — qGPU 架构、隔离策略与资源声明规则
- [GPU 监控指标获取](https://cloud.tencent.com/document/product/457/90912) — 通过 DCGM 获取 GPU 使用率、显存等监控指标
- [GPU 故障检测与自愈](https://cloud.tencent.com/document/product/457/127503) — GPU 故障自动检测与隔离

## 控制台替代

[TKE 控制台 → 集群 → 工作负载](https://console.cloud.tencent.com/tke2/cluster)：在 qgpu 组件与 GPU 节点池就绪后，按官方顺序重建组件并创建带 `tke.cloud.tencent.com/app-class` annotation 的工作负载，区分在线/离线/普通 Pod。
