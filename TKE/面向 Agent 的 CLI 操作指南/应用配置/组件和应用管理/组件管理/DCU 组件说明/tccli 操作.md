# DCU 组件说明（tccli）

> 对照官方：[DCU 组件说明](https://cloud.tencent.com/document/product/457/130456) · page_id `130456`

## 概述

海光 DCU 组件（标识符：`dcu-gpu`）用于在 TKE 集群中管理和调度海光 DCU（Deep Computing Unit）加速器卡资源，支持整卡分配和虚拟化分配两种模式，高效利用国产计算资源，同时提供 Prometheus 标准的设备监控指标。

### 使用场景

| 模式 | 说明 | 适用场景 |
|------|------|---------|
| 整卡模式（`full`） | 将整张 DCU 卡作为独占资源分配给业务容器，默认调度所有 DCU 节点 | 需要最大算力且不共享资源的应用 |
| 虚拟化模式（`vdcu`） | 基于 HAMi，将一张 DCU 卡虚拟化为多个 vDCU，允许多容器共享一张 DCU 卡的算力和显存 | 推理服务、轻量训练等需要细粒度 GPU 资源分配的场景 |

### 部署在集群内的 Kubernetes 对象

| 对象名称 | 类型 | 说明 | 命名空间 |
|---------|------|------|---------|
| dcu-label-node | DaemonSet | 自动检测并标记含海光 DCU 设备的节点 | kube-system |
| dcu-device-plugin | DaemonSet | 基础 Device Plugin，向 kubelet 注册整卡 DCU 资源（仅整卡模式生效） | kube-system |
| dcu-device-plugin-hami | DaemonSet | 基于 HAMi 的 Device Plugin，处理 vDCU 虚拟算力拆分和注册（仅 vDCU 模式生效） | kube-system |
| vdcu-admission-webhook | Deployment | 拦截器，自动向业务 Pod 注入 vDCU 相关环境变量和资源配置 | kube-system |
| vdcu-scheduler-plugin | Deployment | 自定义调度器，支持 binpack 或 spread 策略进行 vDCU 节点和卡级资源调度 | kube-system |
| dcu-exporter | DaemonSet | DCU 设备指标采集器，暴露 Prometheus 标准格式监控指标 | kube-system |

## 前置条件

- [环境准备](../../../../环境准备.md)
- 已 [连接集群](../../../../集群配置/集群管理/连接集群/tccli 操作.md)，kubectl 可正常访问 APIServer
- 集群 Kubernetes 版本 ≥ 1.16
- 节点已安装海光 DCU 驱动和 HyHAL 运行时环境
- vDCU 模式：节点需打上 `dcu=on` 标签
- 集群状态 `Running`

```bash
# 检查集群状态
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

- 需要 CAM 权限：`tke:DescribeClusters`、`tke:DescribeAddon`、`tke:InstallAddon`、`tke:UpdateAddon`、`tke:DeleteAddon`

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 查看集群信息 | `tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'` | 是 |
| 查看 DCU 组件详情 | `tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName dcu-gpu` | 是 |
| 查看组件可配置参数 | `tccli tke DescribeAddonValues --region <Region> --ClusterId <ClusterId> --AddonName dcu-gpu` | 是 |
| 安装 DCU 组件 | `tccli tke InstallAddon --region <Region> --ClusterId <ClusterId> --AddonName dcu-gpu` | 否 |
| 修改运行模式 | `tccli tke UpdateAddon --region <Region> --ClusterId <ClusterId> --AddonName dcu-gpu --RawValues <json>` | 否 |
| 卸载组件 | `tccli tke DeleteAddon --region <Region> --ClusterId <ClusterId> --AddonName dcu-gpu` | 是 |

### 组件参数说明

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `mode` | `full` | 部署模式：`full`（整卡）或 `vdcu`（虚拟 DCU） |
| `config.exporterPort` | `16080` | dcu-exporter 暴露的 Prometheus 指标端口 |
| `config.logThreshold` | `INFO` | 默认组件日志级别 |
| `vdcuScheduler.nodeSchedulerPolicy` | `binpack` | vDCU 节点级调度策略（仅 vDCU 模式）：`binpack`（集中）/ `spread`（分散） |
| `vdcuScheduler.gpuSchedulerPolicy` | `spread` | vDCU 节点内多卡调度策略（仅 vDCU 模式） |

## 操作步骤

### 步骤 1：查看 DCU 组件可配置参数（控制面）

```bash
tccli tke DescribeAddonValues --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName dcu-gpu
# expected: 返回可配置参数列表（mode、config.exporterPort 等）
```

```json
{
  "Values": "<Values>",
  "DefaultValues": "<DefaultValues>",
  "RequestId": "<RequestId>"
}
```

### 步骤 2：安装 DCU 组件（整卡模式，控制面）

```bash
tccli tke InstallAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName dcu-gpu \
    --AddonVersion <AddonVersion> \
    --RawValues '{"mode":"full"}'
# expected: exit 0，组件安装请求已提交
```

### 步骤 3：安装 DCU 组件（vDCU 虚拟化模式，控制面）

```bash
tccli tke InstallAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName dcu-gpu \
    --AddonVersion <AddonVersion> \
    --RawValues '{"mode":"vdcu","config":{"exporterPort":"16080","logThreshold":"INFO"},"vdcuScheduler":{"nodeSchedulerPolicy":"binpack","gpuSchedulerPolicy":"spread"}}'
# expected: exit 0
```

### 步骤 4：修改运行模式（控制面）

从整卡模式切换为 vDCU 模式：

```bash
tccli tke UpdateAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName dcu-gpu \
    --AddonVersion <AddonVersion> \
    --RawValues '{"mode":"vdcu","vdcuScheduler":{"nodeSchedulerPolicy":"binpack","gpuSchedulerPolicy":"spread"}}'
# expected: exit 0
```

### 步骤 5：节点标签配置（vDCU 模式，数据面）

使用 vDCU 模式时，需为节点打标签：

```bash
kubectl label node <NodeName> dcu=on
# expected: node/<NodeName> labeled
```

> **注意**：kubectl 数据面当前因公网端点被 CAM 策略 strategyId:240463971 拒绝而不可达。以下 kubectl 命令为参考格式，需在内网/VPN 环境执行。

## 验证

### 控制面（tccli）

```bash
tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName dcu-gpu \
    | jq '.Addons[0] | {AddonName, AddonVersion, Status}'
# expected: Status "Running"，AddonName "dcu-gpu"
```

```json
{
  "Addons": "<Addons>",
  "AddonName": "<AddonName>",
  "AddonVersion": "<AddonVersion>",
  "RawValues": "<RawValues>",
  "Phase": "<Phase>",
  "Reason": "<Reason>"
}
```

### 数据面（kubectl）

```bash
kubectl get pods -n kube-system -l app.kubernetes.io/name=dcu-gpu
# expected: DCU 组件 Pod Running

kubectl describe node <NodeName> | grep dcu
# expected: 整卡模式显示 dcu.cn/sugon.com: <数量>
```

```text
NAME  STATUS  AGE
...
```

> **注意**：kubectl 数据面当前因公网端点被 CAM 策略 strategyId:240463971 拒绝而不可达。以下 kubectl 命令为参考格式，需在内网/VPN 环境执行。

## 清理

卸载 DCU 组件前，需先清除使用 DCU 资源的工作负载：

```bash
kubectl delete pods --all -n <namespace> --field-selector spec.nodeName=<dcu-node>
# expected: 相关 Pod 已删除
```

卸载组件：

```bash
tccli tke DeleteAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName dcu-gpu
# expected: exit 0，组件卸载请求已提交

# 验证已卸载
tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName dcu-gpu \
    | jq '.Addons[] | select(.AddonName == "dcu-gpu")'
# expected: 空结果
```

```json
{
  "Addons": "<Addons>",
  "AddonName": "<AddonName>",
  "AddonVersion": "<AddonVersion>",
  "RawValues": "<RawValues>",
  "Phase": "<Phase>",
  "Reason": "<Reason>"
}
```

> **注意：** 卸载组件后，节点上的 `dcu=on` 标签需手动移除：`kubectl label node <NodeName> dcu-`

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 组件 Pod 无法启动 | `kubectl get pods -n kube-system -l app.kubernetes.io/name=dcu-gpu` 查看 Pod 状态；`kubectl describe pod` | 节点未安装海光 DCU 驱动和 HyHAL 运行时环境 | 确认节点已安装 DCU 驱动和 HyHAL；检查 DaemonSet 是否调度到含 DCU 设备的节点 |
| vDCU 模式下 Pod 无法调度 | `kubectl get nodes -l dcu=on` 确认节点标签 | 节点未打上 `dcu=on` 标签或 vdcu-scheduler-plugin 异常 | 为节点打标签 `kubectl label node <NodeName> dcu=on`；检查 vdcu-scheduler-plugin 是否 Running |
| DCU 设备未被识别 | `kubectl logs -n kube-system -l app=dcu-label-node` 查看 DaemonSet 日志 | dcu-label-node 未正确检测到设备 | 检查 dcu-label-node DaemonSet 日志 |
| `Unable to connect to the server` | `kubectl cluster-info` 检查连通性 | 公网端点被 CAM 策略 strategyId:240463971 以 `tke:clusterExtranetEndpoint=true` 条件拒绝 | 通过 IOA/VPN 连接内网，或使用同 VPC CVM 执行 kubectl 命令 |

## 下一步

- [组件的生命周期管理](../组件的生命周期管理/tccli 操作.md) — 组件的安装、升级与卸载
- [扩展组件概述](../扩展组件概述/tccli 操作.md) — 查看其他 GPU 相关组件（nvidia-gpu、QGPU）
- [海光 DCU 官方文档](https://www.hygon.cn/) — DCU 驱动安装与配置

## 控制台替代

访问 [TKE 控制台 - 组件管理](https://console.cloud.tencent.com/tke2/cluster)，点击 **新建**，勾选 **dcu-gpu** 完成安装。
