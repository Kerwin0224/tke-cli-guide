# 健康检查（tccli）

> 对照官方：[健康检查](https://cloud.tencent.com/document/product/457/47416) · page_id `47416`

## 概述

集群健康检查功能是 TKE 为集群提供检查各个资源状态及运行情况的服务，覆盖资源状态（kube-apiserver、kube-scheduler、etcd、kubelet、节点、工作负载等）和运行情况（参数配置、高可用、Request/Limit/反亲和/PDB/健康检查配置等）。检查过程中会在集群内自动创建 namespace `tke-cluster-inspection` 并安装 DaemonSet 采集节点信息，完成后自动清理全部临时资源。

**当前 API 能力**：tccli 仅提供只读查询接口（概览、列表、详情）。检查任务的触发（批量检查、立即检查、自动定时检查）为控制台专属功能，暂无对应用写 API。

## 前置条件

- [环境准备](../../../环境准备.md)
- kubectl 不可达：演示集群 cls-xxxxxxxx 为托管集群，公网端点可能受 CAM 策略限制（strategyId:240463971）；内网无 VPN 时不可达。

### 环境检查

```bash
tccli --version
# 预期输出: tccli version X.X.X
```

### 资源检查

```bash
tccli tke DescribeClusters --ClusterIds '["cls-xxxxxxxx"]' --region ap-guangzhou
```

**预期输出**：

```json
{
  "TotalCount": 1,
  "Clusters": [
    {
      "ClusterId": "cls-xxxxxxxx",
      "ClusterName": "my-test-cluster",
      "ClusterStatus": "Running",
      "ClusterVersion": "1.30.0",
      "ClusterType": "MANAGED_CLUSTER"
    }
  ],
  "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看检查结果概览（按集群聚合） | `tccli tke DescribeClusterInspectionResultsOverview --ClusterIds '["cls-xxxxxxxx"]' --region ap-guangzhou` | 是 |
| 查看检查结果列表（历史检查记录） | `tccli tke ListClusterInspectionResults --ClusterIds '["cls-xxxxxxxx"]' --region ap-guangzhou` | 是 |
| 查看单次检查项详情 | `tccli tke ListClusterInspectionResultsItems --ClusterId cls-xxxxxxxx --region ap-guangzhou` | 是 |
| 批量检查 / 立即检查 | 控制台操作（暂无对应 tccli 写 API） | — |
| 自动定时检查 | 控制台操作（暂无对应 tccli 写 API） | — |

## 操作步骤

### 步骤 1：查看检查结果概览

一次查看多个集群的最近一次检查摘要，按 `ResourceStatus`（资源状态）和 `RuntimeStatus`（运行情况）两个维度汇总正常/异常计数。

```bash
tccli tke DescribeClusterInspectionResultsOverview --ClusterIds '["cls-xxxxxxxx"]' --region ap-guangzhou
```

**预期输出**：

```json
{
  "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "InspectionResultsOverview": [
    {
      "ClusterId": "cls-xxxxxxxx",
      "ResourceStatus": {
        "TotalCount": 12,
        "NormalCount": 10,
        "AbnormalCount": 2
      },
      "RuntimeStatus": {
        "TotalCount": 8,
        "NormalCount": 7,
        "AbnormalCount": 1
      }
    }
  ]
}
```

### 步骤 2：查看检查结果列表

获取指定集群的历史检查记录，每条记录包含检查名称、状态、起止时间和总体统计。

```bash
tccli tke ListClusterInspectionResults --ClusterIds '["cls-xxxxxxxx"]' --region ap-guangzhou
```

**预期输出**：

```json
{
  "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "InspectionResults": [
    {
      "ClusterId": "cls-xxxxxxxx",
      "Name": "inspection-20241001-001",
      "Status": "Completed",
      "StartTime": "2024-10-01T10:00:00Z",
      "EndTime": "2024-10-01T10:05:00Z",
      "Summary": {
        "TotalCount": 20,
        "NormalCount": 18,
        "AbnormalCount": 2
      }
    }
  ]
}
```

### 步骤 3：查看单次检查项详情

深入查看某一类检查类别下所有检查项的详细结果，包括异常项的原因和建议。

```bash
tccli tke ListClusterInspectionResultsItems --ClusterId cls-xxxxxxxx --region ap-guangzhou
```

**预期输出**：

```json
{
  "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "InspectionItems": [
    {
      "Name": "kubelet-status-check",
      "Category": "ResourceStatus",
      "Level": "Normal",
      "Description": "kubelet 的状态",
      "Detail": "kubelet 正常运行，24h内无重启"
    },
    {
      "Name": "workload-request-limit-check",
      "Category": "RuntimeStatus",
      "Level": "Abnormal",
      "Description": "工作负载的 Request 和 Limit 配置",
      "Detail": "Deployment nginx-deploy 存在未设置资源限制的容器",
      "Severity": "warn",
      "Reason": "容器 nginx 未设置 resources.requests 和 resources.limits",
      "Suggestion": "建议为容器配置资源限制，有益于资源规划、Pod调度和集群可用性"
    }
  ]
}
```

### 检查项分类说明

| 类别 | 说明 | 典型检查项 |
|------|------|-----------|
| `ResourceStatus` | 集群内关键资源的状态 | kube-apiserver 状态（仅独立集群）、kube-scheduler 状态（仅独立集群）、kube-controller-manager 状态（仅独立集群）、etcd 状态（仅独立集群）、kubelet 状态、节点状态、工作负载状态（Deployment/StatefulSet/DaemonSet） |
| `RuntimeStatus` | 集群运行时配置的合理性 | 参数配置、高可用配置（多副本/跨可用区）、Request/Limit 设置、反亲和性规则、PDB 配置、健康检查配置、HPA-IP 配置 |

> **注意**：独立集群（INDEPENDENT_CLUSTER）额外检查 kube-apiserver、kube-scheduler、kube-controller-manager 和 etcd 的控制面组件状态。托管集群（MANAGED_CLUSTER）仅检查数据面资源。

## 验证

### 控制面验证

```bash
# 确认概览接口可正常返回
tccli tke DescribeClusterInspectionResultsOverview --ClusterIds '["cls-xxxxxxxx"]' --region ap-guangzhou
# 预期: exit 0，InspectionResultsOverview 数组非空
```

```json
{
  "Statistics": [],
  "HealthyLevel": "<HealthyLevel>",
  "Count": 0,
  "Diagnostics": [],
  "Catalogues": [],
  "CatalogueLevel": "<CatalogueLevel>",
  "CatalogueName": "<CatalogueName>"
}
```

```bash
# 确认详情接口可正常返回
tccli tke ListClusterInspectionResultsItems --ClusterId cls-xxxxxxxx --region ap-guangzhou
# 预期: exit 0，InspectionItems 数组非空
```

```json
{
  "RequestId": "..."
}
```

### 验证维度

| 验证维度 | 命令 | 预期 |
|---------|------|------|
| 概览可查询 | `DescribeClusterInspectionResultsOverview` | `InspectionResultsOverview` 非空 |
| 历史记录可查 | `ListClusterInspectionResults` | `InspectionResults` 数组非空 |
| 详情项可查 | `ListClusterInspectionResultsItems` | `InspectionItems` 数组非空 |
| 无权限错误 | 上述任一命令 | 不返回 `UnauthorizedOperation` |

## 清理

健康检查为只读操作，无需清理业务资源。检查过程中 TKE 自动创建的 namespace `tke-cluster-inspection` 和 DaemonSet 会在检查完成后自动清理，无需手动干预。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DescribeClusterInspectionResultsOverview` 返回空数组 `"InspectionResultsOverview": []` | 检查是否在控制台触发过检查 | 集群从未执行过健康检查，无历史数据 | 登录控制台 > 运维中心 > 健康检查，触发一次"立即检查"后重新查询 |
| `ListClusterInspectionResults` 返回空数组 | 同上 | 同上 | 同上 |
| `ListClusterInspectionResultsItems` 返回空数组 | 确认检查记录是否存在 | 检查记录存在但类别下无检查项（极少见），或集群从未执行过检查 | 先执行 `ListClusterInspectionResults` 确认有已完成（`Completed` 状态）的检查记录 |
| 返回 `UnauthorizedOperation` | `tccli configure list` 检查凭据 | 子账号缺少 `tke:DescribeClusterInspectionResultsOverview` / `tke:ListClusterInspectionResults` / `tke:ListClusterInspectionResultsItems` 权限 | 联系主账号授予 `QcloudTKEFullAccess` 或对应 Action 权限 |
| 返回 `InvalidParameter.ClusterId` | 检查 ClusterId 格式（应以 `cls-` 开头） | 集群 ID 格式错误或集群不存在 | 使用 `tccli tke DescribeClusters --region ap-guangzhou` 确认集群 ID |

### 操作成功但结果异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 异常检查项数较多 | 逐条查看 `InspectionItems[].Reason` 和 `Suggestion` | 集群配置存在多处不合理之处 | 根据每条异常项的 `Suggestion` 字段逐项修复 |
| Node 高可用检查异常 | 检查节点分布：`tccli tke DescribeClusterInstances --ClusterId cls-xxxxxxxx --region ap-guangzhou` | 单节点集群，或节点集中在单一可用区 | 在多可用区部署节点，确保高可用 |
| HPA-IP 配置异常 | 排查集群剩余 Pod IP 数 | 当前集群剩余 Pod IP 数不足以满足 HPA 扩容的最大数 | 扩容集群 CIDR 或调整 HPA 最大副本数 |
| 工作负载 Request/Limit 检查异常 | 查看异常项 Detail 中的具体工作负载名 | 工作负载未配置资源限制 | 为指定工作负载补充 `resources.requests` 和 `resources.limits` |
| 独立集群控制面组件异常 | 检查具体组件状态（仅独立集群有此类检查项） | kube-apiserver、kube-scheduler、kube-controller-manager 或 etcd 状态异常 | 联系 TKE 运维团队排查控制面组件 |

## 下一步

- [风险推送](../风险推送/tccli%20操作.md) — 风险事件 EventBridge 推送配置
- [监控管理](../监控管理/tccli%20操作.md) — 集群指标监控与告警
- [日志管理](../日志管理/tccli%20操作.md) — 集群日志采集与分析
- [审计管理](../审计管理/tccli%20操作.md) — 集群操作审计

## 控制台替代

登录 [容器服务控制台](https://console.cloud.tencent.com/tke2) > 运维中心 > 健康检查。支持批量检查、立即检查、自动定时检查三种触发方式，检查完成后可在控制台查看可视化报告，包括异常项分布、历史趋势对比等 CLI 无法直接呈现的图表信息。
