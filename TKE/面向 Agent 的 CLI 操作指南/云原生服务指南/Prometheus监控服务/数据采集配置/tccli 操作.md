# 数据采集配置

> 对照官方：[数据采集配置](https://cloud.tencent.com/document/product/457/71899) · page_id `71899`

## 概述

通过 `DescribePrometheusConfig` 查询和 `ModifyPrometheusConfig` 修改 Prometheus 监控实例的数据采集配置。采集配置定义了 agent 需要抓取的指标目标，支持通过 ServiceMonitor/PodMonitor CRD 或直接配置 scrape_config 两种方式。配置生效后，agent 自动按规则采集目标指标并上报到托管实例。

## 前置条件

- [环境准备](../../../环境准备.md)
- 已有 Prometheus 监控实例，状态为 `Running`
- 已关联至少一个 TKE 集群，agent 状态为 `running`

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
#    monitor:DescribePrometheusInstances, monitor:DescribePrometheusClusterAgents
# 验证：执行 DescribePrometheusInstances 确认权限
tccli monitor DescribePrometheusInstances --region <Region>
# expected: exit 0，返回实例列表

# 4. 验证 monitor 配置读写权限
tccli monitor DescribePrometheusConfig --region <Region> \
    --InstanceId '<InstanceId>' --ClusterId '<ClusterId>'
# expected: exit 0（配置为空时正常返回）
```

### 资源检查

```bash
# 5. 确认 Prometheus 实例 Running
tccli monitor DescribePrometheusInstances --region <Region> \
    --InstanceIds '["<InstanceId>"]'
# expected: InstanceStatus: 2 (Running)

# 6. 确认 agent 状态
tccli monitor DescribePrometheusClusterAgents --region <Region> \
    --InstanceId '<InstanceId>' --ClusterId '<ClusterId>'
# expected: Status: "running"

# 7. 查看当前采集配置
tccli monitor DescribePrometheusConfig --region <Region> \
    --InstanceId '<InstanceId>' --ClusterId '<ClusterId>'
# expected: 返回当前 scrape_config，可能为空
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看采集配置 | `tccli monitor DescribePrometheusConfig --region <Region> --InstanceId '<InstanceId>' --ClusterId '<ClusterId>'` | 是 |
| 修改采集配置 | `tccli monitor ModifyPrometheusConfig --region <Region> --cli-input-json file://config.json` | 否 |

## 操作步骤

### 步骤 1：查看当前采集配置

#### 选择依据

先查看当前配置，了解已有哪些 scrape target，再决定需要新增或修改的配置。采集配置包含两类内容：
- **基础指标采集**（系统自动配置）：node-exporter、kube-state-metrics、kubelet 等基础指标
- **自定义采集**：用户通过 ServiceMonitor/PodMonitor CRD 或 scrape_config 声明的自定义指标

```bash
tccli monitor DescribePrometheusConfig --region <Region> \
    --InstanceId '<InstanceId>' \
    --ClusterId '<ClusterId>'
# expected: exit 0，返回 config 内容
```

预期输出（示例）：

```json
{
    "Config": "scrape_configs:\n- job_name: kubernetes-pods\n  kubernetes_sd_configs:\n  - role: pod\n  relabel_configs:\n  - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]\n    action: keep\n    regex: true\n",
    "RequestId": "..."
}
```

| 占位符 | 说明 | 获取方式 |
|--------|------|---------|
| `<InstanceId>` | Prometheus 实例 ID | `tccli monitor DescribePrometheusInstances --region <Region>` |
| `<ClusterId>` | TKE 集群 ID | `tccli tke DescribeClusters --region <Region>` |
| `REGION` | 地域 | `tccli configure list` |

### 步骤 2：修改采集配置

#### 选择依据

- **基础指标**：agent 关联后自动采集 node-exporter、kube-state-metrics、kubelet 等基础指标，通常无需额外配置
- **自定义指标**：如需采集应用暴露的 Prometheus 指标（如 ngx_http_stub_status_module、JVM metrics），通过 ServiceMonitor CRD 或 scrape_config 声明
- **ServiceMonitor vs scrape_config**：推荐使用 ServiceMonitor CRD（声明式、Kubernetes 原生），直接写 scrape_config 适合简单场景或非标准端口

#### 最小修改（新增一个 scrape job）

`config-minimal.json`：

```json
{
    "InstanceId": "<InstanceId>",
    "ClusterId": "<ClusterId>",
    "Config": "scrape_configs:\n- job_name: 'my-app'\n  scrape_interval: 30s\n  static_configs:\n  - targets: ['my-app-svc.default.svc.cluster.local:9090']\n"
}
```

```bash
tccli monitor ModifyPrometheusConfig --region <Region> \
    --cli-input-json file://config-minimal.json
# expected: exit 0，返回 RequestId
```

预期输出：

```json
{
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

| 占位符 | 说明 |
|--------|------|
| `<InstanceId>` | Prometheus 实例 ID |
| `<ClusterId>` | TKE 集群 ID |

#### 增强配置（多 job + kubernetes_sd_configs）

`config-enhanced.json`：

```json
{
    "InstanceId": "<InstanceId>",
    "ClusterId": "<ClusterId>",
    "Config": "scrape_configs:\n- job_name: 'kubernetes-service-endpoints'\n  scrape_interval: 30s\n  kubernetes_sd_configs:\n  - role: endpoints\n  relabel_configs:\n  - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape]\n    action: keep\n    regex: true\n  - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_path]\n    action: replace\n    target_label: __metrics_path__\n    regex: (.+)\n- job_name: 'my-custom-app'\n  scrape_interval: 15s\n  static_configs:\n  - targets: ['app-svc:8080']\n    labels:\n      env: 'production'\n"
}
```

```bash
tccli monitor ModifyPrometheusConfig --region <Region> \
    --cli-input-json file://config-enhanced.json
# expected: exit 0，返回 RequestId
```

> **注意**：`ModifyPrometheusConfig` 是**全量覆盖**操作，不是追加。提交前务必先用 `DescribePrometheusConfig` 获取当前配置，合并后再提交，避免覆盖已有的采集规则。

### 安全更新配置流程（避免覆盖已有规则）

```bash
# 1. 获取当前配置
tccli monitor DescribePrometheusConfig --region <Region> \
    --InstanceId '<InstanceId>' --ClusterId '<ClusterId>' \
    | jq -r '.Config' > current-config.yaml

# 2. 编辑配置，追加新的 scrape job
# （手动编辑 current-config.yaml，追加新规则）

# 3. 提交更新后的配置
CONFIG=$(cat current-config.yaml | python3 -c "import sys,json; print(json.dumps(sys.stdin.read()))")
tccli monitor ModifyPrometheusConfig --region <Region> \
    --InstanceId '<InstanceId>' \
    --ClusterId '<ClusterId>' \
    --Config "$CONFIG"
# expected: exit 0

# 4. 验证配置已生效
tccli monitor DescribePrometheusConfig --region <Region> \
    --InstanceId '<InstanceId>' --ClusterId '<ClusterId>'
# expected: 配置包含新增的 scrape job
```

## 验证

| 维度 | 命令 | 预期 |
|------|------|------|
| 配置一致性 | `tccli monitor DescribePrometheusConfig --region <Region> --InstanceId '<InstanceId>' --ClusterId '<ClusterId>'` | 返回的 Config 包含修改后的 scrape job |
| scrape target 可达 | 在集群内测试目标端点：`kubectl run -it --rm debug --image=busybox -- wget -qO- TARGET_URL` | 返回 metrics 数据 |
| agent 采集状态 | 查看 agent 日志中的 scrape 记录（如果有日志访问权限） | 无 scrape 错误 |

```bash
# 验证配置生效
tccli monitor DescribePrometheusConfig --region <Region> \
    --InstanceId '<InstanceId>' --ClusterId '<ClusterId>' | jq -r '.Config' | grep 'my-app'
# expected: 输出包含 "my-app" 的 scrape job 配置行
```

## 清理

> **警告**：修改采集配置可能导致指标数据缺失一段时间。如果是要停止采集某个 job，直接修改配置移除对应 scrape job 即可，无需删除配置。无采集配置时 agent 仍会采集基础指标。
> **计费提醒**：采集配置本身不产生费用。费用由数据上报量决定，减少采集目标可降低存储成本。

### 1. 查看当前配置

```bash
tccli monitor DescribePrometheusConfig --region <Region> \
    --InstanceId '<InstanceId>' --ClusterId '<ClusterId>' | jq -r '.Config'
# 查看当前所有 scrape job
```

### 2. 移除自定义 scrape job（通过覆盖更新）

```bash
# 获取当前配置，编辑移除目标 job，重新提交
# （参考上方"安全更新配置流程"，提交时不包含要移除的 job）
tccli monitor ModifyPrometheusConfig --region <Region> \
    --cli-input-json file://cleaned-config.json
# expected: exit 0
```

### 3. 验证已移除

```bash
tccli monitor DescribePrometheusConfig --region <Region> \
    --InstanceId '<InstanceId>' --ClusterId '<ClusterId>' | jq -r '.Config' | grep 'my-app'
# expected: 无输出（目标 job 已移除）
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DescribePrometheusConfig` 返回 `ResourceNotFound` | `tccli monitor DescribePrometheusInstances --region <Region> --InstanceIds '["<InstanceId>"]'` 验证实例存在 | InstanceId 不存在或已销毁 | 确认 InstanceId 正确 |
| `ModifyPrometheusConfig` 返回 `InvalidParameter` | 检查 Config 字段的 YAML 语法 | scrape_config 格式错误或非合法 YAML | 用 `echo 'CONFIG_CONTENT' \| python3 -c "import sys,yaml; yaml.safe_load(sys.stdin.read()); print('OK')"` 验证 YAML 合法性 |
| `ModifyPrometheusConfig` 返回 `FailedOperation` | 检查 Config 内容是否超过长度限制 | Config 过长或包含不支持的特性 | 简化 scrape_config，移除不支持的 relabel 规则 |
| `ModifyPrometheusConfig` 返回 `UnauthorizedOperation.CamNoAuth` | 检查 CAM 策略 | 缺少 `monitor:ModifyPrometheusConfig` 权限 | 联系管理员授权 `monitor:ModifyPrometheusConfig` |

### 操作成功但异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 修改配置后 Describe 看到的配置未变化 | `tccli monitor DescribePrometheusConfig --region <Region> --InstanceId '<InstanceId>' --ClusterId '<ClusterId>'` | 配置同步有延迟，需约 30 秒 | 等待 30-60 秒后重新 Describe |
| 配置生效但 target 无数据 | 在集群内 `kubectl get endpoints TARGET_SVC` 确认端点存在 | Service 没有 Endpoint（Pod 未就绪或 selector 不匹配） | 检查目标 Service 和 Pod 状态，确认 Pod 已 Ready |
| 基础指标不再上报 | `tccli monitor DescribePrometheusConfig --region <Region> --InstanceId '<InstanceId>' --ClusterId '<ClusterId>'` 检查基础 job | 提交配置时覆盖了基础指标采集 job | 重新获取默认基础配置并合并提交 |
| 配置后 agent 状态变为 error | `tccli monitor DescribePrometheusClusterAgents --region <Region> --InstanceId '<InstanceId>' --ClusterId '<ClusterId>'` | Config 格式错误导致 agent 无法 reload | 先用 Describe 获取当前配置，修正错误后重新提交 |

## 下一步

- [精简监控指标](../精简监控指标/tccli%20操作.md) — 过滤不需要的指标以控制成本
- [告警历史](../告警历史/tccli%20操作.md) — 基于采集的指标配置告警规则
- [销毁监控实例](../销毁监控实例/tccli%20操作.md) — 彻底清理监控实例

## 控制台替代

控制台：容器服务控制台 → 运维功能 → Prometheus 监控 → 选择实例 → 数据采集配置。
