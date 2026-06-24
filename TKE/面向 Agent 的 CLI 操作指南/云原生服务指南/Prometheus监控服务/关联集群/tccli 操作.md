# 关联集群

> 对照官方：[关联集群](https://cloud.tencent.com/document/product/457/71898) · page_id `71898`

## 概述

通过 `CreatePrometheusClusterAgent` 将 TKE 集群关联到已创建的 Prometheus 监控实例。该操作会在 TKE 集群中部署 agent 组件（prometheus-agent、node-exporter、kube-state-metrics 等），自动采集集群的监控指标并上报到托管 Prometheus 实例。

## 前置条件

- [环境准备](../../../环境准备.md)
- 已有 Prometheus 监控实例，状态为 `Running`（见 [创建监控实例](../创建监控实例/tccli%20操作.md)）
- 已有 TKE 集群，状态为 `Running`，且与 Prometheus 实例在同一 VPC

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    monitor:CreatePrometheusClusterAgent, monitor:DescribePrometheusClusterAgents
#    monitor:DescribePrometheusInstances
#    tke:DescribeClusters
# 验证：执行 DescribePrometheusInstances 确认权限
tccli monitor DescribePrometheusInstances --region <Region>
# expected: exit 0，返回实例列表

# 验证 TKE 权限
tccli tke DescribeClusters --region <Region>
# expected: exit 0，返回集群列表
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
# 4. 确认 Prometheus 实例状态为 Running
tccli monitor DescribePrometheusInstances --region <Region> \
    --InstanceIds '["<InstanceId>"]'
# expected: InstanceStatus: 2 (Running)

# 5. 确认 TKE 集群状态为 Running，且与实例在同一 VPC
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["<ClusterId>"]'
# expected: ClusterStatus: "Running"，ClusterNetworkSettings.VpcId 与实例 VpcId 一致

# 6. 检查集群是否已关联 Prometheus 实例（一个集群只能关联一个实例）
tccli monitor DescribePrometheusClusterAgents --region <Region> \
    --InstanceId '<InstanceId>' \
    --ClusterId '<ClusterId>'
# expected: 未关联时返回空，已关联时返回 agent 信息
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

### 集群与实例约束

- 一个 TKE 集群只能关联一个 Prometheus 监控实例
- 集群和实例必须在同一 VPC 内
- 关联操作会在集群中创建 agent 组件，需确保集群有足够资源

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 关联集群 | `tccli monitor CreatePrometheusClusterAgent --region <Region> --cli-input-json file://input.json` | 否 |
| 查看关联集群 | `tccli monitor DescribePrometheusClusterAgents --region <Region> --InstanceId '<InstanceId>' --ClusterId '<ClusterId>'` | 是 |
| 解除关联 | 通过控制台操作（CLI 无直接解关联 API） | 否 |

## 操作步骤

### 步骤：关联 TKE 集群到 Prometheus 实例

#### 选择依据

- **Agent 版本**：不指定版本时自动安装最新稳定版，适合大多数场景。如需指定版本，可通过 `AgentVersion` 参数控制。
- **关联后行为**：关联操作是异步的，agent 安装需要 1-2 分钟。安装完成后自动开始采集集群的基础指标。
- **一个集群一个实例**：如果集群已关联其他 Prometheus 实例，需要先在控制台解除旧关联。

#### 最小关联（自动安装最新 agent）

```bash
# CreatePrometheusClusterAgent 需要以下必填参数：
#   InstanceId — Prometheus 实例 ID
#   ClusterId — TKE 集群 ID
#   ClusterType — 集群类型，TKE 集群固定为 "tke"
#   Region — 地域

tccli monitor CreatePrometheusClusterAgent --region <Region> \
    --InstanceId '<InstanceId>' \
    --ClusterId '<ClusterId>' \
    --ClusterType tke
# expected: exit 0，返回 RequestId
```

预期输出：

```json
{
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<InstanceId>` | Prometheus 实例 ID | 必须状态为 Running | `tccli monitor DescribePrometheusInstances --region <Region>` |
| `<ClusterId>` | TKE 集群 ID | 必须状态为 Running、同 VPC | `tccli tke DescribeClusters --region <Region>` |
| `REGION` | 地域 | 与实例和集群一致 | `tccli configure list` |

#### 增强配置（指定 agent 版本）

```bash
tccli monitor CreatePrometheusClusterAgent --region <Region> \
    --InstanceId '<InstanceId>' \
    --ClusterId '<ClusterId>' \
    --ClusterType tke \
    --AgentVersion '<AgentVersion>'
# expected: exit 0，返回 RequestId
```

| 字段 | 说明 |
|------|------|
| `--AgentVersion` | 可选，指定 agent 版本号。不指定则自动安装最新版本 |

### 轮询等待 agent 就绪

```bash
# 轮询直到 agent 状态正常，通常需要 1-2 分钟
tccli monitor DescribePrometheusClusterAgents --region <Region> \
    --InstanceId '<InstanceId>' \
    --ClusterId '<ClusterId>'
# expected: 返回 agent 列表，Status 为 "running"
```

预期输出（agent 就绪后）：

```json
{
    "Agents": [
        {
            "ClusterId": "cls-example",
            "ClusterName": "my-cluster",
            "InstanceId": "prom-example",
            "Status": "running",
            "AgentVersion": "v1.0.0",
            "ClusterType": "tke"
        }
    ],
    "TotalCount": 1,
    "RequestId": "..."
}
```

## 验证

| 维度 | 命令 | 预期 |
|------|------|------|
| agent 状态 | `tccli monitor DescribePrometheusClusterAgents --region <Region> --InstanceId '<InstanceId>' --ClusterId '<ClusterId>'` | `Status: "running"` |
| agent 版本 | 同上，检查 `AgentVersion` | 返回具体版本号 |
| 关联关系 | 同上，检查 `ClusterId`、`InstanceId` | 与关联参数一致 |

```bash
# 综合验证
tccli monitor DescribePrometheusClusterAgents --region <Region> \
    --InstanceId '<InstanceId>' \
    --ClusterId '<ClusterId>' | jq '.Agents[0] | {ClusterId, InstanceId, Status, AgentVersion}'
# expected: Status: "running", ClusterId/InstanceId 与关联参数一致
```

## 清理

> **警告**：解除集群关联（移除 agent）需要登录 [Prometheus 监控控制台](https://console.cloud.tencent.com/tke2/prometheus) 操作。CLI 未提供直接解除关联的 API。解除关联后集群将停止向该实例上报监控数据。
> **计费提醒**：解除集群关联不会停止 Prometheus 实例的计费。如需停止计费，需 [销毁监控实例](../销毁监控实例/tccli%20操作.md)。

### 1. 清理前状态检查

```bash
tccli monitor DescribePrometheusClusterAgents --region <Region> \
    --InstanceId '<InstanceId>' --ClusterId '<ClusterId>'
# 确认当前 agent 状态和集群信息
```

### 2. 解除关联（控制台操作）

无法通过 CLI 直接解除关联。登录 [Prometheus 监控控制台](https://console.cloud.tencent.com/tke2/prometheus)，找到对应实例 → 集群管理 → 选择目标集群 → 解除关联。

### 3. 验证已解除

```bash
tccli monitor DescribePrometheusClusterAgents --region <Region> \
    --InstanceId '<InstanceId>' --ClusterId '<ClusterId>'
# expected: TotalCount: 0 或不再包含目标集群
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `CreatePrometheusClusterAgent` 返回 `ResourceNotFound` | `tccli monitor DescribePrometheusInstances --region <Region> --InstanceIds '["<InstanceId>"]'` 验证实例存在 | InstanceId 不存在或已销毁 | 确认 InstanceId 正确，实例状态为 Running |
| `CreatePrometheusClusterAgent` 返回 `ResourceNotFound.ClusterNotFound` | `tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'` 验证集群存在 | ClusterId 不存在或已删除 | 确认 ClusterId 正确，集群状态为 Running |
| `CreatePrometheusClusterAgent` 返回 `FailedOperation.AgentAlreadyInstalled` | `tccli monitor DescribePrometheusClusterAgents --region <Region> --InstanceId '<InstanceId>' --ClusterId '<ClusterId>'` 查已有关联 | 该集群已关联了一个 Prometheus 实例（一个集群只能关联一个） | 先在控制台解除旧关联，再重新关联新实例 |
| `CreatePrometheusClusterAgent` 返回 `FailedOperation.VpcMismatch` | 对比实例 VPC 和集群 VPC | 实例和集群不在同一 VPC | 在同一 VPC 中创建新的 Prometheus 实例，或在目标 VPC 中部署 TKE 集群 |
| `CreatePrometheusClusterAgent` 返回 `UnauthorizedOperation.CamNoAuth` | 检查 CAM 策略 | 缺少 `monitor:CreatePrometheusClusterAgent` 权限 | 联系管理员授权 `monitor:CreatePrometheusClusterAgent` |
| `DescribePrometheusClusterAgents` 返回 `InvalidParameter` | 检查参数值 | InstanceId 或 ClusterId 格式错误 | 确认参数格式，实例 ID 格式为 `prom-xxxxxxxx`，集群 ID 格式为 `cls-xxxxxxxx` |

### 操作成功但异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `CreatePrometheusClusterAgent` 返回成功但 agent 列表为空 | `tccli monitor DescribePrometheusClusterAgents --region <Region> --InstanceId '<InstanceId>' --ClusterId '<ClusterId>'` | agent 安装尚未完成 | 等待 1-2 分钟后重新查询 |
| agent 状态长时间为 `installing` | 同上，查看 `Status` 字段 | 集群资源不足导致 agent 无法调度 | 检查集群节点资源，确认有足够的 CPU/内存；查看集群中 agent 相关 Pod 状态 `kubectl get pods -n tke-monitor` |
| agent 状态为 `error` | 同上，查看完整 agent 信息 | agent 安装异常 | 保留 region、InstanceId、ClusterId、RequestId → 登录控制台查看 agent 安装日志 |
| agent running 但无数据上报 | 检查采集配置：`tccli monitor DescribePrometheusConfig --region <Region> --InstanceId '<InstanceId>' --ClusterId '<ClusterId>'` | 采集配置为空或未启用 | 配置采集规则（见 [数据采集配置](../数据采集配置/tccli%20操作.md)） |

## 下一步

- [数据采集配置](../数据采集配置/tccli%20操作.md) — 配置指标采集规则
- [精简监控指标](../精简监控指标/tccli%20操作.md) — 过滤不需要的指标，控制成本
- [告警历史](../告警历史/tccli%20操作.md) — 查询告警历史记录

## 控制台替代

控制台：容器服务控制台 → 运维功能 → Prometheus 监控 → 选择实例 → 集群管理 → 关联集群。
