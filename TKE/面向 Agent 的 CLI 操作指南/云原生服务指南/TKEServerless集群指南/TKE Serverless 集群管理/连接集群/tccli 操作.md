# 连接集群

> 对照官方：[连接集群](https://cloud.tencent.com/document/product/457/39814) · page_id `39814`

## 概述

TKE Serverless 集群创建完成后，通过 `DescribeEKSClusterCredential` 获取集群的 kubeconfig 凭证文件，即可使用 kubectl 连接和管理集群。支持以下三种访问方式：本地 kubectl 直连、Cloud Shell 在线访问、以及通过 kubeconfig 文件集成到 CI/CD 流水线。

**注意**：Serverless 集群的 API Server 默认通过公网 CLB 暴露（如在创建时启用 `PublicLB`），也可使用内网 CLB 访问（需启用 `InternalLB`）。

## 前置条件

- [环境准备](../../../../环境准备.md)

### 集群可达性检查

```bash
# 1. 检查 tccli 版本与凭据
tccli --version
# expected: tccli version >= 1.0.0

tccli configure list
# expected: secretId, secretKey, region 均已配置

# 2. 确认集群处于 Running 状态
tccli tke DescribeEKSClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterStatus: Running

# 3. 获取集群凭证
tccli tke DescribeEKSClusterCredential --region <Region> \
    --ClusterId CLUSTER_ID
# expected: exit 0，返回 kubeconfig 内容（base64 编码）
```

```json
{
  "TotalCount": "<TotalCount>",
  "Clusters": "<Clusters>",
  "ClusterId": "<ClusterId>",
  "ClusterName": "<ClusterName>",
  "VpcId": "<VpcId>",
  "SubnetIds": "<SubnetIds>"
}
```

### 前提条件说明（Before you begin）

连接 Serverless 集群前需确认：
- 集群处于 `Running` 状态，API Server 可正常访问
- 本地已安装 kubectl（版本与集群 Kubernetes 版本兼容，建议偏差不超过 1 个小版本）
- 如集群启用内网访问，本地网络需能连通集群 VPC（如通过 VPN/专线/云联网）
- Cloud Shell 方式无需安装任何本地工具，通过控制台浏览器即可操作

```bash
# 安装 kubectl（macOS）
brew install kubectl
# 安装 kubectl（Linux）
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl && sudo mv kubectl /usr/local/bin/

kubectl version --client
# expected: Client Version 显示已安装版本
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 获取 kubeconfig | `tccli tke DescribeEKSClusterCredential --region <Region> --ClusterId CLUSTER_ID` | 是 |
| 开启公网访问 | `tccli tke CreateEKSCluster --cli-input-json file://input.json` 中设置 `PublicLB.Enabled: true` | 否 |
| 开启内网访问 | 同上，设置 `InternalLB.Enabled: true` | 否 |
| Cloud Shell 访问 | 控制台内一键操作 | - |

## 操作步骤

### 步骤 1: 获取 kubeconfig

```bash
tccli tke DescribeEKSClusterCredential --region <Region> \
    --ClusterId CLUSTER_ID
# expected: exit 0，返回包含 kubeconfig 的响应
```

预期输出：

```json
{
    "Kubeconfig": "YXBpVmVyc2lvbjogdjEKY2x1c3RlcnM6Ci0gY2x1c3RlcjoK...",
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

### 步骤 2: 解码并写入 kubeconfig 文件

`Kubeconfig` 字段返回的是 base64 编码的 kubeconfig 内容，需解码后保存：

```bash
# 方式 A: jq + base64 解码（推荐）
tccli tke DescribeEKSClusterCredential --region <Region> \
    --ClusterId CLUSTER_ID | jq -r '.Kubeconfig' | base64 -d > ~/.kube/config-eks-CLUSTER_ID

# 方式 B: 使用 tccli 的 --filter 提取
tccli tke DescribeEKSClusterCredential --region <Region> \
    --ClusterId CLUSTER_ID --filter 'Kubeconfig' | base64 -d > ~/.kube/config-eks-CLUSTER_ID

# expected: 文件创建成功，内容为有效的 kubeconfig YAML
```

```json
{
  "Addresses": "<Addresses>",
  "Type": "<Type>",
  "Ip": "<Ip>",
  "Port": "<Port>",
  "Credential": "<Credential>",
  "CACert": "<CACert>"
}
```

### 步骤 3: 验证集群连接

```bash
# 使用指定 kubeconfig 连接
kubectl --kubeconfig ~/.kube/config-eks-CLUSTER_ID cluster-info
# expected: Kubernetes control plane is running at https://...

# 检查节点（Serverless 集群显示虚拟节点）
kubectl --kubeconfig ~/.kube/config-eks-CLUSTER_ID get nodes
# expected: 列出虚拟节点（如 eklet-subnet-xxx）

# 设置为默认 context（可选）
export KUBECONFIG=~/.kube/config-eks-CLUSTER_ID
kubectl cluster-info
# expected: 同上
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `CLUSTER_ID` | 集群 ID | `cls-xxxxxxxx` 格式 | `tccli tke DescribeEKSClusters --region <Region>` |
| `REGION` | 地域 | 如 `ap-guangzhou` | `tccli configure list` |

### 步骤 4: Cloud Shell 访问（可选）

如不方便安装本地工具，可通过控制台的 Cloud Shell 功能直接在浏览器中操作集群。Cloud Shell 预装了 kubectl 并自动配置了集群凭证。

控制台入口：容器服务控制台 → 集群 → 选择目标集群 → 点击「命令行」按钮。

## 验证

| 维度 | 命令 | 预期 |
|------|------|------|
| kubeconfig 有效 | `kubectl --kubeconfig ~/.kube/config-eks-CLUSTER_ID cluster-info` | 返回 Kubernetes master 地址 |
| 节点可见 | `kubectl --kubeconfig ~/.kube/config-eks-CLUSTER_ID get nodes` | 列出虚拟节点 |
| API 可访问 | `kubectl --kubeconfig ~/.kube/config-eks-CLUSTER_ID api-resources` | 列出所有 API 资源 |
| 权限正常 | `kubectl --kubeconfig ~/.kube/config-eks-CLUSTER_ID auth can-i create pods` | `yes` |

```bash
# 综合连接验证
kubectl --kubeconfig ~/.kube/config-eks-CLUSTER_ID cluster-info && \
kubectl --kubeconfig ~/.kube/config-eks-CLUSTER_ID get nodes && \
kubectl --kubeconfig ~/.kube/config-eks-CLUSTER_ID get ns
# expected: 三个命令均 exit 0
```

## 清理

kubeconfig 文件可随时删除，不影响集群运行：

```bash
# 删除本地 kubeconfig 文件
rm -f ~/.kube/config-eks-CLUSTER_ID
# expected: exit 0

# 清除 KUBECONFIG 环境变量（如已设置）
unset KUBECONFIG
```

> **注意**：kubeconfig 凭证有时效性，过期后需重新获取。请勿将 kubeconfig 提交到版本控制系统。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DescribeEKSClusterCredential` 返回 `ResourceNotFound` | `tccli tke DescribeEKSClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'` 确认集群存在 | 集群 ID 不存在或地域不对（此为操作错误） | 确认 ClusterId 和 region 正确匹配 |
| `DescribeEKSClusterCredential` 返回 `FailedOperation.ClusterState` | `tccli tke DescribeEKSClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'` 查看状态 | 集群非 Running 状态 | 等待集群变为 Running 后重试 |
| `kubectl cluster-info` 返回 `Unable to connect to the server` | `nc -zv API_SERVER_HOST 443` 检测连通性 | 网络不通或 API Server 地址不可达 | 检查本地网络，确认公网/内网访问已启用 |
| `kubectl cluster-info` 返回 `x509: certificate signed by unknown authority` | 检查 kubeconfig 中的 certificate-authority-data | kubeconfig 中的 CA 证书不完整或已损坏 | 重新执行 `DescribeEKSClusterCredential` 获取新 kubeconfig |
| `kubectl cluster-info` 返回 `Unauthorized` | 检查 kubeconfig 中的 client-certificate-data / token | 凭证过期或无效 | 重新获取 kubeconfig |

### 操作成功但异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| kubectl 命令响应极慢 | `kubectl --kubeconfig ~/.kube/config-eks-CLUSTER_ID get --raw /healthz` 检查延迟 | 网络延迟高或 API Server 负载高 | 切换到同地域的 Cloud Shell 访问，或启用内网 CLB |
| kubeconfig 频繁过期 | 检查 kubeconfig 中的 `expiration` 时间戳 | kubeconfig 凭证设有过期时间 | 定期刷新凭证，或使用 `--token` 模式替代证书模式 |
| `kubectl get nodes` 显示 NotReady | `kubectl describe node NODE_NAME` 查看节点状态 | 底层虚拟节点异常 | 等待自动恢复，Serverless 集群节点由平台管理 |

## 下一步

- [工作负载管理](../../Kubernetes%20对象管理/工作负载管理/tccli%20操作.md) — 部署第一个工作负载
- [服务管理](../../Kubernetes%20对象管理/服务管理/tccli%20操作.md) — 通过 Service 暴露服务
- [监控和告警](../../运维中心/监控和告警/tccli%20操作.md) — 配置集群监控

## 控制台替代

控制台：容器服务控制台 → 集群 → 选择目标集群 → 基本信息页 → 点击「集群凭证」获取 kubeconfig，或点击「命令行」进入 Cloud Shell。
