---
doc_type: How-to
subtype: 6A
fused: true
---
# Prometheus 监控入门

> 控制台: [容器服务控制台 - Prometheus 监控](https://console.cloud.tencent.com/tke2/prometheus)
> 初始化已有 Prometheus 实例、关联集群 Agent、查询采集目标。Prometheus 是 TKE 监控的核心。异步操作。
>
> 官方文档：[可观测体系概述](https://cloud.tencent.com/document/product/457/118975)
>
> 配额：Prometheus 实例规格与数据存储周期以产品购买页为准，TKE 侧无额外配额限制。[配额说明](https://cloud.tencent.com/document/product/457/9087)

## 触发条件

- 已有 `prom-` 实例尚未初始化，需用 `RunPrometheusInstance` 初始化后再接入集群
- `DescribePrometheusClusterAgents` 不含目标集群，Prometheus 实例已初始化但集群未装 Agent
- `DescribePrometheusTargets` → 目标全 down 或无数据，Agent 已关联但采集未生效 — 看 [故障恢复](#故障恢复)


## 概述

Prometheus 监控三步：取得已有实例 ID 并初始化 → 关联集群 Agent（采集） → 查询目标与指标。`RunPrometheusInstance` 不创建实例；它初始化 `prom-` 实例，实例购买/创建走控制台或 TMP 对应服务。

| 操作 | 接口 | 作用 |
|:-----|:-----|:-----|
| 初始化实例 | `RunPrometheusInstance` | 初始化已有 Prometheus 实例；不负责购买/创建实例 |
| 关联 Agent | `CreatePrometheusClusterAgent` | 集群内装 Agent，上报指标 |
| 查询目标 | `DescribePrometheusTargets` | 看采集目标状态 |
| 查询实例 | `DescribePrometheusInstancesOverview` | 看实例列表 |

> Prometheus 是 TKE 2018-05-25 旧版独有功能（2022-05-01 新版无）。本文覆盖入口 3 个核心操作，其余见 [告警配置](prometheus-alerting.md)/[配置与模板](prometheus-config.md)/[Agent 管理](prometheus-agent.md)。

## 准备工作

### 环境检查

```bash
tccli --version
# expected: tccli 版本号

tccli tke DescribeClusterStatus --region ap-guangzhou --filter "ClusterStatusSet[?ClusterId=='<CLUSTER_ID>'] | [0].ClusterState"
# expected: "Running"
```

### 资源检查

```bash
# Prometheus 权限探测（本账号未授权时返回 UnauthorizedOperation，须先 CAM 开通）
tccli tke DescribePrometheusInstancesOverview --region ap-guangzhou --Limit 3
# expected: 实例列表（空则无实例）；若 Error.Code=UnauthorizedOperation → 在 CAM 授予 Prometheus/TMP 相关策略后再继续本篇

# 子网（创建实例需指定）
tccli vpc DescribeSubnets --region <REGION> --Filters '[{"Name":"vpc-id","Values":["<VPC_ID>"]}]' \
  --filter "SubnetSet[].SubnetId" --output text
# expected: 子网 ID 列表
```

> ⚠️ **CAM 前置（两层）**：  
> 1. **用户策略**：`DescribePrometheusInstancesOverview` 等在未授权账号返回 `UnauthorizedOperation`（消息含 Operation denied by Cloud API）。仅有 TKE 集群用户权限不够；须单独开通 Prometheus/TMP 相关用户 CAM 策略。  
> 2. **服务相关角色**：官方创建监控实例前需授权 **`TKE_QCSLinkedRoleInPrometheusService`**（名称以控制台/当前账号为准），见下节。  
> 未授权时本篇后续 Create/Run 同样会被拒，勿将 CAM 拒绝误判为参数错误。

### 服务相关角色

> 官方 [创建监控实例](https://cloud.tencent.com/document/product/457/71897)：**服务授权** 创建 LinkedRole，授权 Prometheus 访问相关云产品。与用户 `QcloudTKEFullAccess` 不是同一层。

```bash
# 探测
tccli cam DescribeRoleList --Page 1 --Rp 100 \
  --filter "List[?contains(RoleName,'Prometheus')].RoleName" --output text
# expected: 含 TKE_QCSLinkedRoleInPrometheusService 或官方当前 Linked 角色名；空 → 补齐

# 服务相关角色创建（QCSServiceName 须与角色载体文档一致；不确定时用控制台 Prometheus 首次「服务授权」）
tccli cam CreateServiceLinkedRole help --detail
# expected: 入参含 QCSServiceName[]

# 控制台主路径：容器服务 → 云原生服务 → Prometheus → 新建 → 按弹窗同意授权
# CLI：tccli cam CreateServiceLinkedRole --QCSServiceName '["<QCS_SERVICE_NAME>"]'
# expected: 角色出现在 DescribeRoleList；载体名见 https://cloud.tencent.com/document/product/598/85165
```

| 项 | 说明 |
|:---|:-----|
| 用户 CAM vs LinkedRole | 用户策略决定**你**能否调 API；LinkedRole 决定**监控服务**能否访问云资源 |
| 总表 | [配置凭证 — 服务角色](../../getting-started/credentials.md#服务角色tke--ipamd--as--可观测) |

## 关键字段

> 完整入参以 `tccli tke RunPrometheusInstance` / `CreatePrometheusClusterAgent` / `DescribePrometheusTargets` 的 `help --detail` 为准。

### RunPrometheusInstance

| 字段 | 类型 | 必填 | 约束 | 填错时的错误 |
|:------|------|:--------:|------------|---------------|
| InstanceId | string | 是 | 已有 Prometheus 实例 ID（`prom-` 开头），不是自定义实例名 | `InvalidParameterValue` |
| SubnetId | string | 否 | 初始化时可改用的新子网；省略则使用实例原子网 | `ResourceNotFound.SubnetId` |

> `RunPrometheusInstance` 仅初始化已有实例，返回 `RequestId`，不返回新实例 ID。先从控制台或 `DescribePrometheusInstancesOverview` 取得 `prom-` 实例 ID。

### CreatePrometheusClusterAgent

| 字段 | 类型 | 必填 | 约束 | 填错时的错误 |
|:------|------|:--------:|------------|---------------|
| InstanceId | string | 是 | Prometheus 实例 ID | `ResourceNotFound` |
| Agents | list | 是 | 每项必填 `Region`、`ClusterType`、`ClusterId`、`EnableExternal` | `InvalidParameterValue` |

### DescribePrometheusTargets

| 字段 | 类型 | 必填 | 约束 | 填错时的错误 |
|:------|------|:--------:|------------|---------------|
| InstanceId | string | 是 | Prometheus 实例 ID | `ResourceNotFound` |
| ClusterType | string | 是 | 集群类型 | — |
| ClusterId | string | 是 | 集群 ID | `ResourceNotFound` |
| Filters | list | 否 | 过滤器 | `InvalidParameterValue` |

## 操作步骤

### 步骤 1：决策 — 实例规格

#### 为什么用独立 Prometheus 实例

- **独立实例（推荐）**: 独立 Prometheus 服务，数据隔离，可长期存储
- **集群内置**: 集群内临时 Prometheus，数据随集群销毁
- **默认推荐**: 生产用独立实例；测试可用集群内置
- **可更换**： 独立实例创建后可改规格，但不能转为集群内置

### 步骤 2：初始化已有实例

先从控制台或实例概览取得 `prom-` 实例 ID；若账号尚无实例，先在 Prometheus/TMP 产品入口创建，不能用本 Action 代替购买。

```bash
tccli tke RunPrometheusInstance --version 2018-05-25 --region ap-guangzhou \
  --InstanceId "<INSTANCE_ID>" --SubnetId "<SUBNET_ID>"
# expected: exit 0，返回 RequestId；再用 DescribePrometheusInstanceInitStatus 轮询初始化状态
```

| 占位符 | 含义 | 约束 | 如何获取 |
|:------------|:-----|:-----|:---------|
| `<INSTANCE_ID>` | Prometheus 实例 ID | `prom-` 开头的已有实例 | `DescribePrometheusInstancesOverview` 或控制台 |
| `<SUBNET_ID>` | 初始化使用的 VPC 子网 ID | 可省略；传入时须存在 | `tccli vpc DescribeSubnets` |

### 步骤 3：关联集群 Agent

```bash
tccli tke CreatePrometheusClusterAgent --version 2018-05-25 --region ap-guangzhou \
  --InstanceId "<INSTANCE_ID>" \
  --Agents '[{"Region":"ap-guangzhou","ClusterType":"tke","ClusterId":"<CLUSTER_ID>","EnableExternal":false}]'
# expected: exit 0，返回 RequestId；再用 DescribePrometheusClusterAgents 确认关联
```

> `Agents` 是数组，可一次关联多个集群。每项的 `Region`、`ClusterType`、`ClusterId`、`EnableExternal` 均必填；`ClusterType` 必须使用当前接口支持的集群类型值，执行前以 `CreatePrometheusClusterAgent help --detail` 和目标集群类型为准。

### 步骤 4：查询采集目标

```bash
tccli tke DescribePrometheusTargets --version 2018-05-25 --region ap-guangzhou \
  --InstanceId "<INSTANCE_ID>" --ClusterId "<CLUSTER_ID>"
# expected: 采集目标列表，含 up 状态
```

### 步骤 5：验证

```bash
# 查看实例状态
tccli tke DescribePrometheusInstancesOverview --region ap-guangzhou \
  --filter "Instances[].{id:InstanceId,name:InstanceName,state:InstanceStatus}"
# expected: 实例状态为运行中
```

| 维度 | 命令 | 预期 |
|:-----|:-----|:-----|
| 实例运行 | `DescribePrometheusInstancesOverview` → `InstanceStatus` | 运行中 |
| Agent 就绪 | `DescribePrometheusClusterAgents`（见 [Agent 管理](prometheus-agent.md)） | Agent Ready |
| 采集目标 up | `DescribePrometheusTargets` | 目标 `up` 状态为 1 |
| 指标可查 | `DescribePrometheusRecordRules`（见 [配置](prometheus-config.md)） | 指标返回 |

## 清理

> **副作用警告**：删除 Prometheus 实例会销毁所有监控数据。Agent 会从集群卸载。
>
> ⚠️ **高危操作**：删除 Prometheus 实例不可逆，所有历史监控数据将永久丢失且无法恢复。[常见高危操作](https://cloud.tencent.com/document/product/457/39539)

```bash
# 1. 卸载 Agent
tccli tke DeletePrometheusClusterAgent --version 2018-05-25 --region ap-guangzhou \
  --InstanceId "<INSTANCE_ID>" --Agents '[{"ClusterId":"<CLUSTER_ID>"}]'
# expected: exit 0

# 2. 删除实例（实例删除属于 monitor 服务，Actions 为 DestroyPrometheusInstance；tke 的 DeletePrometheus* 仅覆盖 Agent/Config/Rule/Template）
tccli monitor DestroyPrometheusInstance --version 2018-07-24 --region ap-guangzhou \
  --InstanceId "<INSTANCE_ID>"
# expected: exit 0
```

> 实例删除 Action 为 `monitor DestroyPrometheusInstance`（tke 无实例级删除 Action）。按量计费实例销毁后停止计费；包年包月实例销毁按退费规则处理。

## 故障恢复

### 命令返回错误 (exit ≠ 0)

| 现象 | 诊断 | 根因 | 修复 |
|:--------|:----------|:------------|:-----|
| `InvalidParameterValue.InstanceId` | 查 `DescribePrometheusInstancesOverview` 是否存在该 `prom-` ID | 把自定义名称当成实例 ID，或实例不存在 | 改用已有 `prom-` 实例 ID；无实例先从产品入口创建 |
| `ResourceNotFound.SubnetId` | `tccli vpc DescribeSubnets` | 子网不存在 | 用存在子网 |
| `ResourceNotFound` (InstanceId) | `DescribePrometheusInstancesOverview` | Prometheus 实例不存在 | 到 Prometheus/TMP 产品入口创建实例并取得新的 `prom-` ID；`RunPrometheusInstance` 只初始化已有实例 |
| `FailedOperation` | `DescribeClusterStatus` 查看集群状态 | 集群非 Running | 等集群 Running |
| `UnknownAction` | 检查 `--version` | 未带 `--version 2018-05-25` 误走新版 | 显式带 `--version 2018-05-25` |

### 命令成功但状态不对 (exit = 0)

| 现象 | 诊断 | 根因 | 修复 |
|:--------|:----------|:------------|:-----|
| 实例长时间 Creating | `DescribePrometheusInstancesOverview` | 子网资源不足 | 换子网或提工单 |
| Agent 关联但目标全 down | `DescribePrometheusTargets` | Agent 未就绪或网络不通 | 查 Agent Pod 日志，确认安全组 |
| 无指标数据 | `DescribePrometheusRecordRules` | 采集配置未生效 | 见 [配置与模板](prometheus-config.md) |

## 实例与模板同步查询

> Prometheus 实例详情、初始化状态、模板同步状态查询。

```bash
# 查询实例详情 (按 InstanceId)
tccli tke DescribePrometheusInstance --InstanceId "<PROM_INSTANCE_ID>" --region <REGION>
# expected: exit 0, 实例配置详情

# 查询实例初始化状态 (创建后轮询用)
tccli tke DescribePrometheusInstanceInitStatus --InstanceId "<PROM_INSTANCE_ID>" --region <REGION>
# expected: exit 0, 初始化阶段与进度

# 查询实例概览列表 (支持 Filters 过滤)
tccli tke DescribePrometheusOverviews --Limit 10 --region <REGION>
# expected: exit 0, 实例概览列表
```

> ⚠️ Prometheus 接口需单独授权。`DescribePrometheusOverviews` 未授权时返回 `UnauthorizedOperation: 您未授权访问该接口。请求由云API拦截`。需在 CAM 开通 Prometheus 相关权限。

```bash
# 查询模板同步状态 (按 TemplateId)
tccli tke DescribePrometheusTempSync --TemplateId "<TEMPLATE_ID>" --region <REGION>
# expected: exit 0, 模板同步目标列表

tccli tke DescribePrometheusTemplateSync --TemplateId "<TEMPLATE_ID>" --region <REGION>
# expected: exit 0, 模板同步详情

# 删除模板同步 (TemplateId + Targets[] 定位)
tccli tke DeletePrometheusTempSync --TemplateId "<TEMPLATE_ID>" --region <REGION> \
  --Targets '[{"Region":"<REGION>","InstanceId":"<PROM_INSTANCE_ID>"}]'
# expected: exit 0

tccli tke DeletePrometheusTemplateSync --TemplateId "<TEMPLATE_ID>" --region <REGION> \
  --Targets '[{"Region":"<REGION>","InstanceId":"<PROM_INSTANCE_ID>"}]'
# expected: exit 0
```

> `DescribePrometheusTempSync`/`TemplateSync` 用 `TemplateId` 查同步状态；`DeletePrometheusTempSync`/`TemplateSync` 用 `TemplateId`+`Targets[]`（Region+InstanceId）定位删除目标。模板同步配置见 [配置与模板](prometheus-config.md)。

## 收尾确认

实例、Agent、采集目标与指标查询分别按上文“步骤 5：验证”执行；不要用实例存在性代替整条采集链路的终态证据。仅需确认实例本身存在时，可运行：

```bash
tccli tke DescribePrometheusInstance --region <REGION> --InstanceId <INSTANCE_ID> \
  --filter "{name:Name,id:InstanceId}"
# expected: id 与目标实例一致，name 非空；此结果只确认实例存在，不证明 Agent、Target 或指标链路正常
```

---

## 下一步

- [Prometheus 告警配置](prometheus-alerting.md) — 告警策略与规则
- [Prometheus 配置与模板](prometheus-config.md) — 模板同步与记录规则
- [Prometheus Agent 管理](prometheus-agent.md) — Agent 安装与采集目标
- [可观测性概览](index.md) — 监控全景
- [故障排查](../troubleshooting.md) — 监控无数据诊断
