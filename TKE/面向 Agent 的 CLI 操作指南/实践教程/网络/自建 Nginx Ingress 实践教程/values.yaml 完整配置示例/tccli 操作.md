# values.yaml 完整配置示例（tccli）

> 对照官方：[values.yaml 完整配置示例](https://cloud.tencent.com/document/product/457/104865) · page_id `104865`

## 概述

本页提供 Nginx Ingress Controller Helm Chart 的完整 `values.yaml` 配置示例，涵盖 Controller、Service、Metrics、Admission Webhooks 等核心组件的常用参数。可作为生产环境配置的起点，根据实际需求调整。

> 本页为配置参考页面，以概念性说明为主。完整的 CLI 安装命令参见 [快速开始](../快速开始/tccli 操作.md)。

> **kubectl 数据面不可达**：CAM 拒绝公网端点（strategyId:240463971），内网需 VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

## 前置条件

查看其他自建 Nginx Ingress 操作页面的前置条件（[快速开始](../快速开始/tccli 操作.md#前置条件)），本参考页面聚焦配置说明。

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 查看集群已安装组件 | `tccli tke DescribeAddon` | 是 |
| 安装 Nginx Ingress 插件 | `tccli tke InstallAddon --AddonName NginxIngress` | 否 |
| 卸载 Nginx Ingress 插件 | `tccli tke UninstallAddon --AddonName NginxIngress` | 是 |
| 安装 Helm Chart | `helm install` | 否 |
| 查看 Helm 配置 | `helm get values` | 是 |
| 查看默认 values | `helm show values ingress-nginx/ingress-nginx` | 是 |

## 关键字段说明

以下说明 `values.yaml` 中主要配置块的关键字段。

| 字段路径 | 类型 | 必填 | 取值与约束 | 影响 |
|------|------|:--:|------|------|
| `controller.replicaCount` | Integer | 是 | ≥ 1，生产建议 ≥ 3 | Pod 副本数，直接影响可用性和吞吐 |
| `controller.image.registry` | String | 否 | 镜像仓库地址，国内可替换为 `ccr.ccs.tencentyun.com` | 加速镜像拉取 |
| `controller.image.tag` | String | 否 | 版本标签，如 `v1.10.0` | 锁定版本，避免意外升级 |
| `controller.ingressClassResource.name` | String | 是 | IngressClass 名称，默认 `nginx`。多实例时须唯一 | Ingress 通过此名称选择 Controller |
| `controller.ingressClassResource.controllerValue` | String | 是 | 格式 `k8s.io/ingress-nginx`，多实例时须唯一 | 匹配 IngressClass `spec.controller` 字段 |
| `controller.electionID` | String | 否 | Leader 选举标识，多实例时必须不同 | 防止选举冲突 |
| `controller.service.type` | String | 是 | `LoadBalancer`（推荐）或 `NodePort` | `LoadBalancer` 自动创建 CLB |
| `controller.service.annotations` | Object | 否 | CLB 注解（带宽、子网、已有 CLB ID） | 自定义 CLB 行为 |
| `controller.service.externalTrafficPolicy` | String | 否 | `Local`（保留源 IP）或 `Cluster`（默认） | `Local` 保留源 IP 但要求 Pod 均匀分布 |
| `controller.metrics.enabled` | Boolean | 否 | `true` 启用 Prometheus 指标 | 开启后 `:10254/metrics` 暴露指标 |
| `controller.admissionWebhooks.enabled` | Boolean | 否 | `true` 启用 Ingress 准入校验 | 防止无效 Ingress 规则提交 |
| `controller.resources.limits` | Object | 否 | CPU/内存上限 | 资源限制防止 OOM |
| `controller.resources.requests` | Object | 否 | CPU/内存请求 | 调度器根据请求分配资源 |
| `controller.affinity` | Object | 否 | Pod 亲和性/反亲和性 | 控制 Pod 分布 |
| `controller.tolerations` | Array | 否 | 污点容忍 | 允许 Pod 调度到有污点的节点 |
| `controller.topologySpreadConstraints` | Array | 否 | 拓扑分布约束 | 按 Zone/Node 均匀分布 |
| `controller.config` | Object | 否 | Nginx 配置片段（proxy-body-size 等） | 影响 Nginx 运行时行为 |

## 操作步骤

### 查看默认 values.yaml

```bash
# 添加 Helm 仓库（如未添加）
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

# 查看 Chart 版本列表
helm search repo ingress-nginx/ingress-nginx --versions
# expected: 显示可用版本列表

# 查看当前最新版本的默认 values（参考格式）
helm show values ingress-nginx/ingress-nginx
# expected: 输出完整的默认 values.yaml
```

### 使用自定义 values.yaml 安装/升级

```bash
# 安装
helm install nginx-ingress ingress-nginx/ingress-nginx \
    --namespace ingress-nginx \
    --create-namespace \
    -f custom-values.yaml
# expected: STATUS: deployed

# 升级
helm upgrade nginx-ingress ingress-nginx/ingress-nginx \
    --namespace ingress-nginx \
    -f custom-values.yaml
# expected: STATUS: deployed
```

### 查看当前配置

```bash
# 查看当前 Release 的自定义值
helm get values nginx-ingress -n ingress-nginx
# expected: 显示安装时传入的自定义 values

# 查看完整合并后的配置（包括默认值）
helm get values nginx-ingress -n ingress-nginx --all
# expected: 完整的合并配置
```

### tccli 插件管理命令（与 Helm 自建对比）

```bash
# 查看 Nginx Ingress 插件（TKE 官方版本）
tccli tke DescribeAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName NginxIngress

# 安装 Nginx Ingress 插件
tccli tke InstallAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName NginxIngress \
    --AddonVersion "1.8.0"

# 卸载插件
tccli tke UninstallAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName NginxIngress
```

## values.yaml 完整配置示例

以下为生产环境推荐的 `values.yaml` 参考配置。使用前请根据实际环境调整占位符。

`nginx-ingress-production.yaml`：

```yaml
# ============================================================
# controller — Nginx Ingress Controller 核心配置
# ============================================================
controller:
  # 副本数：生产环境推荐 3
  replicaCount: 3

  # 镜像配置
  image:
    registry: registry.k8s.io
    image: ingress-nginx/controller
    tag: "v1.10.0"
    pullPolicy: IfNotPresent

  # IngressClass 定义
  ingressClassResource:
    name: nginx
    enabled: true
    default: false
    controllerValue: k8s.io/ingress-nginx

  # Leader 选举 ID（多实例时必须不同）
  electionID: ingress-nginx-leader

  # 资源限制
  resources:
    requests:
      cpu: 100m
      memory: 256Mi
    limits:
      cpu: 1
      memory: 1Gi

  # 副本亲和性（反亲和，不同节点）
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app.kubernetes.io/name: ingress-nginx
          topologyKey: kubernetes.io/hostname

  # 拓扑分布约束（跨可用区）
  topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app.kubernetes.io/name: ingress-nginx

  # 探针配置
  livenessProbe:
    httpGet:
      path: /healthz
      port: 10254
    initialDelaySeconds: 30
    periodSeconds: 10
    timeoutSeconds: 5
    successThreshold: 1
    failureThreshold: 3

  readinessProbe:
    httpGet:
      path: /healthz
      port: 10254
    initialDelaySeconds: 10
    periodSeconds: 5
    timeoutSeconds: 3
    successThreshold: 1
    failureThreshold: 3

  # 优雅终止时间
  terminationGracePeriodSeconds: 300

  # PDB 最小可用数
  minAvailable: 1

  # 高并发优化
  maxWorkerConnections: 65536
  keepalive: 300

  # Nginx 运行时配置
  config:
    proxy-body-size: "100m"
    proxy-connect-timeout: "5"
    proxy-read-timeout: "60"
    proxy-send-timeout: "60"
    upstream-keepalive-connections: "512"
    use-gzip: "true"
    server-tokens: "false"
    # JSON 格式日志（便于 CLS 检索）
    log-format-upstream: '{"time":"$time_iso8601","remote_addr":"$remote_addr","host":"$host","method":"$request_method","uri":"$uri","status":$status,"request_time":$request_time,"upstream_response_time":$upstream_response_time,"upstream_addr":"$upstream_addr","http_user_agent":"$http_user_agent"}'

  # Service 配置
  service:
    type: LoadBalancer
    externalTrafficPolicy: Local
    annotations:
      # 公网 CLB 带宽配置（可选）
      service.kubernetes.io/qcloud-loadbalancer-internet-charge-type: "TRAFFIC_POSTPAID_BY_HOUR"
      service.kubernetes.io/qcloud-loadbalancer-internet-max-bandwidth-out: "500"
      # 内网 CLB 子网配置（公网与内网二选一）
      # service.kubernetes.io/qcloud-loadbalancer-internal-subnetid: "SUBNET_ID"
      # 复用已有 CLB（可选）
      # service.kubernetes.io/loadbalance-id: "CLB_ID"
      # CLB 直连 Pod（VPC-CNI 模式）
      # service.cloud.tencent.com/direct-access: "true"

  # Prometheus Metrics
  metrics:
    enabled: true
    serviceMonitor:
      enabled: true
      namespace: "CLUSTER_ID"

  # Admission Webhooks（Ingress 规则校验）
  admissionWebhooks:
    enabled: true
    failurePolicy: Fail
    timeoutSeconds: 10

# ============================================================
# defaultBackend — 默认后端（404 页面）
# ============================================================
defaultBackend:
  enabled: true
  image:
    registry: registry.k8s.io
    image: defaultbackend-amd64
    tag: "1.5"
  resources:
    requests:
      cpu: 10m
      memory: 20Mi
    limits:
      cpu: 50m
      memory: 50Mi

# ============================================================
# rbac — RBAC 配置
# ============================================================
rbac:
  create: true
  scope: false          # false = 集群级别，true = 命名空间级别

# ============================================================
# serviceAccount — ServiceAccount
# ============================================================
serviceAccount:
  create: true
  name: nginx-ingress
```

> **占位符说明**：
> - `SUBNET_ID`：内网 CLB 子网 ID。公网 CLB 场景删除此注解。
> - `CLB_ID`：复用已有 CLB 时填写。未指定时自动创建新 CLB。
> - `CLUSTER_ID`：TKE 集群 ID，用于 ServiceMonitor 命名空间。

## 验证

### 控制面（tccli）

```bash
# 验证插件状态（如使用插件）
tccli tke DescribeAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName NginxIngress
# expected: Status "Running" 或 "NotInstalled"（自建 Helm 方式）
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

### 数据面（需 VPN/IOA）

```bash
# 验证 Helm Release 配置
helm get values nginx-ingress -n ingress-nginx
# expected: 显示自定义值

# 验证配置是否生效
kubectl -n ingress-nginx get deployment nginx-ingress-ingress-nginx-controller -o yaml | grep -E "replicas|image:"
# expected: 显示对应配置值

# 验证 Nginx 运行时配置
kubectl -n ingress-nginx exec -it deployment/nginx-ingress-ingress-nginx-controller -- nginx -T 2>/dev/null | head -100
# expected: 生成的 nginx.conf 反映 values.yaml 配置
```

## 清理

> **⚠️ 注意**：本页为配置参考，无独立的清理操作。如需卸载 Nginx Ingress，参见对应操作页面（[快速开始](../快速开始/tccli 操作.md#清理)）的清理步骤。

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `helm upgrade` 后配置未生效 | `kubectl -n ingress-nginx get pods` 检查 Pod 是否重启 | Helm 升级后 Controller 未重启，旧 Nginx 配置仍生效 | 手动重启：`kubectl -n ingress-nginx rollout restart deployment nginx-ingress-ingress-nginx-controller` |
| `helm install` 失败（镜像拉取超时） | `kubectl -n ingress-nginx describe pod` 查看 Events | 默认镜像仓库 `registry.k8s.io` 在国内拉取慢或不稳定 | 使用腾讯云镜像：`controller.image.registry: ccr.ccs.tencentyun.com` 或配置 `imagePullSecrets` |
| `admissionWebhooks` 导致 Ingress 创建失败 | `kubectl describe ingress <NAME>` 查看 Events | Webhook 证书问题或 Webhook Service 不可达 | `kubectl -n ingress-nginx get svc` 确认 admission Service 存在；如 Webhook 不必要，设置 `controller.admissionWebhooks.enabled: false` |
| Ingress 不生效（ADDRESS 为空） | `kubectl describe ingress <NAME>` 查看 Events | IngressClass 名称不匹配或 Controller 未监听该 Class | 确认 `spec.ingressClassName` 与 `controller.ingressClassResource.name` 一致 |

## 下一步

- [快速开始](https://cloud.tencent.com/document/product/457/104857) — 最小化安装和 Ingress 测试
- [自定义负载均衡器](https://cloud.tencent.com/document/product/457/104858) — CLB Service Annotation 详细配置
- [高并发场景优化](https://cloud.tencent.com/document/product/457/104859) — Worker 进程、keepalive、sysctl 调优
- [高可用配置优化](https://cloud.tencent.com/document/product/457/104860) — PDB、反亲和性、探针调优
- [安装多个 Nginx Ingress Controller](https://cloud.tencent.com/document/product/457/104863) — 多实例 IngressClass 隔离

## 控制台替代

通过 [TKE 控制台 - 组件管理](https://console.cloud.tencent.com/tke2/cluster/sub/list/addon/cluster?rid=1) 安装 Nginx Ingress 插件，或通过 [Helm 应用市场](https://console.cloud.tencent.com/tke2/market) 部署自定义 Chart。
