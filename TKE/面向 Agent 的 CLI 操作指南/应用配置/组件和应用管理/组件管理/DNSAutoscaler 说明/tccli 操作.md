# DNSAutoscaler 说明（tccli）

> 对照官方：https://cloud.tencent.com/document/product/457/49305

## 概述

DNSAutoscaler 是 DNS 自动水平伸缩组件。它以 Deployment 形式运行，获取集群的节点数和核心数，根据预设的伸缩策略自动调整 DNS 副本数。支持两种伸缩模式：**Linear（线性模式）** 和 **Ladder（阶梯模式）**。

**Linear 模式公式**

- `replicas = max(ceil(cores × 1/coresPerReplica), ceil(nodes × 1/nodesPerReplica))`
- `replicas = min(replicas, max)`
- `replicas = max(replicas, min)`

**Linear 模式示例**

```yaml
data:
  linear: |-
    {
      "coresPerReplica": 2,
      "nodesPerReplica": 1,
      "min": 1,
      "max": 100,
      "preventSinglePointFailure": true
    }
```

**Ladder 模式示例**

根据预定义的阈值阶梯表，取节点数和核心数分别计算所得副本数的较大值。

```yaml
data:
  ladder: |-
    {
      "coresToReplicas":
      [
        [ 1, 1 ],
        [ 64, 3 ],
        [ 512, 5 ],
        [ 1024, 7 ],
        [ 2048, 10 ],
        [ 4096, 15 ]
      ],
      "nodesToReplicas":
      [
        [ 1, 1 ],
        [ 2, 2 ]
      ]
    }
```

**示例**：集群 100 节点 / 400 核心，nodesToReplicas 得 2（100 > 2 向下取整），coresToReplicas 得 3（400 落入 64-512 区间向下取整），最终副本数取大值为 3。

## 前置条件

阅读 [环境准备](../../../../环境准备.md) 与 [连接集群](../../../../集群配置/集群管理/连接集群/tccli 操作.md)。

```bash
tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'
# expected: ClusterStatus "Running"
```

**集群与版本要求**

- Kubernetes 版本 **≥ 1.8**。
- 集群的 DNS 服务器工作负载必须为 `deployment/coredns`。

**注意事项**

CoreDNS 的水平伸缩可能导致部分副本短暂不可用。强烈建议在安装本组件前完成相关优化配置，以最大化 DNS 服务的可用性。参见「配置 CoreDNS 平滑升级」文档。

**组件权限（Kubernetes RBAC）**

```yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: tke-dns-autoscaler
rules:
  - apiGroups:
      - ""
    resources:
      - nodes
    verbs:
      - list
      - watch
  - apiGroups:
      - ""
    resources:
      - replicationcontrollers/scale
    verbs:
      - get
      - update
  - apiGroups:
      - extensions
      - apps
    resources:
      - deployments/scale
      - replicasets/scale
    verbs:
      - get
      - update
  - apiGroups:
      - ""
    resources:
      - configmaps
    verbs:
      - get
      - create
```

**功能说明**

| 功能 | 涉及对象 | 操作 |
|------|----------|------|
| 监控集群中 Node 资源变化 | node | list/watch |
| 修改 coredns 的 Deployment 副本数 | replicationcontrollers/scale, deployments/scale, replicasets/scale | get/update |
| 获取 ConfigMap 参数配置；无参数时创建默认 ConfigMap | configmap | get/create |

**部署的 Kubernetes 对象**

| 对象名称 | 类型 | 资源占用 | 命名空间 |
|----------|------|----------|----------|
| tke-dns-autoscaler | Deployment | 每节点 20m CPU，10Mi 内存 | kube-system |
| dns-autoscaler | ConfigMap | — | kube-system |
| tke-dns-autoscaler | ServiceAccount | — | kube-system |
| tke-dns-autoscaler | ClusterRole | — | — |
| tke-dns-autoscaler | ClusterRoleBinding | — | — |

**CAM 权限**

- `tke:DescribeClusters`、`tke:DescribeAddon`、`tke:InstallAddon`、`tke:UpdateAddon`、`tke:DeleteAddon`。

## 控制台与 CLI 参数映射

| 操作 | 控制台 | tccli |
|------|--------|-------|
| 查看集群状态 | 集群列表 | `tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'` |
| 查看组件 | 组件管理 | `tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName DNSAutoscaler` |
| 安装组件 | 组件管理 - 安装 | `tccli tke InstallAddon --region <Region> --ClusterId <ClusterId> --AddonName DNSAutoscaler --AddonVersion <version>` |
| 更新组件 | 组件管理 - 更新 | `tccli tke UpdateAddon --region <Region> --ClusterId <ClusterId> --AddonName DNSAutoscaler --AddonVersion <version>` |
| 卸载组件 | 组件管理 - 卸载 | `tccli tke DeleteAddon --region <Region> --ClusterId <ClusterId> --AddonName DNSAutoscaler` |

**组件配置参数**

| 参数 | 说明 | 类型 | 默认值 |
|------|------|------|--------|
| `global.image.host` | 镜像仓库地址 | string | `ccr.ccs.tencentyun.com` |
| `global.podSpec.tolerations` | 额外容忍（追加到默认 CriticalAddonsOnly 后） | list | `[]` |
| `global.podSpec.priorityClassName` | Pod 优先级类名 | string | `system-cluster-critical` |
| `autoscaler.replicas` | Autoscaler 自身副本数 | int | `1` |
| `autoscaler.resources.requests.cpu` | CPU 请求 | string | `20m` |
| `autoscaler.resources.requests.memory` | 内存请求 | string | `10Mi` |
| `autoscaler.resources.limits.cpu` | CPU 限制（空表示不设） | string | `""` |
| `autoscaler.resources.limits.memory` | 内存限制（空表示不设） | string | `""` |

## 操作步骤

### 1. 安装 DNSAutoscaler 组件

默认使用 Ladder 模式：

```bash
tccli tke InstallAddon \
    --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName DNSAutoscaler \
    --AddonVersion <version>
# expected: RequestId 返回，Addon 进入 Running
```

**默认 Ladder 策略配置**（写入 `kube-system` 的 `configmap/dns-autoscaler`）：

```yaml
data:
  ladder: |-
    {
      "coresToReplicas": 
      [
        [ 1, 1 ],
        [ 128, 3 ],
        [ 512, 4 ]
      ],
      "nodesToReplicas": 
      [
        [ 1, 1 ],
        [ 2, 2 ]
      ]
    }
```

详细配置参考 [cluster-proportional-autoscaler](https://github.com/kubernetes-sigs/cluster-proportional-autoscaler)。

### 2. 修改伸缩策略

> **注意**：kubectl 数据面当前因公网端点被 CAM 策略 strategyId:240463971 拒绝而不可达。以下 kubectl 命令为参考格式，需在内网/VPN 环境执行。

**切换为 Linear 模式：**

```bash
kubectl edit cm dns-autoscaler -n kube-system
```

将 `data` 部分替换为：

```yaml
data:
  linear: |-
    {
      "coresPerReplica": 2,
      "nodesPerReplica": 1,
      "min": 1,
      "max": 100,
      "preventSinglePointFailure": true
    }
```

保存后自动生效。

## 验证

```bash
tccli tke DescribeAddon \
    --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName DNSAutoscaler
# expected: Phase "Running"
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

> **注意**：kubectl 数据面当前因公网端点被 CAM 策略 strategyId:240463971 拒绝而不可达。以下 kubectl 命令为参考格式，需在内网/VPN 环境执行。

验证 Deployment 运行状态：

```bash
kubectl get deploy tke-dns-autoscaler -n kube-system
# expected: READY 副本可用
```

```text
NAME  STATUS  AGE
...
```

查看当前伸缩策略配置：

```bash
kubectl get cm dns-autoscaler -n kube-system -o yaml
# expected: 显示 Ladder 配置
```

```text
NAME  STATUS  AGE
...
```

验证 CoreDNS 副本数已被自动调整：

```bash
kubectl get deploy coredns -n kube-system
# expected: 副本数已按策略调整
```

```text
NAME  STATUS  AGE
...
```

## 清理

```bash
tccli tke DeleteAddon \
    --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName DNSAutoscaler
# expected: RequestId 返回，组件卸载完成
```

> **注意**：kubectl 数据面当前因公网端点被 CAM 策略 strategyId:240463971 拒绝而不可达。以下 kubectl 命令为参考格式，需在内网/VPN 环境执行。

```bash
kubectl get deploy tke-dns-autoscaler -n kube-system
# expected: Error from server (NotFound): deployments.apps "tke-dns-autoscaler" not found
```

```text
NAME  STATUS  AGE
...
```

> **计费提醒**：DNSAutoscaler 组件本身不产生额外费用。

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 副本数未自动调整 | 检查 `kubectl get deploy -n kube-system` 确认 CoreDNS 部署名称 | 集群 DNS 工作负载不是 `deployment/coredns`（如为 `deployment/kube-dns`） | 迁移到 CoreDNS 后再使用本组件 |
| ConfigMap 修改后不生效 | 检查 autoscaler 是否热加载配置 | autoscaler 未热加载 | 重启 autoscaler Pod：`kubectl rollout restart deploy tke-dns-autoscaler -n kube-system` |
| CoreDNS 副本频繁变化 | 检查伸缩策略阈值 | 策略过于敏感，临界点频繁触发 | 调整 Ladder 阶梯表或 Linear 参数，增大阈值区间 |
| Unable to connect（kubectl） | 检查 CAM 策略 | 公网端点被 CAM 策略 strategyId:240463971 拒绝（`tke:clusterExtranetEndpoint=true`） | 调整 CAM 策略或通过内网/VPN 访问 |

## 下一步

- [配置 CoreDNS 平滑升级](https://cloud.tencent.com/document/product/457/xxxxx)
- [cluster-proportional-autoscaler 官方文档](https://github.com/kubernetes-sigs/cluster-proportional-autoscaler)
- [NodeLocalDNSCache 说明（tccli）](../NodeLocalDNSCache%20说明/tccli%20操作.md)
- [TKE 组件管理概述](https://cloud.tencent.com/document/product/457/51082)
- [CoreDNS 自定义配置](https://cloud.tencent.com/document/product/457/41893)

## 控制台替代

可在 TKE 控制台对应集群的「组件管理」页面安装、更新、卸载 DNSAutoscaler 组件，操作等效于上述 `InstallAddon`/`UpdateAddon`/`DeleteAddon` 调用。入口：[容器服务控制台 - 组件管理](https://console.cloud.tencent.com/tke2/cluster?rid=<Region>)。
