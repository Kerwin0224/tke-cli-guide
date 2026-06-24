# 删除节点池（tccli）

> 对照官方：[删除节点池](https://cloud.tencent.com/document/product/457/43738) · page_id `43738`

## 概述

删除集群中的节点池。可选择**同时销毁池内所有 CVM 实例**（`KeepInstance: false`）或**仅移出集群保留实例**（`KeepInstance: true`）。

⚠️ **级联清理警告**：删除节点池会级联删除池内所有节点。如果选择销毁 CVM，系统盘和数据盘也将一并释放。操作不可逆，务必在删除前确认池内无关键业务负载。

## 前置条件

- [环境准备](../../../../环境准备.md)：`tccli` 已配置。
- 池内节点已排空（`kubectl drain`）或确认可销毁。
- 如有缩容保护，先通过 `SetNodePoolNodeProtection` 解除。

## 控制台与 CLI 参数映射

| 控制台操作 | tccli API | 幂等 |
|------------|-----------|------|
| 删除节点池（销毁实例） | `DeleteClusterNodePool`（`KeepInstance: false`） | 否 |
| 删除节点池（保留实例） | `DeleteClusterNodePool`（`KeepInstance: true`） | 否 |
| 从池中移出指定节点 | `RemoveNodeFromNodePool` | 否 |
| 删除前确认池存在 | `DescribeClusterNodePools` | 是 |

## 操作步骤

---

### 步骤 1：确认删除目标

列出待删除节点池及其包含的节点：

```bash
tccli tke DescribeClusterNodePools --ClusterId <ClusterId> --region ap-guangzhou --output json \
  --filter "NodePoolSet[?NodePoolId=='<NodePoolId>'].{NodePoolId:NodePoolId,Name:Name,LifeState:LifeState,NodeCount:NodeCountSummary.AutoscalingAdded}"
```

```json
[
  {
    "NodePoolId": "np-example",
    "Name": "example-np",
    "LifeState": "normal",
    "NodeCount": { "Joined": 2, "Total": 2 }
  }
]
```

---

### 步骤 2：排空池内节点（如有业务负载）

如果池内有运行业务 Pod 的节点，先对每个节点执行封锁与驱逐：

```bash
# 获取池内节点列表
tccli tke DescribeClusterInstances --ClusterId <ClusterId> --region ap-guangzhou --output json \
  --filter "InstanceSet[?NodePoolId=='<NodePoolId>'].InstanceId | []"
```

```json
["ins-example-1", "ins-example-2"]
```

在可访问 APIServer 的环境中对每个节点执行：

```bash
kubectl cordon <NodeName>
kubectl drain <NodeName> --ignore-daemonsets --delete-emptydir-data --force
```

> **可达性要求**：kubectl 需 APIServer 端点可达。若无法 drain，可将期望节点数缩至 0 让 AS 自动移出节点（需 `EnableAutoscale: true`）。

---

### 步骤 3：调用删除 API

**场景 A：删除节点池并销毁 CVM 实例**（最常用）

```bash
tccli tke DeleteClusterNodePool --cli-input-json '{"ClusterId":"<ClusterId>","NodePoolIds":["<NodePoolId>"],"KeepInstance":false}' --region ap-guangzhou --output json
```

```json
{ "RequestId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890" }
```

**场景 B：仅删除节点池，保留 CVM 实例**

```bash
tccli tke DeleteClusterNodePool --cli-input-json '{"ClusterId":"<ClusterId>","NodePoolIds":["<NodePoolId>"],"KeepInstance":true}' --region ap-guangzhou --output json
```

> `KeepInstance: true` 时节点从集群移除，CVM 实例保留在账号下，需手动在 CVM 控制台处理。

**场景 C：仅从节点池移出部分节点（不删池）**

```bash
tccli tke RemoveNodeFromNodePool --cli-input-json '{"ClusterId":"<ClusterId>","NodePoolId":"<NodePoolId>","InstanceIds":["<InstanceId>"]}' --region ap-guangzhou --output json
```

---

### 步骤 4：验证删除结果

```bash
tccli tke DescribeClusterNodePools --ClusterId <ClusterId> --region ap-guangzhou --output json \
  --filter "NodePoolSet[?NodePoolId=='<NodePoolId>'].NodePoolId"
# expected: exit 0, output is empty or null (node pool gone)
```

```json
{
  "NodePoolSet": "<NodePoolSet>",
  "NodePoolId": "<NodePoolId>",
  "Name": "<Name>",
  "ClusterInstanceId": "<ClusterInstanceId>",
  "LifeState": "<LifeState>",
  "LaunchConfigurationId": "<LaunchConfigurationId>"
}
```

## 验证

### Control plane (tccli)

- `DescribeClusterNodePools` 不再包含已删除的 `NodePoolId`。
- 若 `KeepInstance: false`，`DescribeClusterInstances` 中池内节点的 `InstanceState` 变为 `terminating` 直至消失。
- 若 `KeepInstance: true`，CVM 实例仍在 CVM 控制台可见，但不再关联到 TKE 集群。

## 清理

- `KeepInstance: false`：CVM 和磁盘已随池删除，无需额外清理。
- `KeepInstance: true`：保留的 CVM 需在 CVM 控制台手动释放，云硬盘需单独清理。

## 排障

| 现象 | 原因 | 处理 |
|------|------|------|
| 删除失败：节点池非空 | 池内仍有 `Joined` 节点 | 先缩容至 0 或 `RemoveNodeFromNodePool` 移出全部节点 |
| 删除失败：存在缩容保护 | 有节点开启了缩容保护 | `SetNodePoolNodeProtection` 解除保护 |
| 删除卡在 `deleting` | 伸缩组删除流程阻塞 | 检查 AS 控制台伸缩组状态；若长期未完成可工单联系 |
| CVM 销毁后数据盘残留 | `KeepInstance: false` 仅销毁 CVM 随系统盘，独立数据盘可能未自动释放 | 在云硬盘控制台检查并手动删除残留云硬盘 |
| `RemoveNodeFromNodePool` 后节点仍显示在池中 | AS 未同步 | 刷新列表，等待 AS 同步（通常 1-2 分钟） |

## 下一步

- [创建节点池](../创建节点池/tccli%20操作.md)：重建节点池
- [查看节点池](../查看节点池/tccli%20操作.md)：确认集群内剩余节点池
- [移出节点](../../常用操作/移出节点/tccli%20操作.md)：仅移除单节点

## 控制台替代

控制台 → **集群** → 目标集群 → **节点管理** → 选择节点池 → **更多** → **删除**。控制台会提示是否同时销毁 CVM 实例。
