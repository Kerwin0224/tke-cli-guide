---
doc_type: How-to
subtype: 6A
fused: true
---
> 官方文档：[节点概述](https://cloud.tencent.com/document/product/457/32201) · [常见高危操作](https://cloud.tencent.com/document/product/457/39539)
>
> 配额：升级无额外配额限制。[配额说明](https://cloud.tencent.com/document/product/457/9087)
>
> ⚠️ **高危操作**：升级期间节点不可用；升级失败致失联；操作系统或组件版本不受集群支持。

# 升级注册节点

> 在目标机器侧维护注册节点的操作系统，并在变更前后核验节点版本和就绪状态。
> 控制台: [容器服务 - 节点池 - 注册节点](https://console.cloud.tencent.com/tke2/nodepool)

TCCLI 不提供注册节点操作系统或节点组件的升级 Action。注册脚本用于把机器注册到节点池；现有 API/help 没有承诺重新执行脚本会升级操作系统、kubelet 或容器运行时，因此不能把“重新注册”当作“升级完成”。

> 注册节点机器由你自备并维护。操作系统升级应使用组织批准且受该操作系统支持的运维流程；具体包管理、镜像替换与跨版本路径需以操作系统供应方和 TKE 当期兼容性要求为准，本文不猜测未裁决的升级命令。

## 触发条件

- 操作系统供应方要求安装安全更新，或计划变更节点操作系统版本。
- 你需要确认升级前后 kubelet、容器运行时与节点就绪状态。

## 准备工作

- 已注册节点（见[创建注册节点（专线版）](dedicated-line.md)）。
- 已获取集群 kubeconfig（见[配置凭证](../../../getting-started/credentials.md)）。
- 已安装 kubectl 且可连通集群 apiserver。
- 已确认目标操作系统仍在 TKE 注册节点支持范围内，并准备机器级备份与回退方案。
- 一次只处理一个节点；先确认 PodDisruptionBudget、本地数据与替代容量允许驱逐。

## 操作步骤

### 步骤 1：记录升级前版本

> kubectl（K8s 原生命令，非 tccli；TCCLI 管 TKE 抽象层不提供机器操作系统升级能力）
```bash
kubectl get node <NODE_NAME> \
  -o jsonpath='{.status.nodeInfo.osImage}{"\n"}{.status.nodeInfo.kubeletVersion}{"\n"}{.status.nodeInfo.containerRuntimeVersion}{"\n"}'
# expected: 记录 OS、kubelet 与容器运行时版本，作为升级后对照
```

### 步骤 2：隔离并安全驱逐

> kubectl（K8s 原生命令，非 tccli；TCCLI 不提供 cordon/drain 能力）
```bash
kubectl cordon <NODE_NAME>
# expected: node/<NODE_NAME> cordoned

kubectl drain <NODE_NAME> --ignore-daemonsets --delete-emptydir-data
# expected: 节点不可调度，允许驱逐的工作负载已迁移；PDB 阻断时先处理阻断原因，不要强行跳过
```

### 步骤 3：执行受支持的机器侧升级

在目标机器上按组织批准的操作系统运维流程执行升级，并按供应方要求重启。不要仅重新执行注册脚本后便宣告操作系统或组件已升级。

若升级导致注册组件损坏、需要重新注册，按[创建注册节点（专线版）](dedicated-line.md#步骤-4获取节点注册脚本)重新获取并执行注册脚本；这是故障恢复/重新注册，不是升级动作。

### 步骤 4：验证后恢复调度

> kubectl（K8s 原生命令，非 tccli；TCCLI 不管理机器版本或 Node 调度）
```bash
kubectl get node <NODE_NAME> \
  -o jsonpath='{.status.nodeInfo.osImage}{"\n"}{.status.nodeInfo.kubeletVersion}{"\n"}{.status.nodeInfo.containerRuntimeVersion}{"\n"}{.status.conditions[?(@.type=="Ready")].status}{"\n"}'
# expected: 版本与批准的升级目标一致，Ready=True

kubectl uncordon <NODE_NAME>
# expected: node/<NODE_NAME> uncordoned
```

## 验证

- OS、kubelet、容器运行时版本均与升级前记录和批准目标逐项对照。
- `Ready=True` 后才恢复调度。
- 恢复调度后确认业务 Pod 能在该节点正常启动；仅 `Ready=True` 不能证明业务兼容性。

## 故障恢复

| 现象 | 根因 | 修复 |
|:-----|:-----|:-----|
| 升级后版本未达到目标 | 机器侧升级未完成或重启未生效 | 保持 cordon，按操作系统供应方流程检查升级状态；不要用注册脚本替代升级 |
| 节点 `NotReady` | 网络中断、节点组件失败或版本不兼容 | 保持隔离，检查机器到集群网络与组件日志；必要时执行已准备的机器级回退 |
| 节点不再注册 | 注册组件损坏或节点身份失效 | 按专线版创建篇重新获取脚本并注册，再重新执行版本与就绪验收 |

## 收尾确认

> kubectl（K8s 原生命令，非 tccli；TCCLI 不提供机器升级终态）
```bash
kubectl get node <NODE_NAME> \
  -o custom-columns='NAME:.metadata.name,OS:.status.nodeInfo.osImage,KUBELET:.status.nodeInfo.kubeletVersion,RUNTIME:.status.nodeInfo.containerRuntimeVersion,READY:.status.conditions[?(@.type=="Ready")].status,UNSCHEDULABLE:.spec.unschedulable'
# expected: 版本符合批准目标，READY=True，UNSCHEDULABLE=<none>
```

## 清理

本篇不创建 TKE 侧新资源；机器侧升级残留（临时包、备份镜像）按组织运维规范处理。若升级失败且决定下线节点，见 [移除注册节点](remove.md)。

## 下一步

- 编辑节点池：[编辑注册节点池](edit-pool.md)
- 移除节点：[移除注册节点](remove.md)
