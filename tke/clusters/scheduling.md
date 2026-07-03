---
doc_type: How-to
subtype: 6A
fused: false
---
# 调度策略

> 集群调度器插件配置（`SchedulerPolicy`），控制 Pod 调度行为。集群级配置，属集群属性而非节点操作。

## 概述

调度策略（SchedulerPolicy）配置 K8s 调度器插件，决定 Pod 如何被调度到节点。通过 `DescribeClusterSchedulerPolicy` 查询、`ModifyClusterSchedulerPolicy` 修改。

> 调度策略控制的是"Pod 放到哪个节点"（资源适配/亲和性），与安全无关——故归集群配置而非集群加固。

## 触发条件

- 默认调度器不满足业务需求（需资源适配/特定调度逻辑，如 `NodeResourcesFit`）— 用 `ModifyClusterSchedulerPolicy` 配自定义插件
- 需查看集群当前调度器配置（排障 Pod 调度异常）— 用 `DescribeClusterSchedulerPolicy`
- 已配自定义调度策略需恢复默认 — 跳到 [§回滚](#回滚) 设空数组

## 决策依据

| 选项 | 最佳场景 |
|:-----|:---------|
| 默认调度器（`default-scheduler`） | 大多数场景，TKE 默认 |
| 自定义插件（如 `NodeResourcesFit`） | 需要资源适配/特定调度逻辑 |

## 准备工作

```bash
tccli --version
# expected: tccli 3.1.117.1 或更高

tccli tke DescribeClusters --region <REGION> --filter "Clusters[0].ClusterId"
# expected: 集群 ID（凭证有效，见 [配置凭证](../../getting-started/credentials.md)）
```

| 占位符 | 含义 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<CLUSTER_ID>` | 集群 ID | `cls-` 开头 | `tccli tke DescribeClusters` |
| `<REGION>` | 地域 | 如 `ap-guangzhou` | `tccli tke DescribeRegions` |

## 应用

### 查询调度策略

```bash
tccli tke DescribeClusterSchedulerPolicy --ClusterId "<CLUSTER_ID>" --region <REGION>
# expected: exit 0, SchedulerPolicyConfig[] 含 SchedulerName/PluginConfigs
```
```json
{
    "Policy": "",
    "SchedulerPolicyConfig": [
        {"SchedulerName": "default-scheduler", "PluginConfigs": [{"Name": "NodeResourcesFit"}], "PluginSet": {}}
    ],
    "ClientConnection": {},
    "Extenders": [],
    "HighPerformance": false
}
```

> 响应含 5 个顶层字段：`Policy`（调度策略 JSON 字符串）/`SchedulerPolicyConfig`（调度器配置数组）/`ClientConnection`（客户端连接配置）/`Extenders`（扩展调度器）/`HighPerformance`（高性能模式）。空集群默认 `SchedulerPolicyConfig` 含 `default-scheduler`。

### 修改调度策略

```bash
tccli tke ModifyClusterSchedulerPolicy --ClusterId "<CLUSTER_ID>" --region <REGION> \
  --SchedulerPolicyConfig '[{"SchedulerName":"default-scheduler","PluginConfigs":[{"Name":"NodeResourcesFit","Args":"<BASE64_ARGS>"}],"PluginSet":{}}]'
# expected: exit 0
```

> `--SchedulerPolicyConfig` 是对象数组，每项含 `SchedulerName` + `PluginConfigs`（插件配置）+ `PluginSet`（启用/禁用插件）。**`PluginConfigs[].Args` 必填**——缺 `Args` 报 `InvalidParameter.Param: PARAM_ERROR(pluginConfigs[0].args Unmarshal failed)`。`Args` 是 base64 编码的插件参数 JSON，先用 `DescribeClusterSchedulerPolicy` 取当前 `Args` 值再传入，勿凭空构造。完整结构见 `tccli tke ModifyClusterSchedulerPolicy --generate-cli-skeleton`。

## 验证

```bash
tccli tke DescribeClusterSchedulerPolicy --ClusterId "<CLUSTER_ID>" --region <REGION> \
  --filter "SchedulerPolicyConfig[0].SchedulerName"
# expected: 修改后的 SchedulerName
```

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

```bash
# 一次性核对：调度策略已生效，目标 SchedulerName 出现在配置中
tccli tke DescribeClusterSchedulerPolicy --ClusterId "<CLUSTER_ID>" --region <REGION> \
  --filter "SchedulerPolicyConfig[0].SchedulerName" --output text
# expected: 返回目标 SchedulerName（如 default-scheduler）→ 调度策略配置闭环完成
```

## 下一步

- [集群加固](../security/index.md) — 审计/加密/删除保护/准入控制（OPA）
- [配置集群属性与运行时](configure.md) — 其他集群级配置
