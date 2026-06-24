# 高可用配置优化（tccli）

> 对照官方：[高可用配置优化](https://cloud.tencent.com/document/product/457/104860) · page_id `104860`

## 概述

通过多副本部署、Pod 反亲和性、PodDisruptionBudget、拓扑分布约束和优雅终止等机制，确保 Nginx Ingress Controller 在节点故障、滚动更新、维护操作等场景下持续可用。

**可用性对照**：

| 配置 | 单点故障风险 | 配置后 |
|------|------------|--------|
| 单副本 | 节点/容器故障 → 流量全断 | — |
| 多副本 + 反亲和 | — | 副本分布不同节点，单节点故障不影响 |
| 无 PDB | 滚动更新/排空时全部 Pod 同时终止 | — |
| PDB minAvailable=1 | — | 维护期间至少 1 个 Pod 保持可用 |
| 无健康检查 | 启动慢的 Pod 过早接收流量 | — |
| Readiness Probe | — | Pod 就绪后才接入流量 |

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
# 验证
tccli tke DescribeClusters --region <Region>
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
# 5. 确认集群节点数
tccli tke DescribeClusterInstances --region <Region> \
    --ClusterId CLUSTER_ID
# expected: 至少 2 个节点（Running），推荐 ≥ 3

# 6. 确认 Nginx Ingress 已安装
helm list -n ingress-nginx
# expected: nginx-ingress 状态 deployed

# 7. 确认当前 Pod 分布
kubectl -n ingress-nginx get pods -o wide
# expected: 显示 Pod 所在节点
```

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 查看集群节点 | `tccli tke DescribeClusterInstances` | 是 |
| 查看工作负载 | `kubectl get deployment` | 是 |
| 配置反亲和性 | `helm upgrade`（修改 values） | 否 |
| 创建 PDB | `kubectl apply -f pdb.yaml` | 否 |
| 修改副本数 | `helm upgrade` 或 `kubectl scale` | 否 |

## 关键字段说明

以下说明 Nginx Ingress Controller 高可用相关的 Helm Values 配置项。

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `controller.replicaCount` | Integer | 是 | ≥ 2，推荐 ≥ 3。奇数副本数避免脑裂 | 单副本 → 无高可用保障 |
| `controller.minAvailable` | Integer | 否 | PDB 的最小可用数，默认 1。须 < replicaCount | 设为 replicaCount → 滚动更新卡住 |
| `controller.affinity.podAntiAffinity` | Object | 否 | 反亲和性配置，建议 `requiredDuringSchedulingIgnoredDuringExecution` 硬反亲和 | 不配置 → 多副本可能调度到同节点，节点故障全挂 |
| `controller.topologySpreadConstraints` | Array | 否 | 拓扑分布约束（K8s 1.19+），按 zone/node 均匀分布 | 不配置 → 多副本可能集中在少数 zone |
| `controller.healthCheckPath` | String | 否 | 就绪探针路径，默认 `/healthz` | 路径不存在 → Pod 永远不会 Ready |
| `controller.livenessProbe` | Object | 否 | 存活探针配置 | 过于激进 → 正常 Pod 被误杀重启 |
| `controller.readinessProbe` | Object | 否 | 就绪探针配置 | 过于宽松 → 未就绪 Pod 接收流量导致连接拒绝 |
| `controller.terminationGracePeriodSeconds` | Integer | 否 | 优雅终止时间（秒），默认 300 | 过短 → 连接未排空即终止，导致丢请求 |

## 操作步骤

### 步骤 1：配置多副本 + 反亲和性

#### 选择依据

- **副本数**：3 副本是生产环境最低推荐值。奇数副本避免 Leader 选举脑裂。Nginx Ingress Controller 为无状态服务，多副本天然支持。
- **反亲和性**：`requiredDuringScheduling`（硬反亲和）强制每节点最多 1 个 Pod，节点数 ≥ 副本数。`preferredDuringScheduling`（软反亲和）优先但不强制，节点数不足时也能调度。
- **拓扑分布约束**：按可用区（zone）和节点（hostname）双重分散，防止单可用区故障影响。

#### 最小高可用配置

`nginx-ingress-ha-minimal.yaml`：

```yaml
controller:
  replicaCount: 3
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app.kubernetes.io/name: ingress-nginx
          topologyKey: kubernetes.io/hostname
  minAvailable: 1
```

```bash
helm upgrade nginx-ingress ingress-nginx/ingress-nginx \
    --namespace ingress-nginx \
    -f nginx-ingress-ha-minimal.yaml
# expected: STATUS: deployed
```

> **kubectl 数据面不可达**：CAM 拒绝公网端点（strategyId:240463971），内网需 VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

#### 增强配置（含拓扑约束、探针调优）

`nginx-ingress-ha-enhanced.yaml`：

```yaml
controller:
  replicaCount: 3
  minAvailable: 2
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app.kubernetes.io/name: ingress-nginx
        topologyKey: kubernetes.io/hostname
  topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app.kubernetes.io/name: ingress-nginx
  terminationGracePeriodSeconds: 300
  livenessProbe:
    initialDelaySeconds: 30
    periodSeconds: 10
    timeoutSeconds: 5
    successThreshold: 1
    failureThreshold: 3
  readinessProbe:
    initialDelaySeconds: 10
    periodSeconds: 5
    timeoutSeconds: 3
    successThreshold: 1
    failureThreshold: 3
```

```bash
helm upgrade nginx-ingress ingress-nginx/ingress-nginx \
    --namespace ingress-nginx \
    -f nginx-ingress-ha-enhanced.yaml
# expected: STATUS: deployed
```

### 步骤 2：手动创建 PDB（如果 Helm 不支持 minAvailable）

`nginx-ingress-pdb.yaml`（参考格式）：

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: nginx-ingress-pdb
  namespace: ingress-nginx
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: ingress-nginx
```

```bash
kubectl apply -f nginx-ingress-pdb.yaml
# expected: poddisruptionbudget.policy/nginx-ingress-pdb created
```

## 验证

### 控制面（tccli）

```bash
# 验证节点数量和分布
tccli tke DescribeClusterInstances --region <Region> \
    --ClusterId CLUSTER_ID
# expected: 至少 3 个节点 Running
```

```json
{
  "TotalCount": "<TotalCount>",
  "InstanceSet": "<InstanceSet>",
  "InstanceId": "<InstanceId>",
  "InstanceRole": "<InstanceRole>",
  "FailedReason": "<FailedReason>",
  "InstanceState": "<InstanceState>"
}
```

### 数据面（需 VPN/IOA）

```bash
# 验证 Pod 副本数
kubectl -n ingress-nginx get pods -o wide
# expected: 3 个 Pod Running，分布在不同节点

# 验证 PDB 状态
kubectl -n ingress-nginx get pdb
# expected: nginx-ingress-pdb 显示 ALLOWED DISRUPTIONS

# 验证反亲和性生效
kubectl -n ingress-nginx get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.nodeName}{"\n"}{end}'
# expected: 每个 Pod 在不同节点上

# 验证拓扑分布
kubectl -n ingress-nginx get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.nodeName}{"\n"}{end}' | sort -k2
# expected: 按可用区分组，偏差 ≤ maxSkew

# 验证探针配置
kubectl -n ingress-nginx get deployment nginx-ingress-ingress-nginx-controller -o yaml | grep -A5 "livenessProbe\|readinessProbe"
# expected: 显示探针路径、初始延迟等配置

# 验证滚动更新不中断（模拟）
kubectl -n ingress-nginx rollout restart deployment nginx-ingress-ingress-nginx-controller
# 同时从另一终端持续 curl EXTERNAL_IP
# expected: 滚动更新期间流量不中断（少数连接重置属正常）
```

## 清理

### 数据面（需 VPN/IOA）

```bash
# 删除手动创建的 PDB
kubectl delete pdb nginx-ingress-pdb -n ingress-nginx
# expected: poddisruptionbudget.policy "nginx-ingress-pdb" deleted

# 恢复默认副本数
helm upgrade nginx-ingress ingress-nginx/ingress-nginx \
    --namespace ingress-nginx \
    --set controller.replicaCount=1 \
    --reset-values
# expected: STATUS: deployed
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `helm upgrade` 后 Pod 一直 Pending | `kubectl -n ingress-nginx describe pod <POD_NAME>` | 硬反亲和性要求每节点最多 1 个 Pod，但节点数 < 副本数 | 增加节点或改用软反亲和性 `preferredDuringScheduling` |
| `kubectl apply pdb` 返回 `cannot be applied` | `kubectl -n ingress-nginx get pdb` 检查是否已存在同名 PDB | PDB 名称冲突 | 删除已有 PDB 或使用不同名称 |
| `kubectl drain node` 被 PDB 阻止 | `kubectl -n ingress-nginx describe pdb` | `minAvailable` 设得过高，驱逐 Pod 后不满足最小可用数 | 降低 `minAvailable` 或临时删除 PDB 后再 drain |

### 配置生效后功能异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 滚动更新时流量中断 | `kubectl -n ingress-nginx logs -l app.kubernetes.io/name=ingress-nginx --tail=50` | 优雅终止时间不足，Pod 强制 kill 时仍有未完成连接 | 增加 `terminationGracePeriodSeconds` 至 300 以上 |
| Pod 频繁重启（liveness 误杀） | `kubectl -n ingress-nginx describe pod <POD_NAME>` 查看 Events 中的 `Liveness probe failed` | 探针超时太紧或失败阈值太低 | 增加 `initialDelaySeconds`、`timeoutSeconds` 和 `failureThreshold`（正常启动慢的大容器需要更宽容的探针） |
| Pod Ready 但流量 503 | `kubectl -n ingress-nginx logs -l app.kubernetes.io/name=ingress-nginx | grep health` | 就绪探针通过但 Nginx 实际未完成配置加载 | 确保 `/healthz` 端点检查了 Nginx 内部状态，而非仅返回 200 |
| 节点故障后 Pod 漂移慢 | `kubectl -n ingress-nginx get events` | PDB + 反亲和性共同导致 Pod 无法调度到剩余节点 | 确保剩余节点数 ≥ minAvailable；临时删除 PDB 加速恢复 |

## 下一步

- [高并发场景优化](https://cloud.tencent.com/document/product/457/104859) — Worker 进程、keepalive、sysctl 调优
- [可观测性集成](https://cloud.tencent.com/document/product/457/104861) — Prometheus 监控、CLS 日志
- [安装多个 Nginx Ingress Controller](https://cloud.tencent.com/document/product/457/104863) — 多 Ingress Class 隔离

## 控制台替代

通过 [TKE 控制台 - 工作负载](https://console.cloud.tencent.com/tke2/cluster/sub/list/deployment/deployment?rid=1) 查看 Pod 分布、配置亲和性和探针。
