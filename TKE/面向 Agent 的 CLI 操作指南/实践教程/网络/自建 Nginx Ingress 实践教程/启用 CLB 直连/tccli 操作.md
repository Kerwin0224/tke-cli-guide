# 启用 CLB 直连（tccli）

> 对照官方：[启用 CLB 直连](https://cloud.tencent.com/document/product/457/104866) · page_id `104866`

## 概述

CLB 直连 Pod 模式（Direct Pod Access）允许 CLB 直接将流量转发到 Pod IP，跳过 NodePort（iptables/IPVS）转发层，降低延迟、提升转发性能，并保留客户端源 IP。

**方案对比**：

| 特性 | NodePort 模式（默认） | CLB 直连模式 |
|------|----------------------|------------|
| 转发路径 | CLB → NodePort → iptables/IPVS → Pod | CLB → Pod IP（直连） |
| 源 IP 保留 | 需 externalTrafficPolicy: Local | 默认保留 |
| 性能 | 多一层 NAT 转发 | 更低延迟 |
| 要求 | 无特殊要求 | 需 VPC-CNI 网络模式（Pod IP 可路由） |

选择 CLB 直连的场景：需要保留客户端源 IP、追求极致转发性能、VPC-CNI 网络模式集群。

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
#    clb:DescribeLoadBalancers, clb:ModifyLoadBalancerAttributes
#    vpc:DescribeSubnets
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
# 5. 确认集群网络模式（必须是 VPC-CNI）
tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterNetworkSettings 中 NetworkType 包含 "VPC-CNI" 字样

# 6. 确认 Nginx Ingress 已安装
helm list -n ingress-nginx
# expected: nginx-ingress 状态 deployed

# 7. 查询 Service 使用的 CLB ID
kubectl -n ingress-nginx get svc nginx-ingress-ingress-nginx-controller \
    -o jsonpath='{.metadata.annotations.service\.kubernetes\.io/loadbalance-id}'
# expected: 输出 lb-xxxxxxxx
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

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 查看集群网络模式 | `tccli tke DescribeClusters` | 是 |
| 查看 CLB 详情 | `tccli clb DescribeLoadBalancers` | 是 |
| 修改 CLB 属性 | `tccli clb ModifyLoadBalancerAttributes` | 否 |
| 修改 Service 注解 | `kubectl annotate svc` | 否 |
| 启用 VPC-CNI 直连 | `kubectl edit svc`（加注解） | 否 |

## 关键字段说明

以下说明 CLB 直连相关的 Service Annotation 和 CLB 属性。

| 字段/注解 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `service.cloud.tencent.com/direct-access` | Annotation | 是 | `"true"` 启用直连，删除或设为 `"false"` 关闭 | 非 VPC-CNI 网络模式 → 设置后不生效且 Pod 不可达 |
| `service.kubernetes.io/qcloud-loadbalancer-backends-label` | Annotation | 否 | 标签选择器，如 `"app=nginx-ingress"`，用于筛选直连后端 Pod | 标签不匹配 → 直连无后端，流量丢失 |
| `ClusterNetworkSettings.NetworkType` | Cluster 属性 | — | 必须是 VPC-CNI 模式（先决条件） | 非 VPC-CNI → 直连不可用，Pod IP 在 CLB 网络外不可达 |
| `externalTrafficPolicy` | Service Spec | 推荐 | `"Local"`（保留源 IP），默认 `"Cluster"` | `Cluster` 模式下源 IP 为 Node IP |

## 操作步骤

### 步骤 1：确认 VPC-CNI 网络模式

#### 选择依据

- **VPC-CNI vs GlobalRouter**：CLB 直连要求 Pod IP 在 VPC 内可路由，仅 VPC-CNI 网络模式的 Pod 有独立的 VPC IP（Elastic Network Interface）。GlobalRouter 模式下 Pod IP 在 overlay 网络中，CLB 无法直连。
- 如果集群不是 VPC-CNI 模式，CLB 直连不可用。考虑新建 VPC-CNI 集群或将工作负载迁移。

```bash
tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterNetworkSettings 包含 VPC-CNI 相关信息
```

```json
{
  "Clusters": [
    {
      "ClusterId": "cls-example",
      "ClusterName": "MANAGED_CLUSTER",
      "ClusterNetworkSettings": {
        "ClusterCIDR": "10.1.0.0/16",
        "ServiceCIDR": "10.2.0.0/16",
        "VpcId": "vpc-example",
        "EniSubnetIds": ["subnet-example"],
        "Cni": true
      }
    }
  ]
}
```

> 确认 `Cni` 为 `true` 或存在 `EniSubnetIds` 配置，表明使用 VPC-CNI 模式。

### 步骤 2：修改 Service 注解启用 CLB 直连

#### 选择依据

- **启用直连**：设置 `service.cloud.tencent.com/direct-access: "true"` 后，CLB 后端从 Node IP:NodePort 切换为 Pod IP:TargetPort。
- **设置 externalTrafficPolicy**：`Local` 模式下保留源 IP，但要求 Pod 均匀分布在所有节点；如 Pod 仅部分节点有，那些节点上的 NodePort 会返回 503。
- **后端标签**：如只希望特定标签的 Pod 接收流量，通过 Annotation 指定标签选择器。

`nginx-ingress-direct-access.yaml`：

```yaml
controller:
  service:
    type: LoadBalancer
    externalTrafficPolicy: Local
    annotations:
      service.cloud.tencent.com/direct-access: "true"
      service.kubernetes.io/qcloud-loadbalancer-backends-label: "app.kubernetes.io/name=ingress-nginx"
```

```bash
helm upgrade nginx-ingress ingress-nginx/ingress-nginx \
    --namespace ingress-nginx \
    -f nginx-ingress-direct-access.yaml \
    --reuse-values
# expected: STATUS: deployed
```

> **kubectl 数据面不可达**：CAM 拒绝公网端点（strategyId:240463971），内网需 VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

### 步骤 3：手动修改 Service 注解（无需 Helm 升级时）

```bash
# 启用 CLB 直连
kubectl -n ingress-nginx annotate svc nginx-ingress-ingress-nginx-controller \
    service.cloud.tencent.com/direct-access=true --overwrite
# expected: service annotated

# 设置 externalTrafficPolicy（保留源 IP）
kubectl -n ingress-nginx patch svc nginx-ingress-ingress-nginx-controller \
    -p '{"spec":{"externalTrafficPolicy":"Local"}}'
# expected: service patched
```

## 验证

### 控制面（tccli）

```bash
# 查看 CLB 后端目标类型
tccli clb DescribeTargets --region <Region> \
    --LoadBalancerId CLB_ID
# expected: Targets 列表中 Type 为 "ENI"（直连模式）而非 "CVM"
```

```json
{
  "Listeners": [
    {
      "ListenerId": "lbl-example",
      "Protocol": "TCP",
      "Port": 80,
      "Targets": [
        {
          "Type": "ENI",
          "EniIp": "10.1.0.10",
          "Port": 80,
          "Weight": 10
        }
      ]
    }
  ]
}
```

### 数据面（需 VPN/IOA）

```bash
# 验证 Service 注解
kubectl -n ingress-nginx get svc nginx-ingress-ingress-nginx-controller -o yaml | grep -E "direct-access|backends-label"
# expected: 包含 service.cloud.tencent.com/direct-access: "true"

# 验证 Service externalTrafficPolicy
kubectl -n ingress-nginx get svc nginx-ingress-ingress-nginx-controller -o jsonpath='{.spec.externalTrafficPolicy}'
# expected: Local

# 验证 Pod IP 可达性（从同 VPC 内其他 CVM）
kubectl -n ingress-nginx get pods -o wide
# expected: 显示 Pod IP（VPC IP 段）

# 测试外部访问并查看源 IP 是否保留
curl -s EXTERNAL_IP
# expected: 正常返回，后端日志中显示真实客户端 IP
```

## 清理

### 数据面（需 VPN/IOA）

> **⚠️ 警告**：关闭 CLB 直连后，CLB 后端从 Pod IP 切回 Node IP:NodePort，期间可能出现短暂流量中断。

```bash
# 关闭 CLB 直连
kubectl -n ingress-nginx annotate svc nginx-ingress-ingress-nginx-controller \
    service.cloud.tencent.com/direct-access-
# expected: annotation removed

# 恢复 externalTrafficPolicy 为 Cluster
kubectl -n ingress-nginx patch svc nginx-ingress-ingress-nginx-controller \
    -p '{"spec":{"externalTrafficPolicy":"Cluster"}}'
# expected: service patched
```

### 控制面（tccli）

```bash
# 验证 CLB 后端类型已切换
tccli clb DescribeTargets --region <Region> \
    --LoadBalancerId CLB_ID
# expected: Targets 列表中 Type 变为 "CVM"
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl annotate` 返回 `already has a value` | `kubectl -n ingress-nginx get svc -o yaml | grep direct-access` | 注解已存在，需要使用 `--overwrite` | 添加 `--overwrite` 参数：`kubectl annotate ... --overwrite` |
| `helm upgrade` 后直连未生效 | `kubectl -n ingress-nginx get svc -o yaml | grep direct-access` | values 文件注解未正确传递，或 helm 使用了 `--reuse-values` 覆盖 | 确认 values 语法正确，使用 `--reset-values` 避免旧值冲突 |

### 配置生效后功能异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 直连后流量不通（503） | `tccli clb DescribeTargets --region <Region> --LoadBalancerId CLB_ID` 检查 Targets | CLB 后端类型为 "ENI" 但目标 Pod 不存在或不可达 | `kubectl -n ingress-nginx get pods -o wide` 确认 Pod 有 VPC IP；如 Pod 未就绪，检查 Pod 状态 |
| 直连后源 IP 仍为 Node IP | `kubectl -n ingress-nginx get svc -o jsonpath='{.spec.externalTrafficPolicy}'` | `externalTrafficPolicy` 未设为 `Local` | `kubectl patch svc ... externalTrafficPolicy: Local` |
| 部分节点流量不通 | `kubectl -n ingress-nginx get pods -o wide` 查看 Pod 分布 | `externalTrafficPolicy: Local` 下，Pod 未运行的节点上的 NodePort 返回 503 | 增加 Pod 副本或使用 `Cluster` 模式 |
| `DescribeTargets` 返回空 | `kubectl -n ingress-nginx get pods -l app.kubernetes.io/name=ingress-nginx` | 后端标签不匹配，CLB 找不到符合条件的 Pod | 检查 Annotation `qcloud-loadbalancer-backends-label` 与 Pod 标签是否一致 |

## 下一步

- [自定义负载均衡器](https://cloud.tencent.com/document/product/457/104858) — CLB 带宽、网络类型、复用配置
- [高并发场景优化](https://cloud.tencent.com/document/product/457/104859) — Worker 进程、keepalive、sysctl 调优
- [高可用配置优化](https://cloud.tencent.com/document/product/457/104860) — 多副本、PDB、健康检查

## 控制台替代

通过 [TKE 控制台 - 集群详情](https://console.cloud.tencent.com/tke2/cluster) 查看网络模式，通过 [CLB 控制台](https://console.cloud.tencent.com/clb) 查看后端目标类型。
