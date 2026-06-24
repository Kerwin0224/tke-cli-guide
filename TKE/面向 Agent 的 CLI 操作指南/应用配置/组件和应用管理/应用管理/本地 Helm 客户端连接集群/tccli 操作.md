# 本地 Helm 客户端连接集群（tccli）

> 对照官方：[本地 Helm 客户端连接集群](https://cloud.tencent.com/document/product/457/32731) · page_id `32731`

## 概述

通过本地 Helm 客户端连接 TKE 集群。Helm v3 已移除 Tiller 组件，Helm 客户端可直连集群 APIServer，应用相关版本数据直接存储在 Kubernetes 中。使用 TKE 生成的客户端证书访问集群。

## 前置条件

- [环境准备](../../../../环境准备.md)
- 已 [连接集群](../../../../集群配置/集群管理/连接集群/tccli 操作.md)，kubectl 可正常访问 APIServer
- 集群状态 `Running`

```bash
# 检查集群状态
tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'
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

- 需要 CAM 权限：`tke:DescribeClusters`、`tke:DescribeClusterKubeconfig`

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 查看集群信息 | `tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'` | 是 |
| 获取集群 Kubeconfig | `tccli tke DescribeClusterKubeconfig --region <Region> --ClusterId <ClusterId>` | 是 |
| 查看集群访问端点 | `tccli tke DescribeClusterEndpoints --region <Region> --ClusterId <ClusterId>` | 是 |
| 安装 Helm | `curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 && ./get_helm.sh` | 否（重复安装覆盖） |
| 查看 Helm 版本 | `helm version` | 是 |
| 添加 Helm Chart 仓库 | `helm repo add <repo-name> <repo-url>` | 否（重复添加报错） |
| 连接集群（kubectl context） | `kubectl config use-context <context>` / `--kubeconfig <file>` | 是 |

## 操作步骤

### 步骤 1：下载并安装 Helm 客户端（数据面）

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
# expected: Helm installed
```

验证安装：

```bash
helm version
# expected: version.BuildInfo{Version:"v3.x.x", ...}
```

### 步骤 2：配置 Helm Chart 仓库（可选，数据面）

配置 Kubernetes 官方仓库：

```bash
helm repo add stable https://charts.helm.sh/stable
# expected: "stable" has been added
```

配置腾讯云应用市场：

```bash
helm repo add tkemarket https://market-tke.tencentcloudcr.com/chartrepo/opensource-stable
# expected: "tkemarket" has been added
```

配置 TCR 私有 Helm 仓库，参见 [TCR Helm 仓库](https://cloud.tencent.com/document/product/1141/41944)。

### 步骤 3：获取 Kubeconfig（控制面）

```bash
tccli tke DescribeClusterKubeconfig --ClusterId <ClusterId> --region <Region> --output json \
  --filter Kubeconfig > kubeconfig.yaml
# expected: kubeconfig.yaml 生成成功
```

```json
{
  "Kubeconfig": "<Kubeconfig>",
  "RequestId": "<RequestId>"
}
```

> **注意**：kubectl/helm 数据面当前因公网端点被 CAM 策略 strategyId:240463971 拒绝而不可达。以下命令需在内网/VPN 环境执行。

### 步骤 4：连接集群（数据面）

方式一：配置 kubectl context：

```bash
export KUBECONFIG=kubeconfig.yaml
kubectl config use-context <context-name>
# expected: Switched to context "<context-name>"
```

方式二：通过 `--kubeconfig` 参数直接访问：

```bash
helm install <release-name> <chart> \
  --namespace <namespace> \
  --create-namespace \
  --kubeconfig kubeconfig.yaml
# expected: STATUS: deployed
```

### 步骤 5：验证连接（数据面）

```bash
helm list --kubeconfig kubeconfig.yaml -A
helm search repo tkemarket/
# expected: 返回应用列表与 Chart 列表
```

## 验证

### 数据面（kubectl）

```bash
helm version
# expected: version.BuildInfo{Version:"v3.x.x", ...}

helm list -A --kubeconfig kubeconfig.yaml
# expected: 返回应用列表

kubectl --kubeconfig kubeconfig.yaml cluster-info
# expected: Kubernetes control plane is running at ...
```

> **注意**：kubectl 数据面当前因公网端点被 CAM 策略 strategyId:240463971 拒绝而不可达。以下 kubectl 命令为参考格式，需在内网/VPN 环境执行。

## 清理

### 数据面（kubectl）

```bash
helm repo remove stable
helm repo remove tkemarket
# expected: 仓库已移除

rm kubeconfig.yaml
# expected: kubeconfig.yaml 已删除
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Helm 客户端连接 APIServer 失败 | `kubectl --kubeconfig kubeconfig.yaml cluster-info` 检查连通性 | kubeconfig 中 server 地址不正确或端点不可达 | 确认 kubeconfig 中 server 地址正确；内网端点需在 VPC 内，公网端点需已开启 |
| `helm repo add` 失败 | `curl -I <repo-url>` 测试连通性 | 网络不通或仓库 URL 错误 | 检查网络连通性；确认仓库 URL 正确 |
| `Error: Kubernetes cluster unreachable` | `kubectl cluster-info` 检查连通性 | 公网端点被 CAM 策略 strategyId:240463971 以 `tke:clusterExtranetEndpoint=true` 条件拒绝 | 不在公网可达 VPC 时使用 `--kubeconfig` 指定内网 kubeconfig；或使用 Cloud Shell |
| `lookup cls-xxx.ccs.tencent-cloud.com: no such host` | `tccli tke DescribeClusterEndpoints --region <Region> --ClusterId <ClusterId>` 确认端点 | 集群访问未开启或本地不可达内网域名 | 使用 `DescribeClusterEndpoints` 确认端点状态；通过 IOA/VPN 连接内网 |

## 下一步

- [使用应用](../使用应用/tccli 操作.md)
- [应用市场](../应用市场/tccli 操作.md)

## 控制台替代

控制台 **集群 → 基本信息** 复制 kubeconfig；**运维中心 → 应用市场** 管理应用。
