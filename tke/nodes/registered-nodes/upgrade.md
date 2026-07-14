---
doc_type: How-to
subtype: 6A
fused: true
---
> 官方文档：[节点概述](https://cloud.tencent.com/document/product/457/32201) · [常见高危操作](https://cloud.tencent.com/document/product/457/39539)
>
> 配额：升级无额外配额限制。[配额说明](https://cloud.tencent.com/document/product/457/9087)
>
> ⚠️ **高危操作**：升级期间节点不可用；升级失败致失联需重新注册；脚本版本与集群版本不兼容。[常见高危操作](https://cloud.tencent.com/document/product/457/39539)

# 升级注册节点

> 升级注册节点上的操作系统或运行时组件，使其与集群版本保持一致。
> 控制台: [容器服务 - 节点池 - 注册节点](https://console.cloud.tencent.com/tke2/nodepool)

TCCLI 不提供注册节点的「升级」动作。注册节点的升级在目标机器侧通过重新执行注册脚本完成：脚本会拉取并更新节点的 kubelet 与运行时组件，使其与集群控制面版本对齐。

> 注册节点是 K8s 原生对象，tccli 不直接管理其操作系统与组件升级。以下命令为 K8s 原生操作（非 tccli），用于确认节点状态与隔离调度，闭环到 kubectl。

## 触发条件

- 控制台提示注册节点版本偏低，或集群升级后节点组件需同步。
- 你要统一注册节点的 kubelet / 运行时版本。

## 准备工作

- 已注册节点（见[创建注册节点（专线版）](dedicated-line.md)）。
- 已获取集群 kubeconfig（见[配置凭证](../../../getting-started/credentials.md)）。
- 已安装 kubectl 且可连通集群 apiserver。

## 操作步骤

### 步骤 1：查看节点当前版本

<!-- kubectl验证节点版本与标签，非tccli边界 -->
> kubectl（K8s 原生命令，非 tccli；TCCLI 管 TKE 抽象层不提供 K8s 资源操作能力）
```bash
kubectl get nodes -o wide --show-labels
# expected: 列出注册节点及其 Kubelet 版本（标签含注册节点池标识）
```

### 步骤 2：隔离节点调度

<!-- kubectl管理K8s原生Node调度，tccli不管理OS与组件升级，非tccli边界 -->
> kubectl（K8s 原生命令，非 tccli；TCCLI 管 TKE 抽象层不提供 K8s 资源操作能力）
```bash
kubectl cordon <NODE_NAME>
# expected: node/<NODE_NAME> cordoned
```

### 步骤 3：重新执行注册脚本升级组件

在目标机器上重新执行节点池的注册脚本（从控制台或 `DescribeExternalNodeScript` 获取），脚本会更新节点组件版本。

### 步骤 4：恢复调度并校验

<!-- kubectl管理K8s原生Node调度，tccli不管理OS与组件升级，非tccli边界 -->
> kubectl（K8s 原生命令，非 tccli；TCCLI 管 TKE 抽象层不提供 K8s 资源操作能力）
```bash
kubectl uncordon <NODE_NAME>
# expected: node/<NODE_NAME> uncordoned
```

## 验证

<!-- kubectl验证升级后kubelet版本，非tccli边界 -->
> kubectl（K8s 原生命令，非 tccli；TCCLI 管 TKE 抽象层不提供 K8s 资源操作能力）
```bash
kubectl get nodes <NODE_NAME> -o jsonpath='{.status.nodeInfo.kubeletVersion}'
# expected: 返回升级后的 kubelet 版本，与集群控制面一致
```

## 故障恢复

| 现象 | 根因 | 修复 |
|:-----|:-----|:-----|
| 重新执行脚本后版本未变 | 运行时缓存未清理 | 清理节点环境后重跑脚本 |
| 节点 NotReady | 网络中断或组件启动失败 | 检查机器到集群网络与组件日志 |

## 收尾确认

<!-- kubectl验证升级后节点就绪状态，非tccli边界 -->
> kubectl（K8s 原生命令，非 tccli；TCCLI 管 TKE 抽象层不提供 K8s 资源操作能力）
```bash
kubectl get nodes <NODE_NAME> -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}'
# expected: True（节点就绪，版本已对齐）
```

## 下一步

- 编辑节点池：[编辑注册节点池](edit-pool.md)
- 移除节点：[移除注册节点](remove.md)
