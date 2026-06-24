# VPC-CNI（eniipamd）组件介绍

> 对照官方：[VPC-CNI（eniipamd）组件介绍](https://cloud.tencent.com/document/product/457/64919) · page_id `64919`

## 概述

**eniipamd** 是 VPC-CNI 网络方案的核心组件，负责将 VPC 弹性网卡（ENI）的 IP 分配给 Pod。组件管理名为 `eniipamd`，由三个 Kubernetes 工作负载组成：`tke-eni-agent`（节点级 DaemonSet）、`tke-eni-ipamd`（集群级 Deployment）和 `tke-eni-ip-scheduler`（仅固定 IP 模式部署的调度扩展）。三者协同实现 ENI 创建/绑定、IP 分配/释放、策略路由配置等网络功能。

### eniipamd 与 VPC-CNI 模式的关系

| 维度 | VPC-CNI 模式 | eniipamd 组件 |
|------|-------------|--------------|
| 概念层级 | 网络方案（决定 Pod IP 来源） | 组件（实现 VPC-CNI 网络能力的软件） |
| 启用方式 | `CreateCluster` 或 `EnableVpcCniNetworkType` | `InstallAddon`（开启 VPC-CNI 后自动安装，也可手动安装/升级） |
| 管理 API | `EnableVpcCniNetworkType` / `DescribeClusters.Cni` | `DescribeAddon` / `InstallAddon` / `UpdateAddon` |
| 查询字段 | `ClusterNetworkSettings.Cni` + `Property.NetworkType` | `AddonName: "eniipamd"` + `Phase` |
| 依赖关系 | 开启 VPC-CNI 后 eniipamd 自动部署 | 必须先开启 VPC-CNI 模式 |

### 组件架构总览

```
┌────────────────────────────────────────────────────┐
│                  TKE VPC-CNI 模式                     │
├────────────────────────────────────────────────────┤
│  控制面 API: EnableVpcCniNetworkType                  │
│      │                                               │
│      ▼                                               │
│  ┌─────────────────────────────────────────────┐    │
│  │              eniipamd 组件                     │    │
│  ├──────────────────┬──────────────────────────┤    │
│  │ tke-eni-ipamd    │ tke-eni-agent (每节点)    │    │
│  │ (Deployment)     │ (DaemonSet)              │    │
│  │ · CRD 管理       │ · CNI 插件部署           │    │
│  │ · ENI 创建/绑定   │ · 策略路由 & ENI 配置    │    │
│  │ · IP 池管理      │ · GRPC IP 分配/释放      │    │
│  │ · 安全组/EIP 管理 │ · IP 垃圾回收            │    │
│  │                  │ · Device Plugin 上报     │    │
│  ├──────────────────┴──────────────────────────┤    │
│  │ tke-eni-ip-scheduler (仅固定 IP 模式)        │    │
│  │ (Deployment)                                │    │
│  │ · 多子网调度策略                              │    │
│  │ · 子网 IP 充足性检查                          │    │
│  └─────────────────────────────────────────────┘    │
└────────────────────────────────────────────────────┘
```

## 前置条件

- [环境准备](../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:DescribeClusters, tke:DescribeAddon, tke:InstallAddon
#    验证：执行 DescribeClusters 确认权限
tccli tke DescribeClusters --region <Region>
# expected: exit 0，返回集群列表（可为空）

# 4. 确认集群已开启 VPC-CNI 模式
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]' \
    | jq '.Clusters[0].ClusterNetworkSettings.Cni'
# expected: true（VPC-CNI 已开启）
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

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看 eniipamd 组件版本/状态 | `tke DescribeAddon`（`AddonName: "eniipamd"`） | 是 |
| 安装 eniipamd 组件 | `tke InstallAddon`（`AddonName: "eniipamd"`） | 否 |
| 升级 eniipamd 组件 | `tke UpdateAddon` | 否 |
| 开启 VPC-CNI 模式（前置步骤） | `EnableVpcCniNetworkType` | 否 |
| 查看 VPC-CNI 子网 | `DescribeClusters → ClusterNetworkSettings.Subnets` | 是 |
| 查看 tke-eni-agent DaemonSet | `kubectl get ds tke-eni-agent -n kube-system` | 是 |
| 查看 tke-eni-ipamd Deployment | `kubectl get deploy tke-eni-ipamd -n kube-system` | 是 |
| 查看 tke-eni-ip-scheduler | `kubectl get deploy tke-eni-ip-scheduler -n kube-system` | 是 |

### 概念 ↔ API 枚举映射

| 组件 | Kubernetes 资源 | 作用 | 部署条件 |
|------|----------------|------|---------|
| `tke-eni-agent` | DaemonSet（每节点一个 Pod） | CNI 插件部署、策略路由、GRPC IP 分配/释放、Device Plugin 资源上报 | VPC-CNI 开启后自动部署 |
| `tke-eni-ipamd` | Deployment | CRD 管理、ENI 创建/绑定/解绑、IP 池管理、安全组管理、EIP 管理 | VPC-CNI 开启后自动部署 |
| `tke-eni-ip-scheduler` | Deployment | 固定 IP 模式的调度扩展：多子网调度、IP 充足性检查 | 仅固定 IP 模式（`EnableStaticIp: true`） |

## 操作步骤

### 查看 eniipamd 组件版本和状态

`InstallAddon` / `DescribeAddon` 中的 `AddonName` 为固定值 `"eniipamd"`：

```bash
tccli tke DescribeAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName eniipamd
# expected: 返回 eniipamd 组件的版本和运行状态
```

**预期输出：**

```json
{
    "Addons": [
        {
            "AddonName": "eniipamd",
            "AddonVersion": "v3.5.0",
            "Phase": "Running",
            "AddonStatus": "Succeeded"
        }
    ]
}
```

> 常见 Phase：`Running`（正常运行）、`Pending`（安装中）、`Failed`（安装失败）。

### InstallAddon eniipamd（手动安装/重装）

通常开启 VPC-CNI 后 eniipamd 会自动部署，但也可通过 `InstallAddon` 手动安装：

```bash
tccli tke InstallAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName eniipamd \
    --AddonVersion "v3.5.0"
# expected: exit 0, RequestId 非空，组件开始安装
```

> **注意：** `InstallAddon` 仅在 VPC-CNI 已开启（`EnableVpcCniNetworkType` 完成）的集群上有效。若集群未开启 VPC-CNI，须先调用 `EnableVpcCniNetworkType`。

### 查看 tke-eni-agent（节点级 DaemonSet）

```bash
kubectl get ds tke-eni-agent -n kube-system
# expected: 就绪数 = 节点数，每节点一个 tke-eni-agent Pod
```

**预期输出：**

```text
NAME             DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
tke-eni-agent    3         3         3       3            3           <none>          30d
```

### 查看 tke-eni-ipamd（控制面 Deployment）

```bash
kubectl get deploy tke-eni-ipamd -n kube-system
# expected: 1/1 Ready
```

**预期输出：**

```text
NAME             READY   UP-TO-DATE   AVAILABLE   AGE
tke-eni-ipamd    1/1     1            1           30d
```

### 查看 tke-eni-ip-scheduler（固定 IP 模式专属）

```bash
kubectl get deploy tke-eni-ip-scheduler -n kube-system
# expected: 固定 IP 模式下 1/1 Ready；非固定 IP 模式下不存在（Error: not found）
```

```text
NAME  STATUS  AGE
...
```

### 查看组件 Pod 日志（排障用）

```bash
# 查看 tke-eni-ipamd 日志
kubectl logs -n kube-system deploy/tke-eni-ipamd --tail=50

# 查看某节点的 tke-eni-agent 日志
kubectl logs -n kube-system ds/tke-eni-agent --tail=50
```

```text
...log output...
```

## 验证

### Control plane（tccli）

- `DescribeAddon` → eniipamd `AddonName: "eniipamd"`，`Phase: "Running"`，`AddonStatus: "Succeeded"`。
- `DescribeClusters` → `Cni: true`，与 VPC-CNI 模式一致。

### Data plane（kubectl）

- `kubectl get ds tke-eni-agent -n kube-system` → 就绪数 = 节点数。
- `kubectl get deploy tke-eni-ipamd -n kube-system` → `1/1 Ready`。
- `kubectl get deploy tke-eni-ip-scheduler -n kube-system` → 固定 IP 模式 `1/1`；非固定 IP 模式不存在（正常）。

### 验证组件健康度总览

| 检查项 | 命令 | 预期 |
|--------|------|------|
| 组件版本与状态 | `tccli tke DescribeAddon` | `Phase: Running`，`AddonVersion >= v3.5.0` |
| tke-eni-agent 覆盖 | `kubectl get ds tke-eni-agent -n kube-system` | DESIRED = CURRENT = READY = 节点数 |
| tke-eni-ipamd 就绪 | `kubectl get deploy tke-eni-ipamd -n kube-system` | READY = 1/1 |
| VPC-CNI 模式开启 | `DescribeClusters → Cni` | `true` |

## 清理

本页为组件说明与只读查询，无额外资源。

若需卸载 eniipamd（将导致 VPC-CNI 网络中断，谨慎操作），可通过控制台「组件管理」页面或 API 执行。不建议在 CLI 中直接卸载。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DescribeAddon` 无 eniipamd | `tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'` 检查 `Cni` | VPC-CNI 未开启 | 先调用 `EnableVpcCniNetworkType` 开启 VPC-CNI 模式，组件会自动部署 |
| `InstallAddon eniipamd` 返回失败 | 检查集群 `Cni` 是否为 `true` | VPC-CNI 模式未开启，组件无法安装 | 先 `EnableVpcCniNetworkType`，确认成功后再 `InstallAddon` |
| `kubectl get ds tke-eni-agent` 返回 `not found` | `kubectl get ns kube-system` 确认命名空间存在 | 组件未部署或已被删除 | `DescribeAddon` 确认组件状态；若为 Failed 则 `UpdateAddon` 重新部署 |
| `kubectl get deploy tke-eni-ip-scheduler` 返回 `not found` | 检查集群是否启用固定 IP（`EnableStaticIp`） | 非固定 IP 模式下不部署此组件（正常现象） | 无需修复 |
| `DescribeAddon` 返回 `AddonStatus: "Failed"` | 查看组件 Pod 日志 | 组件安装或升级失败 | `kubectl logs -n kube-system deploy/tke-eni-ipamd --tail=100` 查看错误信息 |

### 组件运行异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| tke-eni-agent 部分节点未 Ready | `kubectl describe ds tke-eni-agent -n kube-system` 查看事件 | 节点 ENI 配额耗尽、子网 IP 耗尽或节点资源不足 | 检查节点 ENI 配额（`cvm DescribeInstances` → 机型信息）；检查子网 `AvailableIpAddressCount` |
| Pod 无法获取 VPC IP | `kubectl logs -n kube-system deploy/tke-eni-ipamd --tail=100` 查看 ipamd 日志 | 子网 IP 耗尽或 ENI 配额不足 | 添加容器子网（`AddVpcCniSubnets`）或减少 Pod 密度 |
| 组件版本低于 v3.5.0 | `tccli tke DescribeAddon --AddonName eniipamd` 查看版本 | 组件版本过旧，缺少管理能力 | 通过 `UpdateAddon` 升级至 ≥ v3.5.0 |
| 固定 IP Pod 调度到错误子网 | `kubectl get deploy tke-eni-ip-scheduler -n kube-system` | ip-scheduler 未部署或已崩溃 | 确认 `EnableStaticIp: true`；`kubectl logs` 查看调度器日志 |

## 下一步

- [VPC-CNI 模式介绍](../VPC-CNI%20模式/VPC-CNI%20模式介绍/tccli%20操作.md) -- VPC-CNI 原理与子模式
- [多 Pod 共享网卡模式](../VPC-CNI%20模式/多%20Pod%20共享网卡模式/tccli%20操作.md) -- tke-route-eni 共享网卡原理
- [Pod 间独占网卡模式](../VPC-CNI%20模式/Pod%20间独占网卡模式/tccli%20操作.md) -- tke-direct-eni 独占网卡原理
- [固定 IP 使用方法](../VPC-CNI%20模式/固定%20IP%20模式使用说明/固定%20IP%20使用方法/tccli%20操作.md) -- 固定 IP 模式配置
- [VPC-CNI 模式 Pod 数量限制](../VPC-CNI%20模式/VPC-CNI%20模式%20Pod%20数量限制/tccli%20操作.md) -- ENI/IP 配额与 Pod 密度

## 控制台替代

[TKE 控制台 → 集群](https://console.cloud.tencent.com/tke2/cluster) → 组件管理 → eniipamd：查看组件版本、运行状态，支持安装、升级和配置。
