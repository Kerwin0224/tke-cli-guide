---
doc_type: How-to
subtype: 6B
fused: false
---
# 节点健康检查策略

> 配置 TKE 节点健康检查策略，自动发现节点故障并在允许时自动修复。
> 控制台: [容器服务 - 节点健康检测](https://console.cloud.tencent.com/tke2/node-health)

> ⚠️ 本文档所有 Action 属 **TKE 2022-05-01（官方当前版本）**，调用必须显式 `--version 2022-05-01`。2018-05-25 无此功能域。不带 `--version` 会静默走 2018-05-25 报「未知 Action」。
>
> 官方文档：[节点概述](https://cloud.tencent.com/document/product/457/32201) · [容器服务可观测体系概述](https://cloud.tencent.com/document/product/457/118975)
>
> 配额：无额外配额限制。[配额说明](https://cloud.tencent.com/document/product/457/9087)
>
> ⚠️ **高危操作**：检测规则过严致频繁告警、规则过宽致漏报、误改运行中策略致自愈行为异常。[常见高危操作](https://cloud.tencent.com/document/product/457/39539)

## 触发条件

- 需自动发现节点故障（kubelet 不健康/文件系统只读/内核 oops）并在允许时自动修复 — 创建健康检查策略
- DescribeHealthCheckPolicies 返回空或不含目标规则，需新增策略覆盖 `KubeletUnhealthy`/`RuntimeUnhealthy` 等规则
- `DescribeHealthCheckTemplate` / `CreateHealthCheckPolicy` 报 CAM 拒绝 `tke:CreateHealthCheckPolicy` — 看 [故障恢复](#故障恢复)
- 创建策略后绑定未生效（`DescribeHealthCheckPolicyBindings` 无记录）— 看 [验证](#验证) 与 [收尾确认](#收尾确认)


## 概述

节点健康检查策略（HealthCheckPolicy）按一组规则定期检测节点故障（如 kubelet 不健康、文件系统只读、内核 oops），发现故障后可按规则配置自动修复（重启 runtime / 重启 kubelet）。

健康检查是**节点级**故障自愈，区别于集群巡检（集群级配置扫描，归 Observability）。两者层级与任务不同，不要混淆。

> 集群巡检（`DescribeClusterInspectionResultsOverview` / `ListClusterInspectionResults` / `ListClusterInspectionResultsItems`，2018-05-25）是集群级配置诊断，命令见 [集群巡检与日志配置](../observability/logging.md#集群巡检)。本文档的 `HealthCheckPolicy`（2022-05-01）是节点级故障自愈策略，层级与版本均不同——混淆会调错版本报「未知 Action」。

## 决策依据

### 启用哪些规则

健康检查模板含 15 条规则，分两类（以 `DescribeHealthCheckTemplate` 返回为准）：

| 规则名 | 严重度 | 可自动修复 | 说明 |
|:-------|:------:|:--------:|:-----|
| `KubeletUnhealthy` | Low | ✅ `RestartKubelet` | kubelet healthz 调用失败 |
| `RuntimeUnhealthy` | Low | ✅ `RestartRuntime` | containerd task 列表失败 |
| `ReadonlyFilesystem` | High | ❌ None | 文件系统只读 |
| `OOMKilling` | High | ❌ None | 进程被 OOM kill |
| `TaskHung` | High | ❌ None | 任务阻塞超阈值 |
| `UnregisterNetDevice` | High | ❌ None | 网卡注销 |
| `KernelOopsDivideError` | High | ❌ None | 内核除零错误 |
| `KernelOopsNULLPointer` | High | ❌ None | 内核空指针 |
| `Ext4Error` / `Ext4Warning` | High | ❌ None | ext4 文件系统错误/告警 |
| `IOError` / `MemoryError` | High | ❌ None | IO 错误 / 内存错误 |
| `DockerHung` | High | ❌ None | Docker 任务阻塞 |
| `FDPressure` | Low | ❌ None | 文件描述符过多 |
| `KubeletRestart` | Low | ❌ None | kubelet 重启 |

**只有 `KubeletUnhealthy` 和 `RuntimeUnhealthy` 支持自动修复**（`RepairAction` 非 None）。High 严重度规则只能告警（`ShouldEnable=true, ShouldRepair=false`），不能自动修复——因为这类故障（内核 oops、只读文件系统）重启进程无法解决，需人工介入或重建节点。

> **故障→隔离→恢复路径**：High 规则告警后，人工处理路径是 `kubectl cordon` 隔离节点 → `kubectl drain` 驱逐 Pod → 删除/重建节点。完整隔离与驱逐命令见 [节点实例操作 — 节点隔离与驱逐](instance-ops.md#节点隔离与驱逐kubectl非-tccli)（kubectl，非 tccli）。

### 是否开启自动修复

`AutoRepairEnabled` 仅对可修复规则生效。开启后节点命中规则会自动重启 runtime/kubelet，可能短暂影响该节点上 Pod。生产环境建议先仅启用检测（`AutoRepairEnabled=false`），观察告警再逐步开启自动修复。

## 配置项

| 参数 | 类型 | 必填 | 说明 |
|:-----|:-----|:----:|:-----|
| `ClusterId` | String | 是 | 集群 ID，如 `cls-example` |
| `HealthCheckPolicy.Name` | String | 是 | 策略名，集群内唯一 |
| `HealthCheckPolicy.Rules[].Name` | String | 是 | 规则名，取自模板（如 `KubeletUnhealthy`） |
| `HealthCheckPolicy.Rules[].Enabled` | Boolean | 是 | 是否启用该规则检测 |
| `HealthCheckPolicy.Rules[].AutoRepairEnabled` | Boolean | 是 | 是否自动修复（仅对可修复规则生效） |

> 参数名取自 `CreateHealthCheckPolicy` 入参。

## 应用

### 步骤 1：查询可用规则模板

```bash
tccli tke DescribeHealthCheckTemplate --version 2022-05-01 --region <SUPPORTED_REGION>
# expected: exit 0, HealthCheckTemplate.Rules 含 15 条规则
```
```json
{
    "HealthCheckTemplate": {
        "Rules": [
            {
                "Name": "KubeletUnhealthy",
                "Description": "Call kubelet healthz failed",
                "RepairAction": "RestartKubelet",
                "ShouldEnable": true,
                "ShouldRepair": false,
                "Severity": "Low"
            }
        ]
    }
}
```

> 模板里即使 `RepairAction=RestartKubelet/RestartRuntime`，`ShouldRepair` 默认仍可能为 `false`（建议检测、默认不建议自动修复）。创建策略时由你在 `Rules[].AutoRepairEnabled` 显式打开自动修复；只有 `RepairAction` 非 `None` 的规则（`KubeletUnhealthy`/`RuntimeUnhealthy`）打开才有效。
> 模板在 `ap-guangzhou` / `ap-beijing` / `ap-shanghai` 等地域均可查，内容全局一致（15 条规则）。策略创建/查询用**集群所在地域**。

| 占位符 | 含义 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<SUPPORTED_REGION>` | 查询模板用地域 | 与集群同地域即可；跨地域查模板结果相同 | 如 `ap-guangzhou` |
| `<CLUSTER_ID>` | 集群 ID | 已存在的集群 | `tccli tke DescribeClusters --region <REGION>` |
| `<POLICY_NAME>` | 策略名 | 集群内唯一 | 自定义 |

### 步骤 2：创建健康检查策略

```bash
tccli tke CreateHealthCheckPolicy --version 2022-05-01 --region <REGION> \
  --ClusterId <CLUSTER_ID> \
  --HealthCheckPolicy '{"Name":"<POLICY_NAME>","Rules":[{"Name":"KubeletUnhealthy","Enabled":true,"AutoRepairEnabled":true},{"Name":"ReadonlyFilesystem","Enabled":true,"AutoRepairEnabled":false}]}'
# expected: exit 0, 返回 RequestId
```

### 步骤 3：绑定策略到 Native 节点池

公开 API 没有独立的“绑定策略” Action；绑定写入位于 `ModifyNodePool.Native.HealthCheckPolicyName`。先取得目标 Native 节点池 ID，再执行：

```bash
tccli tke ModifyNodePool --version 2022-05-01 --region <REGION> \
  --ClusterId <CLUSTER_ID> --NodePoolId <NODE_POOL_ID> \
  --Native '{"HealthCheckPolicyName":"<POLICY_NAME>"}'
# expected: exit 0, 返回 RequestId；节点池进入更新流程后恢复 Running
```

### 步骤 4：查询策略与绑定确认

```bash
tccli tke DescribeHealthCheckPolicies --version 2022-05-01 --region <REGION> --ClusterId <CLUSTER_ID>
# expected: exit 0, TotalCount >= 1, HealthCheckPolicies 含新建策略
```
```json
{
    "HealthCheckPolicies": null,
    "TotalCount": 0,
    "RequestId": "..."
}
```

> 上方是空结果示例（策略未创建时）。创建成功后 `TotalCount` 应 ≥ 1，`HealthCheckPolicies` 含策略对象。

## 验证 {#验证}

| 维度 | 命令 | 期望 |
|:-----|:-----|:-----|
| 策略存在 | `DescribeHealthCheckPolicies --ClusterId <CLUSTER_ID>` | `TotalCount >= 1`，含 `<POLICY_NAME>` |
| 规则已启用 | 同上，查策略 `Rules` | 目标规则 `Enabled=true` |
| 绑定生效 | `DescribeHealthCheckPolicyBindings --version 2022-05-01 --ClusterId <CLUSTER_ID>` | 目标策略记录的 `NodePools` 含 `<NODE_POOL_ID>` |
| 自动修复配置 | `DescribeHealthCheckPolicies` → `Rules[].AutoRepairEnabled` | 可修复规则 `AutoRepairEnabled=true`（High 规则须为 false） |

```bash
# 查询策略绑定关系（Filter 无 s，单数；按策略名过滤）
tccli tke DescribeHealthCheckPolicyBindings --version 2022-05-01 --region <REGION> \
  --ClusterId <CLUSTER_ID> \
  --Filter '[{"Name":"HealthCheckPolicyName","Values":["<POLICY_NAME>"]}]' \
  --Offset 0 --Limit 20
# expected: exit 0，返回 HealthCheckPolicyBindings[]+TotalCount
```

## 回滚

```bash
tccli tke DeleteHealthCheckPolicy --version 2022-05-01 --region <REGION> \
  --ClusterId <CLUSTER_ID> --HealthCheckPolicyName <POLICY_NAME>
# expected: exit 0
```

> `DeleteHealthCheckPolicy` 用 `HealthCheckPolicyName`（字符串）而非 ID。修改策略用 `ModifyHealthCheckPolicy`，入参与 `CreateHealthCheckPolicy` 一致。

```bash
# 修改健康检查策略：先 Describe 取得当前完整 Rules，在完整返回基础上修改后回传
tccli tke ModifyHealthCheckPolicy --version 2022-05-01 --region <REGION> \
  --ClusterId <CLUSTER_ID> \
  --HealthCheckPolicy '{"Name":"<POLICY_NAME>","Rules":[{"Name":"KubeletUnhealthy","Enabled":true,"AutoRepairEnabled":true}]}'
# expected: CAM 拦截 AuthFailure.UnauthorizedOperation；授权后 exit 0
```

> `tccli tke ModifyHealthCheckPolicy help --detail` 与 API 入参说明未声明该更新是 replace 还是 merge，因此不作“覆盖式”断言。保守流程是先用 `DescribeHealthCheckPolicies` 读取当前完整 `Rules`，只修改目标字段后回传完整策略，再次 Describe 对比修改前后差异，避免因省略规则产生不确定结果。

## 故障恢复 {#故障恢复}

| 现象 | 根因 | 修复 |
|:-----|:-----|:-----|
| `AuthFailure.UnauthorizedOperation` (tke:CreateHealthCheckPolicy) | CAM 策略要求资源带特定标签（要求 `billing` 标签） | 给集群加授权要求的标签，或申请 `tke:CreateHealthCheckPolicy` 权限 |
| 同名策略已存在 | `HealthCheckPolicy.Name` 集群内唯一 | 先 `DeleteHealthCheckPolicy` 再建，或 `ModifyHealthCheckPolicy` 覆盖 |
| 自动修复未触发 | 规则 `RepairAction=None`（如 `ReadonlyFilesystem`）不可修复 | 仅 `KubeletUnhealthy`/`RuntimeUnhealthy` 可自动修复，其余只能告警 |

> CAM 拒绝样本（ap-guangzhou，标签不匹配）：
> `code:AuthFailure.UnauthorizedOperation message:操作未授权，请检查CAM策略 ... you are not authorized to perform operation (tke:CreateHealthCheckPolicy)`

## 收尾确认 {#收尾确认}

```bash
# 端到端核对：策略绑定生效（仅查策略存在不够，还须核对绑定是否命中节点）
tccli tke DescribeHealthCheckPolicyBindings --version 2022-05-01 --region <REGION> \
  --ClusterId "<CLUSTER_ID>" \
  --Filter '[{"Name":"HealthCheckPolicyName","Values":["<POLICY_NAME>"]}]' \
  --filter "HealthCheckPolicyBindings[?Name=='<POLICY_NAME>'] | [0].{name:Name,nodePools:NodePools}"
# expected: name=<POLICY_NAME>，NodePools 含 <NODE_POOL_ID>（策略已绑定到目标 Native 节点池）

# 策略规则配置核对（Rules[].Enabled/AutoRepairEnabled 是可核对字段，非顶层 Enable）
tccli tke DescribeHealthCheckPolicies --version 2022-05-01 --region <REGION> \
  --ClusterId "<CLUSTER_ID>" \
  --filter "HealthCheckPolicies[?Name=='<POLICY_NAME>'] | [0].{name:Name,rules:Rules}"
# expected: name=<POLICY_NAME>; Rules 含目标规则，Enabled=true 且可修复规则 AutoRepairEnabled 符合预期
```

> 策略绑定生效（`DescribeHealthCheckPolicyBindings` 返回目标策略且 `NodePools` 含目标节点池）+ 规则配置正确（`Rules[].Enabled/AutoRepairEnabled`）= 节点健康检查闭环完成。入参无顶层 `Enable` 字段（只有 `Rules[].Enabled`/`AutoRepairEnabled`），确认时核对节点池绑定，不要查询不存在的 `NodeNames` 或顶层 Enable。

---

## 下一步

- 节点不健康的人工处理（驱逐/移出）：[节点实例操作](instance-ops.md)
- 集群级配置巡检（区别于节点健康检查）：[可观测概览](../observability/index.md)
- 节点池管理：[创建节点池](nodepool-create.md)
