# 注册节点流量接入

> 对照官方：[流量接入](https://cloud.tencent.com/document/product/457/79749) · page_id `79749` · tccli ≥3.1.107 · API 2018-05-25

## 概述

注册节点部署在 IDC 或第三方云环境，默认无法被腾讯云 CLB 直接挂载。通过 CLB 混合云部署特性，可将 CLB 绑定至注册节点，实现四层（TCP/UDP）和七层（HTTP/HTTPS）流量接入。

| 接入方式 | K8s 资源 | CLB 类型 | 适用场景 |
|---------|---------|---------|---------|
| 四层流量接入 | `Service`（`type: LoadBalancer`） | 四层 CLB | TCP/UDP 协议、无需 URL 路由、高性能转发 |
| 七层流量接入 | `Ingress` | 七层 CLB | HTTP/HTTPS 协议、基于域名/路径路由、TLS 终止 |

核心机制：通过 annotation 指定 `hybrid-type: CCN`（云联网）和 `snat-pro-info`（SnatIP 分配子网），CLB 将请求经云联网转发至 IDC 内注册节点。控制面（tccli）管理集群配置和端点，数据面（kubectl）管理 Service/Ingress 资源。

> **注意**：以下 kubectl 命令需在集群端点可达的环境下执行。当前环境内网端点仅 VPC 内可达，公网端点被 CAM 策略限制。如无法直接执行，可通过 IOA/VPN/专线接入 VPC 后操作，或登录同 VPC CVM 执行。

## 前置条件

- [环境准备](../../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 3.1.107

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置，region 为目标地域

# 3. 检查 kubectl 版本
kubectl version --client
# expected: Client Version 已安装，版本 >= 1.28

# 4. 检查 CAM 权限 — 需要以下 Action 名
#    tke:DescribeClusters, tke:DescribeClusterEndpoints
#    tke:DescribeExternalNodeSupportConfig, tke:DescribeClusterKubeconfig
#    vpc:DescribeVpcs, vpc:DescribeSubnets
#    clb:DescribeLoadBalancers
# 验证：执行 DescribeClusters 确认权限
tccli tke DescribeClusters --region <Region>
# expected: exit 0，返回集群列表（可为空）

# 验证 VPC 权限
tccli vpc DescribeVpcs --region <Region>
# expected: exit 0，返回 VPC 列表（可为空）

# 验证 CLB 权限
tccli clb DescribeLoadBalancers --region <Region>
# expected: exit 0，返回 CLB 列表（可为空）
```

**预期输出参考**：

```text
tccli version >= 1.0.0

# tccli configure list 输出示例：
secretId:     AKID********************************
secretKey:    ********************************
region:       ap-guangzhou

# kubectl version --client 输出示例：
Client Version: v1.28.0
```

```json
# DescribeClusters（权限验证）输出示例：
{
    "TotalCount": 1,
    "Clusters": [],
    "RequestId": "..."
}
```

```json
# DescribeVpcs 输出示例：
{
    "TotalCount": 1,
    "VpcSet": [
        {
            "VpcId": "vpc-example",
            "VpcName": "example-vpc",
            "CidrBlock": "10.0.0.0/16"
        }
    ],
    "RequestId": "..."
}
```

### 资源检查

```bash
# 5. 确认集群已启用注册节点支持
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["<ClusterId>"]'
# expected: EnableExternalNode 为 true，ClusterStatus 为 "Running"

# 6. 确认 CLB 混合云部署已开通
#    非 tccli 操作，需在 CLB 控制台确认是否支持混合云部署。
#    如未开通，需通过工单申请。

# 7. 查询可用子网（用于分配 SnatIP）
tccli vpc DescribeSubnets --region <Region> \
    --Filters '[{"Name":"vpc-id","Values":["<VpcId>"]}]'
# expected: 至少返回 1 个子网，AvailableIpCount >= 1

# 8. 查询注册节点支持配置
tccli tke DescribeExternalNodeSupportConfig --region <Region> \
    --ClusterId <ClusterId>
# expected: exit 0，返回 NetworkType 和 Enabled 状态
```

**预期输出参考**：

```json
# DescribeClusters（资源检查）输出示例：
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "cls-example",
            "ClusterName": "example-cluster",
            "ClusterStatus": "Running",
            "EnableExternalNode": true
        }
    ],
    "RequestId": "..."
}
```

```json
# DescribeSubnets 输出示例：
{
    "TotalCount": 2,
    "SubnetSet": [
        {
            "SubnetId": "subnet-example",
            "SubnetName": "example-subnet",
            "VpcId": "vpc-example",
            "CidrBlock": "10.0.1.0/24",
            "AvailableIpAddressCount": 200
        }
    ],
    "RequestId": "..."
}
```

```json
# DescribeExternalNodeSupportConfig 输出示例：
{
    "NetworkType": "HostNetwork",
    "SubnetId": "subnet-example",
    "Enabled": false,
    "Status": "Enabled",
    "EnabledPublicConnect": true,
    "PublicConnectUrl": "https://external.example.com",
    "RequestId": "..."
}
```

### 占位符参考

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<ClusterId>` | 集群 ID | 格式 `cls-xxxxxxxx` | `tccli tke DescribeClusters --region <Region>`，输出 `$.Clusters[*].ClusterId` |
| `<Region>` | 地域 | 如 `ap-guangzhou` | `tccli tke DescribeRegions` |
| `<VpcId>` | VPC 实例 ID | 已存在的 VPC | `tccli vpc DescribeVpcs --region <Region>`，输出 `$.VpcSet[*].VpcId` |
| `<SubnetId>` | 子网 ID | VPC 下子网，可用 IP >= 1 | `tccli vpc DescribeSubnets --region <Region>`，输出 `$.SubnetSet[*].SubnetId` |
| `INTRANET_IP` | 内网端点 IP | VPC 内网地址 | `tccli tke DescribeClusterEndpoints --region <Region> --ClusterId <ClusterId>` |
| `SERVICE_NAME` | Service 名称 | K8s 命名规范，长度 <= 63 | 自定义（用户提供） |
| `APP_NAME` | 后端应用标签 | 需匹配 Pod 的 `app` label | `kubectl get pods -o wide --show-labels`（用户提供） |
| `INGRESS_NAME` | Ingress 名称 | K8s 命名规范，长度 <= 63 | 自定义（用户提供） |
| `NAMESPACE` | 命名空间 | 已存在的 K8s Namespace | `kubectl get ns`（用户提供） |
| `HOST_NAME` | 域名 | 需已备案（国内 CLB） | 自定义，DNS 解析至 CLB VIP（用户提供） |
| `TLS_SECRET_NAME` | TLS 证书 Secret | K8s Secret（类型 `kubernetes.io/tls`） | `kubectl get secret`（用户提供） |
| `API_SERVICE_NAME` | API 后端 Service 名称 | 已存在的 Service | `kubectl get svc`（用户提供） |
| `CLB_VIP_ADDR` | CLB VIP 地址 | 由 CLB 自动分配 | `kubectl get svc <ServiceName> -o jsonpath='{.status.loadBalancer.ingress[0].ip}'` |
| `PUBLIC_CONNECT_URL` | 公网连接地址 | 注册节点公网连接 URL | `tccli tke DescribeExternalNodeSupportConfig`，输出 `$.PublicConnectUrl` |

## 控制台与 CLI 参数映射

| 控制台操作 | CLI 命令 | 幂等 |
|-----------|---------|:--:|
| 查看集群状态 | `tccli tke DescribeClusters` | 是 |
| 查询端点状态 | `tccli tke DescribeClusterEndpoints` | 是 |
| 查询注册节点支持 | `tccli tke DescribeExternalNodeSupportConfig` | 是 |
| 获取 kubeconfig | `tccli tke DescribeClusterKubeconfig` | 是 |
| 创建四层流量接入 | `kubectl apply -f service.yaml` | 是 |
| 创建七层流量接入 | `kubectl apply -f ingress.yaml` | 是 |
| 查看 Service | `kubectl get svc SERVICE_NAME` | 是 |
| 查看 Ingress | `kubectl get ingress INGRESS_NAME` | 是 |
| 删除 Service | `kubectl delete svc SERVICE_NAME` | 是 |
| 删除 Ingress | `kubectl delete ingress INGRESS_NAME` | 是 |

## 关键字段说明

以下说明 Service 和 Ingress 中用于注册节点流量接入的关键 annotation 字段。

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `service.cloud.tencent.com/hybrid-type` | String | 是 | `CCN`（云联网），当前唯一支持值 | 缺少此 annotation → CLB 仅挂载云上节点，注册节点不被加入后端 |
| `service.cloud.tencent.com/snat-pro-info` | JSON String | 是 | `{"snatIps":[{"subnetId":"<SubnetId>"}]}`，subnetId 为 VPC 子网 ID | 缺少此 annotation → CLB 后端无法绑定 IDC IP |
| `kubernetes.io/ingress.class` | String | 是 | `qcloud`，指定使用腾讯云 CLB Ingress Controller | 缺少此 annotation → Ingress 不被 CLB Ingress Controller 处理 |
| `ingress.cloud.tencent.com/hybrid-type` | String | 是 | `CCN`（云联网），当前唯一支持值 | 缺少此 annotation → CLB Ingress 后端仅挂载云上节点 |
| `ingress.cloud.tencent.com/snat-pro-info` | JSON String | 是 | `{"snatIps":[{"subnetId":"<SubnetId>"}]}` | 缺少此 annotation → CLB Ingress 后端无法绑定 IDC IP |
| `externalTrafficPolicy` | String | 否 | `Local`（推荐）或 `Cluster`。`Local` 保留客户端源 IP 且避免跨专线转发 | 使用 `Cluster` 模式 → 流量可能在节点间转发，增加专线带宽消耗、丢失客户端源 IP |

## 操作步骤

### 步骤 1：确认注册节点支持已启用

#### 选择依据

注册节点流量接入依赖 CLB 混合云部署，前提是集群已启用注册节点支持（`EnableExternalNode`）。必须先确认此配置。

- **网络模式**：当前集群使用 `HostNetwork` 模式（由 `EnableExternalNodeSupport` 时自动指定），CLB 流量通过 CCN（云联网）转发至 IDC 内注册节点。
- **如何确认**：`tccli tke DescribeExternalNodeSupportConfig --region <Region> --ClusterId <ClusterId>`

```bash
# 查询集群基本信息 — 确认 EnableExternalNode 已启用
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["<ClusterId>"]'
# expected: exit 0，EnableExternalNode 为 true，ClusterStatus 为 "Running"
```

**预期输出**：

```json
{
    "Clusters": [
        {
            "ClusterId": "cls-example",
            "ClusterName": "CLUSTER_NAME",
            "ClusterStatus": "Running",
            "ClusterVersion": "1.32.2",
            "ClusterType": "MANAGED_CLUSTER",
            "EnableExternalNode": true,
            "ClusterNetworkSettings": {
                "ClusterCIDR": "10.0.0.0/16",
                "ServiceCIDR": "10.1.0.0/20",
                "VpcId": "<VpcId>"
            },
            "ContainerRuntime": "containerd",
            "DeletionProtection": false
        }
    ]
}
```

```bash
# 查询注册节点支持配置 — 确认网络模式和公网连接信息
tccli tke DescribeExternalNodeSupportConfig --region <Region> \
    --ClusterId <ClusterId>
# expected: exit 0，NetworkType 为 "HostNetwork"
```

**预期输出**：

```json
{
    "NetworkType": "HostNetwork",
    "SubnetId": "<SubnetId>",
    "Enabled": false,
    "Status": "Enabled",
    "EnabledPublicConnect": true,
    "PublicConnectUrl": "PUBLIC_CONNECT_URL"
}
```

### 步骤 2：获取 kubeconfig

```bash
# 查询集群端点状态
tccli tke DescribeClusterEndpoints --region <Region> \
    --ClusterId <ClusterId>
# expected: exit 0，返回内网/公网端点信息
```

**预期输出**：

```json
{
    "ClusterExternalEndpoint": "",
    "ClusterIntranetEndpoint": "INTRANET_IP",
    "ClusterDomain": "CLUSTER_ID.ccs.tencent-cloud.com",
    "ClusterExternalDomain": "CLUSTER_ID.ccs.tencent-cloud.com"
}
```

```bash
# 获取 kubeconfig（内网端点）
tccli tke DescribeClusterKubeconfig --region <Region> \
    --ClusterId <ClusterId> \
    --IsExtranet false | jq -r '.Kubeconfig' > ~/.kube/config
# expected: exit 0，kubeconfig 写入 ~/.kube/config
# ⚠️ 此命令会覆盖已有 kubeconfig 文件。如有其他集群配置，建议先备份：
#    cp ~/.kube/config ~/.kube/config.bak
```

**预期输出**：

```json
{
    "Kubeconfig": "<BASE64_KUBECONFIG>",
    "RequestId": "..."
}
```

### 步骤 3：四层流量接入（LoadBalancer Service）

> **警告**：创建 `type: LoadBalancer` 的 Service 将自动创建 CLB 实例（计费资源）。公网 CLB 按流量/带宽计费，内网 CLB 免实例费但可能产生内网带宽费用。详见 [CLB 计费说明](https://cloud.tencent.com/document/product/214/42935)。

> **注意**：以下 kubectl 命令需在集群端点可达的环境下执行。

#### 选择依据

*为什么选这些配置而不是其他：*

- **混合云类型**：annotation `service.cloud.tencent.com/hybrid-type: CCN`。CCN（云联网）是当前注册节点流量接入的唯一支持通道。如果不指定此 annotation，CLB 仅挂载云上 CVM 节点，注册节点不会被加入后端。
- **SnatIP**：annotation `service.cloud.tencent.com/snat-pro-info` 指定 SnatIP 的子网。CLB 需要 SnatIP（VPC 内网 IP）才能在 CLB 和后端 IDC 服务器之间做源地址转换。约束：单个 CLB 实例最多 10 个 SnatIP；单个 SnatIP + 单个后端服务最大 5.5 万连接。
- **流量策略**：`externalTrafficPolicy: Local`。`Local` 模式保留客户端源 IP，且避免流量通过 NAT 在节点间转发，减少跨专线带宽消耗。使用 `Cluster` 模式（默认）会导致流量可能在节点间转发，增加专线带宽消耗且丢失客户端源 IP。

#### 最小创建（只含必填 annotation）

`service-minimal.yaml`：

```yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    service.cloud.tencent.com/hybrid-type: CCN
    service.cloud.tencent.com/snat-pro-info: '{"snatIps": [{"subnetId":"<SubnetId>"}]}'
  name: SERVICE_NAME
spec:
  type: LoadBalancer
  externalTrafficPolicy: Local
  ports:
    - port: 80
      targetPort: 80
  selector:
    app: APP_NAME
```

```bash
kubectl apply -f service-minimal.yaml
# expected: service/SERVICE_NAME created
```

**预期输出**：

```text
service/SERVICE_NAME created
```

#### 增强配置（加健康检查、多端口、会话保持）

`service-enhanced.yaml`：

```yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    service.cloud.tencent.com/hybrid-type: CCN
    service.cloud.tencent.com/snat-pro-info: '{"snatIps": [{"subnetId":"<SubnetId>"}]}'
    service.cloud.tencent.com/health-check-flag: "on"
    service.cloud.tencent.com/health-check-type: "tcp"
    service.cloud.tencent.com/health-check-interval: "5"
    service.cloud.tencent.com/health-check-timeout: "2"
    service.cloud.tencent.com/health-check-healthnum: "3"
    service.cloud.tencent.com/health-check-unhealthnum: "3"
  name: SERVICE_NAME
spec:
  type: LoadBalancer
  externalTrafficPolicy: Local
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800
  ports:
    - name: http
      port: 80
      targetPort: 80
    - name: https
      port: 443
      targetPort: 443
  selector:
    app: APP_NAME
```

```bash
kubectl apply -f service-enhanced.yaml
# expected: service/SERVICE_NAME configured
```

**预期输出**：

```text
service/SERVICE_NAME configured
```

### 步骤 4：七层流量接入（CLB Ingress）

> **注意**：以下 kubectl 命令需在集群端点可达的环境下执行。

#### 选择依据

*为什么选这些配置而不是其他：*

- **Ingress Class**：annotation `kubernetes.io/ingress.class: qcloud`。指定使用腾讯云 CLB Ingress Controller 而非其他 Ingress Controller（如 nginx）。不指定此 annotation 则 Ingress 不会被 CLB Ingress Controller 处理。
- **混合云类型**：annotation `ingress.cloud.tencent.com/hybrid-type: CCN`。与 Service 相同，CCN 是注册节点流量接入的唯一通道。
- **SnatIP**：annotation `ingress.cloud.tencent.com/snat-pro-info`。CLB Ingress 同样需要 SnatIP 将七层请求转发至 IDC 内注册节点的后端 Service。

#### 最小创建（只含必填 annotation）

`ingress-minimal.yaml`：

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: qcloud
    ingress.cloud.tencent.com/hybrid-type: CCN
    ingress.cloud.tencent.com/snat-pro-info: '{"snatIps": [{"subnetId":"<SubnetId>"}]}'
  name: INGRESS_NAME
  namespace: NAMESPACE
spec:
  rules:
    - host: HOST_NAME
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: SERVICE_NAME
                port:
                  number: 80
```

```bash
kubectl apply -f ingress-minimal.yaml
# expected: ingress.networking.k8s.io/INGRESS_NAME created
```

**预期输出**：

```text
ingress.networking.k8s.io/INGRESS_NAME created
```

#### 增强配置（加 TLS、多路由、自定义 CLB 配置）

`ingress-enhanced.yaml`：

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: qcloud
    ingress.cloud.tencent.com/hybrid-type: CCN
    ingress.cloud.tencent.com/snat-pro-info: '{"snatIps": [{"subnetId":"<SubnetId>"}]}'
    ingress.cloud.tencent.com/internet-charge-type: "TRAFFIC_POSTPAID_BY_HOUR"
    ingress.cloud.tencent.com/listen-ports: '[{"HTTPS":443}]'
  name: INGRESS_NAME
  namespace: NAMESPACE
spec:
  tls:
    - hosts:
        - HOST_NAME
      secretName: TLS_SECRET_NAME
  rules:
    - host: HOST_NAME
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: SERVICE_NAME
                port:
                  number: 80
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: API_SERVICE_NAME
                port:
                  number: 8080
```

```bash
kubectl apply -f ingress-enhanced.yaml
# expected: ingress.networking.k8s.io/INGRESS_NAME configured
```

**预期输出**：

```text
ingress.networking.k8s.io/INGRESS_NAME configured
```

## 验证

### 控制面（tccli）

```bash
# 验证集群和端点状态无变化
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["<ClusterId>"]'
# expected: ClusterStatus 仍为 "Running"，EnableExternalNode 仍为 true
```

**预期输出**：

```json
{
    "Clusters": [
        {
            "ClusterId": "cls-example",
            "ClusterStatus": "Running",
            "EnableExternalNode": true
        }
    ]
}
```

### 数据面（kubectl）

> **注意**：以下 kubectl 命令需在集群端点可达的环境下执行。

```bash
# 验证 Service 已创建并获取 CLB VIP
kubectl get svc SERVICE_NAME
# expected: EXTERNAL-IP 列显示 CLB 分配的 VIP
```

**预期输出**：

```text
NAME            TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)        AGE
SERVICE_NAME    LoadBalancer   10.1.x.x       CLB_VIP_ADDR    80:3xxxx/TCP   1m
```

```bash
# 提取 CLB VIP 地址（供后续连通性测试使用）
kubectl get svc SERVICE_NAME -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
# expected: 返回 CLB 分配的 VIP 地址，如 1.2.3.4
```

```bash
# 验证 Service 详情 — 确认 annotation 生效
kubectl get svc SERVICE_NAME -o yaml
# expected: metadata.annotations 包含 hybrid-type: CCN 和 snat-pro-info
```

**预期输出**：

```yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    service.cloud.tencent.com/hybrid-type: CCN
    service.cloud.tencent.com/snat-pro-info: '{"snatIps":[{"subnetId":"<SubnetId>"}]}'
  name: SERVICE_NAME
spec:
  type: LoadBalancer
  externalTrafficPolicy: Local
status:
  loadBalancer:
    ingress:
      - ip: CLB_VIP_ADDR
```

```bash
# 验证 Ingress 已创建
kubectl get ingress INGRESS_NAME -n NAMESPACE
# expected: ADDRESS 列显示 CLB VIP
```

**预期输出**：

```text
NAME            CLASS    HOSTS         ADDRESS         PORTS   AGE
INGRESS_NAME    qcloud   HOST_NAME     CLB_VIP_ADDR    80      1m
```

```bash
# 验证 Ingress 详情 — 确认 annotation 生效
kubectl get ingress INGRESS_NAME -n NAMESPACE -o yaml
# expected: metadata.annotations 包含 hybrid-type: CCN 和 snat-pro-info
```

**预期输出**：

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: qcloud
    ingress.cloud.tencent.com/hybrid-type: CCN
    ingress.cloud.tencent.com/snat-pro-info: '{"snatIps":[{"subnetId":"<SubnetId>"}]}'
  name: INGRESS_NAME
  namespace: NAMESPACE
spec:
  rules:
    - host: HOST_NAME
status:
  loadBalancer:
    ingress:
      - ip: CLB_VIP_ADDR
```

验证维度：

| 维度 | 命令 | 预期 |
|------|------|------|
| 集群状态 | `tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'` | `ClusterStatus: "Running"`，`EnableExternalNode: true` |
| Service 就绪 | `kubectl get svc SERVICE_NAME` | `EXTERNAL-IP` 非空（CLB VIP 已分配） |
| Service annotation | `kubectl get svc SERVICE_NAME -o yaml` | 含 `hybrid-type: CCN` 和 `snat-pro-info` |
| Ingress 就绪 | `kubectl get ingress INGRESS_NAME -n NAMESPACE` | `ADDRESS` 非空（CLB VIP 已分配） |
| Ingress annotation | `kubectl get ingress INGRESS_NAME -n NAMESPACE -o yaml` | 含 `ingress.class: qcloud`、`hybrid-type: CCN` 和 `snat-pro-info` |
| 连通性（四层） | `curl http://CLB_VIP_ADDR/` | HTTP 200（如后端 Pod 正常响应） |
| 连通性（七层） | `curl -H "Host: HOST_NAME" http://CLB_VIP_ADDR/` | HTTP 200（如后端 Pod 正常响应） |

## 清理

> **警告**：删除 Service/Ingress 时，关联的 CLB 实例可能同步删除（取决于 CLB 的删除保护配置）。CLB 实例删除后，关联的 SnatIP 和 CCN 绑定关系会被释放。如果同一 CLB 实例被多个 Service 共享，删除其中部分 Service 可能仍保留 CLB，请确认后再操作。

### 数据面（kubectl）

> **注意**：以下 kubectl 命令需在集群端点可达的环境下执行。

#### 1. 清理前状态检查

```bash
# 查看当前 Service
kubectl get svc SERVICE_NAME
# expected: 确认是待删除的目标 Service
```

**预期输出**：

```text
NAME            TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)        AGE
SERVICE_NAME    LoadBalancer   10.1.x.x       CLB_VIP_ADDR    80:3xxxx/TCP   10m
```

```bash
# 查看当前 Ingress
kubectl get ingress INGRESS_NAME -n NAMESPACE
# expected: 确认是待删除的目标 Ingress
```

**预期输出**：

```text
NAME            CLASS    HOSTS         ADDRESS         PORTS   AGE
INGRESS_NAME    qcloud   HOST_NAME     CLB_VIP_ADDR    80      10m
```

#### 2. 删除 Ingress（先删 Ingress，后删 Service）

```bash
kubectl delete ingress INGRESS_NAME -n NAMESPACE
# expected: ingress.networking.k8s.io "INGRESS_NAME" deleted
```

**预期输出**：

```text
ingress.networking.k8s.io "INGRESS_NAME" deleted
```

#### 3. 删除 Service

```bash
kubectl delete svc SERVICE_NAME
# expected: service "SERVICE_NAME" deleted
# 注意：关联的 CLB 实例可能同步删除
```

**预期输出**：

```text
service "SERVICE_NAME" deleted
```

#### 4. 验证数据面已清理

```bash
kubectl get svc SERVICE_NAME
# expected: Error from server (NotFound): services "SERVICE_NAME" not found
```

**预期输出**：

```text
Error from server (NotFound): services "SERVICE_NAME" not found
```

```bash
kubectl get ingress INGRESS_NAME -n NAMESPACE
# expected: Error from server (NotFound): ingresses.networking.k8s.io "INGRESS_NAME" not found
```

**预期输出**：

```text
Error from server (NotFound): ingresses.networking.k8s.io "INGRESS_NAME" not found
```

#### 5. 验证 CLB 实例已清理

Service/Ingress 删除后，关联的 CLB 实例通常同步释放。建议验证 CLB 实例已无残留：

```bash
# 查询 CLB 实例列表（确认无残留）
tccli clb DescribeLoadBalancers --region <Region>
# expected: 列表中不含本次创建的 LoadBalancer（如 CLB VIP 不在返回结果中）
```

**预期输出**：

```json
{
    "TotalCount": 0,
    "LoadBalancerSet": [],
    "RequestId": "..."
}
```

### 控制面（tccli）

注册节点流量接入的 Service/Ingress 删除后，控制面资源（集群、注册节点）不受影响。仅当不再使用集群时才需在控制面执行清理。

```bash
# 验证集群状态无变化
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["<ClusterId>"]'
# expected: ClusterStatus 为 "Running"，集群未被误删
```

**预期输出**：

```json
{
    "Clusters": [
        {
            "ClusterId": "cls-example",
            "ClusterStatus": "Running"
        }
    ]
}
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl` 返回 `http: server gave HTTP response to HTTPS client` | 检查 kubeconfig 中 server 地址协议：`kubectl config view \| grep server` | API Server 以 HTTP 协议响应（非 HTTPS），端点协议配置不匹配 | 将 kubeconfig 中 server 地址的 `https://` 改为 `http://` 重试。如仍失败，改用 DescribeClusterSecurity token 方式连接 |
| `kubectl` 返回 `Error from server (InternalError): an error on the server ("unknown")` | 检查端点可达性：`curl -v http://INTRANET_IP` | 内网端点 TCP 可达但 API Server 返回内部错误——HTTP 协议下认证不兼容 | 使用 DescribeClusterSecurity 获取 token 方式连接，或通过公网连接端点重试 |
| `kubectl` 返回 `context deadline exceeded (Client.Timeout exceeded while awaiting headers)` | 检查网络连通性：`telnet INTRANET_IP 80` 或 `nc -zv INTRANET_IP 80` | DescribeClusterSecurity token 通过 HTTP 连接超时——认证握手失败 | 确认网络路径通畅（需在 VPC 内或通过 IOA/VPN/专线）。保留 RequestId 以备查询 |
| `kubectl` 返回 `no such host` | `nslookup CLUSTER_ID.ccs.tencent-cloud.com` | 内网域名仅 VPC 内 DNS 可解析——公网环境无法解析 | 改用内网 IP 直接连接（从 `DescribeClusterEndpoints` 获取），或接入 VPC 后使用域名 |
| `CreateClusterEndpoint --IsExtranet true` 返回 `InvalidParameter.Param` | 检查返回错误详情中的 RequestId | 公网端点被组织级 CAM 策略 `strategyId:240463971` 以 `condition tke:clusterExtranetEndpoint=true` 硬拒绝。此为环境限制，非命令错误 | 使用内网端点（`--IsExtranet false`）作为替代方案。需通过 IOA/VPN/专线 接入 VPC 后连接，或申请公网端点 CAM 策略豁免。保留 RequestId: `3bbab8d0-7acb-4fad-a5d5-11e7131e81a3` 备查 |

### 创建成功但流量不通

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Service `EXTERNAL-IP` 已分配但无法访问 | `kubectl get svc SERVICE_NAME -o yaml \| grep hybrid-type` | 未指定 `hybrid-type` annotation，CLB 仅挂载云上节点，注册节点不被加入后端 | 添加 annotation `service.cloud.tencent.com/hybrid-type: CCN` 并重新 apply |
| Ingress `ADDRESS` 已分配但返回 503 | `kubectl get ingress INGRESS_NAME -o yaml \| grep snat-pro-info` | 未指定 `snat-pro-info` annotation，CLB 后端无法绑定 IDC IP | 添加 annotation `service.cloud.tencent.com/snat-pro-info`（Service）或 `ingress.cloud.tencent.com/snat-pro-info`（Ingress）并指定 `subnetId` |
| 流量延迟高或源 IP 丢失 | `kubectl get svc SERVICE_NAME -o yaml \| grep externalTrafficPolicy` | `externalTrafficPolicy` 为默认值 `Cluster`，流量可能在节点间转发 | 设置为 `externalTrafficPolicy: Local` 并重新 apply |
| CLB 健康检查异常 | 查看 CLB 控制台健康检查状态 | 注册节点上后端 Pod 未就绪或端口未开放 | `kubectl get pods -l app=APP_NAME` 确认 Pod Running；检查安全组和防火墙规则 |
| Ingress 创建后无 `ADDRESS` | `kubectl describe ingress INGRESS_NAME -n NAMESPACE` 查看 Events | 未指定 `kubernetes.io/ingress.class: qcloud`，Ingress 不被 CLB Ingress Controller 处理 | 添加 annotation `kubernetes.io/ingress.class: qcloud` 并重新 apply |
| CLB 后端无注册节点 | `kubectl get svc SERVICE_NAME -o yaml \| grep hybrid-type` | 未指定 `hybrid-type` annotation | 添加 annotation `service.cloud.tencent.com/hybrid-type: CCN` 或 `ingress.cloud.tencent.com/hybrid-type: CCN` 并重新 apply |

### 操作成功但 kubectl 不可达

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| kubectl 连接内网端点全部失败（HTTP/HTTPS 均不可用） | 按顺序尝试：HTTPS → HTTP → DescribeClusterSecurity token → 公网连接端点 → 域名 | 内网端点仅 VPC 内可达，公网端点被 CAM 策略 hard deny | 通过 IOA/VPN/专线 接入 VPC 后使用内网端点；或在同 VPC CVM 上执行 kubectl 命令；或向 CAM 管理员申请公网端点策略豁免（strategyId:240463971） |

## 下一步

- [注册节点概述](https://cloud.tencent.com/document/product/457/79655) — 了解注册节点的概念和架构
- [创建注册节点](https://cloud.tencent.com/document/product/457/79748) — 创建注册节点并加入集群
- [TKE Service 管理](https://cloud.tencent.com/document/product/457/45490) — Service 更多配置选项
- [TKE Ingress 管理](https://cloud.tencent.com/document/product/457/45685) — Ingress 更多配置选项
- [TKE CLB 混合云部署](https://cloud.tencent.com/document/product/457) — CLB 混合云部署详细说明

## 控制台替代

[TKE 控制台 → 集群 → 服务与路由](https://console.cloud.tencent.com/tke2/cluster)：通过控制台创建 Service（LoadBalancer）和 Ingress 管理注册节点流量接入。
