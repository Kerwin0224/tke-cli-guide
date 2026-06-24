# rdma-agent 说明（tccli）

> 对照官方：[rdma-agent 说明](https://cloud.tencent.com/document/product/457/116719) · page_id `116719`

## 概述

rdma-agent 是 TKE 提供的网络组件，自动检测节点上的 RDMA 设备并注册到 Kubernetes，支持 RDMA 网络虚拟化，使 Pod 可以使用高性能 RDMA 数据传输能力。

### 核心职责

1. **设备发现与上报**：扫描节点 RDMA 网卡设备，通过 Device Plugin 机制向 Kubelet 上报 RDMA 资源。
2. **设备分配与挂载**：Pod 创建时返回 RDMA 字符设备列表（`/dev/infiniband/*`）供挂载，并配置 cgroup 规则确保 NCCL 等库正确访问 RDMA 设备。
3. **RDMA 网卡管理**：生成并管理 RDMA CNI 配置，管理 Pod 内 RDMA Bond 网卡。

### 网络架构

支持两种模式：**独占 RDMA 网卡**和**共享 RDMA 网卡**。

| 特性 | 独占 RDMA 网卡 | 共享 RDMA 网卡 |
|------|---------------|---------------|
| 资源模型 | 物理资源（实际 bond 数量） | 虚拟资源（1000） |
| 网卡使用 | 独占宿主机 bond | 共享宿主机 bond |
| IP 分配 | 复用宿主机 bond IP | 宿主机 Bond 需扩容子网，从子网内分配独立 IP |
| 设备挂载 | 挂载已分配 bond 对应的设备 | 挂载全部 RDMA 设备 |
| GID Index | 固定为宿主机 GID | 动态，需 Pod 内脚本动态查询 |
| 拓扑感知 | 独占部分宿主机网卡，以 NUMA 拓扑分配 | 共享全部宿主机网卡，拓扑较简单 |
| 推荐场景 | 单 Hostnetwork 独占宿主机 Node；单 Pod 分配 8、4、2 或 1 卡（2 或 1 卡不保证 GPU-NIC PCIE-Switch 感知） | 单双卡场景且有高 PCI-SWITCH 拓扑感知要求；多 Hostnetwork Pod 共享宿主机 Node |

## 前置条件

- [环境准备](../../../../环境准备.md)
- 已 [连接集群](../../../../集群配置/集群管理/连接集群/tccli 操作.md)，kubectl 可正常访问 APIServer

集群状态检查：

```bash
tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'
# expected: ClusterStatus "Running"
```

```json
{
  "TotalCount": "<TotalCount>",
  "Clusters": "<Clusters>",
  "ClusterId": "<ClusterId>",
  "ClusterName": "<ClusterName>",
  "ClusterDescription": "<ClusterDescription>",
  "ClusterVersion": "<ClusterVersion>"
}
```

### 前置条件

- Kubernetes 版本 **≥ 1.20**。
- 仅支持部署在具有 RDMA 设备的实例上。
- RDMA netns 必须为 `shared` 模式，否则组件无法正常工作。
- 使用共享 RDMA 网卡非 hostnetwork 模式时，确保高性能计算集群（THCC）支持 `/28` 或更大子网。

### 前置检查

**Step 1: 检查 RDMA netns 模式**

```bash
/opt/mellanox/iproute2/sbin/rdma system show
```
```text
# expected: netns shared
```

**Step 2: 若非 `shared` 模式，修复之**

```bash
rm /etc/modprobe.d/ib_core.conf
/opt/mellanox/iproute2/sbin/rdma system set netns shared
```

如果上述命令失败，需**重启节点**。

**Step 3: 共享 RDMA 网卡模式检查子网支持**

```bash
curl http://169.254.0.23/meta-data/rdma-subnet | grep intMask
```
```text
# expected: intMask 为 /28 或更大
```

确保 `intMask` 值为 `/28` 或更大，否则 IPVLAN 虚拟化不受支持（仅能使用 HostNetwork 方式）。

### CAM 权限

| 操作 | API 权限 |
|------|----------|
| 安装组件 | `tke:InstallAddon` |
| 查询组件信息 | `tke:DescribeAddon` |
| 查询组件参数 | `tke:DescribeAddonValues` |
| 更新组件 | `tke:UpdateAddon` |
| 删除组件 | `tke:DeleteAddon` |

### 部署的 Kubernetes 对象

| 对象名称 | 类型 | 默认资源占用 | 命名空间 |
|----------|------|-------------|----------|
| tke-rdma-agent-conf | ConfigMap | — | kube-system |
| tke-rdma-shared-agent | DaemonSet | 每节点 0.01 核 CPU，100MB 内存 | kube-system |

## 控制台与 CLI 参数映射

核心 tccli 命令：

| 控制台操作 | tccli 命令 | 说明 |
|-----------|-----------|------|
| 查询集群状态 | `tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'` | 验证集群 Running |
| 查询组件信息 | `tccli tke DescribeAddon --ClusterId <ClusterId> --AddonName rdma-agent` | 获取组件版本与状态 |
| 安装组件 | `tccli tke InstallAddon --ClusterId <ClusterId> --AddonName rdma-agent --AddonVersion <AddonVersion>` | 安装 rdma-agent |
| 更新组件 | `tccli tke UpdateAddon --ClusterId <ClusterId> --AddonName rdma-agent --AddonVersion <AddonVersion>` | 升级版本或更新参数 |
| 删除组件 | `tccli tke DeleteAddon --ClusterId <ClusterId> --AddonName rdma-agent` | 卸载组件 |

组件参数映射：

| Console 参数 | tccli 参数 | 幂等性 |
|-------------|-----------|--------|
| 组件选择（勾选 rdma-agent） | `AddonName: "rdma-agent"` | 是 |
| 组件版本 | `AddonVersion` | 否 |
| CpuRequest | `RawValues` | 否 |
| CpuLimit | `RawValues` | 否 |
| MemoryRequest | `RawValues` | 否 |
| MemoryLimit | `RawValues` | 否 |
| KubeletRootDir | `RawValues` | 否 |
| NodeSelector | `RawValues` | 否 |

### 组件参数配置

| 参数 | 说明 |
|------|------|
| CpuRequest | 每节点请求 CPU，默认 10m |
| CpuLimit | 每节点最大 CPU，默认 50m |
| MemoryRequest | 每节点请求内存，默认 100Mi |
| MemoryLimit | 每节点最大内存，默认 500Mi |
| KubeletRootDir | 节点上 Kubelet 根目录，默认 `/var/lib/kubelet`。如修改过 kubelet 根目录需同步更新 |
| NodeSelector | 节点标签键值对，默认 `feature.node.kubernetes.io/tke-shared-rdma=true`。可自定义以匹配不同标签 |

## 操作步骤

### 1. 安装组件（控制面）

```bash
tccli tke InstallAddon \
    --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName rdma-agent \
    --AddonVersion <AddonVersion>
```

### 2. 设置节点标签（数据面）

> **注意**：kubectl 数据面当前因公网端点被 CAM 策略 strategyId:240463971 拒绝而不可达。以下 kubectl 命令为参考格式，需在内网/VPN 环境执行。

组件默认仅部署到带有 `feature.node.kubernetes.io/tke-shared-rdma=true` 标签的节点。

为已有节点添加标签：

```bash
kubectl label node <NodeName> feature.node.kubernetes.io/tke-shared-rdma=true
```

> 替换 `<NodeName>` 为实际节点名，可通过 `kubectl get node` 查询。

批量查看已打标签的节点：

```bash
kubectl get nodes -l feature.node.kubernetes.io/tke-shared-rdma=true
```
```text
# expected:
# NAME            STATUS   ROLES    AGE   VERSION
# 10.0.0.1        Ready    <none>   1d    v1.30.0
# 10.0.0.2        Ready    <none>   1d    v1.30.0
```

## 验证

### 控制面（tccli）

```bash
tccli tke DescribeAddon \
    --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName rdma-agent
```
```json
# expected:
# {
#     "Response": {
#         "Addons": [
#             {
#                 "AddonName": "rdma-agent",
#                 "AddonVersion": "1.0.0",
#                 "RawValues": "",
#                 "Phase": "Running",
#                 "Reason": ""
#             }
#         ],
#         "RequestId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
#     }
# }
```

### 数据面（kubectl）

> **注意**：kubectl 数据面当前因公网端点被 CAM 策略 strategyId:240463971 拒绝而不可达。以下 kubectl 命令为参考格式，需在内网/VPN 环境执行。

验证 DaemonSet 运行状态：

```bash
kubectl get ds tke-rdma-shared-agent -n kube-system
```
```text
# expected:
# NAME                    DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR                                         AGE
# tke-rdma-shared-agent   2         2         2       2            2           feature.node.kubernetes.io/tke-shared-rdma=true   5m
```

验证 RDMA 资源已上报到节点：

```bash
kubectl describe node <NodeName> | grep tke-shared-rdma
```
```text
# expected:
# tke-shared-rdma:  8
# tke-shared-rdma:  8
```

## 清理

### 控制面（tccli）

```bash
tccli tke DeleteAddon \
    --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName rdma-agent
```

### 数据面（kubectl）

> **注意**：kubectl 数据面当前因公网端点被 CAM 策略 strategyId:240463971 拒绝而不可达。以下 kubectl 命令为参考格式，需在内网/VPN 环境执行。

```bash
kubectl get ds tke-rdma-shared-agent -n kube-system
# expected: Error from server (NotFound): daemonsets.apps "tke-rdma-shared-agent" not found
```

```text
NAME  STATUS  AGE
...
```

> **计费提醒**：rdma-agent 组件本身不产生额外费用。RDMA 实例按规格计费，与组件无关。

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| DaemonSet 未调度到 RDMA 节点 | `kubectl get nodes -l feature.node.kubernetes.io/tke-shared-rdma=true` 无返回 | 节点缺少 `feature.node.kubernetes.io/tke-shared-rdma=true` 标签 | 执行 `kubectl label node <NodeName> feature.node.kubernetes.io/tke-shared-rdma=true` |
| Pod 启动后 RDMA 设备不可用 | `/opt/mellanox/iproute2/sbin/rdma system show` 显示 `netns exclusive` | RDMA netns 为 `exclusive` 模式 | 执行 `/opt/mellanox/iproute2/sbin/rdma system set netns shared`，必要时重启节点 |
| 共享模式非 HostNetwork Pod IP 分配失败 | `curl http://169.254.0.23/meta-data/rdma-subnet \| grep intMask` 掩码过小 | 子网掩码不足（< /28） | 确保 intMask 为 `/28` 或更大；否则仅能使用 HostNetwork |
| 组件安装后无 RDMA 资源上报 | `ibstat` 无设备输出 | 节点无 RDMA 设备或设备未识别 | 确认节点为 RDMA 实例（如 GPU 机型）；执行 `ibstat` 检查 RDMA 设备状态 |
| Unable to connect（kubectl 数据面不可达） | 公网端点连接超时 | CAM 策略 strategyId:240463971 拒绝 `tke:clusterExtranetEndpoint=true` | 在内网/VPN 环境执行命令；或调整 CAM 策略放通公网端点权限 |

## 下一步

- [在 TKE 上使用 RDMA 加速 AI 推理和训练](https://cloud.tencent.com/document/product/457/xxxxx)
- [TKE GPU 节点使用说明](https://cloud.tencent.com/document/product/457/32205)
- [Kubernetes Device Plugin 机制](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/)
- [TKE 组件管理概述](https://cloud.tencent.com/document/product/457/51082)
- [NVIDIA NCCL 官方文档](https://docs.nvidia.com/deeplearning/nccl/)

## 控制台替代

[容器服务控制台 - 组件管理 - rdma-agent](https://console.cloud.tencent.com/tke2/cluster?rid=<Region>)
