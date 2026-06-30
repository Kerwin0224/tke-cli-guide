# eNP（ebpf Network Policies）使用说明（tccli）

> 对照官方：[eNP（ebpf Network Policies）使用说明](https://cloud.tencent.com/document/product/457/115877) · page_id `115877`

## 概述

eNP（eBPF Network Policies）组件是基于 eBPF 技术的 NetworkPolicy 实现。通过在 Pod 所在宿主机 veth 的 tc 上挂载 eBPF 程序，在网卡处理报文时根据五元组（源地址、源端口、目的地址、目的端口、传输协议）匹配 NetworkPolicy 规则，判断报文放通或丢弃。eNP 引入 CRD 存储 NetworkPolicy 关联实体，tke-network-policy-controller 为每个 Pod 计算 Policy-CRD，agent 通过 DaemonSet 注入方式在 Pod 网卡上挂载 filter eBPF 程序。

### 组件架构

1. tke-network-policy-controller 以 Deployment 部署在控制面，list/watch NetworkPolicy 并为每个 Pod 计算 Policy-CRD。
2. tke-network-policy-agent 通过 DaemonSet 注入方式在业务 Pod 的 network namespace 内默认网卡（eth0）上挂载 filter eBPF 程序。
3. agent 根据 Policy-CRD 在 eBPF map 中维护放通规则，eBPF 程序据此放通或丢弃报文。

### 组件架构 — 限制

1. OS 版本：TencentOS Server 3.1 及 3.2
2. TKE 1.22 及以上集群版本
3. 网络插件：共享网卡 VPC-CNI
4. 仅支持超级节点
5. 仅 IPv4 集群，暂不支持 IPv4/IPv6 双栈

### eNP 组件 e2e 测试情况

eNP 通过了 Kubernetes 社区 NetworkPolicy e2e 测试（基于 release-1.26），通过 70+ 个测试 case，覆盖场景包括：

- default-deny-ingress / default-deny-all 策略
- 基于 PodSelector / NamespaceSelector 的允许规则
- MatchExpressions 支持
- 多策略叠加、策略更新、策略删除
- Ingress/Egress 独立控制
- TCP/UDP/SCTP 协议区分

不支持的场景：

1. Pod 为 Service 后端时，通过 egress 放通对该 Pod 访问，如通过 Service IP 访问会被拒绝（底层 eBPF 在 Pod 网卡 egress 处实现，Service IP 尚未 DNAT）
2. 不支持 named port 类型规则

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

### 限制条件

| 限制项 | 要求 |
|--------|------|
| OS | TencentOS Server 3.1 / 3.2 |
| Kubernetes 版本 | TKE 1.22 及以上 |
| 网络插件 | 共享网卡 VPC-CNI |
| 节点类型 | 仅支持超级节点 |
| IP 协议 | 仅 IPv4，暂不支持双栈 |

### CAM 权限

| 操作 | API 权限 |
|------|----------|
| 安装组件 | `tke:InstallAddon` |
| 查询组件信息 | `tke:DescribeAddon` |
| 查询组件参数 | `tke:DescribeAddonValues` |
| 更新组件 | `tke:UpdateAddon` |
| 删除组件 | `tke:DeleteAddon` |

## 控制台与 CLI 参数映射

核心 tccli 命令：

| 控制台操作 | tccli 命令 | 说明 |
|-----------|-----------|------|
| 查询集群状态 | `tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'` | 验证集群 Running |
| 查询组件信息 | `tccli tke DescribeAddon --ClusterId <ClusterId> --AddonName enp` | 获取组件版本与状态 |
| 安装组件 | `tccli tke InstallAddon --ClusterId <ClusterId> --AddonName enp --AddonVersion <AddonVersion>` | 安装 eNP |
| 更新组件 | `tccli tke UpdateAddon --ClusterId <ClusterId> --AddonName enp --AddonVersion <AddonVersion>` | 升级版本或更新参数 |
| 删除组件 | `tccli tke DeleteAddon --ClusterId <ClusterId> --AddonName enp` | 卸载组件 |

控制台与 kubectl 操作映射：

| 控制台 | tccli / kubectl |
|--------|----------------|
| 安装 eNP 组件 | `InstallAddon --AddonName enp` |
| 创建 NetworkPolicy | `kubectl apply -f network-policy.yaml` |
| 查看 NetworkPolicy | `kubectl get networkpolicies` |
| 查看 eNP 组件状态 | `DescribeAddon --AddonName enp` |
| 查看 eNP CRD | `kubectl get policy-crd` |

## 操作步骤

### 1. 安装 eNP 组件（控制面）

```bash
tccli tke InstallAddon --region <Region> \
  --ClusterId <ClusterId> \
  --AddonName enp \
  --AddonVersion <AddonVersion>
```

### 2. 查看组件部署架构（数据面）

> **注意**：kubectl 数据面当前因公网端点被 CAM 策略 strategyId:240463971 拒绝而不可达。以下 kubectl 命令为参考格式，需在内网/VPN 环境执行。

组件架构：

1. tke-network-policy-controller 以 Deployment 部署在控制面，list/watch NetworkPolicy 并为每个 Pod 计算 Policy-CRD
2. tke-network-policy-agent 通过 DaemonSet 注入方式在业务 Pod 的 network namespace 内默认网卡（eth0）上挂载 filter eBPF 程序
3. agent 根据 Policy-CRD 在 eBPF map 中维护放通规则，eBPF 程序据此放通或丢弃报文

查看组件状态：

```bash
kubectl get deploy -n kube-system tke-network-policy-controller
kubectl get ds -n kube-system tke-network-policy-agent
```

### 3. 创建 NetworkPolicy（数据面）

示例：default-deny-ingress 策略：

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: <namespace>
spec:
  podSelector: {}
  policyTypes:
  - Ingress
```

```bash
kubectl apply -f default-deny-ingress.yaml
```

### 4. 允许指定 Pod 通信的 NetworkPolicy（数据面）

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-specific
  namespace: <namespace>
spec:
  podSelector:
    matchLabels:
      app: <app-label>
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: frontend
    ports:
    - protocol: TCP
      port: 80
```

```bash
kubectl apply -f allow-specific.yaml
```

```text
# command executed successfully
```

### 5. 查看 NetworkPolicy（数据面）

```bash
kubectl get networkpolicies -A
kubectl describe networkpolicy <policy-name> -n <namespace>
```

```text
NAME  STATUS  AGE
...
```

## 验证

### 控制面（tccli）

```bash
tccli tke DescribeAddon --region <Region> \
  --ClusterId <ClusterId> \
  --AddonName enp
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

### 数据面（kubectl）

> **注意**：kubectl 数据面当前因公网端点被 CAM 策略 strategyId:240463971 拒绝而不可达。以下 kubectl 命令为参考格式，需在内网/VPN 环境执行。

```bash
kubectl get networkpolicies -A
kubectl get pods -n kube-system | grep tke-network-policy
```

```text
NAME  STATUS  AGE
...
```

## 清理

### 数据面（kubectl）

> **注意**：kubectl 数据面当前因公网端点被 CAM 策略 strategyId:240463971 拒绝而不可达。以下 kubectl 命令为参考格式，需在内网/VPN 环境执行。

```bash
kubectl delete networkpolicy <policy-name> -n <namespace>
```

### 控制面（tccli）

```bash
tccli tke DeleteAddon --region <Region> \
  --ClusterId <ClusterId> \
  --AddonName enp
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| eNP 组件安装失败 | 检查集群 OS、K8s 版本、网络插件、节点类型 | 不满足限制条件 | 确认 TencentOS Server 3.1/3.2、TKE 1.22+、共享网卡 VPC-CNI、仅超级节点、仅 IPv4 |
| NetworkPolicy 不生效 | `kubectl logs -n kube-system tke-network-policy-controller-xxx` 查看异常 | 组件运行异常 | 排查 controller/agent 日志并重启对应 Pod |
| 通过 Service IP 访问被拒绝 | 直接访问 Service IP 失败，Pod IP 可通 | eBPF 在 Pod 网卡 egress 处实现，Service IP 尚未 DNAT | 当前不支持通过 Service IP 放通后端 Pod，改用 Pod IP 访问 |
| named port 规则不生效 | 使用 named port 类型规则 | eNP 暂不支持 named port | 改用数字端口号 |
| Unable to connect（kubectl 数据面不可达） | 公网端点连接超时 | CAM 策略 strategyId:240463971 拒绝 `tke:clusterExtranetEndpoint=true` | 在内网/VPN 环境执行命令；或调整 CAM 策略放通公网端点权限 |

## 下一步

- [Network Policy 说明](../NetworkPolicy 说明/tccli 操作.md)
- [NginxIngress 说明](../NginxIngress 说明/tccli 操作.md)

## 控制台替代

控制台 **集群 → 组件管理 → 新建** 安装 eNP 组件；**集群 → 网络 → NetworkPolicy** 管理网络策略。
