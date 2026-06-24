# NodeLocalDNSCache 说明（tccli）

> 对照官方：https://cloud.tencent.com/document/product/457/49423

## 概述

NodeLocal DNSCache 以 DaemonSet 形式在集群节点上运行 DNS 缓存代理，用于提升 DNS 性能。在 ClusterFirst DNS 模式下，Pod 原本通过 kube-dns serviceIP 进行 DNS 查询，其中 iptables 规则（经由 kube-proxy）将查询转换到 kube-dns/CoreDNS 端点。安装 NodeLocal DNSCache 后，Pod 直接访问同节点上的 DNS 缓存代理（地址 `169.254.20.10`），避免了 iptables DNAT 规则和连接跟踪。本地缓存代理在缓存未命中时查询 kube-dns 服务（默认 `cluster.local` 后缀域名）。

## 前置条件

阅读 [环境准备](../../../../环境准备.md) 与 [连接集群](../../../../集群配置/集群管理/连接集群/tccli 操作.md)。

```bash
tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'
# expected: ClusterStatus "Running"
```

**集群与版本要求**

- Kubernetes 版本 **≥ 1.14**。
- 集群网络模式必须为 **VPC-CNI**，不支持 GlobalRouter（GR）模式。
- VPC-CNI 支持 kube-proxy iptables 和 ipvs 两种模式。
- 集群中不得修改 DNS 服务工作负载名称或标签。`kube-system` 命名空间中必须存在：
  - `service/kube-dns`
  - `deployment/kube-dns` 或 `deployment/coredns`，且带有标签 `k8s-app: kube-dns`

**IPVS 模式额外要求**

- **独立集群 IPVS 模式**：`add-pod-eni-ip-limit-webhook` ClusterRole 需具备以下权限：

```yaml
- apiGroups:
  - ""
  resources:
  - configmaps
  - secrets
  - namespaces
  - services
  verbs:
  - list
  - watch
  - get
  - create
  - update
  - delete
  - patch
```

- **独立集群和托管集群 IPVS 模式**：`tke-eni-ip-webhook` 命名空间中的 `add-pod-eni-ip-limit-webhook` Deployment 镜像版本须 **≥ v0.0.6**。

**不支持场景**

| 受限场景 | 原因 |
|----------|------|
| 超级节点（EKS）上的 Pod | 超级节点没有真实宿主机运行 NodeLocal DNS Cache。新版 EKS 会自动忽略 169.254.20.10；旧版可能出现 DNS 解析异常 |
| 独占网卡模式（`tke-direct-eni`）Pod | Pod 网络流量绕过宿主机网络栈，无法访问 Local DNS 缓存服务 169.254.20.10 |
| Cilium Overlay 网络模式 | Cilium 有自己的 DNS 处理机制，与 NodeLocal DNS Cache 不兼容 |
| GlobalRouter（GR）网络模式 | GR 集群不包含 `tke-eni-ip-webhook` 组件（仅 VPC-CNI 有），因此不支持 |

**部署的 Kubernetes 对象**

| 对象名称 | 类型 | 资源占用 | 命名空间 |
|----------|------|----------|----------|
| node-local-dns | DaemonSet | 每节点 50m CPU，5MiB 内存 | kube-system |
| kube-dns-upstream | Service | — | kube-system |
| node-local-dns | ServiceAccount | — | kube-system |
| node-local-dns | ConfigMap | — | kube-system |

**CAM 权限**

- `tke:DescribeClusters`、`tke:DescribeAddon`、`tke:InstallAddon`、`tke:UpdateAddon`、`tke:DeleteAddon`。

## 控制台与 CLI 参数映射

| 操作 | 控制台 | tccli |
|------|--------|-------|
| 查看集群状态 | 集群列表 | `tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'` |
| 查看组件 | 组件管理 | `tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName NodeLocalDNSCache` |
| 安装组件 | 组件管理 - 安装 | `tccli tke InstallAddon --region <Region> --ClusterId <ClusterId> --AddonName NodeLocalDNSCache --AddonVersion <version>` |
| 更新组件 | 组件管理 - 更新 | `tccli tke UpdateAddon --region <Region> --ClusterId <ClusterId> --AddonName NodeLocalDNSCache --AddonVersion <version>` |
| 卸载组件 | 组件管理 - 卸载 | `tccli tke DeleteAddon --region <Region> --ClusterId <ClusterId> --AddonName NodeLocalDNSCache` |

## 操作步骤

### 1. 安装 NodeLocalDNSCache 组件

```bash
tccli tke InstallAddon \
    --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName NodeLocalDNSCache \
    --AddonVersion <version>
# expected: RequestId 返回，Addon 进入 Running
```

### 2. 配置 DNSConfig 自动注入

DNSConfig 自动注入由 `tke-eni-ip-webhook` 组件实现，而非 NodeLocal DNS Cache 本身。集群级别的注入开关在 `tke-eni-ip-webhook` 组件设置中配置（默认 `localdns=false`）。

**三级配置优先级（从高到低）**

| 优先级 | 级别 | 配置方式 | 说明 |
|--------|------|----------|------|
| 最高 | Pod 级别 | Pod 标签 `localdns-injector=enabled\|disabled` | 精细化逐 Pod 控制 |
| 中 | Namespace 级别 | Namespace 标签 `localdns-injector=enabled\|disabled` | 按命名空间批量控制 |
| 最低 | 集群级别 | 在 TKE 控制台组件管理中配置 `tke-eni-ip-webhook` 的 `localdns` 参数 | 集群默认行为（默认 `false`） |

> **注意**：kubectl 数据面当前因公网端点被 CAM 策略 strategyId:240463971 拒绝而不可达。以下 kubectl 命令为参考格式，需在内网/VPN 环境执行。

**场景 1：集群默认禁用（localdns=false），为特定命名空间启用：**

```bash
kubectl label namespace production localdns-injector=enabled
```

**场景 2：集群默认启用（localdns=true），为特定命名空间禁用：**

```bash
kubectl label namespace sensitive-app localdns-injector=disabled
```

**场景 3：命名空间已启用，为特定 Pod 禁用：**

```bash
kubectl run special-pod \
    --image=nginx \
    --labels="localdns-injector=disabled"
```

**场景 4：集群和命名空间均已禁用，为特定 Pod 启用：**

```bash
kubectl run test-pod \
    --image=nginx \
    --labels="localdns-injector=enabled"
```

**推荐 CoreDNS 配置**

安装 NodeLocalDNSCache 后，推荐修改 CoreDNS 配置如下：

```
template ANY HINFO . {
    rcode NXDOMAIN
}

forward . /etc/resolv.conf {
    prefer_udp
}
```

## 验证

```bash
tccli tke DescribeAddon \
    --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName NodeLocalDNSCache
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

验证 DaemonSet 是否在所有节点上运行：

```bash
kubectl get ds node-local-dns -n kube-system
# expected: DESIRED/CURRENT/READY 一致
```

```text
NAME  STATUS  AGE
...
```

验证 DNS 缓存地址可访问（在任意 Pod 中执行）：

```bash
kubectl run -it --rm debug --image=busybox --restart=Never -- nslookup kubernetes.default.svc.cluster.local 169.254.20.10
# expected: Server: 169.254.20.10，解析成功
```

## 清理

```bash
tccli tke DeleteAddon \
    --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName NodeLocalDNSCache
# expected: RequestId 返回，组件卸载完成
```

> **注意**：kubectl 数据面当前因公网端点被 CAM 策略 strategyId:240463971 拒绝而不可达。以下 kubectl 命令为参考格式，需在内网/VPN 环境执行。

```bash
kubectl get ds node-local-dns -n kube-system
# expected: Error from server (NotFound): daemonsets.apps "node-local-dns" not found
```

```text
NAME  STATUS  AGE
...
```

> **计费提醒**：NodeLocalDNSCache 组件本身不产生额外费用。

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Pod 无法解析 DNS | 检查集群网络模式 `kubectl get cm -n kube-system tke-eni-ipam-controller-config -o yaml` | 非 VPC-CNI（GR 模式不支持） | 卸载组件或切换至 VPC-CNI |
| 超级节点上的 Pod DNS 异常 | 检查 EKS 版本 | 旧版 EKS 未忽略 169.254.20.10 | 升级 EKS 版本；如无法升级，将超级节点 Pod 从注入范围排除 |
| DNSConfig 未注入到 Pod | 检查 `tke-eni-ip-webhook` 版本与开关 | 集群级开关未启用（`localdns=false`）或镜像版本 < v0.0.6 | 确认镜像 ≥ v0.0.6，在组件管理中设置 `localdns=true` |
| Cilium 模式下 DNS 异常 | 检查网络插件 | Cilium 与 NodeLocal DNS Cache 不兼容 | 卸载 NodeLocalDNSCache 组件 |
| Unable to connect（kubectl） | 检查 CAM 策略 | 公网端点被 CAM 策略 strategyId:240463971 拒绝（`tke:clusterExtranetEndpoint=true`） | 调整 CAM 策略或通过内网/VPN 访问 |

## 下一步

- [DNSAutoscaler 说明（tccli）](../DNSAutoscaler%20说明/tccli%20操作.md)
- [CoreDNS 配置自定义域名解析](https://cloud.tencent.com/document/product/457/41893)
- [Kubernetes 官方 NodeLocal DNSCache 文档](https://kubernetes.io/docs/tasks/administer-cluster/nodelocaldns)
- [TKE VPC-CNI 网络模式](https://cloud.tencent.com/document/product/457/34993)
- [ip-masq-agent 说明](https://cloud.tencent.com/document/product/457/121346)

## 控制台替代

可在 TKE 控制台对应集群的「组件管理」页面安装、更新、卸载 NodeLocalDNSCache 组件，操作等效于上述 `InstallAddon`/`UpdateAddon`/`DeleteAddon` 调用。入口：[容器服务控制台 - 组件管理](https://console.cloud.tencent.com/tke2/cluster?rid=<Region>)。
