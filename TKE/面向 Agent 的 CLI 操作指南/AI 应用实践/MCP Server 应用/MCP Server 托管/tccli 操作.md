# 托管 MCP Server 到 TKE（tccli）

> 对照官方：[MCP Server 托管](https://cloud.tencent.com/document/product/457/124005) · page_id `124005`

## 概述

将 Streamable HTTP 类型的 MCP Server 部署到 TKE 集群，通过 LoadBalancer Service 暴露公网入口供外部 MCP Client 调用。

MCP（Model Context Protocol）是 AI Agent 与外部工具交互的开放协议。TKE 支持托管 Streamable HTTP 模式的 MCP Server，相较于 stdio 模式（仅限本地进程通信），Streamable HTTP 模式天然支持跨网络部署和远程访问。

> **kubectl 数据面不可达**：CAM 拒绝公网端点（strategyId:240463971），内网需 VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。控制面操作通过 tccli 完成。

## 前置条件

- [环境准备](../../../环境准备.md)

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
#    tcr:DescribeRepositories, tcr:CreateRepository
#    cvm:DescribeInstances, cvm:DescribeSecurityGroups
# 验证 TKE 权限
tccli tke DescribeClusters --region <Region>
# expected: exit 0，返回集群列表（可为空）

# 4. 检查 kubectl（数据面操作需要，需 VPN/IOA）
kubectl version --client
# expected: 显示 Client Version
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
# 5. 查询可用 TKE 集群
tccli tke DescribeClusters --region <Region>
# expected: 至少返回 1 个 Running 状态的集群

# 6. 检查目标集群节点资源
tccli tke DescribeClusterInstances --region <Region> \
    --ClusterId CLUSTER_ID
# expected: 至少 1 个 Running 状态的节点，有足够资源供新 Pod 调度

# 7. 查询可用 TCR 仓库（如需推送镜像）
tccli tcr DescribeRepositories --region <Region> \
    --RegistryId REGISTRY_ID
# expected: 返回仓库列表，或准备创建新仓库
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

- 集群类型：`MANAGED_CLUSTER`（托管集群），控制面由腾讯云运维
- 推荐 K8s 版本：>= 1.30.0
- 容器运行时：containerd（TKE 托管集群强制）

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看集群列表 | `DescribeClusters` | 是 |
| 查看集群节点 | `DescribeClusterInstances` | 是 |
| 获取 kubeconfig | `DescribeClusterKubeconfig` | 是 |
| 创建 TCR 仓库 | `CreateRepository`（tcr） | 否 |
| 查看 TCR 仓库 | `DescribeRepositories`（tcr） | 是 |

> 创建 Deployment、Service 等 K8s 资源属于数据面操作，通过 `kubectl` 完成（需 VPN/IOA）。

## 操作步骤

### 步骤 1：确认目标集群

```bash
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterStatus "Running"，ClusterType "MANAGED_CLUSTER"
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
            "ClusterType": "MANAGED_CLUSTER",
            "ClusterVersion": "1.30.0",
            "ClusterNodeNum": 3
        }
    ],
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `REGION` | 地域 | 如 `ap-guangzhou` | `tccli configure list` |
| `CLUSTER_ID` | 集群 ID | 格式 `cls-xxxxxxxx` | `tccli tke DescribeClusters` |

### 步骤 2：准备 MCP Server 镜像（TCR）

#### 选择依据

- **仓库类型**：使用 TCR 个人版（免费，无 SLA 保障）或企业版（推荐生产环境）。个人版适合开发测试，企业版具备 SLA 保障和更高并发能力。
- **镜像可见性**：建议设为公开镜像，便于 TKE 集群直接拉取，无需配置 imagePullSecret。

```bash
# 创建公开 TCR 个人版仓库（如尚不存在）
tccli tcr CreateRepository --region <Region> \
    --RegistryId REGISTRY_ID \
    --RepositoryName REPOSITORY_NAME \
    --Public true \
    --BriefDescription "MCP Server image repository"
# expected: exit 0，返回 RequestId，代表仓库创建成功
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `REGISTRY_ID` | 实例 ID | 个人版用 `tcr-` 前缀字符串 | `tccli tcr DescribeInstances` |
| `REPOSITORY_NAME` | 仓库名称 | 长度 1-100 | 自定义，如 `mcp-server` |

#### 推送镜像到 TCR

```bash
# 1. 登录 TCR
docker login ccr.ccs.tencentyun.com --username YOUR_UIN

# 2. 构建并推送 MCP Server 镜像
docker build -t ccr.ccs.tencentyun.com/NAMESPACE/REPOSITORY_NAME:TAG .
docker push ccr.ccs.tencentyun.com/NAMESPACE/REPOSITORY_NAME:TAG
# expected: push 成功，sha256 摘要已输出
```

### 步骤 3：部署 MCP Server 工作负载（数据面，需 VPN/IOA）

以下为 kubectl 参考命令。需先获取 kubeconfig：

```bash
# 获取 kubeconfig
tccli tke DescribeClusterKubeconfig --region <Region> --ClusterId CLUSTER_ID \
    | jq -r '.Kubeconfig' > ~/.kube/config-mcp
export KUBECONFIG=~/.kube/config-mcp
# expected: kubeconfig 写入成功
```

**最小部署**（仅必填字段）：

`mcp-server-minimal.yaml`：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mcp-server
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mcp-server
  template:
    metadata:
      labels:
        app: mcp-server
    spec:
      containers:
      - name: mcp-server
        image: ccr.ccs.tencentyun.com/NAMESPACE/REPOSITORY_NAME:TAG
        ports:
        - containerPort: 8000
---
apiVersion: v1
kind: Service
metadata:
  name: mcp-server-svc
  namespace: default
spec:
  type: LoadBalancer
  selector:
    app: mcp-server
  ports:
  - port: 8000
    targetPort: 8000
```

```bash
kubectl apply -f mcp-server-minimal.yaml
# expected: deployment.apps/mcp-server created, service/mcp-server-svc created
```

### 步骤 4：获取公网访问入口

```bash
kubectl get svc mcp-server-svc -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
# expected: 返回公网 IP 地址（如 43.xxx.xxx.xxx）
```

```text
NAME  STATUS  AGE
...
```

## 验证

### 控制面（tccli）

```bash
# 验证集群状态
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

### 数据面（需 VPN/IOA）

```bash
# 1. 确认 Pod 为 Running
kubectl get pods -l app=mcp-server
# expected: STATUS 为 Running，READY 1/1

# 2. 确认 Service 已分配公网 IP
kubectl get svc mcp-server-svc
# expected: EXTERNAL-IP 不为 <pending>

# 3. 测试 MCP Server 连通性
ping MCP_SERVER_IP
# expected: 能 ping 通

# 4. 测试端口可达（假设端口 8000）
telnet MCP_SERVER_IP 8000
# expected: Connected（或 curl 返回 HTTP 响应）

# 5. 测试 MCP 端点（如服务暴露 /mcp 路径）
curl http://MCP_SERVER_IP:8000/mcp
# expected: 返回 MCP Server 响应
```

```text
NAME  STATUS  AGE
...
```

## 清理

> **警告**：`kubectl delete -f` 会删除 Deployment 和 Service。如 Service 为 LoadBalancer 类型，关联的 CLB 将被释放。删除前确认 IP 未被生产使用。

### 数据面清理（需 VPN/IOA，在控制面之前）

```bash
# 1. 清理前状态检查
kubectl get all -l app=mcp-server
# 确认是待删除的目标资源

# 2. 删除工作负载和 Service
kubectl delete -f mcp-server-minimal.yaml
# expected: deployment.apps "mcp-server" deleted, service "mcp-server-svc" deleted

# 3. 验证已删除
kubectl get pods -l app=mcp-server
# expected: No resources found
```

```text
NAME  STATUS  AGE
...
```

### 控制面清理（tccli）

```bash
# 4. 删除 TCR 仓库（如不再使用）
tccli tcr DeleteRepository --region <Region> \
    --RegistryId REGISTRY_ID \
    --RepositoryName REPOSITORY_NAME
# ⚠️ 将删除仓库及所有镜像 Tag，不可恢复

# 5. 验证已删除
tccli tcr DescribeRepositories --region <Region> \
    --RegistryId REGISTRY_ID
# expected: 目标仓库不再出现
```

```json
{
  "RepositoryList": "<RepositoryList>",
  "Name": "<Name>",
  "Namespace": "<Namespace>",
  "CreationTime": "<CreationTime>",
  "Public": "<Public>",
  "Description": "<Description>"
}
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `CreateRepository` 返回 `LimitExceeded` | `tccli tcr DescribeRepositories --region <Region> --RegistryId REGISTRY_ID` 查看已有仓库数 | 个人版仓库数量达到上限（此为环境限制，非命令错误） | 删除不再使用的仓库后重试，或升级至企业版 |
| Docker push 返回 `unauthorized` | `docker login ccr.ccs.tencentyun.com` 重试登录 | TCR 凭据过期或 UIN 不匹配 | 在控制台获取正确的长期访问凭证，重新 docker login |
| `kubectl apply` 返回 `Forbidden` | `tccli tke DescribeClusterKubeconfig --region <Region> --ClusterId CLUSTER_ID` | CAM 策略拒绝公网端点访问（strategyId:240463971） | 通过 VPN/IOA 连接内网后执行 |

### 部署成功但服务不可达

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Pod 长时间 Pending | `kubectl describe pod -l app=mcp-server` 查看 Events | 资源请求过高或集群资源不足 | 降低 resources.requests 或扩容节点 |
| Service EXTERNAL-IP 为 `<pending>` | `kubectl describe svc mcp-server-svc` 查看 Events | CLB 创建中或配额不足 | 等待 2-5 分钟；如持续 pending，检查 CLB 配额 |
| ping/telnet 不通公网 IP | `kubectl get svc mcp-server-svc` 确认 EXTERNAL-IP | 安全组未放行端口 8000 | 在 CLB 安全组中放行对应端口 |
| MCP Client 连接后无响应 | `kubectl logs -l app=mcp-server` 查看日志 | MCP Server 启动失败或配置错误 | 检查 MCP Server 日志排查应用层问题 |

## 下一步

- [Agent 代码沙箱工具部署](../../Agent%20智能体应用/Agent%20代码沙箱工具部署/tccli%20操作.md) -- page_id `123916`
- [Embedding 模型部署](../../AI%20模型部署/Embedding%20模型部署/tccli%20操作.md) -- page_id `124002`
- [创建 TKE 托管集群](../../../集群配置/集群管理/创建集群/tccli%20操作.md)
- [TCR 企业版快速入门](https://cloud.tencent.com/document/product/1141/40295)
- [MCP 协议规范](https://modelcontextprotocol.io/)

## 控制台替代

[容器服务控制台 - 工作负载](https://console.cloud.tencent.com/tke2/cluster)
