# 应用市场（tccli）

> 对照官方：[应用市场](https://cloud.tencent.com/document/product/457/46432) · page_id `46432`

## 概述

腾讯云容器服务应用市场按照集群类型（集群、Serverless 集群、边缘集群、注册集群）、应用场景（AI、数据库、大数据、工具、日志分析、监控、CI/CD、存储、网络、博客、开发、安全）等分类方式，提供 Helm Chart、容器镜像、软件服务等。可通过应用市场快速完成应用创建。

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
| 添加应用市场 Helm 仓库 | `helm repo add tkemarket https://market-tke.tencentcloudcr.com/chartrepo/opensource-stable` | 否（重复添加报错） |
| 查看 Chart 列表 | `helm search repo tkemarket/` | 是 |
| 查看应用详情 | `helm show chart tkemarket/<chart>` | 是 |
| 查看应用 README | `helm show readme tkemarket/<chart>` | 是 |
| 查看可配置参数 | `helm show values tkemarket/<chart>` | 是 |
| 创建应用 | `helm install <release> tkemarket/<chart> -n <namespace>` | 否（重复创建报错） |
| 删除应用 | `helm uninstall <release> -n <namespace>` | 是 |

## 操作步骤

### 步骤 1：添加应用市场 Helm 仓库（数据面）

```bash
helm repo add tkemarket https://market-tke.tencentcloudcr.com/chartrepo/opensource-stable
helm repo update
# expected: "tkemarket" has been added；Update Complete
```

> **注意**：kubectl/helm 数据面当前因公网端点被 CAM 策略 strategyId:240463971 拒绝而不可达。以下命令需在内网/VPN 环境执行。

### 步骤 2：查看应用（数据面）

查看所有可用 Chart：

```bash
helm search repo tkemarket/
# expected: 返回 Chart 列表
```

按关键词搜索：

```bash
helm search repo tkemarket/<keyword>
# expected: 返回匹配的 Chart
```

查看 Chart 详情：

```bash
helm show chart tkemarket/<chart-name>
helm show readme tkemarket/<chart-name>
helm show values tkemarket/<chart-name>
# expected: 返回 Chart 元数据、README、可配置参数
```

### 步骤 3：创建应用（数据面）

```bash
helm install <release-name> tkemarket/<chart-name> \
  --namespace <namespace> \
  --create-namespace \
  --set <key1>=<value1> \
  --set <key2>=<value2>
# expected: STATUS: deployed
```

或者使用自定义 values.yaml 文件：

```bash
helm show values tkemarket/<chart-name> > custom-values.yaml
# 编辑 custom-values.yaml
helm install <release-name> tkemarket/<chart-name> \
  --namespace <namespace> \
  --create-namespace \
  -f custom-values.yaml
# expected: STATUS: deployed
```

### 步骤 4：查看已创建的应用（数据面）

```bash
helm list -A
helm list -n <namespace>
helm status <release-name> -n <namespace>
# expected: 返回应用列表与状态
```

## 验证

### 数据面（kubectl）

```bash
helm search repo tkemarket/
# expected: 返回 Chart 列表

helm list -A
# expected: 返回已部署应用

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
# expected: release "release-name" uninstalled

helm repo remove tkemarket
# expected: "tkemarket" has been removed
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `helm search repo tkemarket/` 无结果 | `helm repo list` 确认仓库已添加 | 仓库索引未更新 | 执行 `helm repo update` 更新仓库索引 |
| 仓库连接超时 | `curl -I https://market-tke.tencentcloudcr.com` 测试连通性 | 网络不通 | 检查网络连通性，确保可访问 `market-tke.tencentcloudcr.com` |
| Chart 安装失败 | `helm status <release-name> -n <namespace>` 查看失败原因；`kubectl describe pod -n <namespace>` | Chart 参数错误或依赖未满足 | 查看 helm status 和 kubectl describe pod 排查 |
| 筛选应用场景 | `helm search repo tkemarket/<key>` 关键词搜索 | 控制台分类方式仅在控制台可用 | 通过关键词搜索；官方控制台分类方式需通过控制台查看 |
| `Kubernetes cluster unreachable` | `kubectl cluster-info` 检查连通性 | 公网端点被 CAM 策略 strategyId:240463971 以 `tke:clusterExtranetEndpoint=true` 条件拒绝 | 通过 IOA/VPN 连接内网，或使用同 VPC CVM 执行 helm 命令 |

## 下一步

- [使用应用](../使用应用/tccli 操作.md)
- [本地 Helm 客户端连接集群](../本地 Helm 客户端连接集群/tccli 操作.md)

## 控制台替代

控制台 **运维中心 → 应用市场** 浏览和筛选 Chart，单击 **创建应用** 按需配置参数。
