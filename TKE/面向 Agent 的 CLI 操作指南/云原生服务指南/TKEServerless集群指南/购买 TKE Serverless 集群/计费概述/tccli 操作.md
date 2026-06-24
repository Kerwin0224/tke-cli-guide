# 计费概述

> 对照官方：[计费概述](https://cloud.tencent.com/document/product/457/39807) · page_id `39807`

## 概述

TKE Serverless 集群采用**按量计费**模式，按 Pod 实际使用的资源规格和运行时长计费。与标准 TKE 集群不同，Serverless 集群**无集群管理费**，也无需支付 CVM 节点费用。

计费构成：
- **计费主体**：每个 Pod 独立计费，以 Pod 的 `resources.requests` 声明的规格为准。
- **计费维度**：vCPU 核数 x 运行时长 + 内存 GB 数 x 运行时长 + GPU 卡数 x 运行时长。
- **计费粒度**：按秒计费，按小时结算。Pod 从 `Pending` 状态（分配底层资源成功后）开始计费，到 `Terminated` 状态停止。
- **计费周期**：每个自然小时生成账单，扣费一次。
- **无最低消费**：无 Pod 运行时不产生任何费用。

与标准 TKE 的计费对比：

| 维度 | 标准 TKE（托管） | TKE Serverless |
|------|-----------------|----------------|
| 集群管理费 | 有（按集群规格） | 无 |
| CVM 节点费 | 有（按节点规格 x 时长） | 无 |
| Pod 计费 | 间接（节点费覆盖） | 直接（按 Pod 规格 x 时长） |
| 网络流量 | 公网出流量按量 | 公网出流量按量 |
| 存储 | CBS/CFS 按量 | CBS/CFS 按量 |
| 适用场景 | 长期稳态运行 | 弹性突发、CI/CD |

## 前置条件

- [环境准备](../../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看集群列表（确认运行 Pod 数量） | `DescribeEKSClusters` | 是 |
| 查看 Pod 资源规格 | kubectl（`kubectl describe pod`） | 是 |
| 查看计费账单 | 无 CLI API（计费中心查看） | — |
| 查看账户余额 | 无 CLI API（计费中心查看） | — |

> **注意**：TKE Serverless 的计费明细（按 Pod 粒度）目前仅可通过 [计费中心控制台](https://console.cloud.tencent.com/expense/overview) 查看，无对应 tccli API。以下 tccli 操作用于确认当前资源使用量。

## 操作步骤

### 步骤 1：查看当前运行的 Serverless 集群

```bash
tccli tke DescribeEKSClusters --region <Region>
# expected: exit 0，返回所有 Serverless 集群及其状态
```

**预期输出**：

```json
{
    "TotalCount": 2,
    "Clusters": [
        {
            "ClusterId": "cls-xxxxxxxx",
            "ClusterName": "prod-serverless",
            "Status": "Running",
            "CreatedTime": "2025-01-15T10:00:00Z"
        }
    ],
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

### 步骤 2：查看集群中运行的 Pod 及其资源规格

> **需 kubectl 可达环境**（参考 [连接集群](../../TKE%20Serverless%20集群管理/连接集群/tccli%20操作.md)）

```bash
# 配置 kubeconfig 后查看所有命名空间的 Pod
kubectl get pods -A -o wide
# expected: 列出所有 Pod 及其所在节点

# 查看 Pod 的 CPU/内存 request（决定计费规格）
kubectl get pods -A -o jsonpath='{range .items[*]}{.metadata.namespace}{"/"}{.metadata.name}{"\t"}{.spec.containers[0].resources.requests.cpu}{"\t"}{.spec.containers[0].resources.requests.memory}{"\n"}{end}'
# expected: 输出 namespace/podname 对应 CPU 和内存 request
```

**预期输出示例**：

```
default/nginx-deployment-xxxxx    500m    512Mi
default/api-server-xxxxx          1       2Gi
kube-system/coredns-xxxxx         100m    128Mi
```

### 步骤 3：估算当月费用

以 Pod `resources.requests` 为例：

- CPU: 1 核，内存: 2Gi，运行 24 小时 x 30 天 = 720 小时
- 费用 = (1 核 x CPU 单价 + 2GB x 内存单价) x 720 小时

> **实际单价**请参考 [TKE Serverless 定价文档](https://cloud.tencent.com/document/product/457/39806)，不同地域单价可能不同。费用以计费中心实时出账金额为准。

### 步骤 4：费用控制最佳实践

| 实践 | 说明 |
|------|------|
| 设置 Pod `resources.requests` 为实际需求的最小值 | 计费按 `requests` 而非 `limits`，合理设置可降低费用 |
| 为临时任务使用短生命周期 Pod（Job/CronJob） | 任务完成后 Pod 自动终止，停止计费 |
| 利用弹性伸缩 HPA 按需扩缩容 | 低负载时缩容至最小副本数 |
| 及时清理不用的集群和工作负载 | 删除无用 Pod 立即停止计费 |
| 设置计费告警 | 在计费中心配置阈值告警，避免意外超额 |

## 验证

```bash
# 确认当前正在运行的 Pod 数量（决定当前计费项）
kubectl get pods -A --field-selector=status.phase=Running | wc -l
# expected: 返回 Running 状态的 Pod 数（含标题行）

# 确认集群无异常运行中的多余 Pod
kubectl get pods -A | grep -v 'Running\|NAME'
# expected: 无输出（所有 Pod 正常 Running，无异常状态）
```

```text
NAME  STATUS  AGE
...
```

## 清理

本页面为只读操作，无需清理。

> 若需停止所有计费，删除所有工作负载或删除集群。参考 [创建集群](../../TKE%20Serverless%20集群管理/创建集群/tccli%20操作.md) 的「清理」节（`DeleteEKSCluster`）。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl get pods` 返回 `Unable to connect to the server` | `cat ~/.kube/config` 检查 server 地址 | kubeconfig 未配置或已过期 | 执行 `tccli tke DescribeEKSClusterCredential --ClusterId CLUSTER_ID --region <Region>` 重新获取 |
| `DescribeEKSClusters` 返回 `TotalCount: 0` 但计费中心仍有扣费 | 确认是否在其他地域有集群 | 当前地域参数不匹配 | 切换 `--region` 到其他地域逐一检查 |
| 计费中心金额与预期不符 | `kubectl get pods -A` 复核运行中 Pod 数量 | 可能有未知 Pod 持续运行（如泄漏的 Job Pod） | 排查并清理无用 Pod |

### 操作成功但异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Pod 已删除但仍在计费 | `kubectl get pods -A` 确认 Pod 是否彻底终止 | Pod 处于 `Terminating` 状态，终止过程未完成 | `kubectl delete pod POD_NAME --force --grace-period=0` 强制删除 |
| 费用远高于预期 | 逐一检查 Pod `resources.requests` 规格 | 容器 requests 声明过大 | 调整 Deployment/Pod 的 `resources.requests` 为实际需求值重新部署 |
| 集群已删除但仍有计费 | 确认集群的 CBS/CFS 存储卷是否未删除 | 集群删除不会自动删除 PVC 关联的 CBS 磁盘 | 在 [CBS 控制台](https://console.cloud.tencent.com/cvm/cbs) 手动删除未绑定的 CBS 磁盘 |

## 下一步

- [购买限制](../购买限制/tccli%20操作.md) — 了解资源配额和限制
- [欠费说明](../欠费说明/tccli%20操作.md) — 了解欠费后的影响和处理
- [集群生命周期](../../TKE%20Serverless%20集群管理/集群生命周期/tccli%20操作.md) — 集群状态与计费的关系

## 控制台替代

[TKE 控制台 → 创建 Serverless 集群](https://console.cloud.tencent.com/tke2/ecluster/create)：创建页面列出各资源规格及对应单价，集群详情页可查看运行中的 Pod 分布。
