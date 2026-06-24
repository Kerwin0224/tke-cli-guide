# 腾讯云 Prometheus 一键关联监控容器服务（tccli）

> 对照官方：[腾讯云 Prometheus 一键关联监控容器服务](https://cloud.tencent.com/document/product/457/90923) · page_id `90923`

## 概述

通过腾讯云 Prometheus 监控服务（TMP）一键关联 TKE 托管集群，自动部署 Prometheus Agent 并完成 Kubernetes 集群级指标（Node、Pod、Deployment、StatefulSet 等）的采集与上报。TMP 提供免运维、自动扩缩容的 Prometheus 兼容服务，与 TKE 深度集成，支持 Grafana 预置大盘和告警规则一键导入。数据保存时长支持 15 天（免费额度）至 180 天。

> **kubectl 数据面不可达**：CAM拒绝公网端点(strategyId:240463971)，内网需VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

## 前置条件

### 环境检查

```bash
tccli --version
# expected: tccli version >= 1.0.0

tccli configure list
# expected: secretId 与 secretKey 已配置，默认 region 为 ap-guangzhou
```

验证 CAM 权限——以下为关联操作所需全部 CAM Action，请逐一确认：

```bash
tccli monitor DescribePrometheusInstances \
    --region <Region>
# expected: 返回 Prometheus 实例列表，所需 CAM Action: monitor:DescribePrometheusInstances

tccli tke DescribeClusters \
    --region <Region> \
    --ClusterIds '["CLUSTER_ID"]'
# expected: 返回集群详情，所需 CAM Action: tke:DescribeClusters

tccli monitor DescribePrometheusAgents \
    --region <Region> \
    --InstanceId PROM_INSTANCE_ID
# expected: 返回 Agent 列表（可能为空），所需 CAM Action: monitor:DescribePrometheusAgents
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

| 参数 | 说明 | 示例值 |
|------|------|--------|
| CLUSTER_ID | TKE 托管集群 ID | cls-xxxxxxxx |
| REGION | 腾讯云地域 | ap-guangzhou |
| PROM_INSTANCE_ID | Prometheus 监控实例 ID | prom-xxxxxxxx |
| VPC_ID | TKE 集群所在 VPC（新建 TMP 实例需同 VPC） | vpc-xxxxxxxx |
| SUBNET_ID | VPC 子网 ID（创建 TMP 实例时指定） | subnet-xxxxxxxx |

```bash
tccli tke DescribeClusters \
    --region <Region> \
    --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterStatus 为 "Running"，ClusterType 为 "MANAGED_CLUSTER"，ClusterVersion >= 1.30.0

tccli monitor DescribePrometheusInstances \
    --region <Region> \
    --InstanceIds '["PROM_INSTANCE_ID"]'
# expected: InstanceStatus 为 "Running"，VpcId 与 TKE 集群 VPC 一致
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

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 创建 TMP 实例 | `tccli monitor CreatePrometheusInstance --region <Region> --cli-input-json file://create-tmp.json` | 否（重复创建会产生多个计费实例） |
| 查询 Prometheus 实例列表 | `tccli monitor DescribePrometheusInstances --region <Region>` | 是 |
| 一键关联 TKE 集群 | `tccli monitor CreatePrometheusAgent --region <Region> --cli-input-json file://agent.json` | 否（重复关联会报 AlreadyExists） |
| 查看 Agent 关联状态 | `tccli monitor DescribePrometheusAgents --region <Region> --InstanceId PROM_INSTANCE_ID` | 是 |
| 修改 Agent 采集配置 | `tccli monitor ModifyPrometheusAgent --region <Region> --cli-input-json file://modify-agent.json` | 否 |
| 解除集群关联 | `tccli monitor DeletePrometheusAgent --region <Region> --InstanceId PROM_INSTANCE_ID --AgentId AGENT_ID` | 否（删除不可逆） |

## 操作步骤

### 步骤 1：创建 Prometheus 监控实例（如尚未创建）

#### 选择依据

- **TMP vs 自建 Prometheus**：TMP 免运维、自动扩缩容、与 TKE 深度集成，无需自行管理 Prometheus Server 和存储
- **数据保存时长**：15 天（免费额度）、30 天、90 天、180 天可选，按量计费
- **VPC 要求**：TMP 实例必须与目标 TKE 集群在同一 VPC 内，否则网络不通
- **Grafana 集成**：TMP 自带 Grafana 并预置 TKE 监控大盘

#### 最小创建

```bash
cat > create-tmp.json <<'EOF'
{
    "InstanceName": "tke-monitor",
    "DataRetentionTime": 15,
    "VpcId": "VPC_ID",
    "SubnetId": "SUBNET_ID",
    "Zone": "ap-guangzhou-3",
    "TagSpecification": [
        {
            "ResourceType": "prometheus",
            "Tags": [
                {
                    "Key": "env",
                    "Value": "production"
                }
            ]
        }
    ]
}
EOF
tccli monitor CreatePrometheusInstance \
    --region <Region> \
    --cli-input-json file://create-tmp.json
# expected: 返回 InstanceId（格式 prom-xxxxxxxx），RequestId 非空
```

记录返回的 `InstanceId`，后续步骤使用 `PROM_INSTANCE_ID` 代替。

#### 增强配置

如需更长的数据保存期或指定 Grafana 密码：

```bash
cat > create-tmp-enhanced.json <<'EOF'
{
    "InstanceName": "tke-monitor",
    "DataRetentionTime": 30,
    "VpcId": "VPC_ID",
    "SubnetId": "SUBNET_ID",
    "Zone": "ap-guangzhou-3",
    "GrafanaPassword": "GRAFANA_ADMIN_PASSWORD",
    "EnableGrafana": true
}
EOF
tccli monitor CreatePrometheusInstance \
    --region <Region> \
    --cli-input-json file://create-tmp-enhanced.json
# expected: 返回 InstanceId，Grafana 自动部署
```

### 步骤 2：一键关联 TKE 集群

通过 `CreatePrometheusAgent` 接口将 TKE 集群关联到 TMP 实例，系统自动在集群中部署 Prometheus Agent 组件：

```bash
cat > agent.json <<'EOF'
{
    "InstanceId": "PROM_INSTANCE_ID",
    "Agents": [
        {
            "ClusterId": "CLUSTER_ID",
            "ClusterType": "tke",
            "Region": "REGION",
            "EnableExternal": false
        }
    ]
}
EOF
tccli monitor CreatePrometheusAgent \
    --region <Region> \
    --cli-input-json file://agent.json
# expected: exit 0，RequestId 非空，无 Error 信息
```

关联成功后，系统自动完成以下操作：
- 在 TKE 集群 `tke-monitor` 命名空间中部署 Prometheus Agent（StatefulSet + DaemonSet）
- 自动创建 ServiceMonitor 和 PodMonitor 采集 Kubernetes 基础指标
- 注册 TKE 集群节点、Pod、容器等 target 到 TMP 实例
- 部署 kube-state-metrics 获取 Kubernetes 资源对象状态

### 步骤 3：查看关联状态

```bash
tccli monitor DescribePrometheusAgents \
    --region <Region> \
    --InstanceId PROM_INSTANCE_ID
# expected: Agents 数组中包含已关联的集群信息
```

预期输出：

```json
{
    "Agents": [
        {
            "ClusterId": "CLUSTER_ID",
            "ClusterName": "managed-cluster",
            "ClusterType": "tke",
            "Region": "ap-guangzhou",
            "Status": "Running",
            "AgentId": "agent-xxxxxxxx"
        }
    ],
    "TotalCount": 1,
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

### 步骤 4：修改采集配置（可选）

调整 Agent 的数据采集频率和指标过滤规则：

```bash
cat > modify-agent.json <<'EOF'
{
    "InstanceId": "PROM_INSTANCE_ID",
    "AgentId": "AGENT_ID",
    "Config": "{\"scrape_interval\":\"30s\",\"scrape_timeout\":\"10s\"}"
}
EOF
tccli monitor ModifyPrometheusAgent \
    --region <Region> \
    --cli-input-json file://modify-agent.json
# expected: exit 0，采集配置已更新
```

## 验证

### 控制面（tccli）

```bash
tccli monitor DescribePrometheusInstances \
    --region <Region> \
    --InstanceIds '["PROM_INSTANCE_ID"]'
# expected: InstanceStatus 为 "Running"，GrafanaURL 非空，DataRetentionTime 为设定值

tccli monitor DescribePrometheusAgents \
    --region <Region> \
    --InstanceId PROM_INSTANCE_ID
# expected: Agents[0].Status 为 "Running"

tccli monitor DescribePrometheusTargets \
    --region <Region> \
    --InstanceId PROM_INSTANCE_ID
# expected: Targets 列表非空，包含 kubelet、kube-state-metrics、node-exporter 等 target，Health 均为 "up"
```

### 数据面（需 VPN/IOA）

验证集群中 Agent 组件部署状态：

```bash
kubectl get pods -n tke-monitor
# expected: prometheus-agent 相关 Pod（StatefulSet + DaemonSet）状态均为 Running

kubectl get servicemonitor -n tke-monitor
# expected: 自动创建的 ServiceMonitor 列表，含 kubelet、cadvisor 等

kubectl get statefulset -n tke-monitor
# expected: prometheus-agent StatefulSet，READY 1/1
```

```text
NAME  STATUS  AGE
...
```

通过 Grafana 验证数据（Grafana URL 由控制面 DescribePrometheusInstances 返回）：

```bash
echo "Grafana URL 请从 tccli monitor DescribePrometheusInstances 输出中获取"
# 浏览器访问 Grafana URL -> Dashboards -> TKE Cluster Monitoring
```

## 清理

在执行清理前注意以下事项：
- **计费警告**：`PROM_INSTANCE_ID` 为按量计费资源，解关联集群后 Prometheus 实例仍会产生费用，如不再使用请务必执行实例销毁
- **数据丢失警告**：销毁 Prometheus 实例将删除所有历史监控数据且不可恢复，请提前导出 Grafana 大盘配置或关键告警规则
- **解关联影响**：解除集群关联后，该集群的监控数据停止上报，关联的告警规则将失效
- **Agent 清理**：解除关联后集群中的 `tke-monitor` 命名空间及 Agent 组件会自动清理

### 数据面（需 VPN/IOA）

解关联后验证 Agent 组件已清理（解关联操作见控制面步骤）：

```bash
kubectl get namespace tke-monitor
# expected: Error from server (NotFound): namespaces "tke-monitor" not found
```

```text
NAME  STATUS  AGE
...
```

### 控制面（tccli）

步骤 1：解除集群关联（记录 AgentId）：

```bash
AGENT_ID=$(tccli monitor DescribePrometheusAgents \
    --region <Region> \
    --InstanceId PROM_INSTANCE_ID \
    | jq -r '.Agents[0].AgentId')
echo "AGENT_ID=$AGENT_ID"
# expected: AGENT_ID=agent-xxxxxxxx

tccli monitor DeletePrometheusAgent \
    --region <Region> \
    --InstanceId PROM_INSTANCE_ID \
    --AgentId "$AGENT_ID"
# expected: exit 0，RequestId 非空
```

步骤 2：确认解关联成功：

```bash
tccli monitor DescribePrometheusAgents \
    --region <Region> \
    --InstanceId PROM_INSTANCE_ID
# expected: Agents 为空数组，TotalCount 为 0
```

步骤 3：销毁 Prometheus 实例（需二次确认，不可逆）：

```bash
tccli monitor DeletePrometheusInstance \
    --region <Region> \
    --InstanceId PROM_INSTANCE_ID
# expected: exit 0，RequestId 非空
```

步骤 4：验证已销毁：

```bash
tccli monitor DescribePrometheusInstances \
    --region <Region> \
    --InstanceIds '["PROM_INSTANCE_ID"]'
# expected: TotalCount 为 0，或 InstanceStatus 为 "Terminating"
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| CreatePrometheusAgent 关联失败 | `tccli monitor DescribePrometheusAgents --region <Region> --InstanceId PROM_INSTANCE_ID` | TMP 实例与 TKE 集群不在同一 VPC，或集群已关联其他 Prometheus 实例 | 确认 TMP 实例 `VpcId` 与 TKE 集群 VPC 一致；一个 TKE 集群只能关联一个 TMP 实例，先解关联旧的 |
| 关联后 Target 持续 "down" | `tccli monitor DescribePrometheusTargets --region <Region> --InstanceId PROM_INSTANCE_ID` | Agent 组件未成功部署或网络不通 | `kubectl get pods -n tke-monitor` 检查 Agent Pod 状态；`kubectl logs -n tke-monitor -l app=prometheus-agent` 查看启动日志；确认 TKE 安全组允许 TMP 实例网段入站 |
| 部分指标缺失 | `tccli monitor DescribePrometheusTargets --region <Region> --InstanceId PROM_INSTANCE_ID \| jq '.Targets[] \| select(.Health!="up")'` | 采集配置中未包含对应 ServiceMonitor 或采集器 | `kubectl get servicemonitor -A` 确认目标 ServiceMonitor 存在且匹配；修改 Agent 配置添加自定义采集规则 |
| 权限不足 CreatePrometheusAgent 报错 | 查看 tccli 终端输出的错误信息（含 `UnauthorizedOperation` 关键字） | CAM 子账号缺少 `monitor:CreatePrometheusAgent` 或 `tke:DescribeClusterSecurity` 权限 | 在 CAM 控制台为子账号授权策略 `QcloudMonitorFullAccess` 和 `QcloudTKEFullAccess`，或创建自定义策略包含所需 Action |

## 下一步

- [一键接入腾讯云应用性能监控 APM](../一键接入腾讯云应用性能监控%20APM/tccli%20操作.md) -- page_id `107997`
- [JVM 接入](../JVM%20接入/tccli%20操作.md) -- page_id `112791`
- [MySQL Exporter 接入](../MySQL%20Exporter%20接入/tccli%20操作.md) -- page_id `112792`

## 控制台替代

[TMP 控制台](https://console.cloud.tencent.com/monitor/prometheus) -> 选择实例 -> 关联集群 -> 一键关联 TKE。
