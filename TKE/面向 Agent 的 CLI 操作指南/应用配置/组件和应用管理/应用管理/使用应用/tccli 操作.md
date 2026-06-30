# 使用应用（tccli）

> 对照官方：[使用应用](https://cloud.tencent.com/document/product/457/32730) · page_id `32730`

## 概述

通过容器服务控制台或本地 Helm 客户端对 TKE 应用进行创建、更新、回滚、删除操作。应用管理基于 Helm Chart 实现，支持应用市场和第三方来源。应用管理仅支持 Kubernetes 1.8 版本以上集群。

## 前置条件

- [环境准备](../../../../环境准备.md)
- 已 [连接集群](../../../../集群配置/集群管理/连接集群/tccli 操作.md)，kubectl 可正常访问 APIServer
- 已安装 Helm v3 客户端（参见 [本地 Helm 客户端连接集群](../本地 Helm 客户端连接集群/tccli 操作.md)）
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
| 创建应用 | `helm install <release> <chart> -n <namespace>` | 否（重复创建报错） |
| 更新应用 | `helm upgrade <release> <chart> -n <namespace>` | 是 |
| 回滚应用 | `helm rollback <release> <revision> -n <namespace>` | 否 |
| 删除应用 | `helm uninstall <release> -n <namespace>` | 是 |
| 查看应用列表 | `helm list -A` | 是 |
| 查看应用详情 | `helm get all <release> -n <namespace>` | 是 |
| 查看版本历史 | `helm history <release> -n <namespace>` | 是 |
| 添加应用市场 Chart 仓库 | `helm repo add tkemarket https://market-tke.tencentcloudcr.com/chartrepo/opensource-stable` | 是 |
| 添加第三方 Helm Repo | `helm repo add <repo-name> <repo-url>` | 是 |

## 操作步骤

### 步骤 1：创建应用（数据面）

#### 从应用市场安装

```bash
helm repo add tkemarket https://market-tke.tencentcloudcr.com/chartrepo/opensource-stable
helm repo update
helm search repo tkemarket/<chart-name>
helm install <release-name> tkemarket/<chart-name> \
  --namespace <namespace> \
  --create-namespace \
  --set <key>=<value>
# expected: STATUS: deployed
```

#### 从第三方 Chart 安装

```bash
helm repo add <repo-name> <repo-url>
helm repo update
helm install <release-name> <repo-name>/<chart-name> \
  --namespace <namespace> \
  --create-namespace
# expected: STATUS: deployed
```

或直接使用 Chart 包地址：

```bash
helm install <release-name> http://<chart-server>/<chart>.tgz \
  --namespace <namespace> \
  --create-namespace
# expected: STATUS: deployed
```

> **注意**：kubectl/helm 数据面当前因公网端点被 CAM 策略 strategyId:240463971 拒绝而不可达。以下命令需在内网/VPN 环境执行。

### 步骤 2：查看应用列表（数据面）

```bash
helm list -A
helm list -n <namespace>
# expected: 返回应用列表
```

### 步骤 3：更新应用（数据面）

```bash
helm upgrade <release-name> tkemarket/<chart-name> \
  --namespace <namespace> \
  --set <key>=<new-value> \
  --reuse-values
# expected: STATUS: deployed
```

### 步骤 4：回滚应用（数据面）

查看历史版本：

```bash
helm history <release-name> -n <namespace>
# expected: 返回版本历史
```

回滚到指定版本：

```bash
helm rollback <release-name> <revision> -n <namespace>
# expected: Rollback was a success
```

### 步骤 5：查看应用详情（数据面）

```bash
helm get all <release-name> -n <namespace>
helm get values <release-name> -n <namespace>
helm get manifest <release-name> -n <namespace>
# expected: 返回应用完整信息
```

### 步骤 6：删除应用（数据面）

```bash
helm uninstall <release-name> -n <namespace>
# expected: release "<release-name>" uninstalled
```

## 验证

### 数据面（kubectl）

```bash
helm list -A
# expected: 返回已部署应用

helm status <release-name> -n <namespace>
# expected: STATUS: deployed

kubectl get pods -n <namespace> -l app.kubernetes.io/instance=<release-name>
# expected: Pod Running
```

```text
NAME  STATUS  AGE
...
```

> **注意**：kubectl 数据面当前因公网端点被 CAM 策略 strategyId:240463971 拒绝而不可达。以下 kubectl 命令为参考格式，需在内网/VPN 环境执行。

## 清理

### 数据面（kubectl）

```bash
helm uninstall <release-name> -n <namespace>
# expected: release "<release-name>" uninstalled
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `helm install` 失败 | `helm version` 确认版本；`helm search repo <chart>` 确认 Chart 可访问 | Helm 版本过低或 Chart 地址不可达 | 使用 Helm v3；确认 Chart 地址可访问 |
| 应用状态异常 | `helm status <release-name> -n <namespace>` 查看失败原因 | Chart 参数错误或 Pod 启动失败 | `kubectl describe pod -n <namespace>` 排查 |
| 回滚失败 | `helm history <release-name> -n <namespace>` 确认版本历史存在 | 目标 revision 不存在 | 验证版本历史后重试 |
| 找不到应用市场的 Chart | `helm repo list` 确认仓库已添加 | 仓库索引未更新 | `helm repo update` 更新仓库索引 |
| `Kubernetes cluster unreachable` | `kubectl cluster-info` 检查连通性 | 公网端点被 CAM 策略 strategyId:240463971 以 `tke:clusterExtranetEndpoint=true` 条件拒绝 | 通过 IOA/VPN 连接内网，或使用同 VPC CVM 执行 helm 命令 |

## 下一步

- [本地 Helm 客户端连接集群](../本地 Helm 客户端连接集群/tccli 操作.md)
- [应用市场](../应用市场/tccli 操作.md)

## 控制台替代

控制台 **运维中心 → 应用市场**，选择 Cluster、单击 **新建**，配置应用来源（应用市场 / 第三方）、Chart 地址和参数后完成创建。
