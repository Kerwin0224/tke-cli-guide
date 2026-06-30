# tke-eni-ip-webhook 说明（tccli）

> 对照官方：[tke-eni-ip-webhook 说明](https://cloud.tencent.com/document/product/457/123793) · page_id `123793`

## 概述

tke-eni-ip-webhook 是腾讯云 TKE 在 VPC-CNI 网络模式下用于创建 Pod 的钩子组件。该组件在每次创建 Pod 时会接收 kube-apiserver 请求，并向新 Pod 添加对应的网络资源请求，实现对用户透明的网络资源调度。

**部署在集群内的 Kubernetes 对象（仅独立集群）：**

以下对象仅部署在 TKE 独立集群（Master 和 Etcd 由用户自行维护）中。托管集群（Master 和 Etcd 由平台维护）中，这些对象由平台管理，不会部署到用户集群。

| Kubernetes 对象名称 | 类型 | 资源占用 | 所属 Namespaces |
|---|---|---|---|
| add-pod-eni-ip-limit-webhook | ServiceAccount | - | tke-eni-ip-webhook |
| add-pod-eni-ip-limit-webhook | ClusterRole | - | - |
| add-pod-eni-ip-limit-webhook | ClusterRoleBinding | - | - |
| add-pod-eni-ip-limit-webhook | Service | - | tke-eni-ip-webhook |
| add-pod-eni-ip-limit-webhook | Deployment | 0.01 核 CPU, 30MB 内存 | tke-eni-ip-webhook |

**使用限制：** 仅适用于 VPC-CNI 网络模式下的 TKE 标准集群。

## 前置条件

- [环境准备](../../../../环境准备.md)
- 已 [连接集群](../../../../集群配置/集群管理/连接集群/tccli 操作.md)，kubectl 可正常访问 APIServer
- 集群网络模式为 VPC-CNI

确认集群状态正常：

```bash
tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'
```

```json
{
  "TotalCount": 0,
  "Clusters": [],
  "ClusterId": "<ClusterId>",
  "ClusterName": "<ClusterName>",
  "ClusterDescription": "<ClusterDescription>",
  "ClusterVersion": "<ClusterVersion>",
  "ClusterOs": "<ClusterOs>",
  "ClusterType": "<ClusterType>"
}
```

**CAM 权限要求：**

| 操作 | 所需权限 |
|------|---------|
| 安装组件 | `tke:InstallAddon` |
| 查看组件状态 | `tke:DescribeAddon` |
| 查看组件配置 | `tke:DescribeAddonValues` |
| 更新组件 | `tke:UpdateAddon` |
| 卸载组件 | `tke:DeleteAddon` |

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|---|---|---|
| 查看集群列表 | `tccli tke DescribeClusters` | 是 |
| 查看组件详情 | `tccli tke DescribeAddon --AddonName tke-eni-ip-webhook` | 是 |
| 安装组件 | `tccli tke InstallAddon --AddonName tke-eni-ip-webhook --AddonVersion <AddonVersion>` | 否（已安装时报错 ResourceAlreadyExists） |
| 更新组件配置 | `tccli tke DescribeAddonValues --AddonName tke-eni-ip-webhook` + `tccli tke UpdateAddon --AddonName tke-eni-ip-webhook` | 否 |
| 卸载组件 | `tccli tke DeleteAddon --AddonName tke-eni-ip-webhook` | 是（已删除时返回 NotFound） |

## 操作步骤

### 1. 安装 tke-eni-ip-webhook 组件（控制面）

```bash
tccli tke InstallAddon \
    --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName tke-eni-ip-webhook \
    --AddonVersion <AddonVersion>
```

输出示例：

```json
{
    "Response": {
        "RequestId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
    }
}
```

### 2. 验证网络资源自动注入（数据面）

> **注意**：kubectl 数据面当前因公网端点被 CAM 策略 strategyId:240463971 拒绝而不可达。以下 kubectl 命令为参考格式，需在内网/VPN 环境执行。

以下 kubectl 验证仅适用于 TKE **独立集群**。托管集群中该组件由平台管理，相关对象不在用户集群中。

创建一个测试 Pod，不手动配置网络资源请求：

```bash
kubectl run test-eni \
    --image nginx:latest \
    --restart Never
```

检查 Pod 是否被自动注入网络资源请求：

```bash
kubectl get pod test-eni -o yaml | \
    grep -A 6 'resources:'
```

输出示例（共享网卡模式）：

```yaml
    resources:
      requests:
        tke.cloud.tencent.com/eni-ip: "1"
      limits:
        tke.cloud.tencent.com/eni-ip: "1"
```

**组件支持的自动注入网络资源类型：**

| 网络模式 | Resource Key |
|---|---|
| 共享网卡（Shared ENI） | `tke.cloud.tencent.com/eni-ip` |
| 独立网卡/独占网卡（Independent/Exclusive ENI） | `tke.cloud.tencent.com/direct-eni` |
| 中继子网卡（Rel

```json
{
  "Addons": [],
  "AddonName": "<AddonName>",
  "AddonVersion": "<AddonVersion>",
  "RawValues": "<RawValues>",
  "Phase": "<Phase>",
  "Reason": "<Reason>",
  "CreateTime": "<CreateTime>",
  "RequestId": "<RequestId>"
}
```ay Sub-ENI） | `tke.cloud.tencent.com/sub-eni` |
| 弹性公网 IP（Elastic Public IP） | `tke.cloud.tencent.com/eip` |

## 验证

### Control plane（tccli）

```bash
tccli tke DescribeAddon \
    --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName tke-eni-ip-webhook
```

```json
{
  "RequestId": "..."
}
```

**预期输出：** 组件状态为 `Running`，AddonName 为 `tke-eni-ip-webhook`。

### Data plane（kubectl）

> **注意**：kubectl 数据面当前因公网端点被 CAM 策略 strategyId:240463971 拒绝而不可达。以下 kubectl 命令为参考格式，需在内网/VPN 环境执行。

> **注意：** 以下 kubectl 验证仅适用于 TKE **独立集群**。托管集群中该组件由平台管理，相关对象不在用户集群中，请使用 `tccli tke DescribeAddon` 确认组件状态。

检查 Deployment 运行状态：

```bash
kubectl get deployment add-pod-eni-ip-limit-webhook \
    -n tke-eni-ip-webhook
```

```text
NAME  STATUS  AGE
...
```

**预期输出：** `READY` = `1/1`。

检查 MutatingWebhookConfiguration：

```bash
kubectl get mutatingwebhookconfigurations \
    add-pod-eni-ip-limit-webhook
```

## 清理

> **计费提醒：** tke-eni-ip-webhook 组件本身不产生额外费用。卸载后新创建的 Pod 将不再自动注入网络资源请求，可能导致 VPC-CNI 模式下 Pod 无法正确获取 IP 而调度失败，请确保集群网络模式已切换为非 VPC-CNI 后再

```json
{
  "Addons": [],
  "AddonName": "<AddonName>",
  "AddonVersion": "<AddonVersion>",
  "RawValues": "<RawValues>",
  "Phase": "<Phase>",
  "Reason": "<Reason>",
  "CreateTime": "<CreateTime>",
  "RequestId": "<RequestId>"
}
```卸载。

卸载前确认集群网络模式：

> **注意**：kubectl 数据面当前因公网端点被 CAM 策略 strategyId:240463971 拒绝而不可达。以下 kubectl 命令为参考格式，需在内网/VPN 环境执行。

```bash
kubectl get nodes -o yaml | \
    grep tke.cloud.tencent.com/eni-ip
```

```text
NAME  STATUS  AGE
...
```

如有输出，说明集群仍在使用 VPC-CNI 模式，**请勿卸载**该组件。

卸载组件：

```bash
tccli tke DeleteAddon \
    --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName tke-eni-ip-webhook
```

验证已卸载：

```bash
tccli tke DescribeAddon \
    --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName tke-eni-ip-webhook
```

```json
{
  "RequestId": "..."
}
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Pod 创建后未被注入 `tke.cloud.tencent.com/eni-ip` 资源请求 | `kubectl describe mutatingwebhookconfigurations add-pod-eni-ip-limit-webhook` 检查配置状态 | webhook 未正常运行或 MutatingWebhookConfiguration 缺失 | 确认 webhook Pod 运行正常；重新安装组件 |
| 托管集群中 `kubectl get` 找不到 webhook 相关对象 | `tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName tke-eni-ip-webhook` 确认组件状态 | 托管集群由平台管理，对象不在用户命名空间中 | 仅在独立集群中使用 kubectl 验证，托管集群通过 `tccli tke DescribeAddon` 确认状态 |
| 安装后 VPC-CNI 模式下 Pod 仍 Pending | `kubectl describe node <NodeName> \| grep tke.cloud.tencent.com/eni-ip` 查看节点 ENI IP 配额 | 网络资源不足或配额已达上限 | 扩容 ENI IP 配额或增加节点 |
| webhook Deployment CrashLoopBackOff | `kubectl logs -n tke-eni-ip-webhook deployment/add-pod-eni-ip-limit-webhook` 查看错误日志 | 独立集群中 kube-apiserver 网络不可达 | 确认 API Server 连通性 |
| `Unable to connect to the server` | `kubectl cluster-info` 检查连通性 | 公网端点被 CAM 策略 strategyId:240463971 以 `tke:clusterExtranetEndpoint=true` 条件拒绝 | 通过 IOA/VPN 连接内网，或使用同 VPC CVM 执行 kubectl 命令 |

## 下一步

- [VPC-CNI 网络模式说明](https://cloud.tencent.com/document/product/457/xxxxx) — 了解 VPC-CNI 网络原理和配额限制
- [tke-eni 说明](https://cloud.tencent.com/document/product/457/xxxxx) — ENI 分配组件说明
- [使用 TKE 独立集群](https://cloud.tencent.com/document/product/457/xxxxx) — 独立集群部署与管理

## 控制台替代

[组件管理 - 新建组件](https://console.cloud.tencent.com/tke2/cluster)
