---
doc_type: How-to
subtype: 6A
fused: false
---
# 调度策略

> 集群调度器插件配置（`SchedulerPolicy`），控制 Pod 调度行为。集群级配置，属集群属性而非节点操作。

> 官方文档：[基本概念](https://cloud.tencent.com/document/product/457/45598) · [常见高危操作](https://cloud.tencent.com/document/product/457/39539)

## 概述

调度策略（SchedulerPolicy）配置 K8s 调度器插件，决定 Pod 如何被调度到节点。通过 `DescribeClusterSchedulerPolicy` 查询、`ModifyClusterSchedulerPolicy` 修改。

> 调度策略控制的是"Pod 放到哪个节点"（资源适配/亲和性），与安全无关——故归集群配置而非集群加固。

> 配额：无额外限制。[配额说明](https://cloud.tencent.com/document/product/457/9087)

## 触发条件

- 默认调度器不满足业务需求（需资源适配/特定调度逻辑，如 `NodeResourcesFit`）— 用 `ModifyClusterSchedulerPolicy` 配自定义插件
- 需查看集群当前调度器配置（排障 Pod 调度异常）— 用 `DescribeClusterSchedulerPolicy`
- 已配自定义调度策略需恢复默认 — 跳到 [§回滚](#回滚) 设空数组

## 决策依据

| 选项 | 最佳场景 |
|:-----|:---------|
| 默认调度器（`default-scheduler`） | 通用场景，TKE 默认 |
| 自定义插件（如 `NodeResourcesFit`） | 需要资源适配/特定调度逻辑 |

## 准备工作

```bash
tccli --version
# expected: 最新版本或更高

tccli tke DescribeClusters --region <REGION> --filter "Clusters[0].ClusterId"
# expected: 集群 ID（凭证有效，见 [配置凭证](../../getting-started/credentials.md)）
```

| 占位符 | 含义 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<CLUSTER_ID>` | 集群 ID | `cls-` 开头 | `tccli tke DescribeClusters` |
| `<REGION>` | 地域 | 如 `ap-guangzhou` | `tccli tke DescribeRegions` |

## 应用

> ⚠️ **高危操作**：调度策略变更影响全集群 Pod 调度；新策略与节点资源不匹配 → 新 Pod 卡 `Pending`；修改只影响新调度，已运行 Pod 不重调度；先在测试环境验证。[常见高危操作](https://cloud.tencent.com/document/product/457/39539)

### 查询调度策略

```bash
tccli tke DescribeClusterSchedulerPolicy --ClusterId "<CLUSTER_ID>" --region <REGION>
# expected: exit 0, SchedulerPolicyConfig[] 含 SchedulerName/PluginConfigs
```
```json
{
    "Policy": "",
    "SchedulerPolicyConfig": [
        {"SchedulerName": "default-scheduler", "PluginConfigs": [{"Name": "NodeResourcesFit", "Args": "<base64>"}], "PluginSet": {"Enabled": [{"Name": "DefaultPreemption", "Weight": 0}, {"Name": "NodeResourcesFit", "Weight": 1}]}}
    ]
}
```

> 响应含 5 个顶层字段：`Policy`（调度策略 JSON 字符串）/`SchedulerPolicyConfig`（调度器配置数组）/`ClientConnection`（客户端连接配置）/`Extenders`（扩展调度器）/`HighPerformance`（高性能模式）。空集群默认 `SchedulerPolicyConfig` 含 `default-scheduler`。

### 修改调度策略

```bash
tccli tke ModifyClusterSchedulerPolicy --ClusterId "<CLUSTER_ID>" --region <REGION> \
  --SchedulerPolicyConfig '[{"SchedulerName":"default-scheduler","PluginConfigs":[{"Name":"NodeResourcesFit","Args":"<BASE64_ARGS>"}],"PluginSet":{}}]'
# expected: exit 0
```

> `--SchedulerPolicyConfig` 是对象数组，每项含 `SchedulerName` + `PluginConfigs`（插件配置）+ `PluginSet`（启用/禁用插件）。**`PluginConfigs[].Args` 必填**——缺 `Args` 报 `InvalidParameter.Param: PARAM_ERROR(pluginConfigs[0].args Unmarshal failed)`。`Args` 是 base64 编码的插件参数 JSON，先用 `DescribeClusterSchedulerPolicy` 取当前 `Args` 值再传入，勿凭空构造。完整结构见 `tccli tke ModifyClusterSchedulerPolicy help --detail`。

### 配置扩展调度器（Extenders）

接自定义调度器扩展（如 GPU/NUMA/重调度器），通过 `Extenders` 对象数组传入：

```bash
tccli tke ModifyClusterSchedulerPolicy --ClusterId "<CLUSTER_ID>" --region <REGION> \
  --Extenders '[{"ExtenderClientConfig":{"url":"http://<EXTENDER_SVC>:8080","cert":"","key":""},"ManagedResources":["*"]}]'
# expected: exit 0
```

> `Extenders[]` 每项含 `ExtenderClientConfig`（扩展器连接配置 url/cert/key）+ `ManagedResources`（该扩展器管理的资源列表）。用于接入集群外的自定义调度扩展服务。`ClientConnection`（kubeconfig 连接配置）与 `HighPerformance`（高性能调度模式）是同层独立配置。

## 验证

异步生效，检查 ≥4 个维度：

```bash
tccli tke DescribeClusterSchedulerPolicy --ClusterId "<CLUSTER_ID>" --region <REGION> \
  --filter "SchedulerPolicyConfig[0].{name:SchedulerName,plugins:PluginConfigs,set:PluginSet}"
# expected: 修改后的 SchedulerName + PluginConfigs 含目标插件 + PluginSet 启用列表
```

| 维度 | 命令 | 预期 |
|:-----|:-----|:-----|
| SchedulerName | `DescribeClusterSchedulerPolicy` → `SchedulerPolicyConfig[0].SchedulerName` | 目标调度器名（如 `default-scheduler`） |
| PluginConfigs | 同上 → `PluginConfigs[].Name`/`Args` | 含配置的插件（如 `NodeResourcesFit`）+ Args 非空 |
| PluginSet | 同上 → `PluginSet.Enabled[]` | 启用插件列表含目标插件 |
| Extenders | 同上 → `Extenders[]` | 若配了扩展调度器，含 url/ManagedResources |

## 故障恢复

| 现象 | 诊断命令 | 根因 | 修复 |
|:-----|:---------|:-----|:-----|
| `InvalidParameter.Param: PARAM_ERROR(pluginConfigs[0].args Unmarshal failed)` | 检查 `PluginConfigs[]` 是否含 `Args` 字段 | `PluginConfigs[].Args` 必填，缺 `Args` 反序列化失败 | 先 `DescribeClusterSchedulerPolicy` 取当前 `Args` 值（base64）传入，勿省略 |
| `InvalidParameter` | 检查 `--SchedulerPolicyConfig` JSON 格式 | 配置非合法 JSON 或结构错 | 用 `--generate-cli-skeleton` 对照结构 |
| `UnauthorizedOperation.CamNoAuth` | 查子账号权限 | 无 `tke:ModifyClusterSchedulerPolicy` 权限 | CAM 授权 `QcloudTKEFullAccess` |
| 集群状态不允许 | `DescribeClusterStatus` 查状态 | 集群非 Running | 等集群就绪再修改 |

## 回滚

```bash
# 恢复默认调度策略（SchedulerPolicyConfig 设为空数组）
tccli tke ModifyClusterSchedulerPolicy --ClusterId "<CLUSTER_ID>" --region <REGION> --SchedulerPolicyConfig '[]'
# expected: exit 0
```

## 收尾确认

> kubectl（K8s 原生命令，非 tccli；TCCLI 管 TKE 抽象层不提供 K8s 资源操作能力）
<!-- tccli管调度策略CRUD，kubectl get pods查K8s层Pod调度观测，非tccli边界 -->
```bash
# SchedulerName + PluginConfigs + Extenders 三项一次性核对（分项查字段存在后，再核对配置项协同生效）
tccli tke DescribeClusterSchedulerPolicy --ClusterId "<CLUSTER_ID>" --region <REGION> \
  --filter "SchedulerPolicyConfig[0].{name:SchedulerName,plugins:PluginConfigs[0].Name,extenders:Extenders[0].ExtenderClientConfig.url}"
# expected: name=目标调度器 + plugins=目标插件 + extenders=扩展器 url（若未配 Extenders 则 null）

# 端到端核对：Pod 是否按新策略调度（仅查配置字段不够，还须确认配置是否影响 Pod 调度）
kubectl get pods -A -o wide --kubeconfig <KUBECONFIG> | head -10
# expected: Pod 列表返回，NODE 列显示已调度到节点（调度策略生效后新调度的 Pod 按策略分布）
```

> SchedulerName + PluginConfigs + Extenders 三项合一 = 调度策略配置闭环完成。**能力边界**：策略修改只影响**新调度**的 Pod，已运行 Pod 不重调度；若新策略与节点资源不匹配（如 `NodeResourcesFit` 请求超节点容量），新 Pod 会卡 `Pending`——用 `kubectl describe pod <POD>` 看 `FailedScheduling` 事件诊断。

## 下一步

- [集群加固](../security/index.md) — 审计/加密/删除保护/准入控制（OPA）
- [配置集群属性与运行时](configure.md) — 其他集群级配置
