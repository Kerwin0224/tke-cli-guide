# Pod 一直处于 Pending 状态（tccli）

> 对照官方：[Pod 一直处于 Pending 状态](https://cloud.tencent.com/document/product/457/42945) · page_id `42945`

## 概述

排查 Pod 长时间 Pending 问题：资源不足、PVC 未绑定、节点选择器不匹配、污点、亲和性无法满足等。通过 tccli 检查集群和节点资源，配合 kubectl 查看调度失败原因。

> **kubectl 数据面不可达**：CAM拒绝公网端点(strategyId:240463971)，内网需VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

## 前置条件

- [环境准备](../../环境准备.md)
- 已获取集群 kubeconfig（如需 kubectl 操作）

### 环境检查

```bash
tccli --version
# 预期输出: tccli version X.X.X
```

### 资源检查

```bash
tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]'
```

**预期输出**：

```json
{
  "TotalCount": 1,
  "Clusters": [
    {
      "ClusterId": "cls-xxxxxxxx",
      "ClusterName": "my-test-cluster",
      "ClusterStatus": "Running",
      "ClusterVersion": "1.30.0",
      "ClusterType": "MANAGED_CLUSTER"
    }
  ]
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli / CLI | 幂等 |
|---|---|---|
| 查看集群状态 | `tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]'` | 是 |
| 查看节点实例 | `tccli tke DescribeClusterInstances --region ap-guangzhou --ClusterId cls-xxxxxxxx` | 是 |
| 查看调度失败原因 | `kubectl describe pod <pod>`（需 VPN/IOA） | 是 |
| 查看节点资源 | `kubectl top nodes`（需 VPN/IOA） | 是 |
| 查看节点标签和污点 | `kubectl describe node <node>`（需 VPN/IOA） | 是 |

## 操作步骤

### 步骤1：控制面检查集群和节点

```bash
tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]'
```

**预期输出**：

```json
{
  "Clusters": [
    {
      "ClusterId": "cls-xxxxxxxx",
      "ClusterStatus": "Running",
      "ClusterVersion": "1.30.0"
    }
  ]
}
```

```bash
tccli tke DescribeClusterInstances --region ap-guangzhou --ClusterId cls-xxxxxxxx
```

**预期输出**：

```json
{
  "TotalCount": 3,
  "InstanceSet": [
    {
      "InstanceId": "ins-xxxxxxxx",
      "InstanceRole": "WORKER",
      "InstanceState": "running"
    }
  ]
}
```

### 步骤2：数据面查看调度失败原因（需 VPN/IOA）

```bash
kubectl describe pod <pod> -n <ns> | grep -A15 Events
# 预期: Events 中显示调度失败的具体原因
```

```text
Name:         ...
Status:       Running
...
```

### 步骤3：按 Events Reason 分类诊断

| Events Reason | 根因 | 修复 |
|---|---|---|
| `FailedScheduling` + `0/N nodes are available: insufficient cpu/memory` | 所有节点 CPU 或内存余量不足 | 扩容节点或降低 Pod requests |
| `FailedScheduling` + `node(s) didn't match node selector` | Pod 的 nodeSelector 无节点匹配 | 检查并修正 nodeSelector 标签，或为节点添加对应标签 |
| `FailedScheduling` + `node(s) had taint` | Pod 未容忍节点的污点（Taint） | 添加 toleration 或移除节点污点 |
| `pod has unbound immediate PersistentVolumeClaims` | PVC 未绑定到可用 PV | `kubectl get pvc` 查看 PVC 状态 |
| `FailedScheduling` + `node(s) didn't match pod affinity/anti-affinity` | 亲和性/反亲和性规则无法满足 | 调整 affinity 规则或增加节点 |

### 步骤4：检查节点资源（需 VPN/IOA）

```bash
# 查看各节点资源使用
kubectl top nodes
# 预期: CPU% 和 MEMORY% 列

# 查看节点可分配资源详情
kubectl describe node <node> | grep -A8 "Allocated resources"
# 预期: 对比 Capacity 和 Allocated，判断是否有余量
```

```text
Name:         ...
Status:       Running
...
```

### 步骤5：检查节点污点和标签（需 VPN/IOA）

```bash
kubectl get nodes --show-labels
# 预期: 确认各节点 LABELS

kubectl describe node <node> | grep -A5 Taints
# 预期: 查看节点是否存在 NoSchedule 或 NoExecute 污点
```

```text
NAME  STATUS  AGE
...
```

## 验证

### 控制面（tccli）

```bash
tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]'
```

**预期输出**：

```json
{
  "Clusters": [
    {
      "ClusterId": "cls-xxxxxxxx",
      "ClusterStatus": "Running"
    }
  ]
}
```

### 数据面（需 VPN/IOA）

```bash
kubectl get pod <pod> -n <ns>
# 预期: STATUS 变为 Running

kubectl get nodes
# 预期: 所有节点 Ready
```

```text
NAME  STATUS  AGE
...
```

## 清理

无需特殊清理（排障页）。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|---|---|---|---|
| Events 显示资源不足但 `kubectl top nodes` 显示有余量 | 检查节点可分配资源（Allocatable）而非总容量 | 系统预留资源占用，或已有 Pod 的 limits 预留了未使用的资源 | 检查节点 Allocatable 与 Capacity 差异；降低已有 Pod 的 requests |
| PVC 一直 Pending | `kubectl describe pvc` 查看事件 | StorageClass 未定义 provisioner 或无可用 PV | 安装 CBS-CSI provisioner；检查 StorageClass 配置 |

### 操作成功但异常

| 现象 | 诊断 | 根因 | 修复 |
|---|---|---|---|
| Pod 被驱逐后持续 Pending | 原节点资源释放但新节点资源不足 | 驱逐后的 Pod 重新调度但集群整体资源紧张 | 扩容节点或配置 Cluster Autoscaler（CA）自动扩容 |
| Deployment 部分 Pod Pending | `kubectl describe pod` 发现特定节点问题 | 渲染节点存在磁盘压力或网络异常 | Cordon 问题节点：`kubectl cordon <node>` |

## 下一步

- [节点常见报错与处理](../节点常见报错与处理/tccli%20操作.md) -- page_id `89869`
- [Pod 一直处于 Terminating 状态](../Pod%20状态异常与处理措施/Pod%20一直处于%20Terminating%20状态/tccli%20操作.md) -- page_id `43238`

## 控制台替代

[容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) -> 工作负载 -> Pod -> 事件 Tab。
