# 内存压缩监控

> 对照官方：[压缩监控](https://cloud.tencent.com/document/product/457/102625) · page_id `102625`

## 概述

开启内存压缩后，可通过云监控指标和 QosAgent 日志两个维度监控压缩效果。控制面确认节点池配置和节点信息，数据面通过 QosAgent 日志和 Prometheus 指标查看压缩率、回收量等关键指标。

| 监控维度 | 数据来源 | 适用场景 |
|---------|---------|---------|
| 云监控（QCE/TKE） | 腾讯云可观测平台 | 集中监控多集群、配置告警 |
| QosAgent 日志 | kubectl logs | 单节点排查、压缩行为验证 |
| Prometheus 指标 | 集群内 Prometheus | 自定义看板、Grafana 可视化 |

## 前置条件

- [环境准备](../../../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置，region 为目标地域

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:DescribeClusterNodePools, tke:DescribeClusterNodePoolDetail
#    tke:DescribeClusterInstances, monitor:DescribeBaseMetrics
# 验证：执行 DescribeClusterNodePools 确认权限
tccli tke DescribeClusterNodePools --region <Region> --ClusterId CLUSTER_ID
# expected: exit 0，返回节点池列表（可为空）

# 4. 检查 kubectl 可用
kubectl version --client
# expected: Client Version 显示版本号
```

```json
{
  "NodePoolSet": "<NodePoolSet>",
  "NodePoolId": "<NodePoolId>",
  "Name": "<Name>",
  "ClusterInstanceId": "<ClusterInstanceId>",
  "LifeState": "<LifeState>",
  "LaunchConfigurationId": "<LaunchConfigurationId>"
}
```

### 资源检查

```bash
# 5. 确认内存压缩已开启
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId CLUSTER_ID \
    --NodePoolId NODE_POOL_ID \
    | jq '.NodePool.Annotations'
# expected: 含 tke.cloud.tencent.com/memory-compression-enabled: "true"

# 6. 确认 QosAgent 正常运行
kubectl get pods -n kube-system -l app=qos-agent
# expected: 返回 Running 状态的 Pod 列表

# 7. 确认 Prometheus（如需）已安装
kubectl get pods -n monitoring
# expected: 如集群已安装 Prometheus，返回相关 Pod
```

```json
{
  "NodePool": "<NodePool>",
  "NodePoolId": "<NodePoolId>",
  "Name": "<Name>",
  "ClusterInstanceId": "<ClusterInstanceId>",
  "LifeState": "<LifeState>",
  "LaunchConfigurationId": "<LaunchConfigurationId>"
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看节点池压缩配置 | `DescribeClusterNodePoolDetail` | 是 |
| 查看节点列表 | `DescribeClusterInstances` | 是 |
| 查看云监控指标 | `monitor GetMonitorData` | 是 |
| 查看 QosAgent 日志 | kubectl logs -n kube-system | 是 |

## 操作步骤

### 步骤 1：控制面 — 确认节点池和节点信息

```bash
# 查看节点池压缩配置
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId CLUSTER_ID \
    --NodePoolId NODE_POOL_ID \
    | jq '{Annotations: .NodePool.Annotations, NodeCount: .NodePool.NodeCountSummary}'
# expected: 返回压缩相关 Annotations 和节点数量

# 查看节点池内节点列表
tccli tke DescribeClusterInstances --region <Region> \
    --ClusterId CLUSTER_ID \
    --Filters '[{"Name":"node-pool-id","Values":["NODE_POOL_ID"]}]'
# expected: 返回节点实例列表，含 InstanceId 和 InstanceState
```

**预期输出**：

```json
{
    "InstanceSet": [
        {
            "InstanceId": "ins-example-001",
            "InstanceRole": "WORKER",
            "InstanceState": "running",
            "NodePoolId": "np-example",
            "CreatedTime": "2025-06-10T08:00:00Z"
        }
    ],
    "RequestId": "..."
}
```

### 步骤 2：数据面 — 查看 QosAgent 压缩日志

```bash
# 查看最近 QosAgent 日志
kubectl logs -n kube-system -l app=qos-agent --tail=50
# expected: 日志中包含 memory_compression、pagecache_reclaim 等关键字

# 按节点筛选日志（替换 NODE_NAME 为实际节点名）
kubectl logs -n kube-system -l app=qos-agent --tail=50 \
    --field-selector spec.nodeName=NODE_NAME
# expected: 该节点的 QosAgent 日志

# 实时跟踪压缩行为
kubectl logs -n kube-system -l app=qos-agent -f | grep -i "compression\|reclaim"
# expected: 实时输出压缩事件日志
```

**预期输出**（示例）：

```text
2025-06-10 08:15:30 INFO memory_compression: pagecache_reclaim node=NODE_NAME reclaimed=256MB threshold=80%
2025-06-10 08:15:31 INFO memory_compression: cold_memory_identified node=NODE_NAME cold_pages=512MB
2025-06-10 08:16:00 INFO memory_compression: cycle_complete node=NODE_NAME duration=30s freed=200MB
```

### 步骤 3：数据面 — 查看节点内存指标

```bash
# 查看节点内存使用情况
kubectl top node NODE_NAME
# expected: 显示 CPU 和 MEMORY(bytes) 使用量和百分比

# 查看节点详细内存信息
kubectl describe node NODE_NAME | grep -A 10 "Allocated resources"
# expected: 显示资源分配详情

# 进入节点查看内存统计
kubectl debug node/NODE_NAME -it --image=busybox -- cat /proc/meminfo
# expected: MemTotal, MemFree, Cached (PageCache) 等信息
```

```text
NAME  STATUS  AGE
...
```

### 步骤 4：控制面 — 拉取云监控指标

```bash
# 查询 TKE 节点内存使用率指标
tccli monitor GetMonitorData --region <Region> \
    --Namespace QCE/TKE \
    --MetricName K8sNodeMemUsage \
    --Instances '[{"Dimensions":[{"Name":"tke_cluster_instance_id","Value":"CLUSTER_ID"},{"Name":"node","Value":"NODE_NAME"}]}]' \
    --StartTime START_TIME \
    --EndTime END_TIME \
    --Period 60
# expected: 返回时序内存使用率数据

# START_TIME/END_TIME 格式: 2025-06-10T08:00:00+08:00
```

**预期输出**：

```json
{
    "DataPoints": [
        {
            "Dimensions": [{"Name": "node", "Value": "node-example-001"}],
            "Timestamps": ["2025-06-10 08:00:00", "2025-06-10 08:01:00"],
            "Values": [65.5, 62.3]
        }
    ],
    "RequestId": "..."
}
```

### 步骤 5：数据面 — Prometheus 指标查询（如集群已安装 Prometheus）

```bash
# 端口转发 Prometheus
kubectl port-forward -n monitoring svc/prometheus-k8s 9090:9090

# curl 查询压缩相关指标（在另一个终端执行）
curl -s 'http://localhost:9090/api/v1/query?query=qos_agent_memory_compression_pagecache_reclaimed_bytes' \
    | jq '.data.result[] | {node: .metric.node, value: .value[1]}'
# expected: 按节点返回压缩回收的字节数

# 查询压缩触发次数
curl -s 'http://localhost:9090/api/v1/query?query=qos_agent_memory_compression_trigger_count' \
    | jq '.data.result[] | {node: .metric.node, value: .value[1]}'
# expected: 按节点返回压缩触发次数
```

## 验证

### 控制面（tccli）

| 维度 | 检查内容 | 预期 |
|------|---------|------|
| 压缩配置生效 | `DescribeClusterNodePoolDetail` → `Annotations` | 含 `memory-compression-enabled: "true"` |
| 节点状态正常 | `DescribeClusterInstances` → `InstanceState` | 全部为 `running` |
| 云监控数据 | `monitor GetMonitorData` → `DataPoints` | 返回非空时序数据 |

### 数据面（kubectl）

| 维度 | 检查内容 | 预期 |
|------|---------|------|
| QosAgent 运行 | `kubectl get pods -n kube-system -l app=qos-agent` | 全部 Running |
| 压缩日志 | `kubectl logs -n kube-system -l app=qos-agent --tail=20` | 含 `pagecache_reclaim` 或 `compression` 字样 |
| 内存变化 | `kubectl top node NODE_NAME` | 压缩触发后内存使用率下降 |
| PageCache 变化 | 进入节点执行 `cat /proc/meminfo` | Cached 值在压缩触发后减少 |

```bash
# 综合验证：确认压缩功能正常
kubectl logs -n kube-system -l app=qos-agent --tail=20 | grep -i "compression\|reclaim"
# expected: 输出压缩相关日志，包含回收量信息
```

```text
NAME  STATUS  AGE
...
```

## 清理

本页为监控说明页，无资源需清理。

> 如需停止监控，关闭 `kubectl port-forward` 进程即可。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `monitor GetMonitorData` 返回 `InvalidParameterValue` | 检查 `--Namespace` 是否为 `QCE/TKE`，`--MetricName` 是否存在 | Namespace 或 MetricName 拼写错误 | 用 `monitor DescribeBaseMetrics --region <Region> --Namespace QCE/TKE` 查询可用指标名 |
| `kubectl top node` 返回 `error: metrics not available yet` | `kubectl get pods -n kube-system -l k8s-app=metrics-server` | metrics-server 未安装或未就绪 | 等待 metrics-server Pod Running 后重试 |

### 监控数据异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| QosAgent 日志无压缩事件 | `kubectl describe configmap -n kube-system qos-agent-config` 检查配置 | 内存压缩未启用，或阈值设置过高 | 确认 Annotation `memory-compression-enabled` 为 `"true"`；适当降低阈值 |
| 云监控无数据 | `tccli tke DescribeClusterNodePoolDetail --region <Region> --ClusterId CLUSTER_ID --NodePoolId NODE_POOL_ID` 确认节点池类型 | 非原生节点池不支持内存压缩，无对应监控指标 | 确认节点池 Type 为 `Native` |
| 压缩日志有错误 | `kubectl logs -n kube-system -l app=qos-agent --tail=100 \| grep -i error` | QosAgent 压缩模块异常 | 查看具体错误信息；如为配置错误，修正 qos-agent-config 后重启 DaemonSet |
| Prometheus 无压缩指标 | `kubectl get servicemonitor -n monitoring` 确认指标采集 | ServiceMonitor 未配置 QosAgent 指标采集 | 添加 QosAgent ServiceMonitor 或检查 Prometheus target 配置 |

## 下一步

- [使用说明](../使用说明/tccli%20操作.md) — page_id `102456`
- [原生节点功能支持说明](../../原生节点功能支持说明/tccli%20操作.md)
- [原生节点底层宿主机异常告警](../../原生节点底层宿主机异常告警/tccli%20操作.md) — page_id `121135`

## 控制台替代

[容器服务控制台 → 集群 → 节点管理 → 原生节点 → 监控](https://console.cloud.tencent.com/tke2/cluster)：在节点详情页查看内存使用率曲线和 QosAgent 相关指标。
