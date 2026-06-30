# Pod 一直处于 Pending 状态（tccli）

> 对照官方：[Pod 一直处于 Pending 状态](https://cloud.tencent.com/document/product/457/42948) · page_id `42948`

## 概述

Pod `Pending` 表示调度器已接收但尚未分配到节点。常见原因：资源不足、节点亲和性不满足、污点未容忍、PVC 未绑定。通过 tccli 检查集群状态，配合 kubectl 查看调度失败原因和节点资源。

> **kubectl 数据面不可达**：CAM拒绝公网端点(strategyId:240463971)，内网需VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

## 前置条件

- [环境准备](../../../环境准备.md)
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
| 查看调度失败原因 | `kubectl describe pod <pod> \| grep -A10 Events`（需 VPN/IOA） | 是 |
| 查看节点资源 | `kubectl top nodes`（需 VPN/IOA） | 是 |
| 查看节点污点 | `kubectl describe node <node> \| grep -A5 Taints`（需 VPN/IOA） | 是 |

## 操作步骤

### 步骤1：控制面检查集群

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

### 步骤2：数据面查看调度失败原因（需 VPN/IOA）

```bash
kubectl describe pod <pod-name> -n <ns> | grep -A10 Events
# 预期: FailedScheduling 事件包含具体原因
```

```text
Name:         ...
Status:       Running
...
```

### 步骤3：按 Events Reason 诊断

| Events Reason | 根因 | 修复 |
|---|---|---|
| `FailedScheduling` + `0/N nodes are available: insufficient cpu/memory` | 所有节点资源余量不足 | 扩容节点或降低 Pod requests |
| `node(s) didn't match node selector` | nodeSelector 无节点匹配 | 修正 selector 标签或为节点添加对应标签 |
| `node(s) had taint` | Pod 未容忍节点污点 | 添加 tolerations 或移除节点污点 |
| `pod has unbound immediate PersistentVolumeClaims` | PVC 未绑定 | `kubectl get pvc` 查看状态 |
| `node(s) didn't match pod affinity/anti-affinity` | 亲和性规则无法满足 | 调整 affinity 规则或增加节点 |

### 步骤4：检查节点资源和可用性（需 VPN/IOA）

```bash
kubectl top nodes
# 预期: CPU(cores) 和 MEMORY(bytes)

kubectl describe node <node> | grep -A8 "Allocated resources"
# 预期: 对比 Capacity 和 Allocated

kubectl get nodes --show-labels
# 预期: 查看节点标签
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
kubectl get pod <pod> -n <ns> -w
# 预期: STATUS 从 Pending 变为 Running
```

```text
NAME  STATUS  AGE
...
```

## 清理

无需特殊清理（排障页）。测试 Pod 修复后可删除。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|---|---|---|---|
| Events 显示资源不足但 `kubectl top nodes` 有余量 | 检查 Allocatable vs Capacity | 系统预留资源占用；已有 Pod limits 预留未使用的资源 | 降低已有 Pod requests；检查节点预留资源比例 |
| PVC Pending 导致 Pod Pending | `kubectl describe pvc` 查看 PVC Events | StorageClass provisioner 未创建 PV | 安装对应 CSI provisioner；检查 StorageClass |

### 操作成功但异常

| 现象 | 诊断 | 根因 | 修复 |
|---|---|---|---|
| Pod 一直 Pending 但 Events 为空 | 手动 `kubectl patch` 触发重新调度 | 调度器队列延迟 | 检查 scheduler 日志；删除重建 Pod |
| 扩容节点后 Pod 仍 Pending | 检查新 Pod 的 affinity 规则 | 亲和性规则约束了调度范围 | 调整 affinity 规则允许调度到新节点 |

## 下一步

- [Pod 一直处于 Terminating 状态](../Pod%20一直处于%20Terminating%20状态/tccli%20操作.md) -- page_id `43238`
- [节点常见报错与处理](../../节点常见报错与处理/tccli%20操作.md) -- page_id `89869`

## 控制台替代

[容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) -> 工作负载 -> Pod -> 事件 Tab。
