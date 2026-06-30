# 连接集群（tccli）

> 对照官方：[连接集群](https://cloud.tencent.com/document/product/457/39814) · page_id `39814`

## 概述

通过 `kubectl` 连接 TKE Serverless 集群，支持三种方式：Cloud Shell 连接、本地计算机连接、集群内节点连接。本篇聚焦 CLI 方式（方式二和方式三），介绍获取 kubeconfig 及配置 kubectl 的完整流程。

> **重要提示：** Serverless 容器服务已升级为 TKE 标准集群 + 超级节点模式，新建 Serverless 集群入口已关闭。存量用户集群不受影响。

## 前置条件

- [环境准备](../../环境准备.md)
- 已成功[创建集群](../创建集群/tccli%20操作.md)，集群状态为 `Running`
- 集群内存在节点资源（需先创建 Pod 后才能开启公网访问端点）
- 本地已安装 `kubectl`（版本需与服务端 Kubernetes 版本一致或相近）
- 本地已安装 `curl`

### 安装 kubectl

**macOS：**

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/darwin/amd64/kubectl"
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
kubectl version --client
```

**Linux：**

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
kubectl version --client
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|------|
| 获取集群 kubeconfig 凭证 | `DescribeEKSClusterCredential` | 是 |
| 管理集群端点（公网/内网） | `CreateEKSClusterEndpoint` / `DescribeEKSClusterEndpoints` | 否/是 |
| 查询集群 APIServer 信息 | `DescribeEKSClusters` | 是 |

## 操作步骤

### 方式一：通过 Cloud Shell 连接（推荐）

Cloud Shell 集成在控制台中，无需本地配置：

1. 登录 [容器服务控制台](https://console.cloud.tencent.com/tke2) → 集群管理
2. 目标集群右侧 **更多 > 连接集群**
3. 控制台底部出现 Cloud Shell，可直接执行 `kubectl` 命令

此方式无需 CLI 操作，适合临时管理。

### 方式二：通过本地 kubectl 连接

#### 步骤 1：开启集群公网访问端点

```bash
tccli tke CreateEKSClusterEndpoint \
    --ClusterId <ClusterId> \
    --SubnetId <SubnetId> \
    --IsExtranet true \
    --region <Region> \
    --output json
```

参数说明：

| 参数 | 说明 |
|------|------|
| `ClusterId` | 目标集群 ID |
| `SubnetId` | CLB 所在子网 ID |
| `IsExtranet` | `true` 为公网访问，`false` 为内网访问 |

> **安全提醒：** 公网访问端点开启后需配置安全组，放通 443 端口，并严格控制来源 IP。

#### 步骤 2：查询集群端点状态

```bash
tccli tke DescribeEKSClusterEndpoints \
    --ClusterId <ClusterId> \
    --region <Region> \
    --output json
```

```json
{
  "RequestId": "..."
}
```

#### 步骤 3：获取 kubeconfig 凭证

```bash
tccli tke DescribeEKSClusterCredential \
    --ClusterId <ClusterId> \
    --region <Region> \
    --output json
```

示例输出：

```json
{
    "Kubeconfig": "apiVersion: v1\nclusters:\n- cluster:\n    certificate-authority-data: LS0t...\n    server: https://cls-example.ccs.tencent-cloud.com\n  name: cls-example\ncontexts:\n- context:\n    cluster: cls-example\n    user: admin\n  name: cls-example-context-default\ncurrent-context: cls-example-context-default\nkind: Config\nusers:\n- name: admin\n  user:\n    client-certificate-data: LS0t...\n    client-key-data: LS0t...",
    "Expiration": "2025-01-15T10:30:00Z",
    "RequestId": "abc123-def456-ghi789"
}
```

#### 步骤 4：配置 kubeconfig

若 `~/.kube/config` 为空，直接写入：

```bash
tccli tke DescribeEKSClusterCredential \
    --ClusterId <ClusterId> \
    --region <Region> | jq -r '.Kubeconfig' > ~/.kube/config
```

```json
{
  "Addresses": [],
  "Type": "<Type>",
  "Ip": "<Ip>",
  "Port": 0,
  "Credential": "<Credential>",
  "CACert": "<CACert>",
  "Token": "<Token>",
  "PublicLB": "<PublicLB>"
}
```

若已有其他集群配置，合并：

```bash
tccli tke DescribeEKSClusterCredential \
    --ClusterId <ClusterId> \
    --region <

```json
{
  "Addresses": [],
  "Type": "<Type>",
  "Ip": "<Ip>",
  "Port": 0,
  "Credential": "<Credential>",
  "CACert": "<CACert>",
  "Token": "<Token>",
  "PublicLB": "<PublicLB>"
}
```Region> | jq -r '.Kubeconfig' > /tmp/kubeconfig-new

KUBECONFIG=~/.kube/config:/tmp/kubeconfig-new kubectl config view --merge --flatten > ~/.kube/config
export KUBECONFIG=~/.kube/config
```

切换 context 并验证：

```bash
kubectl config get-contexts
kubectl config use-context <ContextName>
kubectl get nodes
```

> **注意：** kubectl 不可达（公网端点 CAM 拒绝 strategyId:240463971）。以下命令为正确的 CLI 操作方式，但实际连接受 CAM 策略限制。

示例输出（连通时）：

```text
NAME                  STATUS   ROLES    AGE   VERSION
eklet-subnet-xxx-1    Ready    <none>   2d    v2.x.x
eklet-subnet-xxx-2    Ready    <none>   2d    v2.x.x
```

### 方式三：通过集群内节点连接

适用于集群内部署的 Pod 或已建立内网连接的机器。

#### 步骤 1：获取 kubeconfig

同方式二步骤 3。

#### 步骤 2：替换 server 地址为内网 IP

获取 Kubernetes Service 的 ClusterIP（在集群内通过 kubectl 获取 `kubernetes` Service 的 IP 后），修改 kubeconfig 中的 `server` 字段：

```yaml
# 修改前
server: https://cls-example.ccs.tencent-cloud.com

# 修改后
server: https://<ClusterIP>:443
```

#### 步骤 3：在集群内使用 kubeconfig

```bash
# 将修改后的 kubeconfig 写入集群内节点的 ~/.kube/config
kubectl get nodes
```

## 验证

| 验证项 | 命令 | 期望结果 |
|--------|------|---------|
| 集群端点已开启 | `tke DescribeEKSClusterEndpoints --ClusterId <ClusterId> --region <Region>` | 返回端点列表 |
| 凭证获取成功 | `tke DescribeEKSClusterCredential --ClusterId <ClusterId> --region <Region>` | 返回 `Kubeconfig` 字段 |
| kubectl 连接成功 | `kubectl cluster-info` | 显示集群 API Server 信息 |

## 清理

如需关闭公网访问端点，可删除对应的 CLB 或通过控制台操作。kubeconfig 文件无清理需求，可保留以备后续使用。

## 排障

| 现象 | 原因 | 处理 |
|------|------|------|
| `DescribeEKSClusterCredential` 报 `ResourceNotFound` | 集群 ID 不正确或集群不存在 | 检查 `ClusterId`，确保集群状态为 `Running` |
| `kubectl get nodes` 报 `Unable to connect to the server` | 端点未开启或网络不通 | 检查 `CreateEKSClusterEndpoint` 是否成功，安全组是否放通 443 |
| `kubectl` 报证书错误 | 证书过期或本地时间不同步 | 重新获取 kubeconfig（`DescribeEKSClusterCredential`），同步本地时间 |
| `kubectl` 版本不兼容 | 客户端版本与服务端偏差过大 | 安装与服务端版本一致或 ±1 个 minor 版本的 kubectl |
| 公网端点无法访问 | CAM 策略拒绝（strategyId:240463971） | 联系管理员调整 CAM 策略，放通当前用户/角色的集群访问权限 |

## 下一步

- [工作负载管理](../../Kubernetes%20对象管理/工作负载管理/tccli%20操作.md) — 部署应用至 Serverless 集群
- [服务管理](../../Kubernetes%20对象管理/服务管理/tccli%20操作.md) — 暴露服务入口

## 控制台替代

[容器服务控制台 → 集群 → 目标集群 → 基本信息](https://console.cloud.tencent.com/tke2/cluster) — 在「集群 APIServer 信息」区域查看开启状态并复制 kubeconfig。
