# 监控告警概述（tccli）

> 对照官方：[监控告警概述](https://cloud.tencent.com/document/product/457/34180) · page_id `34180`

## 概述

腾讯云容器服务 TKE 提供集群、节点、工作负载、Pod、Container 五个层面的监控数据收集和展示。良好的监控环境为高可靠性、高可用性和高性能提供保证。通过告警配置可以为不同资源收集不同维度的监控数据，方便掌握资源状况，轻松定位故障。

TKE 基础监控覆盖 Kubernetes 对象核心指标，建议结合腾讯云可观测平台（TCOP）的基础资源监控（云服务器、块存储、负载均衡等）使用。监控数据通过 QCE 命名空间上报：

| QCE 命名空间 | 说明 |
|-------------|------|
| `QCE/TKE2` | TKE 集群维度指标 |
| `QCE/NODE` | 节点维度指标 |
| `QCE/DOCKER` | 容器维度指标 |

如需更细粒度的自定义指标，可使用腾讯云 Prometheus 监控服务。

## 前置条件

- `tccli` 已配置凭证
- kubectl 不可达：演示集群 cls-xxxxxxxx 为托管集群，公网端点可能受 CAM 策略限制（strategyId:240463971）；内网无 VPN 时不可达。以下 kubectl 命令作为文档参考。

```bash
tccli tke DescribeClusters --ClusterIds '["cls-xxxxxxxx"]' --region ap-guangzhou
```

```json
{
  "RequestId": "e.g.-a1b2c3d4-5678-90ab-cdef-0123456789ab",
  "TotalCount": 1,
  "Clusters": [
    {
      "ClusterId": "cls-xxxxxxxx",
      "ClusterName": "my-test-cluster",
      "ClusterType": "MANAGED_CLUSTER",
      "ClusterStatus": "Running",
      "ClusterVersion": "1.30.0",
      "ClusterNodeNum": 2
    }
  ]
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 查看集群列表 | `tccli tke DescribeClusters --ClusterIds '["cls-xxxxxxxx"]' --region ap-guangzhou` | 是 |
| 查看监控数据 | 参见 [查看监控数据](../查看监控数据/tccli 操作.md) | 是 |
| 查看监控指标列表 | 参见 [监控及告警指标列表](../监控及告警指标列表/tccli 操作.md) | 是 |
| 查看告警策略 | `monitor:DescribeAlarmPolicies` | 是 |

## 操作步骤

监控告警概述为概念型页面，涵盖监控和告警两大功能入口。具体操作请参见对应子页面。

### 查看集群基本信息

```bash
tccli tke DescribeClusters --ClusterIds '["cls-xxxxxxxx"]' --region ap-guangzhou
```

```json
{
  "RequestId": "e.g.-a1b2c3d4-5678-90ab-cdef-0123456789ab",
  "TotalCount": 1,
  "Clusters": [
    {
      "ClusterId": "cls-xxxxxxxx",
      "ClusterName": "my-test-cluster",
      "ClusterType": "MANAGED_CLUSTER",
      "ClusterStatus": "Running",
      "ClusterVersion": "1.30.0",
      "ClusterNodeNum": 2
    }
  ]
}
```

### 查询告警策略

通过腾讯云可观测平台 API 查询当前账户下所有告警策略：

```bash
tccli monitor DescribeAlarmPolicies --region ap-guangzhou
```

```json
{
  "RequestId": "e.g.-b2c3d4e5-6789-0abc-def0-123456789abc",
  "TotalCount": 3,
  "Policies": [
    {
      "PolicyId": "policy-abc123",
      "PolicyName": "TKE-集群CPU使用率告警",
      "MonitorType": "MT_QCE",
      "Namespace": "QCE/TKE2",
      "Enable": 1
    }
  ]
}
```

### 快速查看节点资源使用（数据面参考）

以下 kubectl 命令作为文档参考（演示集群 kubectl 不可达）：

```bash
kubectl top nodes
```

```text
NAME             CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
node-10-0-1-5    1200m        30%    3200Mi          40%
node-10-0-1-6    800m         20%    2400Mi          30%
```

### 快速查看 Pod 资源使用（数据面参考）

```bash
kubectl top pods -A
```

```text
NAMESPACE     NAME                              CPU(cores)   MEMORY(bytes)
kube-system   coredns-6d8cf9b84-xm9lk           8m           45Mi
kube-system   kube-proxy-abc123                 5m           30Mi
```

## 验证

### Data plane (kubectl 文档参考)

```bash
kubectl top nodes && kubectl top pods -A 2>/dev/null | head -5
```

```text
NAME             CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
node-10-0-1-5    1200m        30%    3200Mi          40%
NAMESPACE     NAME                    CPU(cores)   MEMORY(bytes)
default       nginx-deploy-xxx        15m          120Mi
```

### 控制面验证

```bash
tccli tke DescribeClusters --ClusterIds '["cls-xxxxxxxx"]' --region ap-guangzhou
```

```json
{
  "TotalCount": 0,
  "Clusters": [],
  "ClusterId": "<ClusterId>",
  "ClusterName": "<ClusterName>",
  "ClusterDescription": "<ClusterDescription>",
  "ClusterVersion": "<ClusterVersion>",
  "ClusterOs": "<ClusterOs>",
  "ClusterType": "<ClusterType>"
}
```

预期：`ClusterStatus` 为 `Running`。

## 清理

监控告警概述为概念型页面，无清理操作。

## 排障

| 现象 | 处理 |
|------|------|
| 基础监控不满足需求 | 使用腾讯云 Prometheus 监控服务，支持采集自定义指标、多集群监控 |
| 无历史监控数据 | TKE 默认提供基础监控，数据保留时间有限；长期存储需配置 Prometheus 或 CLS |
| 告警未触发 | 检查告警策略配置和告警阈值；建议为所有生产集群配置必要告警 |
| DescribeAlarmPolicies 为空 | 确认当前地域有已配置的告警策略；若无，通过控制台或 `monitor:CreateAlarmPolicy` 创建 |

## 下一步

- [查看监控数据](../查看监控数据/tccli 操作.md) — 控制台与 CLI 查看监控数据
- [监控及告警指标列表](../监控及告警指标列表/tccli 操作.md) — 五层监控告警指标参考
- [查看 TKE 集群控制面组件监控](../../控制面组件监控/查看 TKE 集群控制面组件监控/tccli 操作.md) — 控制面组件监控
- [事件日志](../../../事件管理/事件日志/tccli 操作.md) — 集群事件持久化到 CLS

## 控制台替代

登录 [容器服务控制台](https://console.cloud.tencent.com/tke2) > 集群列表 > 单击监控图标，查看集群/节点/工作负载/Pod 监控数据。
