# 调整节点池（tccli）

> 对照官方：[调整节点池](https://cloud.tencent.com/document/product/457/43737) · page_id `43737`

## 概述

调整节点池的**全局配置**（名称、Label、Taint）、**期望节点数**（扩缩容）、**弹性伸缩开关**和**缩容保护**。涉及控制面 API：`ModifyClusterNodePool`、`ModifyNodePoolDesiredCapacityAboutAsg`、`SetNodePoolNodeProtection`。

## 前置条件

- [环境准备](../../../../环境准备.md)：`tccli` 已配置。
- 节点池 `LifeState` 为 `normal`。
- 修改节点池 Label/Taint 会触发**全池节点滚动更新**，请确认业务可接受。

## 控制台与 CLI 参数映射

| 控制台操作 | tccli API | 幂等 |
|------------|-----------|------|
| 修改节点池名称 / 全局配置 | `ModifyClusterNodePool` | 否 |
| 调整期望节点数（手动扩缩容） | `ModifyNodePoolDesiredCapacityAboutAsg` | 否 |
| 启用/停用弹性伸缩 | `ModifyClusterNodePool`（`EnableAutoscale` 字段） | 否 |
| 设置节点缩容保护 | `SetNodePoolNodeProtection` | 否 |
| 修改池内支持的机型 | `ModifyNodePoolInstanceTypes` | 否 |
| 查看当前配置 | `DescribeClusterNodePoolDetail` | 是 |

## 操作步骤

---

### 调整节点池全局配置

修改名称、Label、Taint、弹性伸缩开关、期望节点数范围等：

```bash
tccli tke ModifyClusterNodePool --generate-cli-skeleton --output json > ModifyClusterNodePool.json
```

编辑 `ModifyClusterNodePool.json`，设置目标字段后提交。

**示例：修改节点池名称并添加 Label**

```json
{
  "ClusterId": "<ClusterId>",
  "NodePoolId": "<NodePoolId>",
  "Name": "kerwinwjyan-rewrite-np",
  "Labels": [
    { "Name": "pool-name", "Value": "kerwinwjyan-test" },
    { "Name": "env", "Value": "rewrite" }
  ]
}
```

```bash
tccli tke ModifyClusterNodePool --cli-input-json file://ModifyClusterNodePool.json --region ap-guangzhou --output json
```

```json
{ "RequestId": "7f7fe9d7-85a1-4bb9-80b3-cce0464289b9" }
```

# expected: exit 0, contains "RequestId"

> ⚠️ **决策依据**：Label/Taint 修改会触发全池节点滚动重建。添加 `env=rewrite` 标签作为示例——操作前需提前评估影响范围，预留足够滚动更新时间。参见 [清理警告](#清理)。

**示例：启用弹性伸缩并扩大容量范围**

```json
{
  "ClusterId": "<ClusterId>",
  "NodePoolId": "<NodePoolId>",
  "EnableAutoscale": true,
  "AutoScalingGroupPara": "{\"MaxSize\":10,\"MinSize\":2}"
}
```

> **操作顺序**：先扩大容量上限（`MaxNodesNum`），再调整期望节点数（`DesiredCapacity`）。`DesiredCapacity` 受 `[MinNodesNum, MaxNodesNum]` 约束，必须先扩大容量区间。

---

### 调整节点池期望节点数

手动增减池内节点数量（不改变伸缩范围上下限）：

```bash
tccli tke ModifyNodePoolDesiredCapacityAboutAsg --generate-cli-skeleton --output json > ModifyDesiredCapacity.json
```

编辑 JSON：

```json
{
  "ClusterId": "<ClusterId>",
  "NodePoolId": "<NodePoolId>",
  "DesiredCapacity": 3
}
```

```bash
tccli tke ModifyNodePoolDesiredCapacityAboutAsg --cli-input-json file://ModifyDesiredCapacity.json --region ap-guangzhou --output json
```

```json
{ "RequestId": "176df3db-ad78-4455-bbac-b56eca458824" }
```

> ⚠️ `DesiredCapacity` 必须在 `[MinNodesNum, MaxNodesNum]` 范围内。若设置值超出范围，将返回 `InvalidParameterValue.Size`。请先通过 `ModifyClusterNodePool` 同时传入 `--MaxNodesNum` 和 `--MinNodesNum`，使期望值落入区间内，再重试。参见 [排障](#排障)。

---

### 修改节点池支持的机型

为节点池增加或替换可用机型（不影响已创建的节点）。

> **操作顺序**：先通过 AS API 查询节点池真实机型，再调用 `ModifyNodePoolInstanceTypes`。`DescribeClusterNodePoolDetail` 不返回 `InstanceType`，必须通过 AS 的 `DescribeLaunchConfigurations` 查询。

**步骤 1：查询当前机型**

```bash
tccli as DescribeLaunchConfigurations --LaunchConfigurationIds '["<LaunchConfigurationId>"]' --region <Region>
# expected: 返回 LaunchConfiguration 中已有的 InstanceTypes
```

> 获取 `LaunchConfigurationId`：先通过 `tccli tke DescribeClusterNodePoolDetail` 获取节点池关联的 ASG，再通过 `tccli as DescribeAutoScalingGroups` 获取 LaunchConfigurationId。

**步骤 2：修改机型**

```bash
tccli tke ModifyNodePoolInstanceTypes --generate-cli-skeleton --output json > ModifyInstanceTypes.json
```

编辑 JSON：

```json
{
  "ClusterId": "<ClusterId>",
  "NodePoolId": "<NodePoolId>",
  "InstanceTypes": ["M5.SMALL8"]
}
```

```bash
tccli tke ModifyNodePoolInstanceTypes --cli-input-json file://ModifyInstanceTypes.json --region ap-guangzhou --output json
```

```json
{ "RequestId": "ba0e3e08-4630-48dc-ab68-e6e30caabf4c" }
```

> ⚠️ **不能跨大机型族修改**（如 S5 → M5）。`InstanceTypes` 中的值必须来自当前 AS LaunchConfiguration 已有的大机型族。否则将返回 `InvalidParameter.Param`：`can not change major instance type`。参见 [排障](#排障)。

---

### 设置节点缩容保护

对池内指定节点设置缩容保护，防止弹性伸缩缩容时误删关键节点：

```bash
tccli tke SetNodePoolNodeProtection --generate-cli-skeleton --output json > SetProtection.json
```

编辑 JSON：

```json
{
  "ClusterId": "<ClusterId>",
  "NodePoolId": "<NodePoolId>",
  "InstanceIds": ["<InstanceId>"],
  "ProtectedFromScaleIn": true
}
```

```bash
tccli tke SetNodePoolNodeProtection --cli-input-json file://SetProtection.json --region ap-guangzhou --output json
```

```json
{
  "SucceedInstanceIds": ["ins-dui4re12"],
  "FailedInstanceIds": null
}
```

> **重要**：官方建议勿在**弹性伸缩组控制台**直接修改期望数或启停伸缩，应与 TKE 节点池配置保持一致，避免 TKE 与 AS 状态不一致导致意外行为。

---

### 查看修改后的配置

```bash
tccli tke DescribeClusterNodePoolDetail --ClusterId <ClusterId> --NodePoolId <NodePoolId> --region ap-guangzhou --output json \
  --filter "{Name:NodePool.Name,Labels:NodePool.Labels,EnableAutoscale:NodePool.EnableAutoscale,DesiredCapacity:NodePool.AutoscalingGroupPara.DesiredCapacity}"
```

```json
{
  "Name": "kerwinwjyan-rewrite-np",
  "Labels": [
    { "Name": "pool-name", "Value": "kerwinwjyan-test" },
    { "Name": "env", "Value": "rewrite" }
  ],
  "EnableAutoscale": false,
  "DesiredCapacity": 1
}
```

## 验证

### 控制面（tccli）

- `DescribeClusterNodePoolDetail` 中修改的字段与预期一致（名称、Labels、`EnableAutoscale` 等）。例如 Label 添加后返回 `"env": "rewrite"`，确认已生效。
- `DescribeClusterInstances` 节点数量逐渐趋近期望值（扩缩容异步完成）。如期望值调至 2，节点数应在数分钟内从 1 上升至 2。

### 数据面（kubectl）

```bash
kubectl get nodes -l env=rewrite --show-labels
# expected: 节点数量与 DesiredCapacity 一致，含新 Label
```

```text
NAME  STATUS  AGE
...
```

> **可达性要求**：kubectl 需 APIServer 端点可达。

## 清理

如需回滚：将期望节点数调回原始值，关闭弹性伸缩，移除新增的 Label/Taint。

## 排障

| 现象 | 原因 | 处理 |
|------|------|------|
| 手动缩容后节点未驱逐 | 手动缩容不触发 Pod 驱逐 | 先 `kubectl drain` 再调小 `DesiredCapacity`，或使用弹性伸缩缩容路径 |
| `ModifyNodePoolDesiredCapacityAboutAsg` 修改后节点数未变化 | 弹性伸缩正在处理其他活动 | 等待伸缩活动完成，避免并发修改 |
| `ModifyNodePoolDesiredCapacityAboutAsg` 返回 `InvalidParameterValue.Size`：`需要满足 '最小值 <= 期望实例数 <= 最大值' 的限制` | `DesiredCapacity` 不在 `[MinNodesNum, MaxNodesNum]` 区间内 | 先通过 `ModifyClusterNodePool` 同时传入 `--MaxNodesNum` 和 `--MinNodesNum` 扩大容量区间，再重新设置 `DesiredCapacity` |
| `ModifyNodePoolInstanceTypes` 返回 `InvalidParameter.Param`：`can not change major instance type` | 尝试跨大机型族修改（如 S5 → M5） | 先执行 `tccli as DescribeLaunchConfigurations --LaunchConfigurationIds '["<LaunchConfigurationId>"]' --region <Region>` 查询节点池真实机型，只使用已有机型族内的配置写入 `InstanceTypes` |
| 修改 `EnableAutoscale` 后 ASG 状态异常 | 与 AS 控制台直接修改冲突 | 仅通过 TKE API 修改伸缩配置 |
| Lable/Taint 修改后 Pod 调度异常 | 新 Taint 排斥存量 Pod | 提前评估 `Effect` 影响，使用 `PreferNoSchedule` 做软约束 |

## 下一步

- [查看节点池伸缩记录](../查看节点池伸缩记录/tccli%20操作.md)：验证弹性伸缩活动
- [删除节点池](../删除节点池/tccli%20操作.md)：删除不再使用的节点池
- [设置节点 Label](../../常用操作/设置节点%20Label/tccli%20操作.md)：单节点 Label 操作

## 控制台替代

控制台 → **集群** → 目标集群 → **节点管理** → 点击节点池名称 → **编辑**（修改全局配置）或 **调整数量**（修改期望节点数）。节点右侧的**缩容保护**开关对应 `SetNodePoolNodeProtection`。
