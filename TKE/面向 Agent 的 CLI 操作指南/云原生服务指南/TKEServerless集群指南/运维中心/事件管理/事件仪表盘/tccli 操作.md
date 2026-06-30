# 事件仪表盘

> 对照官方：[事件仪表盘](https://cloud.tencent.com/document/product/457/58212) · page_id `58212`

## 概述

TKE Serverless 集群的事件仪表盘提供 Kubernetes Events 的可视化展示和聚合分析，帮助运维人员快速定位集群问题。

事件来源于 Kubernetes API Server 的 Event 资源，记录集群中资源的状态变更和异常情况：
- **Pod Events**：调度失败、镜像拉取失败、健康检查失败、OOMKilled、Preemption 等。
- **Workload Events**：Deployment/StatefulSet/Job 的扩缩容、更新、回滚等。
- **存储 Events**：PVC 绑定失败、卷挂载失败的 Warning。
- **网络 Events**：Service/Ingress 配置异常。

> **依赖**：事件仪表盘依赖集群已开启[事件日志](../事件日志/tccli%20操作.md)采集功能。未开启时仪表盘仅展示近 1 小时的 Events（K8s 默认保留）。

## 前置条件

- [环境准备](../../../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查 kubectl 并连接集群
kubectl cluster-info
# expected: Kubernetes control plane is running at ...

# 3. 确认集群存在
tccli tke DescribeEKSClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]'
# expected: exit 0，集群存在
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

## 控制台与 CLI 参数映射

| 控制台操作 | tccli / kubectl | 幂等 |
|-----------|-----------|:--:|
| 查看事件仪表盘概览 | 无直接 CLI API（控制台可视化） | — |
| 查看集群 Events 列表 | `kubectl get events -A` | 是 |
| 查看特定资源 Events | `kubectl describe RESOURCE` | 是 |
| 查看集群基本信息 | `DescribeEKSClusters` | 是 |
| 开启事件日志采集 | 控制台操作（无对应 CLI API） | — |
| 配置事件投递（CLS） | 控制台操作（无对应 CLI API） | — |

> **注意**：事件仪表盘为控制台纯可视化功能，无直接 CLI/Dashboard API。以下 kubectl 操作用于在 CLI 环境查看集群 Events，可作为仪表盘的替代查询手段。

## 操作步骤

### 步骤 1：查看集群所有命名空间的 Events

```bash
# 按时间排序查看最新 Events
kubectl get events -A --sort-by='.lastTimestamp'
# expected: 返回集群中所有 Events（可能较多，建议加过滤）

# 仅查看最近的 Events
kubectl get events -A --sort-by='.lastTimestamp' | tail -20
# expected: 返回最近 20 条 Events
```

**预期输出**：

```
NAMESPACE     LAST SEEN   TYPE      REASON              OBJECT                               MESSAGE
default       2m          Normal    Scheduled           pod/nginx-deployment-xxxxx-xxxxx      Successfully assigned default/nginx-deployment-xxxxx-xxxxx to eklet
default       2m          Normal    Pulling             pod/nginx-deployment-xxxxx-xxxxx      Pulling image "nginx:latest"
default       1m          Normal    Pulled              pod/nginx-deployment-xxxxx-xxxxx      Successfully pulled image "nginx:latest"
default       1m          Normal    Created             pod/nginx-deployment-xxxxx-xxxxx      Created container nginx
default       1m          Normal    Started             pod/nginx-deployment-xxxxx-xxxxx      Started container nginx
kube-system   30s         Warning   FailedScheduling    pod/problem-pod                      0/1 nodes are available: 1 Insufficient cpu
```

### 步骤 2：过滤 Warning 级别 Events

```bash
# 仅查看 Warning 类型 Events
kubectl get events -A --field-selector type=Warning --sort-by='.lastTimestamp' | tail -20
# expected: 只返回 Warning 类型 Events，用于快速排查问题

# 按命名空间过滤 Warning Events
kubectl get events -n NAMESPACE --field-selector type=Warning
# expected: 仅返回指定命名空间的 Warning Events
```

```text
NAME  STATUS  AGE
...
```

### 步骤 3：查看特定 Pod 的 Events

```bash
# 查看 Pod 关联的所有 Events
kubectl describe pod POD_NAME -n NAMESPACE | grep -A 50 Events
# expected: 返回该 Pod 的 Events 段落

# 通过 label 过滤 Pod 的 Events
kubectl get events -n NAMESPACE --field-selector involvedObject.name=POD_NAME
# expected: 仅返回与该 Pod 相关的 Events
```

```text
NAME  STATUS  AGE
...
```

### 步骤 4：常见 Events 类型及含义

| Event Type | Reason | 说明 | 严重程度 |
|-----------|--------|------|:--:|
| Normal | `Scheduled` | Pod 已被调度 | 信息 |
| Normal | `Pulling` / `Pulled` | 镜像拉取中/完成 | 信息 |
| Normal | `Created` / `Started` | 容器创建/启动完成 | 信息 |
| Normal | `ScalingReplicaSet` | ReplicaSet 扩缩容 | 信息 |
| Warning | `FailedScheduling` | Pod 调度失败（资源不足、亲和性不满足等） | **高** |
| Warning | `Failed` / `BackOff` | 镜像拉取失败（重试中） | **高** |
| Warning | `FailedMount` | 卷挂载失败 | **高** |
| Warning | `Unhealthy` | 健康检查失败 | **中** |
| Warning | `OOMKilled` | 容器内存超限被 OOM Kill | **高** |
| Warning | `Evicted` | Pod 被驱逐（资源压力） | **高** |
| Warning | `NodeNotReady` | 节点（底层沙箱）不可用 | **高** |

### 步骤 5：事件仪表盘指标类别

| 类别 | 展示内容 | 来源 |
|------|---------|------|
| 事件总览 | 事件总数、Normal/Warning 占比 | K8s Events |
| 事件类型分布 | 按 Reason 分类的分布饼图 | K8s Events |
| Warning 事件 Top | 最多 Warning 的命名空间/资源 | K8s Events |
| 事件趋势 | 按时间聚合的事件量趋势 | K8s Events（持久化后） |
| 异常事件列表 | 实时 Warning Events 流 | K8s Events |
| 命名空间健康度 | 各命名空间 Normal vs Warning | K8s Events |

## 验证

```bash
# 验证集群 Events 可正常查询
kubectl get events -A --sort-by='.lastTimestamp' | wc -l
# expected: 返回 Events 数量（大于 0）

# 验证无高严重性 Warning Events
kubectl get events -A --field-selector type=Warning --sort-by='.lastTimestamp' | tail -10
# expected: 检查无误即可，若有异常 Warning 需排查

# 验证 Pod 调度正常（检查无 FailedScheduling）
kubectl get events -A --field-selector type=Warning | grep -i "FailedScheduling"
# expected: 无输出（无调度失败的 Pod）
```

```text
NAME  STATUS  AGE
...
```

## 清理

本页面为只读操作，无需清理。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl get events` 返回空 | 新集群无活动或 Events 已过期 | K8s 默认 Events 保留 1 小时 | 执行任意 kubectl 操作（如 `kubectl get pods`）产生新 Events 后重试 |
| `kubectl get events` 报 `Unable to connect to the server` | `cat ~/.kube/config` 检查 server | kubeconfig 未配置或集群不可达 | 重新获取：`tccli tke DescribeEKSClusterCredential --ClusterId CLUSTER_ID --region <Region>` |
| 事件仪表盘页面无数据 | 检查集群是否已开启[事件日志采集](../事件日志/tccli%20操作.md) | 事件日志采集未开启 | 在控制台开启事件日志采集并投递到 CLS |
| `kubectl describe pod` 无 Events 段落 | Events 可能已过期（>1h）或 Pod 无状态变更 | Events 默认 1 小时 TTL | 正常现象。操作 Pod 后重新 describe |

### 操作成功但异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 大量 `FailedScheduling` Warning | `kubectl describe pod` 查看被调度失败的 Pod | 集群资源不足、子网 IP 耗尽或亲和性规则不满足 | 扩展子网、添加可用区或调整亲和性规则 |
| 大量 `Failed` (BackOff) Warning | `kubectl describe pod` 查看镜像拉取详情 | 镜像不存在、认证失败或仓库不可达 | 检查镜像地址是否正确，确认 `imagePullSecret` 已配置且有效 |
| 持续出现 `Unhealthy` Warning | `kubectl logs POD_NAME` 检查应用日志 | 应用健康检查端口或路径配置不当 | 调整 `livenessProbe` 和 `readinessProbe` 配置 |
| 出现 `OOMKilled` 且 Pod 频繁重启 | `kubectl describe pod` 查看容器状态 | 容器内存使用超过 `resources.limits.memory` | 增大 `limits.memory` 或排查应用内存泄漏 |
| 事件仪表盘展示与 CLI 不一致 | Events 保留时间不同（CLI 1h vs 仪表盘持久化） | CLI `kubectl get events` 仅看近 1 小时，仪表盘含历史持久化数据 | 正常差异。需查看历史数据使用控制台仪表盘 |

## 下一步

- [事件日志](../事件日志/tccli%20操作.md) — 开启事件日志持久化采集
- [审计仪表盘](../../审计管理/审计仪表盘/tccli%20操作.md) — 查看 API 审计仪表盘
- [审计日志](../../审计管理/审计日志/tccli%20操作.md) — 审计日志采集配置
- [监控和告警](../../监控和告警/tccli%20操作.md) — 基于 Events 配置告警

## 控制台替代

[TKE 控制台 → Serverless 集群 → 运维中心 → 事件仪表盘](https://console.cloud.tencent.com/tke2/ecluster)：控制台提供完整的事件仪表盘可视化，展示事件趋势、分布、Top 分析和实时 Warning 流。建议开启事件日志持久化以获得完整历史数据。
