# CoreDNS 说明（tccli）

> 对照官方：[CoreDNS 说明](https://cloud.tencent.com/document/product/457/129280) · page_id `129280`

## 概述

CoreDNS 是 Kubernetes 集群的 DNS 服务组件，负责为集群内 Service 和 Pod 提供 DNS 解析服务。基于插件链架构，支持灵活的 DNS 配置，包括服务发现、缓存、转发、健康检查等功能。CoreDNS 为集群创建时默认安装，无需手动安装。通过 Corefile 配置文件定义 DNS 行为，支持多种插件组合。

## 前置条件

- [环境准备](../../../../环境准备.md)
- 已 [连接集群](../../../../集群配置/集群管理/连接集群/tccli 操作.md)，kubectl 可正常访问 APIServer
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

- 需要 CAM 权限：`tke:DescribeClusters`、`tke:DescribeAddon`、`tke:UpdateAddon`、`tke:DeleteAddon`

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 查看集群信息 | `tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'` | 是 |
| 查看 CoreDNS 组件详情 | `tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName coredns` | 是 |
| 升级 CoreDNS | `tccli tke UpdateAddon --region <Region> --ClusterId <ClusterId> --AddonName coredns --AddonVersion <TargetVersion>` | 否 |
| 卸载 CoreDNS（不建议） | `tccli tke DeleteAddon --region <Region> --ClusterId <ClusterId> --AddonName coredns` | 是 |
| 修改 Corefile 配置 | `kubectl edit cm coredns -n kube-system` | 是 |

**部署在集群内的 Kubernetes 对象：**

| Kubernetes 对象 | 类型 | 请求资源 | 所属 Namespace |
|----------------|------|---------|---------------|
| coredns | Deployment | 每实例 100m CPU，30Mi 内存 | kube-system |
| coredns | ConfigMap | - | kube-system |
| kube-dns | Service | - | kube-system |
| coredns | ServiceAccount | - | kube-system |
| system:coredns | ClusterRole | - | - |
| system:coredns | ClusterRoleBinding | - | - |

## 操作步骤

### 步骤 1：查看 CoreDNS 组件信息（控制面）

```bash
tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName coredns \
    | jq '.Addons[0] | {AddonName, AddonVersion, Status}'
# expected: 返回 CoreDNS 组件详细信息
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

### 步骤 2：查看 CoreDNS 运行状态（数据面）

```bash
kubectl get deploy coredns -n kube-system -o wide
kubectl get pods -n kube-system -l k8s-app=kube-dns -o wide
# expected: coredns Pod Running
```

```text
NAME  STATUS  AGE
...
```

> **注意**：kubectl 数据面当前因公网端点被 CAM 策略 strategyId:240463971 拒绝而不可达。以下 kubectl 命令为参考格式，需在内网/VPN 环境执行。

### 步骤 3：查看默认 Corefile 配置（数据面）

```bash
kubectl get cm coredns -n kube-system -o yaml
# expected: 返回 Corefile 配置
```

默认 Corefile 配置：

```yaml
.:53 {
    template ANY HINFO . {
        rcode NXDOMAIN
    }
    errors
    health {
        lameduck 30s
    }
    ready
    kubernetes cluster.local. in-addr.arpa ip6.arpa {
        pods insecure
        fallthrough in-addr.arpa ip6.arpa
    }
    prometheus :9153
    forward . /etc/resolv.conf {
        prefer_udp
    }
    cache 30
    reload
    loadbalance
}
```

主要插件说明：

- **errors**：将错误日志输出到标准输出
- **health**：提供健康检查端点（默认端口 8080），lameduck 30s 收到关闭信号后继续服务 30 秒
- **ready**：提供就绪检查端点（默认端口 8181）
- **kubernetes**：Kubernetes 服务发现插件，解析集群内 Service 和 Pod 的 DNS 记录
- **prometheus**：在 9153 端口暴露 Prometheus 指标
- **forward**：将非集群域名的 DNS 请求转发到上游 DNS 服务器
- **cache**：DNS 记录缓存，TTL 30 秒
- **reload**：支持 Corefile 热加载
- **loadbalance**：DNS 记录轮询负载均衡

### 步骤 4：修改 Corefile 配置（数据面）

```bash
kubectl edit cm coredns -n kube-system
# expected: 配置保存成功
```

配置修改后 CoreDNS 会自动热加载，无需重启 Pod。详细配置请参见 [CoreDNS 官方文档](https://coredns.io/manual/toc/)。

### 步骤 5：升级 CoreDNS 组件（控制面）

```bash
tccli tke UpdateAddon --region <Region> \
  --ClusterId <ClusterId> \
  --AddonName coredns \
  --AddonVersion <TargetVersion>
# expected: exit 0，升级进行中
```

> **强烈建议：** 升级前先进行平滑升级配置，参见 [配置 CoreDNS 平滑升级](https://cloud.tencent.com/document/product/457/78005)。

### 组件权限说明

CoreDNS 组件权限为最小权限依赖：

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: system:coredns
rules:
  - apiGroups:
      - '*'
    resources:
      - endpoints
      - services
      - pods
      - namespaces
    verbs:
      - list
      - watch
  - apiGroups:
      - discovery.k8s.io
    resources:
      - endpointslices
    verbs:
      - list
      - watch
```

### 参数说明

| 参数 | 说明 | 类型 | 默认值 |
|------|------|------|--------|
| `global.image.host` | 镜像仓库地址 | string | `ccr.ccs.tencentyun.com` |
| `global.kubednsClusterIP` | kube-dns Service 的 ClusterIP，为空自动分配 | string | `""` |
| `global.cluster.highAvailability` | 是否为高可用集群（控制 zone 拓扑分布约束） | bool | `false` |
| `global.podSpec.tolerations` | 额外的容忍配置 | list | `[]` |
| `global.podSpec.priorityClassName` | Pod 优先级类名 | string | `system-cluster-critical` |
| `coredns.replicas` | CoreDNS 副本数 | int | `2` |
| `coredns.hostNetwork` | 是否使用主机网络 | bool | `false` |
| `coredns.image` | 自定义镜像地址 | string | `""` |
| `coredns.server.port` | DNS 服务端口 | int | `53` |
| `coredns.livenessProbe.port` | 存活探针端口 | int | `8080` |
| `coredns.readinessProbe.port` | 就绪探针端口 | int | `8181` |
| `coredns.resources.requests.cpu` | CPU 请求 | string | `100m` |
| `coredns.resources.requests.mem` | 内存请求 | string | `30Mi` |
| `coredns.resources.limits.cpu` | CPU 限制 | string | `""` |
| `coredns.resources.limits.mem` | 内存限制 | string | `2Gi` |

## 验证

### 控制面（tccli）

```bash
tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName coredns \
    | jq '.Addons[0] | {AddonName, AddonVersion, Status}'
# expected: Status "Running"
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
kubectl get pods -n kube-system -l k8s-app=kube-dns
# expected: coredns Pod Running

kubectl exec -n kube-system deployment/coredns -- nslookup kubernetes.default.svc.cluster.local
# expected: 正常解析到 kubernetes Service
```

```text
NAME  STATUS  AGE
...
```

> **注意**：kubectl 数据面当前因公网端点被 CAM 策略 strategyId:240463971 拒绝而不可达。以下 kubectl 命令为参考格式，需在内网/VPN 环境执行。

## 清理

CoreDNS 为集群必需组件，不建议卸载。如确需清理（仅限非生产集群）：

```bash
tccli tke DeleteAddon --region <Region> --ClusterId <ClusterId> --AddonName coredns
# expected: exit 0，组件卸载请求已提交

# 验证已卸载
tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName coredns \
    | jq '.Addons[] | select(.AddonName == "coredns")'
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

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| DNS 解析失败 | `kubectl get pods -n kube-system -l k8s-app=kube-dns` 查看 Pod 状态；`kubectl logs -n kube-system -l k8s-app=kube-dns` | CoreDNS Pod 异常 | 检查 CoreDNS Pod 是否 Running；查看日志定位错误 |
| Corefile 修改未生效 | `kubectl get cm coredns -n kube-system -o yaml` 检查配置 | 热加载延迟或 Corefile 语法错误 | CoreDNS 支持自动热加载，等待数秒；或检查 Corefile 语法 |
| CoreDNS Pod CrashLoopBackOff | `kubectl describe pod -n kube-system -l k8s-app=kube-dns` | Corefile 配置语法错误 | 检查 Corefile 配置语法；查看 describe 输出 |
| 升级导致 DNS 服务中断 | 升级前检查副本数与可用区分布 | 未配置平滑升级 | 强烈建议先配置平滑升级；确保多副本跨节点/跨 AZ 部署 |
| `Unable to connect to the server` | `kubectl cluster-info` 检查连通性 | 公网端点被 CAM 策略 strategyId:240463971 以 `tke:clusterExtranetEndpoint=true` 条件拒绝 | 通过 IOA/VPN 连接内网，或使用同 VPC CVM 执行 kubectl 命令 |

## 下一步

- [组件高可用](../组件高可用/tccli 操作.md)
- [DNSAutoscaler 说明](../DNSAutoscaler 说明/tccli 操作.md)
- [NodeLocalDNSCache 说明](../NodeLocalDNSCache 说明/tccli 操作.md)
- [配置 CoreDNS 平滑升级](https://cloud.tencent.com/document/product/457/78005)

## 控制台替代

控制台 **集群 → 组件管理** 查看 CoreDNS 状态和版本，**运维中心 → 组件管理** 进行升级操作。修改 Corefile 直接编辑 kube-system 下的 `configmap/coredns`。
