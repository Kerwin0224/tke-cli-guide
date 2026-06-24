# 精简监控指标

> 对照官方：[精简监控指标](https://cloud.tencent.com/document/product/457/71900) · page_id `71900`

## 概述

Prometheus 按存储数据量计费，未经筛选的原始监控指标可能包含大量低频使用的数据。通过 `ModifyPrometheusConfig` 配置 metric_relabel_configs，在采集阶段丢弃不需要的指标或标签，可显著降低存储成本和查询延迟。精简操作在 agent 端执行，不影响监控实例的正常运行。

**常见精简策略**：

| 策略 | 说明 | 成本影响 |
|------|------|:--:|
| 丢弃不必要的指标 | 通过 `metric_relabel_configs` 的 `drop` action 丢弃整个指标 | 高 |
| 丢弃高频但无价值的标签 | 丢弃 `pod_name`、`container_id` 等不需要的高基数标签 | 中 |
| 按标签值过滤 | 只保留特定 namespace 或 deployment 的指标 | 高 |
| 聚合预计算 | 通过 recording rules 只存储聚合后的指标 | 高 |

## 前置条件

- [环境准备](../../../环境准备.md)
- 已有 Prometheus 监控实例，状态为 `Running`
- 已关联 TKE 集群并配置了采集规则（见 [数据采集配置](../数据采集配置/tccli%20操作.md)）

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    monitor:DescribePrometheusConfig, monitor:ModifyPrometheusConfig
#    monitor:DescribePrometheusInstances
# 验证
tccli monitor DescribePrometheusInstances --region <Region>
# expected: exit 0，返回实例列表
```

### 资源检查

```bash
# 4. 确认实例 Running
tccli monitor DescribePrometheusInstances --region <Region> \
    --InstanceIds '["<InstanceId>"]'
# expected: InstanceStatus: 2 (Running)

# 5. 查看当前采集配置（了解哪些指标在被采集）
tccli monitor DescribePrometheusConfig --region <Region> \
    --InstanceId '<InstanceId>' --ClusterId '<ClusterId>'
# expected: 返回当前 scrape_config，了解已有 job 和 target

# 6. 查看 agent 状态
tccli monitor DescribePrometheusClusterAgents --region <Region> \
    --InstanceId '<InstanceId>' --ClusterId '<ClusterId>'
# expected: Status: "running"
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看采集配置 | `tccli monitor DescribePrometheusConfig --region <Region> --InstanceId '<InstanceId>' --ClusterId '<ClusterId>'` | 是 |
| 添加指标过滤 | `tccli monitor ModifyPrometheusConfig --region <Region> --cli-input-json file://config.json` | 否 |

## 操作步骤

### 步骤：通过 metric_relabel_configs 精简指标

#### 选择依据

- **丢弃 vs 保留**：推荐使用 `keep` action 明确保留需要的指标（白名单），而不是用 `drop` 层层排除（黑名单）。白名单模式更安全，不会遗漏意外的低成本指标。
- **正则表达式**：使用正向匹配（`regex`）指定保留或丢弃的条件，黄金信号指标（USE/RED）通常需要保留：`up`、`scrape_duration_seconds`、`container_memory_working_set_bytes`、`container_cpu_usage_seconds_total` 等。
- **高基数标签**：`pod_uid`、`container_id`、`image_id` 等标签会产生极高基数，如非必要建议丢弃。

#### 最小精简（丢弃特定指标）

在需要精简的 scrape job 配置中，新增 `metric_relabel_configs` 段。以下是丢弃 `go_gc_*` 系列指标的最小配置更新：

`config-minimal.json`：

```json
{
    "InstanceId": "<InstanceId>",
    "ClusterId": "<ClusterId>",
    "Config": "scrape_configs:\n- job_name: 'kubernetes-pods'\n  kubernetes_sd_configs:\n  - role: pod\n  relabel_configs:\n  - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]\n    action: keep\n    regex: true\n  metric_relabel_configs:\n  - source_labels: [__name__]\n    regex: 'go_gc_.*'\n    action: drop\n"
}
```

```bash
tccli monitor ModifyPrometheusConfig --region <Region> \
    --cli-input-json file://config-minimal.json
# expected: exit 0，返回 RequestId
```

| 占位符 | 说明 | 获取方式 |
|--------|------|---------|
| `<InstanceId>` | Prometheus 实例 ID | `tccli monitor DescribePrometheusInstances --region <Region>` |
| `<ClusterId>` | TKE 集群 ID | `tccli tke DescribeClusters --region <Region>` |
| `REGION` | 地域 | `tccli configure list` |

#### 增强配置（多规则：丢弃低频指标 + 剥离高基数标签）

`config-enhanced.json`：

```json
{
    "InstanceId": "<InstanceId>",
    "ClusterId": "<ClusterId>",
    "Config": "scrape_configs:\n- job_name: 'kubernetes-pods'\n  kubernetes_sd_configs:\n  - role: pod\n  relabel_configs:\n  - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]\n    action: keep\n    regex: true\n  metric_relabel_configs:\n  - source_labels: [__name__]\n    regex: 'go_gc_.*|go_memstats_.*|process_.*'\n    action: drop\n  - source_labels: [__name__, container_id]\n    regex: 'container_.+;.+'\n    action: drop\n    separator: ';'\n  - regex: '(pod_uid|container_id|image_id)'\n    action: labeldrop\n"
}
```

```bash
tccli monitor ModifyPrometheusConfig --region <Region> \
    --cli-input-json file://config-enhanced.json
# expected: exit 0，返回 RequestId
```

### metric_relabel_configs 常用规则速查

| Action | 用途 | 示例 |
|--------|------|------|
| `keep` | 只保留匹配的指标 | 保留 `container_cpu_*` 和 `container_memory_*` |
| `drop` | 丢弃匹配的指标 | 丢弃 `go_gc_*`、`go_memstats_*` |
| `labeldrop` | 丢弃指定的标签名 | 丢弃 `pod_uid`、`container_id`、`image_id` |
| `labelkeep` | 只保留指定的标签名 | 只保留 `namespace`、`pod`、`container` |
| `replace` | 替换标签值或标签名 | 将长标签名缩短 |

> **注意**：`metric_relabel_configs` 在指标采集后、存储前执行。修改后已存储的历史数据不受影响，仅对新采集的指标生效。

## 验证

| 维度 | 命令 | 预期 |
|------|------|------|
| 配置已更新 | `tccli monitor DescribePrometheusConfig --region <Region> --InstanceId '<InstanceId>' --ClusterId '<ClusterId>'` | 包含 `metric_relabel_configs` 段 |
| 被丢弃的指标不再上报 | 在控制台或通过 PromQL 查询被丢弃的指标名 | 指标不存在或无新数据点 |

```bash
# 验证配置包含精简规则
tccli monitor DescribePrometheusConfig --region <Region> \
    --InstanceId '<InstanceId>' \
    --ClusterId '<ClusterId>' | jq -r '.Config' | grep 'metric_relabel_configs'
# expected: 输出包含 metric_relabel_configs 行
```

## 清理

> **注意**：移除 metric_relabel_configs 不会删除已存储的历史数据，但新采集的指标将重新包含之前被丢弃的指标，增加存储成本。
> **计费提醒**：恢复全量指标采集后，数据上报量可能大幅增加，按量计费成本随之增加。

### 1. 查看当前配置中的精简规则

```bash
tccli monitor DescribePrometheusConfig --region <Region> \
    --InstanceId '<InstanceId>' --ClusterId '<ClusterId>' | jq -r '.Config' | grep -A 10 'metric_relabel_configs'
# 查看所有精简规则
```

### 2. 移除精简规则（恢复全量采集）

获取当前配置，编辑移除 `metric_relabel_configs` 段，重新提交更新后的配置。

```bash
# 1. 导出当前配置
tccli monitor DescribePrometheusConfig --region <Region> \
    --InstanceId '<InstanceId>' --ClusterId '<ClusterId>' \
    | jq -r '.Config' > current-config.yaml

# 2. 编辑移除 metric_relabel_configs 段

# 3. 提交更新
CONFIG=$(cat current-config.yaml | python3 -c "import sys,json; print(json.dumps(sys.stdin.read()))")
tccli monitor ModifyPrometheusConfig --region <Region> \
    --InstanceId '<InstanceId>' \
    --ClusterId '<ClusterId>' \
    --Config "$CONFIG"
# expected: exit 0
```

### 3. 验证已移除

```bash
tccli monitor DescribePrometheusConfig --region <Region> \
    --InstanceId '<InstanceId>' --ClusterId '<ClusterId>' | jq -r '.Config' | grep 'metric_relabel_configs'
# expected: 无输出（精简规则已移除）
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `ModifyPrometheusConfig` 返回 `InvalidParameter` | 检查 metric_relabel_configs 的 YAML 语法 | metric_relabel_configs 语法错误或在 Config 中缩进不正确 | 用 YAML 验证工具检查语法，确保 `metric_relabel_configs` 在正确的 scrape job 内缩进 |
| `ModifyPrometheusConfig` 返回 `FailedOperation` | 检查规则数量 | 规则过多或正则表达式过于复杂 | 精简规则数量，合并相似规则 |
| `DescribePrometheusConfig` 返回的 Config 为空 | 检查实例和集群 ID | 未关联集群或采集配置未初始化 | 先执行 [关联集群](../关联集群/tccli%20操作.md) |

### 操作成功但异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 精简后预期保留的指标也被丢弃 | `tccli monitor DescribePrometheusConfig --region <Region> --InstanceId '<InstanceId>' --ClusterId '<ClusterId>'` 查看实际规则 | 正则表达式匹配范围过宽，误伤了其他指标 | 收紧正则范围，使用 `promtool check config` 验证规则覆盖范围 |
| 精简后存储成本未明显下降 | 对比精简前后的指标数量和样本数 | 保留的指标中仍有高基数标签或大量时间序列 | 使用 `count({__name__=~".+"}) by (__name__)` 查询 Top N 指标，进一步精简 |
| 配置中已加 metric_relabel_configs 但 Describe 看不到 | 同上 | 配置同步有延迟，约 30-60 秒 | 等待后重新 Describe |
| 配置更新后采集中断 | `tccli monitor DescribePrometheusClusterAgents --region <Region> --InstanceId '<InstanceId>' --ClusterId '<ClusterId>'` 查看 agent 状态 | Config 中存在严重语法错误导致 agent reload 失败 | 检查 agent 状态和日志。先用 Describe 获取正确配置，修正后重新提交 |

## 下一步

- [数据采集配置](../数据采集配置/tccli%20操作.md) — 完整的数据采集配置管理
- [告警历史](../告警历史/tccli%20操作.md) — 基于精简后的指标配置告警
- [计费方式和资源使用](../计费方式和资源使用/tccli%20操作.md) — 了解精简指标对成本的影响

## 控制台替代

控制台：容器服务控制台 → 运维功能 → Prometheus 监控 → 选择实例 → 数据采集 → 编辑 → 添加 metric_relabel_configs。
