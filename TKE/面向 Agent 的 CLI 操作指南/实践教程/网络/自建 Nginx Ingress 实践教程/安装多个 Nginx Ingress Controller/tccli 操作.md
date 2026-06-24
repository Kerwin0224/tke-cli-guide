# 安装多个 Nginx Ingress Controller（tccli）

> 对照官方：[安装多个 Nginx Ingress Controller](https://cloud.tencent.com/document/product/457/104863) · page_id `104863`

## 概述

在同一个 TKE 集群中安装多个 Nginx Ingress Controller 实例，每个实例使用独立的 Ingress Class 和命名空间，实现不同业务线、不同环境的流量隔离。常见场景：内网/公网入口分离、生产/测试流量隔离、不同团队独立管理 Ingress 规则。

**隔离模式**：

| 隔离方式 | 原理 | 适用场景 |
|---------|------|---------|
| Ingress Class 隔离 | 每个 Controller 监听不同 `ingressClassName` 的 Ingress | Controller 实例功能相同，按业务分组 |
| 命名空间隔离 | 不同 Controller 安装在不同命名空间 | 需要独立的权限和资源配额 |
| CLB 隔离 | 每个 Controller 使用独立的 CLB | 对外暴露不同的 CLB 入口 |
| Scope 隔离 | Controller `--watch-namespace` 仅监听特定命名空间 | 精细流量隔离，减少 Controller 内存占用 |

> **kubectl 数据面不可达**：CAM 拒绝公网端点（strategyId:240463971），内网需 VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

## 前置条件

- [环境准备](../../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId、secretKey、region 均已配置

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:DescribeClusters, tke:DescribeClusterKubeconfig
#    clb:DescribeLoadBalancers, vpc:DescribeSubnets
# 验证
tccli tke DescribeClusters --region <Region>
# expected: exit 0

tccli clb DescribeLoadBalancers --region <Region>
# expected: exit 0

# 4. 检查 kubectl 和 Helm（须 VPN/IOA）
kubectl version --client
# expected: Client Version v1.30+
helm version --short
# expected: v3.x
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

### 资源检查

```bash
# 5. 确认现有 Nginx Ingress
helm list -n ingress-nginx
# expected: 显示已有 nginx-ingress Release

# 6. 确认现有 IngressClass
kubectl get ingressclass
# expected: 已有 nginx IngressClass
```

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 查看 IngressClass | `kubectl get ingressclass` | 是 |
| 查看 Helm Release | `helm list -A` | 是 |
| 安装新 Controller | `helm install`（不同 Release 名） | 否 |
| 删除 Controller | `helm uninstall` | 是 |
| 查看 CLB 列表 | `tccli clb DescribeLoadBalancers` | 是 |

## 关键字段说明

多实例安装时的关键 Helm Values 参数：

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `controller.ingressClassResource.name` | String | 是 | 自定义 Ingress Class 名称，如 `nginx-internal`。不能与已有 Class 重名 | 重名 → Ingress 规则可能被错误的 Controller 处理 |
| `controller.ingressClassResource.controllerValue` | String | 是 | 格式 `k8s.io/ingress-nginx-{NAME}`，匹配 IngressClass `spec.controller` 字段 | 不匹配 → Ingress 无法被此 Controller 识别 |
| `controller.ingressClass` | String | 是 | 与 `ingressClassResource.name` 保持一致 | 不一致 → Controller 监听的 Class 与注册的 Class 不匹配 |
| `controller.scope.enabled` | Boolean | 否 | `true` 启用命名空间级别隔离 | 不启用 → 监听所有命名空间的 Ingress |
| `controller.scope.namespace` | String | 否 | 仅监听特定命名空间，`scope.enabled=true` 时必填 | 填错 → Ingress 不生效 |
| `controller.electionID` | String | 否 | 多 Controller 时需不同的 Leader election ID，避免选举冲突 | 相同 → Controller 间错乱 |
| `controller.service.annotations` | Object | 否 | 为新 Controller 配置独立的 CLB（内网/已有 CLB） | 不配置 → 可能创建不必要的公网 CLB |

## 操作步骤

### 步骤 1：安装第二个 Nginx Ingress Controller（内网入口）

#### 选择依据

- **独立 Ingress Class**：第二个 Controller 使用 `nginx-internal` 作为 Ingress Class，与默认的 `nginx` 隔离。应用中通过 `spec.ingressClassName: nginx-internal` 指定流量由内网 Controller 处理。
- **独立命名空间**：安装在 `ingress-nginx-internal` 命名空间，与公共入口隔离。
- **内网 CLB**：通过 Service Annotation 指定内网子网，自动创建内网 CLB。
- **独立 electionID**：防止两个 Controller 的 Leader 选举相互干扰。

#### 最小安装

`nginx-ingress-internal-values.yaml`：

```yaml
controller:
  ingressClassResource:
    name: nginx-internal
    controllerValue: k8s.io/ingress-nginx-internal
  ingressClass: nginx-internal
  electionID: ingress-nginx-internal-leader
  service:
    type: LoadBalancer
    annotations:
      service.kubernetes.io/qcloud-loadbalancer-internal-subnetid: "SUBNET_ID"
```

```bash
helm install nginx-ingress-internal ingress-nginx/ingress-nginx \
    --namespace ingress-nginx-internal \
    --create-namespace \
    -f nginx-ingress-internal-values.yaml
# expected: STATUS: deployed
```

> **kubectl 数据面不可达**：CAM 拒绝公网端点（strategyId:240463971），内网需 VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

### 步骤 2：安装 Scope 限定的 Controller（仅监听特定命名空间）

#### 选择依据

- **Scope 隔离**：Controller 仅监听 `team-a` 命名空间的 Ingress 规则。控制器内存中只维护该命名空间的配置，适合多租户场景。
- **RBAC**：配合 ClusterRole 和 RoleBinding，限制 Controller 只能访问特定命名空间。

`nginx-ingress-scope-team-a-values.yaml`：

```yaml
controller:
  ingressClassResource:
    name: nginx-team-a
    controllerValue: k8s.io/ingress-nginx-team-a
  ingressClass: nginx-team-a
  electionID: ingress-nginx-team-a-leader
  scope:
    enabled: true
    namespace: team-a
  service:
    type: LoadBalancer
    annotations:
      service.kubernetes.io/qcloud-loadbalancer-internal-subnetid: "SUBNET_ID"
```

```bash
helm install nginx-ingress-team-a ingress-nginx/ingress-nginx \
    --namespace ingress-nginx-team-a \
    --create-namespace \
    -f nginx-ingress-scope-team-a-values.yaml
# expected: STATUS: deployed
```

### 步骤 3：验证多个 IngressClass 和 Controller

```bash
# 验证所有 IngressClass
kubectl get ingressclass
# expected: 显示 nginx、nginx-internal、nginx-team-a

# 验证所有 Controller Pod
kubectl -n ingress-nginx get pods
kubectl -n ingress-nginx-internal get pods
kubectl -n ingress-nginx-team-a get pods
# expected: 每个命名空间都有 Running 的 Pod

# 验证 CLB（每个 Controller 的 Service）
kubectl get svc -n ingress-nginx
kubectl get svc -n ingress-nginx-internal
kubectl get svc -n ingress-nginx-team-a
# expected: 每个 Controller 都有独立的 EXTERNAL-IP

# 查看所有 Helm Release
helm list -A
# expected: 显示 3 个 Release（不同命名空间）
```

### 步骤 4：创建使用不同 IngressClass 的 Ingress 规则

`ingress-team-a.yaml`（参考格式）：

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: team-a-app
  namespace: team-a
spec:
  ingressClassName: nginx-team-a
  rules:
  - host: team-a.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: team-a-svc
            port:
              number: 80
```

```bash
kubectl apply -f ingress-team-a.yaml
# expected: ingress.networking.k8s.io/team-a-app created
```

## 验证

### 控制面（tccli）

```bash
# 查看所有 CLB（每个 Controller 创建独立的 CLB）
tccli clb DescribeLoadBalancers --region <Region>
# expected: 多个 CLB 实例，名称包含各 Controller

# 查看集群组件
tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterStatus "Running"，各 Controller 对应 CLB 正常
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

### 数据面（需 VPN/IOA）

```bash
# 验证 IngressClass
kubectl get ingressclass -o yaml | grep -A3 "name: nginx-internal"
# expected: 显示 nginx-internal 的 controller 值

# 验证 Controller 只监听特定命名空间（scope 模式）
kubectl -n ingress-nginx-team-a logs -l app.kubernetes.io/name=ingress-nginx | grep "watching namespace"
# expected: 日志显示 watching namespace team-a

# 验证 Ingress 被正确 Controller 处理
kubectl describe ingress team-a-app -n team-a
# expected: Events 无异常，Admission 无 Webhook 拒绝

# 测试内网访问（从同 VPC 的其他 CVM，须 VPN/IOA）
# curl -H "Host: team-a.example.com" http://INTERNAL_CLB_IP/
```

```text
NAME  STATUS  AGE
...
```

## 清理

### 数据面（需 VPN/IOA）

> **⚠️ 警告**：删除 Controller 时，关联的 CLB 会被自动删除（除非通过 Annotation 复用了已有 CLB）。确认无生产流量后再操作。

```bash
# 1. 删除 Ingress 规则
kubectl delete ingress team-a-app -n team-a
# expected: ingress "team-a-app" deleted

# 2. 卸载 Controller（按需）
helm uninstall nginx-ingress-internal -n ingress-nginx-internal
helm uninstall nginx-ingress-team-a -n ingress-nginx-team-a
# expected: releases uninstalled

# 3. 删除命名空间
kubectl delete ns ingress-nginx-internal ingress-nginx-team-a
# expected: namespaces deleted

# 4. 删除 IngressClass
kubectl delete ingressclass nginx-internal nginx-team-a
# expected: ingressclass deleted
```

### 控制面（tccli）

```bash
# 验证 CLB 已释放
tccli clb DescribeLoadBalancers --region <Region>
# expected: 列表中不再包含已删除 Controller 的 CLB
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `helm install` 返回 `IngressClass already exists` | `kubectl get ingressclass` | IngressClass 名称冲突 | 使用不同的 `ingressClassResource.name` |
| 新 Controller 无法获取 Leader | `kubectl -n ingress-nginx-internal logs -l app.kubernetes.io/name=ingress-nginx | grep leader` | `electionID` 与已有 Controller 冲突 | 设置唯一的 `electionID` |
| 新 Controller Pending（节点资源不足） | `kubectl -n ingress-nginx-internal describe pod` | 集群节点资源不足以运行额外 Controller | 扩容节点或降低新 Controller 的资源请求 |

### 配置生效后功能异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Ingress 使用了自定义 Class 但不生效 | `kubectl describe ingress <NAME>` 查看 Address 字段 | IngressClass 的 `spec.controller` 值与 Controller 的 `controllerValue` 不匹配 | 确保 IngressClass 的 `spec.controller: k8s.io/ingress-nginx-internal` 与 values 中的 `controllerValue` 一致 |
| Scope 模式下 Ingress 不生效 | `kubectl -n <NS> describe ingress` 检查 Events | Ingress 所在命名空间不在 Controller 的 scope.namespace 中 | 将 Ingress 移到正确的命名空间，或修改 Controller scope |
| 多个 Controller 争抢同一 Ingress | `kubectl describe ingress <NAME>` 查看 IngressClass | Ingress 的 `ingressClassName` 为空或不唯一 | 在 Ingress 中明确指定 `spec.ingressClassName` |
| Webhook 拒绝 Ingress（validation error） | `kubectl describe ingress <NAME>` 查看 Events | 多个 Controller 的 ValidatingWebhook 同时拦截 | 查看 Webhook 配置：`kubectl get validatingwebhookconfigurations` |

## 下一步

- [快速开始](https://cloud.tencent.com/document/product/457/104857) — 基础安装和 Ingress 测试
- [自定义负载均衡器](https://cloud.tencent.com/document/product/457/104858) — 每个 Controller 的独立 CLB 配置
- [从 TKE Nginx Ingress 插件迁移到自建 Nginx Ingress](https://cloud.tencent.com/document/product/457/104864) — 迁移到自建方案

## 控制台替代

通过 [TKE 控制台 - 组件管理](https://console.cloud.tencent.com/tke2/cluster/sub/list/addon/cluster?rid=1) 查看已安装组件。多 Ingress Controller 的管理主要通过 Helm CLI 完成。
