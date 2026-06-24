# 调度器高性能模式（tccli）

> 对照官方：[调度器高性能模式](https://cloud.tencent.com/document/product/457/129803) · page_id `129803`

## 概述

调度器高性能模式（Scheduler High-Performance Mode）针对大规模集群（通常节点数 >= 1000）优化 TKE 默认调度器的
调度吞吐和延迟。启用后，调度器以批量处理模式运行，显著降低调度延迟，提升大规模 Pod 创建/调度场景下的性能。

**使用方式**：通过 `tccli tke ModifyClusterAttribute` 启用 `SchedulerHighPerformanceMode` 属性。
仅 `tccli` 控制面操作，无需数据面（kubectl）介入。

## 前置条件

- [环境准备](../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置，region 为目标地域

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:DescribeClusters, tke:ModifyClusterAttribute
# 验证：执行 DescribeClusters 确认权限
tccli tke DescribeClusters --region ap-guangzhou
# expected: exit 0，返回集群列表（可为空）
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

### 资源检查

```bash
# 4. 确认目标集群存在且状态正常
tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["CLUSTER_ID"]'
# expected: TotalCount >= 1，ClusterStatus: "Running"

# 5. 确认集群当前调度器模式（检查 SchedulerHighPerformanceMode 字段）
tccli tke DescribeClusters --region ap-guangzhou \
    --ClusterIds '["CLUSTER_ID"]' \
    | jq '.Clusters[0].SchedulerHighPerformanceMode'
# expected: null 或 false（未启用状态）
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

### 版本与规格选择

- 集群规模建议：节点数 >= 500 时考虑启用，节点数 >= 1000 时推荐启用。小规模集群启用后无明显收益，且会
  增加调度器资源消耗。
- 确认当前节点数：`tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["CLUSTER_ID"]' | jq '.Clusters[0].ClusterNodeNum'`。
- Kubernetes 版本要求：需集群版本支持高性能调度器模式。通过
  `tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["CLUSTER_ID"]' | jq '.Clusters[0].ClusterVersion'` 确认。

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看集群详情 | `DescribeClusters --ClusterIds` | 是 |
| 启用调度器高性能模式 | `ModifyClusterAttribute --SchedulerHighPerformanceMode true` | 是 |
| 关闭调度器高性能模式 | `ModifyClusterAttribute --SchedulerHighPerformanceMode false` | 是 |

## Key parameters

以下说明 `ModifyClusterAttribute` 中与调度器高性能模式相关的参数。

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `SchedulerHighPerformanceMode` | Boolean | 是 | `true`（启用）或 `false`（关闭）。启用后调度器以高性能批次模式运行 | 小规模集群开启无明显收益，且多消耗调度器资源 |
| `ClusterId` | String | 是 | 目标集群 ID，格式 `cls-xxxxxxxx` | 集群不存在 → `ResourceNotFound.ClusterNotFound` |

## 操作步骤

### 步骤 1：查看当前调度器模式

```bash
tccli tke DescribeClusters --region ap-guangzhou \
    --ClusterIds '["CLUSTER_ID"]'
# expected: exit 0，集群信息中包含 SchedulerHighPerformanceMode 字段
```

**预期输出**：

```json
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "cls-xxxxxxxx",
            "ClusterName": "CLUSTER_NAME",
            "ClusterStatus": "Running",
            "ClusterVersion": "1.30.0",
            "ClusterNodeNum": 1200,
            "SchedulerHighPerformanceMode": false
        }
    ]
}
```

| 字段 | 值 | 含义 |
|------|-----|------|
| `SchedulerHighPerformanceMode: false` | 未启用 | 调度器以默认模式运行 |
| `SchedulerHighPerformanceMode: true` | 已启用 | 调度器以高性能批量模式运行 |

### 步骤 2：启用调度器高性能模式

#### 选择依据

- **何时启用**：集群节点数 >= 1000，或频繁出现大量 Pod 同时调度（如大规模滚动更新、批量 Job 提交）导致
  调度延迟 > 30s。
- **何时不启用**：节点数 < 500 的小规模集群。高性能模式会增加调度器内存和 CPU 消耗，小集群
  不划算。
- **影响范围**：启用后影响集群默认调度器（kube-scheduler）的行为，对已调度的 Pod 无影响。
- **可逆操作**：`ModifyClusterAttribute --SchedulerHighPerformanceMode false` 即可关闭，无持久化副作用。

```bash
tccli tke ModifyClusterAttribute --region ap-guangzhou \
    --ClusterId CLUSTER_ID \
    --SchedulerHighPerformanceMode true
# expected: exit 0，返回 RequestId
```

**预期输出**：

```json
{
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

| 占位符 | 说明 | 获取方式 |
|--------|------|---------|
| `CLUSTER_ID` | 目标集群 ID | `tccli tke DescribeClusters --region ap-guangzhou` |

## 验证

### 控制面（tccli）

```bash
# 确认 SchedulerHighPerformanceMode 已变更为 true
tccli tke DescribeClusters --region ap-guangzhou \
    --ClusterIds '["CLUSTER_ID"]' \
    | jq '.Clusters[0].SchedulerHighPerformanceMode'
# expected: true
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

| 维度 | 命令 | 预期 |
|------|------|------|
| 集群状态 | `tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["CLUSTER_ID"]'` | `ClusterStatus: "Running"` |
| 调度器模式 | 同上，检查 `SchedulerHighPerformanceMode` | `true`（启用后） |
| 集群版本 | 同上，检查 `ClusterVersion` | 与启用前一致，版本无变化 |
| 节点数 | 同上，检查 `ClusterNodeNum` | 与启用前一致，节点数无变化 |

> ModifyClusterAttribute 是同步接口，返回 RequestId 即表示配置已生效。

## 清理

> **警告**：关闭调度器高性能模式后，调度器恢复默认处理模式。在大量 Pod 同时调度时可能导致调度延迟增加。
> 如果集群节点数 >= 1000，建议保持启用状态。

### 1. 清理前状态检查

```bash
tccli tke DescribeClusters --region ap-guangzhou \
    --ClusterIds '["CLUSTER_ID"]' \
    | jq '.Clusters[0] | {ClusterStatus, ClusterNodeNum, SchedulerHighPerformanceMode}'
# 确认当前集群状态、节点数和调度器模式
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

### 2. 关闭调度器高性能模式

```bash
tccli tke ModifyClusterAttribute --region ap-guangzhou \
    --ClusterId CLUSTER_ID \
    --SchedulerHighPerformanceMode false
# expected: exit 0，返回 RequestId
```

### 3. 验证已关闭

```bash
tccli tke DescribeClusters --region ap-guangzhou \
    --ClusterIds '["CLUSTER_ID"]' \
    | jq '.Clusters[0].SchedulerHighPerformanceMode'
# expected: false
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

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `ModifyClusterAttribute` 返回 `ResourceNotFound.ClusterNotFound` | `tccli tke DescribeClusters --region ap-guangzhou` 查看集群列表 | 集群 ID 错误或不属于当前账号/地域 | 确认 `--ClusterId` 和 `--region` 与目标集群一致 |
| `ModifyClusterAttribute` 返回 `InvalidParameter.SchedulerHighPerformanceMode` | 检查 `SchedulerHighPerformanceMode` 值是否为布尔类型 | 填写了非布尔值（如字符串 `"true"` 或数字 `1`），tccli 参数类型错误 | 使用 `true` 或 `false`（不加引号） |
| `ModifyClusterAttribute` 返回 `FailedOperation.ClusterVersionNotSupport` | `tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["CLUSTER_ID"]' \| jq '.Clusters[0].ClusterVersion'` 查看集群版本 | 当前集群版本不支持高性能调度器模式（此为环境限制） | 升级集群版本到支持该功能的版本 |
| `ModifyClusterAttribute` 返回 `UnauthorizedOperation` | `tccli configure list` 检查当前凭据 | 子账号缺少 `tke:ModifyClusterAttribute` 权限（此为环境限制） | 联系主账号授予 `QcloudTKEFullAccess` 或最小权限 `tke:ModifyClusterAttribute` |

### 启用后调度性能无明显提升

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 启用后 Pod 调度延迟无明显降低 | `tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["CLUSTER_ID"]' \| jq '.Clusters[0].ClusterNodeNum'` 查看节点数 | 集群规模较小（< 500 节点），默认调度器本身足够快（此为正常行为，非错误） | 小规模集群关闭高性能模式以节省调度器资源 |
| 启用后调度器内存/CPU 消耗明显增加 | `kubectl top pods -n kube-system \| grep scheduler` 查看调度器资源使用（需 kubectl 可达环境） | 调度器高性能模式需要额外内存和 CPU（此为预期行为） | 评估节点规模是否确实需要高性能模式；若节点数 < 500，建议关闭 |
| Pod 调度延迟仍 > 30s | `kubectl get events --all-namespaces \| grep -i "failed\|timeout" \| tail -20` 查看调度事件 | 调度延迟可能由节点资源不足、Pod 亲和性约束等原因导致，非调度器模式问题 | 检查集群资源余量：`kubectl describe nodes`；检查 Pod 调度约束是否过于严格 |

## 下一步

- [集群扩缩容](https://cloud.tencent.com/document/product/457/32194) — 大规模集群的节点扩缩容操作
- [集群管理模式说明](https://cloud.tencent.com/document/product/457/31013) — 托管集群与独立集群的区别
- [调度组件概述](https://cloud.tencent.com/document/product/457/111862) — TKE 调度组件全景
- [Crane 调度器介绍](https://cloud.tencent.com/document/product/457/75472) — 了解 Crane 智能调度能力
- [TKE API 概览](https://cloud.tencent.com/document/product/457/31853) — 完整 API 列表

## 控制台替代

[TKE 控制台 → 集群 → 基本信息](https://console.cloud.tencent.com/tke2/cluster)，在集群详情页找到「调度器高性能模式」开关。
