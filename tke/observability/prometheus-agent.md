---
doc_type: How-to
subtype: 6B
fused: false
---
# 配置 Prometheus 集群采集 Agent

> 在 TKE Prometheus 实例上为集群安装采集 Agent，上报集群内指标；管理 Agent 外部标签与采集目标状态。
> 控制台: [容器服务 - 监控](https://console.cloud.tencent.com/tke2/monitor)
>
> 官方文档：[可观测体系概述](https://cloud.tencent.com/document/product/457/118975)
>
> 配额：单个 Prometheus 实例关联 Agent 集群数无额外配额限制。[配额说明](https://cloud.tencent.com/document/product/457/9087)

> ⚠️ 本页所有 Action 属 **TKE 2018-05-25（默认版本）**。

## 触发条件

- `DescribePrometheusClusterAgents` 不含目标 `ClusterId`，需为本集群安装 Agent 上报指标
- `DescribePrometheusTargets` → 采集目标 `lastError` 非空，Agent 已装但采集失败
- 多集群指标无法区分来源，`DescribePrometheusClusterAgents` → `ExternalLabels` 缺 `cluster` 标签 — 看 [故障恢复](#故障恢复)


## 概述

Prometheus 集群采集 Agent（ClusterAgent）部署在被监控集群内，抓取集群指标（kubelet、kube-state-metrics、节点/容器指标等）并远端写到 TKE Prometheus 实例。一个 Prometheus 实例可对接多个集群的 Agent。

Agent 与 Prometheus 实例的关系：实例是存储与查询后端，Agent 是各集群内的采集器。先有实例（`RunPrometheusInstance` 创建，见 [Prometheus 入门](prometheus.md)），再为本集群装 Agent。

外部标签（ExternalLabels）是 Agent 上报时附加在所有指标上的静态标签，用于在多集群实例中区分指标来源集群。

## 决策依据

### Agent 部署形态

| 选项 | 适用 | 代价 |
|:-----|:-----|:-----|
| `EnableExternal: false`（默认） | Agent 仅在集群内采集，通过 VPC 内网上报 | 需实例与集群网络互通 |
| `EnableExternal: true` | Agent 通过公网端点上报 | 公网带宽与暴露面 |

### 是否启用基础采集

- `NotInstallBasicScrape: false`（默认）：安装 TKE 预置的基础采集规则（kubelet/node 等），安装后即可使用基础指标。
- `NotInstallBasicScrape: true`：不装基础规则，仅采自定义目标，需自配 scrape config。

不确定时保持默认（`false`），先有基础指标再按需裁剪。

## 配置项

> 注意入参混用 `InstanceId`（实例）与 `ClusterId`（集群，小写 d）——契约不同，不可类推。

| 字段 | 类型 | 必填 | 默认值 | 有效值 | 填错的影响 |
|:-----|:-----|:----:|:-------|:-------|:-----------|
| `InstanceId` | String | 是 | — | Prometheus 实例 ID | 实例不存在被拒 |
| `Agents[].Region` | String | 是 | — | 集群地域，如 `ap-guangzhou` | 与集群不符被拒 |
| `Agents[].ClusterType` | String | 是 | — | `tke` / `eks`（非 CreateCluster 的 MANAGED/INDEPENDENT） | 类型错被拒 |
| `Agents[].ClusterId` | String | 是 | — | 已存在的集群 ID | 集群不存在被拒 |
| `Agents[].EnableExternal` | Boolean | 否 | false | true/false | 公网/内网上报选错 |
| `Agents[].NotInstallBasicScrape` | Boolean | 否 | false | true/false | 缺失基础指标 |
| `Agents[].NotScrape` | Boolean | 否 | false | true/false | Agent 不采集 |
| `Agents[].ExternalLabels[].Name` | String | 否 | — | 标签名 | 标签不上报 |
| `Agents[].ExternalLabels[].Value` | String | 否 | — | 标签值 | 标签不上报 |
| `Agents[].InClusterPodConfig` | Object | 否 | — | HostNet/NodeSelector/Tolerations | Agent 调度不符预期 |

## 应用

### 步骤 1：安装 Agent

```bash
tccli tke CreatePrometheusClusterAgent --region <REGION> \
  --InstanceId <PROM_INSTANCE_ID> \
  --Agents '[{"Region":"<REGION>","ClusterType":"tke","ClusterId":"<CLUSTER_ID>","EnableExternal":false}]'
# expected: exit 0, 返回 RequestId
```

| 占位符 | 含义 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<PROM_INSTANCE_ID>` | Prometheus 实例 ID | 已存在实例 | `tccli tke DescribePrometheusInstancesOverview --region <REGION>` |
| `<REGION>` | 地域 | 如 `ap-guangzhou` | `tccli tke DescribeRegions` |
| `<CLUSTER_ID>` | 集群 ID | 已存在集群 | `tccli tke DescribeClusters --region <REGION>` |

> ⚠️ **写操作 CAM 授权**：`CreatePrometheusClusterAgent` 需 `tke:CreatePrometheusClusterAgent` 权限，未授权返回 `AuthFailure.UnauthorizedOperation`。授权后调用成功。

### 步骤 2：配置外部标签

```bash
tccli tke ModifyPrometheusAgentExternalLabels --region <REGION> \
  --InstanceId <PROM_INSTANCE_ID> --ClusterId <CLUSTER_ID> \
  --ExternalLabels '[{"Name":"cluster","Value":"prod-gz"},{"Name":"env","Value":"prod"}]'
# expected: exit 0, 返回 RequestId
```

外部标签建议至少含 `cluster` 标签区分来源集群，多集群统一查询时按此过滤。

### 步骤 3：删除 Agent

```bash
tccli tke DeletePrometheusClusterAgent --region <REGION> \
  --InstanceId <PROM_INSTANCE_ID> \
  --Agents '[{"ClusterType":"tke","ClusterId":"<CLUSTER_ID>","Region":"<REGION>"}]' \
  --Force false
# expected: exit 0, 返回 RequestId
```

`Force: true` 强制删除（即使 Agent 状态异常）；`false` 则在 Agent 正常时才删。删除 Agent 不影响 Prometheus 实例，仅停止该集群指标上报。

## 验证

```bash
tccli tke DescribePrometheusClusterAgents --region <REGION> --InstanceId <PROM_INSTANCE_ID>
# expected: exit 0, 返回 Agent 列表，含目标 ClusterId
```
```json
{
    "Agents": [
        {
            "ClusterId": "cls-example",
            "ClusterType": "tke",
            "Region": "ap-guangzhou",
            "EnableExternal": false,
            "ExternalLabels": [{"Name": "cluster", "Value": "prod-gz"}]
        }
    ],
    "RequestId": "..."
}
```

```bash
tccli tke DescribePrometheusTargets --region <REGION> \
  --InstanceId <PROM_INSTANCE_ID> --ClusterType tke --ClusterId <CLUSTER_ID>
# expected: exit 0, 返回采集目标列表，所有 target lastError 为空
```

| 维度 | 命令 | 期望 |
|:-----|:-----|:-----|
| Agent 已安装 | `DescribePrometheusClusterAgents --InstanceId <PROM_INSTANCE_ID>` | 列表含目标 `ClusterId` |
| 外部标签生效 | 同上，查 `ExternalLabels` | 含配置的标签 |
| 采集目标 up | `DescribePrometheusTargets --InstanceId <PROM_INSTANCE_ID> --ClusterId <CLUSTER_ID>` | 所有 target `lastError` 为空 |
| 实例关联 | `DescribePrometheusAgentInstances --ClusterId <CLUSTER_ID>` | 返回该集群关联的实例 |

```bash
# 按集群查关联的 Prometheus 实例（ClusterId 定位，反向查「这集群被哪个实例采」）
tccli tke DescribePrometheusAgentInstances --region <REGION> --ClusterId <CLUSTER_ID>
# expected: CAM 拦截 UnauthorizedOperation（云API网关层）；授权后返回 Instances[]

# 按实例查 Agent 列表（InstanceId + 分页，区别于 DescribePrometheusClusterAgents）
tccli tke DescribePrometheusAgents --region <REGION> \
  --InstanceId <PROM_INSTANCE_ID> --Offset 0 --Limit 20
# expected: CAM 拦截 UnauthorizedOperation（云API网关层）；授权后返回 Agents[]+Total
```

> `DescribePrometheusAgentInstances`（按集群反向查实例，入参 `ClusterId`）与 `DescribePrometheusAgents`（按实例正向查 Agent，入参 `InstanceId`+分页）方向相反。`DescribePrometheusClusterAgents`（步骤 1 验证用）同向但返回结构略不同。三者按「已知什么查什么」选用。

## 回滚

```bash
tccli tke DeletePrometheusClusterAgent --region <REGION> \
  --InstanceId <PROM_INSTANCE_ID> \
  --Agents '[{"ClusterType":"tke","ClusterId":"<CLUSTER_ID>","Region":"<REGION>"}]' --Force true
# expected: exit 0
```

```bash
# 验证已删除
tccli tke DescribePrometheusClusterAgents --region <REGION> --InstanceId <PROM_INSTANCE_ID>
# expected: 目标 ClusterId 不再出现在 Agent 列表
```

> 仅移除 Agent，Prometheus 实例与已存储的历史指标不受影响。
>
> ⚠️ **高危操作**：误删 Agent 将导致该集群监控数据中断，告警规则可能因此漏报生产故障。[常见高危操作](https://cloud.tencent.com/document/product/457/39539)

## 故障恢复

### 命令返回错误（exit ≠ 0）

| 现象 | 诊断 | 根因 | 修复 |
|:-----|:-----|:-----|:-----|
| `AuthFailure.UnauthorizedOperation` (tke:CreatePrometheusClusterAgent) | 查账号 CAM 策略 | 写 Prometheus Agent 需 `tke:CreatePrometheusClusterAgent` 权限 | 申请该 Action 权限。此为环境限制 |
| `UnauthorizedOperation` 您未授权访问该接口 (DescribePrometheus*) | 同上 | 读 Prometheus 域同样需 CAM 授权，与写操作错误码不同（读=云API拦截层，写=CAM策略层） | 申请 `tke:DescribePrometheus*` 读权限 |
| `Unknown options` | `tccli tke <Action> --generate-cli-skeleton` | 参数名拼错（`ClusterId` 小写 d vs `InstanceId`） | 用 `--generate-cli-skeleton` 输出的真实参数名 |

> ⚠️ **错误码分层**：Prometheus 域写操作（Create/Modify/Delete）返回 `AuthFailure.UnauthorizedOperation`（CAM 策略层，含 `tke:ActionName`），读操作（Describe）返回 `UnauthorizedOperation: 您未授权访问该接口。请求由云API拦截`（云 API 网关层）。两者根因同为缺 CAM 授权，但报错层级不同，排查时注意区分。

### 命令成功但状态不对（exit = 0）

| 现象 | 诊断 | 根因 | 修复 |
|:-----|:-----|:-----|:-----|
| Agent 已装但无指标 | `DescribePrometheusTargets --ClusterId <CLUSTER_ID>` | `NotScrape: true` 或网络不通 | 改 `NotScrape: false`，检查实例与集群 VPC 互通 |
| 采集目标 lastError 非空 | `DescribePrometheusTargets` | 目标 Service/ServiceMonitor 不存在或端口不符 | 修正 scrape 目标配置 |
| 多集群指标无法区分 | `DescribePrometheusClusterAgents` 查 `ExternalLabels` | 未配 `cluster` 外部标签 | `ModifyPrometheusAgentExternalLabels` 补标签 |

## 收尾确认

```bash
# Agent 已接入集群
tccli tke DescribePrometheusClusterAgents --region <REGION> --InstanceId <PROM_INSTANCE_ID> \
  --filter "Agents[].{cluster:ClusterId,region:Region,status:Status}"
# expected: 目标集群 Status=Running

# 检查全部采集目标：先确认非空，再确认没有 Error 非空的目标
tccli tke DescribePrometheusTargets --region <REGION> \
  --InstanceId <PROM_INSTANCE_ID> --ClusterType tke --ClusterId <CLUSTER_ID> \
  --filter "{total:sum(Jobs[].Total),failed:Jobs[].Targets[?Error!=''].{url:Url,state:State,error:Error}[]}"
# expected: total > 0 且 failed=[]；不得截断抽样
```

> `DescribePrometheusTargets` 能证明目标非空且抓取无报错，但不能证明时序样本已写入。`DescribePrometheusRecordRules` 只返回规则配置，也不能作为指标入库证据。指标非空、新鲜时间戳与目标集群标签必须通过实例 `QueryAddress` 对应的 Prometheus 查询入口或控制台执行 PromQL 核验；未完成该查询时，只能确认“Agent 已运行且 Targets 无抓取错误”，不能宣称指标端到端入库。

---

## 下一步

- [Prometheus 入门](prometheus.md) — 创建 Prometheus 实例（装 Agent 的前置）
- [Prometheus 告警配置](prometheus-alerting.md) — 在指标基础上配告警
- [Prometheus 配置与模板](prometheus-config.md) — 管理采集配置与记录规则
