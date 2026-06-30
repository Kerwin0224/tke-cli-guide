# NginxIngress 版本过低导致集群无法升级到 1.22 以上（tccli）

> 对照官方：[NginxIngress 版本过低导致集群无法升级到 1.22 以上](https://cloud.tencent.com/document/product/457/118391) · page_id `118391`

## 概述

Kubernetes 1.22 版本移除了 `extensions/v1beta1` 和 `networking.k8s.io/v1beta1` Ingress API。若集群中仍存在使用旧版 API 的 Ingress 资源，将阻止集群升级到 1.22 及以上版本。需将 Ingress 迁移到 `networking.k8s.io/v1` 并更新 NginxIngress 组件。

> **kubectl 数据面不可达**：CAM拒绝公网端点(strategyId:240463971)，内网需VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

## 前置条件

- [环境准备](../../环境准备.md)：`tccli` 已配置
- 已获取集群 kubeconfig（如需 kubectl 操作）
- 了解 Ingress 资源和 Nginx Ingress Controller 基本概念
- 集群为 MANAGED_CLUSTER 类型，K8s 1.30.0，ap-guangzhou

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 查看集群状态 | `tccli tke DescribeClusterStatus --region <Region> --ClusterIds '["CLUSTER_ID"]'` | 是 |
| 查看 NginxIngress 版本 | `tccli tke DescribeAddon --region <Region> --ClusterId CLUSTER_ID --AddonName nginx-ingress` | 是 |
| 列出所有 Ingress | `kubectl get ingress -A`（需 VPN/IOA） | 是 |
| 查看 Ingress API 版本 | `kubectl get ingress -A -o json \| jq '.items[].apiVersion'`（需 VPN/IOA） | 是 |
| 升级 NginxIngress | `tccli tke UpdateAddon --region <Region> --ClusterId CLUSTER_ID --AddonName nginx-ingress --AddonVersion <target>` | 否 |

## 操作步骤

### 1. 控制面：检查集群和目标版本兼容性

```bash
tccli tke DescribeClusterStatus --region <Region> --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterState "Running"
```

```json
{
  "ClusterStatusSet": "<ClusterStatusSet>",
  "ClusterId": "<ClusterId>",
  "ClusterState": "<ClusterState>",
  "ClusterInstanceState": "<ClusterInstanceState>",
  "ClusterBMonitor": "<ClusterBMonitor>",
  "ClusterInitNodeNum": "<ClusterInitNodeNum>"
}
```

```bash
tccli tke DescribeAddon --region <Region> --ClusterId CLUSTER_ID --AddonName nginx-ingress
# expected: 返回当前 AddonVersion，状态 "Running"
#  注意：若版本 < 1.0.0，则 Ingress 可能使用旧 API
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

### 2. 数据面：扫描使用旧 API 的 Ingress（需 VPN/IOA）

```bash
# 列出所有命名空间的 Ingress 及其 API 版本
kubectl get ingress -A -o custom-columns=NAMESPACE:.metadata.namespace,NAME:.metadata.name,API-VERSION:.apiVersion
# expected: apiVersion 列中出现 networking.k8s.io/v1beta1 或 extensions/v1beta1 即为需迁移

# 列出所有使用旧 API 的 Ingress
kubectl get ingress -A -o json | jq -r '
  .items[] | select(.apiVersion != "networking.k8s.io/v1") |
  "\(.metadata.namespace)/\(.metadata.name) apiVersion=\(.apiVersion)"
'
# expected: 列表中的 Ingress 需迁移
```

```text
NAME  STATUS  AGE
...
```

### 3. 数据面：迁移 Ingress 到 networking.k8s.io/v1（需 VPN/IOA）

**v1beta1 与 v1 的关键差异：**

| 字段 | extensions/v1beta1 | networking.k8s.io/v1 |
|------|-------------------|---------------------|
| `apiVersion` | `extensions/v1beta1` 或 `networking.k8s.io/v1beta1` | `networking.k8s.io/v1` |
| `pathType` | 不存在（默认 ImplementationSpecific） | 必填：`Prefix`、`Exact` 或 `ImplementationSpecific` |
| `serviceName` / `servicePort` | 使用 `serviceName` + `servicePort`（数字或字符串） | 使用 `service.name` + `service.port.number` 或 `service.port.name` |
| `IngressClass` | 通过注解 `kubernetes.io/ingress.class` 指定 | 使用 `spec.ingressClassName` 字段 |
| `backend` 类型 | `backend` 仅支持 serviceName/servicePort | 支持 `service` 和 `resource` 两种后端 |

**步骤 3a：备份现有 Ingress**

```bash
kubectl get ingress <name> -n NAMESPACE -o yaml > ingress-backup.yaml
# expected: 备份文件生成
```

**步骤 3b：转换 YAML 格式**

```yaml
# 迁移前（v1beta1 格式 -- 废弃）示例：
# apiVersion: networking.k8s.io/v1beta1
# kind: Ingress
# metadata:
#   name: example-ingress
#   namespace: NAMESPACE
#   annotations:
#     kubernetes.io/ingress.class: "nginx"
# spec:
#   rules:
#   - host: example.CLUSTER_ID
#     http:
#       paths:
#       - path: /api
#         backend:
#           serviceName: api-service
#           servicePort: 80

# 迁移后（v1 格式）：
# apiVersion: networking.k8s.io/v1
# kind: Ingress
# metadata:
#   name: example-ingress
#   namespace: NAMESPACE
# spec:
#   ingressClassName: nginx
#   rules:
#   - host: example.CLUSTER_ID
#     http:
#       paths:
#       - path: /api
#         pathType: Prefix
#         backend:
#           service:
#             name: api-service
#             port:
#               number: 80
```

```bash
kubectl apply -f ingress-v1.yaml
# expected: ingress.networking.k8s.io/example-ingress configured
```

### 4. 控制面：升级 NginxIngress 组件

```bash
tccli tke UpdateAddon --region <Region> --ClusterId CLUSTER_ID --AddonName nginx-ingress --AddonVersion "<target-version>"
# expected: RequestId 返回，升级已提交
```

## 验证

### 控制面（tccli）

```bash
tccli tke DescribeAddon --region <Region> --ClusterId CLUSTER_ID --AddonName nginx-ingress
# expected: AddonVersion 为目标版本，Status "Running"
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

```bash
tccli tke DescribeClusterStatus --region <Region> --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterState "Running"
```

```json
{
  "ClusterStatusSet": "<ClusterStatusSet>",
  "ClusterId": "<ClusterId>",
  "ClusterState": "<ClusterState>",
  "ClusterInstanceState": "<ClusterInstanceState>",
  "ClusterBMonitor": "<ClusterBMonitor>",
  "ClusterInitNodeNum": "<ClusterInitNodeNum>"
}
```

### 数据面（需 VPN/IOA）

```bash
# 确认所有 Ingress 已使用 v1 API
kubectl get ingress -A -o json | jq '[.items[].apiVersion] | unique'
# expected: ["networking.k8s.io/v1"]

# 确认 Ingress 正常工作
kubectl get ingress -A
# expected: 所有 Ingress ADDRESS 有值

# 确认 NginxIngress Controller Pod 运行正常
kubectl get pods -n kube-system -l app=nginx-ingress
# expected: 所有 Pod Running
```

```text
NAME  STATUS  AGE
...
```

## 清理

排障页通常无需清理。如有测试 Ingress 资源需清理：

```bash
kubectl delete ingress <test-ingress> -n NAMESPACE
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl apply` 迁移后的 Ingress 报错 "unknown field 'serviceName'" | `kubectl explain ingress.spec.rules.http.paths.backend` | YAML 仍使用 v1beta1 的 `serviceName`/`servicePort` 字段而非 v1 的 `service.name`/`service.port` | 将所有 `serviceName` → `service.name`，`servicePort`（整数）→ `service.port.number` |
| `kubectl apply` 报错 "missing required field 'pathType'" | `grep -n "pathType" ingress-v1.yaml` | networking.k8s.io/v1 中 `pathType` 为必填字段 | 在每个 path 中添加 `pathType: Prefix`（最常见场景）或 `pathType: Exact` / `pathType: ImplementationSpecific` |
| `tccli tke UpdateAddon` 报错 "cannot upgrade cluster with deprecated Ingress API" | 参照步骤 2 扫描旧 API Ingress | 集群中存在未迁移的 v1beta1 Ingress 资源 | 完成所有 Ingress 到 v1 格式的迁移后再执行升级 |
| `kubectl apply` 报错 "IngressClass 'nginx' not found" | `kubectl get ingressclass` | IngressClass 资源 `nginx` 未在集群中创建 | 确认 NginxIngress 组件正确安装后 `kubectl get ingressclass` 应存在；若无，升级 NginxIngress 组件版本 |

### 创建成功但状态异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Ingress ADDRESS 为空 | `kubectl describe ingress <name> -n NAMESPACE \| grep -A10 Events` | NginxIngress Controller 版本不兼容，无法识别 v1 Ingress | 升级 NginxIngress 到支持 networking.k8s.io/v1 的版本（通常 >= 0.42.0） |
| 升级后部分路径路由失效 | `kubectl get ingress <name> -n NAMESPACE -o yaml \| grep -A5 pathType` | `pathType: Exact` 过于严格导致路径不匹配 | 将 `pathType` 改为 `Prefix`（匹配路径前缀）；或改为 `ImplementationSpecific` 遵循 Nginx 默认匹配规则 |
| TLS 证书不生效 | `kubectl get ingress <name> -n NAMESPACE -o yaml \| grep -A10 tls` | 迁移时 `tls` 段未正确转换或 Secret 名称变更 | 确认 `spec.tls[].hosts` 和 `spec.tls[].secretName` 完整；确认 Secret 存在：`kubectl get secret <secret> -n NAMESPACE` |
| 旧注解 `kubernetes.io/ingress.class` 与新 `ingressClassName` 冲突 | `kubectl get ingress <name> -n NAMESPACE -o yaml \| grep -E "ingressClassName\|kubernetes.io/ingress.class"` | 同时存在注解和 spec.ingressClassName，NginxIngress Controller 处理逻辑可能冲突 | 删除旧注解 `kubernetes.io/ingress.class`，仅保留 `spec.ingressClassName` |

## 下一步

- [Nginx Ingress 偶现 Connection Refused](../Nginx%20Ingress%20偶现%20Connection%20Refused/tccli%20操作.md) -- page_id `71216`
- [Service&Ingress 常见报错和处理](../Service&Ingress%20常见报错和处理/tccli%20操作.md) -- page_id `75758`

## 控制台替代

[容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) → 集群 → 服务与路由 → Ingress → 查看 YAML → 检查 apiVersion。
