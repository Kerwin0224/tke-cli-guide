# 查看节点池伸缩记录（tccli）

> 对照官方：[查看节点池伸缩记录](https://cloud.tencent.com/document/product/457/48538) · page_id `48538`

## 概述

查看节点池弹性伸缩（Cluster Autoscaler / CA）的历史活动记录。伸缩记录来源于底层**弹性伸缩组**（ASG）的活动日志。控制台需要先**开启集群事件日志**，然后在事件中心按 ASG 或节点池关键字检索。tccli 控制面可查询节点池关联的 ASG ID，再通过 `as` 产品 API 查询伸缩活动详情。

## 前置条件

- [环境准备](../../../../环境准备.md)：`tccli` 已配置。
- 集群已 [开启事件日志](https://cloud.tencent.com/document/product/457)（控制台操作：集群 → 日志 → 事件日志 → 开启）。
- 节点池已启用弹性伸缩（`EnableAutoscale: true`）或至少发生过手动扩缩容。

## 控制台与 CLI 参数映射

| 控制台查看方式 | tccli / API | 幂等 |
|---------------|-------------|------|
| 全局伸缩记录（事件中心） | 事件服务 API `DescribeEvents` + ASG 活动查询 | 是 |
| 单池伸缩记录（节点池详情） | `DescribeClusterNodePoolDetail` → `AutoscalingGroupId` | 是 |
| 获取节点池关联的 ASG ID | `DescribeClusterNodePoolDetail` → `NodePool.AutoscalingGroupId` | 是 |
| 查询 AS 伸缩活动 | `as DescribeAutoScalingActivities`（弹性伸缩产品） | 是 |

> **映射页边界**：本页聚焦 TKE 侧的节点池-ASG 关联与伸缩活动查询入口。`as` 产品 API 的完整参数与返回结构见 [弹性伸缩 API 文档](https://cloud.tencent.com/document/product/377)。

## 操作步骤

---

### 步骤 1：获取节点池关联的伸缩组 ID

每个普通节点池在创建时自动绑定一个弹性伸缩组。获取其 ASG ID：

```bash
tccli tke DescribeClusterNodePoolDetail --ClusterId <ClusterId> --NodePoolId <NodePoolId> --region ap-guangzhou --output json \
  --filter "{NodePoolId:NodePool.NodePoolId,Name:NodePool.Name,AsgId:NodePool.AutoscalingGroupId,EnableAutoscale:NodePool.EnableAutoscale,DesiredCapacity:NodePool.AutoscalingGroupPara.DesiredCapacity}"
```

```json
{
  "NodePoolId": "np-example",
  "Name": "example-np",
  "AsgId": "asg-example",
  "EnableAutoscale": true,
  "DesiredCapacity": 3
}
```

# expected: exit 0, contains "AsgId"

---

### 步骤 2：查询伸缩活动

使用获取到的 `AutoscalingGroupId`，通过弹性伸缩（`as`）产品 API 查询伸缩活动历史：

```bash
tccli as DescribeAutoScalingActivities --AutoScalingGroupIds '["<AsgId>"]' --region ap-guangzhou --output json
```

```json
{
  "ActivitySet": [
    {
      "ActivityId": "asa-example-1",
      "AutoScalingGroupId": "asg-example",
      "StatusCode": "SUCCESSFUL",
      "Cause": "A scale-out activity was triggered by a threshold alarm policy.",
      "Description": "Add 2 instances. Starting from 1 to 3.",
      "StartTime": "2026-06-15T10:30:00Z",
      "EndTime": "2026-06-15T10:32:30Z",
      "ActivityRelatedInstanceSet": ["ins-example-2", "ins-example-3"]
    },
    {
      "ActivityId": "asa-example-2",
      "AutoScalingGroupId": "asg-example",
      "StatusCode": "SUCCESSFUL",
      "Cause": "A scale-in activity was triggered by a threshold alarm policy.",
      "Description": "Remove 1 instances. Starting from 3 to 2.",
      "StartTime": "2026-06-14T08:00:00Z",
      "EndTime": "2026-06-14T08:01:15Z",
      "ActivityRelatedInstanceSet": ["ins-example-1"]
    }
  ],
  "TotalCount": 2,
  "RequestId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
}
```

按状态筛选（如仅查看失败的伸缩活动）：

```bash
tccli as DescribeAutoScalingActivities --AutoScalingGroupIds '["<AsgId>"]' --region ap-guangzhou --output json \
  --filter "ActivitySet[?StatusCode!='SUCCESSFUL']"
```

---

### 步骤 3：在事件中心查看全局伸缩事件（可选）

官方控制台的**全局伸缩记录**依赖事件日志。查询事件需使用事件服务 API（或控制台检索）：

```bash
# 列出集群事件日志配置
tccli tke DescribeLogSwitches --ClusterIds '["<ClusterId>"]' --region ap-guangzhou --output json \
  --filter "SwitchSet[?ClusterId=='<ClusterId>'].{Enable:Enable,LogsetId:LogsetId,TopicId:TopicId}"
```

```json
[
  { "Enable": true, "LogsetId": "logset-example", "TopicId": "topic-example" }
]
```

> 如果 `Enable: false`，需先在控制台或通过 `CreateCLSLogConfig` 开启事件日志服务。伸缩记录才会被采集到 CLS 日志主题中。

---

### 关键字段说明

| 字段 | 含义 |
|------|------|
| `StatusCode` | `SUCCESSFUL`（成功）/ `FAILED`（失败）/ `IN_PROGRESS`（进行中）/ `CANCELLED`（取消） |
| `Cause` | 触发原因（如告警策略、定时任务、手动调整） |
| `ActivityRelatedInstanceSet` | 本次伸缩活动中涉及的 CVM 实例 ID 列表 |
| `StartTime` / `EndTime` | 伸缩活动起止时间 |

## 验证

### Control plane (tccli)

- `DescribeClusterNodePoolDetail` 返回有效的 `AutoscalingGroupId`（非空、非 null）。
- `as DescribeAutoScalingActivities` 返回 `TotalCount >= 0` 及历史活动列表。
- 若节点池启用弹性伸缩且有扩缩容历史，活动列表中 `StatusCode` 应为 `SUCCESSFUL`。

## 清理

只读操作，无资源清理。

## 排障

| 现象 | 原因 | 处理 |
|------|------|------|
| `AutoscalingGroupId` 为空 | 节点池非普通节点池（如原生/超级节点池无 ASG） | 确认节点池类型，非 ASG 节点池通过各自专页查询伸缩 |
| `as DescribeAutoScalingActivities` 返回空 | 该 ASG 无伸缩活动历史 | 确认节点池是否发生过扩缩容（`DesiredCapacity` 是否有变更） |
| 控制台全局伸缩记录为空 | 未开启集群事件日志 | 在控制台 [开启事件日志](https://cloud.tencent.com/document/product/457)，启用后需等待新伸缩事件发生才能查到 |
| `StatusCode: FAILED` | 伸缩失败（如机型库存不足、余额不足等） | 查看 `Cause` 字段的详细描述，排查对应问题 |

## 下一步

- [调整节点池](../调整节点池/tccli%20操作.md)：修改期望节点数触发伸缩
- [查看节点池](../查看节点池/tccli%20操作.md)：查看池配置和当前容量

## 控制台替代

控制台 → **集群** → 目标集群 → **日志** → **事件日志**（全局伸缩记录），或 **节点管理** → 点击节点池 → **伸缩记录**（单池记录）。控制台伸缩记录依赖事件日志启用。
