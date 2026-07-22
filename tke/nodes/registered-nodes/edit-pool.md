---
doc_type: How-to
subtype: 6A
fused: true
---
> 官方文档：[节点概述](https://cloud.tencent.com/document/product/457/32201) · [节点池概述](https://cloud.tencent.com/document/product/457/104096)
>
> 配额：无额外配额限制。[配额说明](https://cloud.tencent.com/document/product/457/9087)
>
> ⚠️ **高危操作**：修改标签/污点致 Pod 误驱逐或重新调度；修改错误致节点异常需重建节点池。[常见高危操作](https://cloud.tencent.com/document/product/457/39539)

# 编辑注册节点池

> 修改已存在注册节点池的名称、标签、污点、删除保护或用户脚本。
> 控制台: [容器服务 - 节点池 - 注册节点](https://console.cloud.tencent.com/tke2/nodepool)

编辑注册节点池通过 `ModifyExternalNodePool` 完成。该动作修改池级配置，触发节点池状态 `updating` → `normal`。

## 触发条件

- 已有注册节点池（`DescribeExternalNodePools` 返回 `LifeState=normal`）。
- 你要调整节点池的 Labels / Taints / 删除保护，或补充节点初始化后执行的用户脚本。

## 准备工作

- 已创建注册节点池（见[创建注册节点（专线版）](dedicated-line.md)）。
- 已配置 tccli 凭证（见[配置凭证](../../../getting-started/credentials.md)）。

## 关键字段

| 参数 | 所属 Action | 必填 | 说明 |
|:-----|:-----------|:----:|:-----|
| `ClusterId` | ModifyExternalNodePool | 是 | 集群 ID |
| `NodePoolId` | ModifyExternalNodePool | 是 | 节点池 ID（单数） |
| `Name` | ModifyExternalNodePool | 否 | 节点池名称 |
| `Labels` | ModifyExternalNodePool | 否 | 注册节点标签 |
| `Taints` | ModifyExternalNodePool | 否 | 注册节点污点 |
| `DeletionProtection` | ModifyExternalNodePool | 否 | 删除保护开关 |
| `UserScript` | ModifyExternalNodePool | 否 | base64 编码的用户脚本，k8s 组件运行后执行；调用方必须保证脚本可重入并自行实现重试逻辑，脚本及其日志位于节点 `/data/ccs_userscript/` |

## 操作步骤

### 步骤 1：查询节点池当前配置

```bash
tccli tke DescribeExternalNodePools --ClusterId <CLUSTER_ID> --region <REGION>
# expected: exit 0, NodePoolSet 含目标池
```

### 步骤 2：修改标签与删除保护

```bash
tccli tke ModifyExternalNodePool --region <REGION> \
  --ClusterId <CLUSTER_ID> --NodePoolId <NODEPOOL_ID> \
  --Labels '[{"Name":"env","Value":"hybrid"}]' \
  --DeletionProtection true
# expected: exit 0; 节点池不存在报 FailedOperation.RecordNotFound
```

### 步骤 3：添加污点

```bash
tccli tke ModifyExternalNodePool --region <REGION> \
  --ClusterId <CLUSTER_ID> --NodePoolId <NODEPOOL_ID> \
  --Taints '[{"Key":"dedicated","Value":"registered","Effect":"NoSchedule"}]'
# expected: exit 0
```

## 验证

```bash
tccli tke DescribeExternalNodePools --ClusterId <CLUSTER_ID> --region <REGION> \
  --filter "NodePoolSet[?NodePoolId=='<NODEPOOL_ID>'].{state:LifeState,labels:Labels,taints:Taints,protect:DeletionProtection}"
# expected: 目标池 LifeState=normal；Labels 含 Name=env、Value=hybrid；
#           Taints 含 Key=dedicated、Value=registered、Effect=NoSchedule；DeletionProtection=true
```

## 回滚

用步骤 1 查到的原值重新 `ModifyExternalNodePool` 逐项改回。

```bash
# 改回原 Labels 与 DeletionProtection（原值取自步骤 1 的 DescribeExternalNodePools）
tccli tke ModifyExternalNodePool --region <REGION> \
  --ClusterId <CLUSTER_ID> --NodePoolId <NODEPOOL_ID> \
  --Labels '[{"Name":"<ORIG_NAME>","Value":"<ORIG_VALUE>"}]' \
  --DeletionProtection false
# expected: exit 0

# 清除步骤 3 添加的污点（传空数组）
tccli tke ModifyExternalNodePool --region <REGION> \
  --ClusterId <CLUSTER_ID> --NodePoolId <NODEPOOL_ID> \
  --Taints '[]'
# expected: exit 0
```

## 故障恢复

| 现象 | 根因 | 修复 |
|:-----|:-----|:-----|
| `FailedOperation.RecordNotFound` | `NodePoolId` 错误 | 用 `DescribeExternalNodePools` 核对节点池 ID |
| 状态长时间 `updating` | 节点池正在应用配置 | 等待状态回到 `normal` 再操作 |

## 收尾确认

跨步骤汇总池级配置，并核对池内节点端到端仍可用（改标签/污点后业务调度前置就绪）。

```bash
# 池级配置汇总（Labels/Taints/删除保护）
tccli tke DescribeExternalNodePools --ClusterId <CLUSTER_ID> --region <REGION> \
  --filter "NodePoolSet[?NodePoolId=='<NODEPOOL_ID>'].{state:LifeState,labels:Labels,taints:Taints,protect:DeletionProtection,name:Name}"
# expected: LifeState=normal；Labels/Taints/DeletionProtection/Name 与本次修改目标一致

# 池内节点端到端可用性（改污点/标签后节点仍在池内且非异常态；衔接下一步调度前置）
tccli tke DescribeExternalNode --ClusterId <CLUSTER_ID> \
  --NodePoolId <NODEPOOL_ID> --region <REGION> \
  --filter "Nodes[].{name:Name,status:Status,unschedulable:Unschedulable}"
# expected: 既有节点 Status 非 Failed；若步骤 3 加了 NoSchedule 污点，新 Pod 不应再调度到这些节点（业务可用边界）
```

> 池级配置符合预期 + 池内节点未因修改进入异常态 = 编辑任务闭合。加污点后的调度效果属端到端业务可用维度，须与「仅池配置写成功」区分。可进入 [移除注册节点](remove.md) 或继续业务调度。

## 下一步

- 移除节点或节点池：[移除注册节点](remove.md)
- 节点健康检查：[节点健康检查策略](../health-check.md)
