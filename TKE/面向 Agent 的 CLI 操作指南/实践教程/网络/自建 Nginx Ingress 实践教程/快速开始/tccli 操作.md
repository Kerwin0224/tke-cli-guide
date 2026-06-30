# 快速开始（tccli）

> 对照官方：[快速开始](https://cloud.tencent.com/document/product/457/104857) · page_id `104857`

## 概述

通过 Helm 在 TKE 集群中安装自建 Nginx Ingress Controller，并通过 CLB 对外暴露 Ingress 服务。核心流程：添加 Helm 仓库 → 安装 Nginx Ingress → 验证 Pod 运行 → 创建 Ingress 规则 → 测试外部访问。

**方案对比**：

| 方案 | InstallAddon（TKE 插件） | Helm 自建 |
|------|--------------------------|----------|
| 升级控制 | TKE 控制台统一管理，版本受限 | Helm 独立管理，可使用社区最新版 |
| CLB 配置 | 自动创建，配置有限 | 支持完整 CLB 注解自定义 |
| 多实例 | 不支持 | 支持安装多个 Controller |
| 适用场景 | 简单场景、快速接入 | 需要高级定制、多实例、灰度发布 |

选择自建 Nginx Ingress 的场景：需要自定义 CLB 带宽/网络类型、多 Ingress Class、或社区最新特性。

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
# expected: secretId、secretKey、region 均已配置，region 为目标地域

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:DescribeClusters, tke:DescribeClusterKubeconfig
#    tke:InstallAddon, tke:DescribeAddon
# 验证
tccli tke DescribeClusters --region <Region>
# expected: exit 0，返回集群列表（可为空）

# 4. 检查 kubectl 可用性（须 VPN/IOA）
kubectl version --client
# expected: Client Version v1.30+

# 5. 检查 Helm 可用性
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
# 6. 确认目标集群状态
tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterStatus "Running"

# 7. 获取 kubeconfig（须 VPN/IOA 方可使用 kubectl）
tccli tke DescribeClusterKubeconfig --region <Region> --ClusterId CLUSTER_ID | jq -r '.Kubeconfig' > ~/.kube/config

# 8. 验证集群可达性（须 VPN/IOA）
kubectl cluster-info
# expected: Kubernetes control plane 地址

# 9. 查询 CLB 配额
tccli clb DescribeQuota --region <Region>
# expected: 剩余配额 ≥ 1
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

### 版本与规格选择

- K8s 版本：推荐 ≥ 1.30.0
- Nginx Ingress Controller：推荐社区最新稳定版
- CLB 类型：默认公网 CLB（`OPEN`），如需内网访问选 `INTERNAL`

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 查看集群列表 | `tccli tke DescribeClusters` | 是 |
| 获取 kubeconfig | `tccli tke DescribeClusterKubeconfig` | 是 |
| 查看集群组件 | `tccli tke DescribeAddon` | 是 |
| 安装 Nginx Ingress（插件） | `tccli tke InstallAddon --AddonName NginxIngress` | 否 |
| 添加 Helm 仓库 | `helm repo add` | 否 |
| 安装 Helm Chart | `helm install` | 否 |
| 创建 Ingress | `kubectl apply -f ingress.yaml` | 否 |
| 查看 CLB 实例 | `tccli clb DescribeLoadBalancers` | 是 |

## 操作步骤

### 步骤 1：添加 Nginx Ingress Helm 仓库

#### 选择依据

- **Helm 仓库**：使用 Kubernetes 官方社区仓库 `ingress-nginx`，由 Kubernetes 社区维护，稳定性最佳。
- 如需国内加速，可替换为腾讯云镜像源。

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
# expected: "Update Complete" 或 "Already up to date"
```

### 步骤 2：安装 Nginx Ingress Controller

#### 选择依据

- **Controller 副本数**：最小 1 副本，生产环境建议 ≥ 2 副本。
- **CLB 类型**：默认公网（`OPEN`），用于外部访问测试；生产环境可选内网 CLB（`INTERNAL`）。
- **Service Type**：`LoadBalancer` 自动创建 CLB，是 TKE 环境的标准做法。

#### 最小安装

创建 `nginx-ingress-values-minimal.yaml`：

```yaml
controller:
  replicaCount: 1
  service:
    type: LoadBalancer
    annotations:
      service.kubernetes.io/qcloud-loadbalancer-internal-subnetid: "SUBNET_ID"
```

```bash
helm install nginx-ingress ingress-nginx/ingress-nginx \
    --namespace ingress-nginx \
    --create-namespace \
    -f nginx-ingress-values-minimal.yaml
# expected: STATUS: deployed
```

> **kubectl 数据面不可达**：CAM 拒绝公网端点（strategyId:240463971），内网需 VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

#### 增强配置（含资源限制、日志）

创建 `nginx-ingress-values-enhanced.yaml`：

```yaml
controller:
  replicaCount: 2
  service:
    type: LoadBalancer
    annotations:
      service.kubernetes.io/qcloud-loadbalancer-internal-subnetid: "SUBNET_ID"
  resources:
    requests:
      cpu: 100m
      memory: 256Mi
    limits:
      cpu: 500m
      memory: 512Mi
  metrics:
    enabled: true
```

```bash
helm upgrade nginx-ingress ingress-nginx/ingress-nginx \
    --namespace ingress-nginx \
    -f nginx-ingress-values-enhanced.yaml
# expected: STATUS: deployed
```

| 参数 | 说明 | 约束 | 获取方式 |
|------|------|------|---------|
| `SUBNET_ID` | 子网 ID（内网 CLB） | 必须，公网 CLB 省略 | `tccli vpc DescribeSubnets --region <Region> | jq '.SubnetSet[].SubnetId'` |
| `replicaCount` | Controller 副本数 | ≥ 1 | 自定义 |

### 步骤 3：查看 CLB 地址

```bash
kubectl -n ingress-nginx get svc nginx-ingress-ingress-nginx-controller
# expected: EXTERNAL-IP 不为空
```

```text
NAME                                         TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)                      AGE
nginx-ingress-ingress-nginx-controller       LoadBalancer   10.2.100.100   EXTERNAL_IP      80:30080/TCP,443:30443/TCP   2m
```

```bash
# 获取 CLB 实例 ID
kubectl -n ingress-nginx get svc nginx-ingress-ingress-nginx-controller \
    -o jsonpath='{.metadata.annotations.service\.kubernetes\.io/loadbalance-id}'
# expected: 输出 CLB 实例 ID
```

```bash
# 查看 CLB 详情
tccli clb DescribeLoadBalancers --region <Region> \
    --LoadBalancerIds '["CLB_ID"]'
```

```json
{
  "LoadBalancerSet": [
    {
      "LoadBalancerId": "lb-example",
      "LoadBalancerName": "nginx-ingress",
      "LoadBalancerType": "OPEN",
      "Status": 1,
      "LoadBalancerVips": ["EXTERNAL_IP"]
    }
  ]
}
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `CLB_ID` | CLB 实例 ID | 格式 `lb-xxxxxxxx` | `kubectl get svc ...` 注解或 `tccli clb DescribeLoadBalancers` |
| `EXTERNAL_IP` | CLB 公网 IP | — | 上述命令输出 |

### 步骤 4：创建测试 Ingress 规则

#### 准备测试应用

`test-app.yaml`：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-test
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-test
  template:
    metadata:
      labels:
        app: nginx-test
    spec:
      containers:
      - name: nginx
        image: nginx:1.25-alpine
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-test-svc
spec:
  selector:
    app: nginx-test
  ports:
  - port: 80
    targetPort: 80
```

```bash
kubectl apply -f test-app.yaml
# expected: deployment.apps/nginx-test created, service/nginx-test-svc created
```

#### 创建 Ingress

`test-ingress.yaml`：

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-test-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: INGRESS_HOST
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-test-svc
            port:
              number: 80
```

```bash
kubectl apply -f test-ingress.yaml
# expected: ingress.networking.k8s.io/nginx-test-ingress created
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `INGRESS_HOST` | 域名，用于 Ingress Host 匹配 | 可先用 IP 测试 | 自定义域名，或绑定 CLB IP 的测试域名 |

## 验证

### 控制面（tccli）

```bash
# 验证 CLB 状态正常
tccli clb DescribeLoadBalancers --region <Region> \
    --LoadBalancerIds '["CLB_ID"]'
# expected: Status 为 1（正常）

# 验证监听器已创建
tccli clb DescribeListeners --region <Region> \
    --LoadBalancerId CLB_ID
# expected: HTTP:80 和 HTTPS:443 监听器已创建
```

### 数据面（需 VPN/IOA）

```bash
# 验证 Helm Release 状态
helm list -n ingress-nginx
# expected: ingress-nginx 状态 deployed

# 验证 Pod 运行
kubectl -n ingress-nginx get pods
# expected: ingress-nginx-controller-* Pod Running

# 验证 Ingress 已创建
kubectl get ingress nginx-test-ingress
# expected: ADDRESS 列显示 CLB IP

# 验证 Service
kubectl get svc -n ingress-nginx
# expected: EXTERNAL-IP 不为空

# 测试外部访问（从可出网的机器）
curl -H "Host: INGRESS_HOST" http://EXTERNAL_IP/
# expected: 返回 Nginx 欢迎页或测试应用响应
```

```text
NAME  STATUS  AGE
...
```

## 清理

### 数据面（需 VPN/IOA）

> **⚠️ 警告**：删除 Ingress 和 Service 后，外部流量将中断。确认无生产流量后再执行。

```bash
# 1. 删除测试 Ingress
kubectl delete ingress nginx-test-ingress
# expected: ingress.networking.k8s.io "nginx-test-ingress" deleted

# 2. 删除测试应用
kubectl delete -f test-app.yaml
# expected: deployment.apps "nginx-test" deleted, service "nginx-test-svc" deleted

# 3. 卸载 Nginx Ingress Helm Release
helm uninstall nginx-ingress -n ingress-nginx
# expected: release "nginx-ingress" uninstalled

# 4. 删除命名空间
kubectl delete ns ingress-nginx
# expected: namespace "ingress-nginx" deleted
```

### 控制面（tccli）

> **⚠️ 警告**：Helm 卸载 Nginx Ingress 时，`LoadBalancer` 类型的 Service 会自动触发 CLB 删除。如果您使用了已存在的 CLB（通过 Service Annotation 指定），卸载 Service 不会删除该 CLB，需手动清理。

```bash
# 验证 CLB 已释放或不再关联
tccli clb DescribeLoadBalancers --region <Region> \
    --LoadBalancerIds '["CLB_ID"]'
# expected: 返回空列表（自动删除）或 CLB 不再有后端

# 如果 CLB 未被自动删除（使用了已有 CLB），手动清理监听器
tccli clb DeleteListener --region <Region> \
    --LoadBalancerId CLB_ID \
    --ListenerId LISTENER_ID
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `helm install` 返回 `Error: INSTALLATION FAILED` | `helm list -n ingress-nginx` 检查是否已存在同名 Release | Helm Release 名称冲突 | `helm uninstall nginx-ingress -n ingress-nginx` 后重试，或使用不同名称 |
| `kubectl apply` 返回 `connection refused` | `kubectl cluster-info` | kubeconfig 过期或集群 API Server 不可达 | 重新获取 kubeconfig：`tccli tke DescribeClusterKubeconfig --region <Region> --ClusterId CLUSTER_ID \| jq -r '.Kubeconfig' > ~/.kube/config`，并通过 VPN/IOA 连接 |
| `kubectl apply` 返回 `Unauthorized` | `kubectl auth can-i create ingress` | CAM 策略拒绝（strategyId:240463971，公网端点不可达） | 通过 VPN/IOA 连接内网端点 |

### 安装成功但功能异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Ingress 无外部 IP | `kubectl -n ingress-nginx get svc` 查看 EXTERNAL-IP | CLB 创建中或子网配置错误 | 等待 2-5 分钟；如超过 10 分钟，检查 `kubectl -n ingress-nginx describe svc` 中的 Events |
| 外部无法访问 | `tccli clb DescribeListeners --region <Region> --LoadBalancerId CLB_ID` | 安全组未放通 80/443 端口 | 配置 CLB 安全组放通入站 80/443 |
| Ingress 返回 404 | `kubectl describe ingress nginx-test-ingress` | Host 不匹配或后端 Service 不存在 | 检查 `curl` 的 `-H "Host: ..."` 是否与 Ingress 规则一致 |
| 后端返回 503 | `kubectl get endpoints nginx-test-svc` 检查是否为空 | 后端 Pod 无 Ready 状态 | `kubectl describe pod -l app=nginx-test` 检查 Pod 状态 |

## 下一步

- [自定义负载均衡器](https://cloud.tencent.com/document/product/457/104858) — 配置 CLB 带宽、网络类型、已有 CLB 复用
- [启用 CLB 直连](https://cloud.tencent.com/document/product/457/104866) — CLB 直连 Pod，提升转发性能
- [高可用配置优化](https://cloud.tencent.com/document/product/457/104860) — 多副本、PDB、健康检查
- [values.yaml 完整配置示例](https://cloud.tencent.com/document/product/457/104865) — 完整 Helm Values 参考

## 控制台替代

通过 [TKE 控制台 - 组件管理](https://console.cloud.tencent.com/tke2/cluster/sub/list/addon/cluster?rid=1) 安装 Nginx Ingress 插件，或通过 [容器镜像服务 TCR](https://console.cloud.tencent.com/tcr) 拉取自定义镜像。
