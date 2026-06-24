# 部署 AI 网关到 TKE（tccli）

> 对照官方：[AI 网关部署](https://cloud.tencent.com/document/product/457/123918) · page_id `123918`

## 概述

在 TKE 集群上部署 Envoy AI Gateway，为大模型服务提供统一的流量管理入口。Envoy AI Gateway 是基于 Envoy Proxy 的 Kubernetes 原生 AI 网关，支持多模型切换、租户认证、配额控制、Token 级限流、内容安全检查等功能。

本页通过 Helm 部署 Envoy Gateway 和 Envoy AI Gateway 组件，并配置 Kimi 等大模型后端作为示例。

> **kubectl 数据面不可达**：CAM 拒绝公网端点（strategyId:240463971），内网需 VPN/IOA。以下 kubectl 和 helm 命令为参考格式，需在内网/VPN 环境下执行。

## 前置条件

- [环境准备](../../../环境准备.md)

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
#    tke:DescribeClusterInstances
# 验证
tccli tke DescribeClusters --region <Region>
# expected: exit 0

# 4. 检查 kubectl（须 VPN/IOA）
kubectl version --client
# expected: Client Version v1.32.x

# 5. 检查 helm（须 VPN/IOA）
helm version
# expected: Version v3.x.x

# 如缺失 helm（TencentOS）：
# curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# 6. 检查 curl
curl --version
# expected: 已安装
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
# 7. 查询可用 TKE 集群（推荐香港地域，避免网络延迟）
tccli tke DescribeClusters --region <Region>
# expected: 至少 1 个 Running 集群

# 8. 检查节点数量和规格
tccli tke DescribeClusterInstances --region <Region> \
    --ClusterId CLUSTER_ID
# expected: >= 3 个 Running 节点，推荐 SA5.LARGE8
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

| 组件 | 推荐版本 | 说明 |
|------|---------|------|
| TKE 集群 | K8s >= 1.30.0 | 推荐香港地域以减少镜像拉取延迟 |
| 节点 | SA5.LARGE8 x 3 | Envoy Gateway 需要至少 3 个节点确保高可用 |
| Envoy Gateway | v0.0.0-latest | Helm Chart，OCI 仓库 |
| Envoy AI Gateway | v0.0.0-latest | Helm Chart，OCI 仓库 |

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看集群列表 | `DescribeClusters` | 是 |
| 查看集群节点 | `DescribeClusterInstances` | 是 |
| 获取 kubeconfig | `DescribeClusterKubeconfig` | 是 |

> Envoy Gateway 和 AI Gateway 的安装、配置全部通过 Helm + kubectl 完成（数据面操作）。

## 操作步骤

### 步骤 1：确认集群并获取 kubeconfig

```bash
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterStatus "Running"
```

**预期输出**（以真实集群 `cls-xxxxxxxx` 为例）：

```json
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "cls-xxxxxxxx",
            "ClusterName": "my-tke-cluster",
            "ClusterStatus": "Running",
            "ClusterVersion": "1.30.0",
            "ClusterType": "MANAGED_CLUSTER",
            "ClusterNodeNum": 3
        }
    ],
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

```bash
# 获取 kubeconfig
tccli tke DescribeClusterKubeconfig --region <Region> --ClusterId CLUSTER_ID \
    | jq -r '.Kubeconfig' > ~/.kube/config
# expected: kubeconfig 写入成功
```

```json
{
  "Kubeconfig": "<Kubeconfig>",
  "RequestId": "<RequestId>"
}
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `REGION` | 地域 | 如 `ap-guangzhou`、`ap-hongkong` | `tccli configure list` |
| `CLUSTER_ID` | 集群 ID | 格式 `cls-xxxxxxxx` | `tccli tke DescribeClusters` |

### 步骤 2：部署 Envoy Gateway（须 VPN/IOA）

#### 选择依据

- **Envoy Gateway**：Kubernetes 原生网关实现，提供 Gateway API 兼容的入口控制器。必须先部署 Envoy Gateway，再在其上部署 AI Gateway。
- **命名空间隔离**：使用独立的 `envoy-gateway-system` namespace 避免与业务组件冲突。

```bash
helm upgrade -i eg oci://docker.io/envoyproxy/gateway-helm \
    --version v0.0.0-latest \
    --namespace envoy-gateway-system \
    --create-namespace
# expected: Release "eg" does not exist. Installing it now. STATUS: deployed
```

等待 Envoy Gateway 就绪：

```bash
kubectl wait --timeout=2m \
    -n envoy-gateway-system deployment/envoy-gateway \
    --for=condition=Available
# expected: deployment.apps/envoy-gateway condition met
```

### 步骤 3：部署 Envoy AI Gateway（须 VPN/IOA）

```bash
helm upgrade -i aieg oci://docker.io/envoyproxy/ai-gateway-helm \
    --version v0.0.0-latest \
    --namespace envoy-ai-gateway-system \
    --create-namespace
# expected: Release "aieg" does not exist. Installing it now. STATUS: deployed
```

等待 AI Gateway 控制器就绪：

```bash
kubectl wait --timeout=2m \
    -n envoy-ai-gateway-system deployment/ai-gateway-controller \
    --for=condition=Available
# expected: deployment.apps/ai-gateway-controller condition met
```

### 步骤 4：配置 AI Gateway（须 VPN/IOA）

```bash
# 1. 应用 Redis 配置（限流和配额需要）
kubectl apply -f https://raw.githubusercontent.com/envoyproxy/ai-gateway/main/manifests/envoy-gateway-config/redis.yaml
# expected: 资源已创建

# 2. 应用 AI Gateway 核心配置
kubectl apply -f https://raw.githubusercontent.com/envoyproxy/ai-gateway/main/manifests/envoy-gateway-config/config.yaml
# expected: 资源已创建

# 3. 应用 RBAC 配置
kubectl apply -f https://raw.githubusercontent.com/envoyproxy/ai-gateway/main/manifests/envoy-gateway-config/rbac.yaml
# expected: 资源已创建

# 4. 重启 Envoy Gateway 加载新配置
kubectl rollout restart -n envoy-gateway-system deployment/envoy-gateway
kubectl wait --timeout=2m \
    -n envoy-gateway-system deployment/envoy-gateway \
    --for=condition=Available
# expected: deployment "envoy-gateway" successfully rolled out
```

### 步骤 5：部署测试后端（须 VPN/IOA）

```bash
# 1. 应用基础测试配置（含模拟后端）
kubectl apply -f https://raw.githubusercontent.com/envoyproxy/ai-gateway/main/examples/basic/basic.yaml
# expected: 资源已创建

# 2. 等待 Gateway Pod 就绪
kubectl wait pods --timeout=2m \
    -l gateway.envoyproxy.io/owning-gateway-name=envoy-ai-gateway-basic \
    -n envoy-gateway-system \
    --for=condition=Ready
# expected: pod 条件满足

# 3. 获取 Gateway 入口地址
export GATEWAY_URL=$(kubectl get gateway/envoy-ai-gateway-basic \
    -o jsonpath='{.status.addresses[0].value}')
echo $GATEWAY_URL
# expected: 输出 IP 或 hostname
```

```text
NAME  STATUS  AGE
...
```

### 步骤 6：验证测试后端（须 VPN/IOA）

```bash
curl -H "Content-Type: application/json" \
    -d '{
        "model": "some-cool-self-hosted-model",
        "messages": [{"role":"system","content": "Hi."}]
    }' \
    $GATEWAY_URL/v1/chat/completions
# expected: HTTP 200，返回 JSON 包含 "I am inevitable."
```

### 步骤 7：接入大模型（以 Kimi 为例）（须 VPN/IOA）

#### 选择依据

- **AIServiceBackend**：定义后端大模型服务的 schema（如 OpenAI 兼容 API）。
- **BackendSecurityPolicy**：通过 Kubernetes Secret 管理 API Key，避免硬编码。
- **AIGatewayRoute**：基于请求头 `x-ai-eg-model` 路由到指定后端，实现多模型透明切换。

**最小配置**：

`ai-gateway-kimi-minimal.yaml`：

```yaml
apiVersion: aigateway.envoyproxy.io/v1alpha1
kind: AIGatewayRoute
metadata:
  name: envoy-ai-gateway-basic-openai
  namespace: envoy-gateway-system
spec:
  targetRefs:
  - group: gateway.networking.k8s.io
    kind: Gateway
    name: envoy-ai-gateway-basic
  rules:
  - matches:
    - headers:
      - name: x-ai-eg-model
        value: kimi-k2-0905-preview
    backendRefs:
    - name: envoy-ai-gateway-basic-openai
---
apiVersion: aigateway.envoyproxy.io/v1alpha1
kind: AIServiceBackend
metadata:
  name: envoy-ai-gateway-basic-openai
  namespace: envoy-gateway-system
spec:
  schema:
    name: OpenAI
  backendRef:
    name: envoy-ai-gateway-basic-openai
---
apiVersion: aigateway.envoyproxy.io/v1alpha1
kind: BackendSecurityPolicy
metadata:
  name: envoy-ai-gateway-basic-openai-apikey
  namespace: envoy-gateway-system
spec:
  targetRefs:
  - group: aigateway.envoyproxy.io
    kind: AIServiceBackend
    name: envoy-ai-gateway-basic-openai
  type: APIKey
  apiKey:
    secretRef:
      name: envoy-ai-gateway-basic-openai-apikey
---
apiVersion: gateway.envoyproxy.io/v1alpha1
kind: Backend
metadata:
  name: envoy-ai-gateway-basic-openai
  namespace: envoy-gateway-system
spec:
  endpoints:
  - fqdn:
      hostname: api.moonshot.cn
      port: 443
---
apiVersion: gateway.envoyproxy.io/v1alpha1
kind: BackendTLSPolicy
metadata:
  name: envoy-ai-gateway-basic-openai-tls
  namespace: envoy-gateway-system
spec:
  targetRefs:
  - group: gateway.envoyproxy.io
    kind: Backend
    name: envoy-ai-gateway-basic-openai
  validation:
    hostname: api.moonshot.cn
    wellKnownCACertificates: "System"
---
apiVersion: v1
kind: Secret
metadata:
  name: envoy-ai-gateway-basic-openai-apikey
  namespace: envoy-gateway-system
type: Opaque
stringData:
  apiKey: "API_KEY"
```

```bash
# 应用配置
kubectl apply -f ai-gateway-kimi-minimal.yaml
# expected: 所有资源已创建

# 等待 Pod 就绪
kubectl wait pods --timeout=2m \
    -l gateway.envoyproxy.io/owning-gateway-name=envoy-ai-gateway-basic \
    -n envoy-gateway-system \
    --for=condition=Ready
# expected: pod 条件满足
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `API_KEY` | Kimi API Key | 格式 `sk-xxx` | 从 [Moonshot 开放平台](https://platform.moonshot.cn/) 获取 |

#### 测试 Kimi 接入

```bash
curl -H "Content-Type: application/json" \
    -d '{
        "model": "kimi-k2-0905-preview",
        "messages": [{"role":"user","content": "Hi."}]
    }' \
    $GATEWAY_URL/v1/chat/completions
# expected: HTTP 200，返回 Kimi 模型的 JSON 补全响应，含 usage 统计
```

## 验证

### 控制面（tccli）

```bash
# 1. 确认集群状态
tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
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

### 数据面（须 VPN/IOA）

```bash
# 2. 检查 Envoy Gateway 组件状态
kubectl get pods -n envoy-gateway-system
# expected: envoy-gateway Pod Running

# 3. 检查 AI Gateway 控制器状态
kubectl get pods -n envoy-ai-gateway-system
# expected: ai-gateway-controller Pod Running

# 4. 验证 Gateway 入口可达
echo $GATEWAY_URL
curl -s -o /dev/null -w "%{http_code}" $GATEWAY_URL/v1/chat/completions
# expected: HTTP 状态码（200 或其他非 5xx）

# 5. 验证模型路由（如已配置 Kimi）
curl -s -H "Content-Type: application/json" \
    -H "x-ai-eg-model: kimi-k2-0905-preview" \
    -d '{"model":"kimi-k2-0905-preview","messages":[{"role":"user","content":"Hi."}]}' \
    $GATEWAY_URL/v1/chat/completions | jq '.model'
# expected: "kimi-k2-0905-preview"
```

```text
NAME  STATUS  AGE
...
```

## 清理

> **警告**：卸载 Helm Release 将删除对应的 Deployment、Service 和 CRD 资源。关联的 CLB 将被释放。清理前确认 Gateway 未被生产流量使用。

### 数据面清理（须 VPN/IOA，在控制面之前）

```bash
# 1. 清理前状态检查
helm list -A
kubectl get gateways -A
# 确认是待清理的资源

# 2. 删除 Kimi 配置（如已部署）
kubectl delete -f ai-gateway-kimi-minimal.yaml
# expected: 所有资源已删除

# 3. 删除测试后端配置
kubectl delete -f https://raw.githubusercontent.com/envoyproxy/ai-gateway/main/examples/basic/basic.yaml
# expected: 资源已删除

# 4. 卸载 AI Gateway
helm uninstall aieg -n envoy-ai-gateway-system
# expected: release "aieg" uninstalled

# 5. 卸载 Envoy Gateway
helm uninstall eg -n envoy-gateway-system
# expected: release "eg" uninstalled

# 6. 验证已删除
kubectl get pods -n envoy-gateway-system
# expected: No resources found
kubectl get pods -n envoy-ai-gateway-system
# expected: No resources found
```

```text
NAME  STATUS  AGE
...
```

### 控制面

集群本身无需清理（共享集群）。如为本页新建专用集群：

```bash
tccli tke DeleteCluster --region <Region> \
    --ClusterId CLUSTER_ID \
    --InstanceDeleteMode terminate
# ⚠️ 将删除集群及所有关联 CVM、磁盘
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Helm install 超时 | `helm list -A` 查看 Release 状态 | OCI 镜像拉取超时（`docker.io` 网络延迟） | 推荐使用香港地域集群；配置镜像拉取加速或代理 |
| `kubectl apply` 返回 `no matches for kind` | `kubectl api-resources \| grep envoyproxy` | CRD 未安装（Helm Release 未成功） | `kubectl get crd \| grep envoyproxy` 确认 CRD；重试 Helm install |
| `kubectl wait` 超时 | `kubectl describe pod POD_NAME -n NAMESPACE` 查看 Events | Pod 调度失败或镜像拉取慢 | 检查节点资源是否充足（>= 3 节点）；等待镜像拉取完成 |
| `curl` 返回 `No matching route found` | `kubectl describe aigatewayroute -A` 查看路由规则 | 模型名不匹配（curl 中 model 值与 YAML 中 `x-ai-eg-model` 不一致） | 确保 curl 请求中的 `model` 字段与 AIGatewayRoute 的 `x-ai-eg-model` 值完全一致 |

### 部署成功但功能异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Gateway 无 EXTERNAL-IP | `kubectl get gateway/envoy-ai-gateway-basic -o yaml` | Service Type 不是 LoadBalancer 或 CLB 配额不足 | 检查 Gateway YAML 的 addresses 字段；如持续 `<pending>`，检查 CLB 配额 |
| API 调用返回 401 | 检查 Secret 中的 apiKey | API Key 无效或 Secret 格式错误 | `kubectl get secret envoy-ai-gateway-basic-openai-apikey -o jsonpath='{.data.apiKey}' \| base64 -d` 验证 key 值 |
| 请求路由到错误模型 | `kubectl logs -n envoy-gateway-system POD_NAME` 查看日志 | 模型名匹配错误或路由未生效 | 检查 `AIGatewayRoute` 的 `x-ai-eg-model` headers 配置 |

## 下一步

- [Embedding 模型部署](../../AI%20模型部署/Embedding%20模型部署/tccli%20操作.md) -- page_id `124002`（自托管大模型推理服务）
- [MCP Server 托管](../../MCP%20Server%20应用/MCP%20Server%20托管/tccli%20操作.md) -- page_id `124005`
- [Agent 代码沙箱工具部署](../../Agent%20智能体应用/Agent%20代码沙箱工具部署/tccli%20操作.md) -- page_id `123916`
- [Envoy AI Gateway 官方文档](https://github.com/envoyproxy/ai-gateway)
- [Gateway API 规范](https://gateway-api.sigs.k8s.io/)

## 控制台替代

[容器服务控制台](https://console.cloud.tencent.com/tke2/cluster)
