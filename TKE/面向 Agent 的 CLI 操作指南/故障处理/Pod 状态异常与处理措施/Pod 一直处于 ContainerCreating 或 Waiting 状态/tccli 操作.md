# Pod 一直处于 ContainerCreating 或 Waiting 状态（tccli）

> 对照官方：[Pod 一直处于 ContainerCreating 或 Waiting 状态](https://cloud.tencent.com/document/product/457/42946) · page_id `42946`

## 概述

Pod 长时间处于 `ContainerCreating` 或 `Waiting` 状态通常由镜像拉取、存储卷挂载、CNI 网络分配等环节阻塞导致。通过 tccli 检查集群状态，配合 kubectl 查看 Pod Events 定位卡住的具体环节。

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
| 查看 Pod Events | `kubectl describe pod <pod>`（需 VPN/IOA） | 是 |
| 查看 PVC 状态 | `kubectl get pvc`（需 VPN/IOA） | 是 |
| 查看 CNI 组件 | `kubectl get pods -n kube-system -l k8s-app=tke-cni`（需 VPN/IOA） | 是 |

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

### 步骤2：数据面查看 Pod Events（需 VPN/IOA）

```bash
kubectl describe pod <pod-name> -n <ns>
# 预期: 重点关注 Events 区域的 Warning 事件
```

```text
Name:         ...
Status:       Running
...
```

### 步骤3：按 Events Reason 诊断

| Events Reason | 根因 | 修复 |
|---|---|---|
| `FailedMount` | PVC 未绑定、Secret/ConfigMap 不存在、存储卷挂载超时 | `kubectl describe pvc <name>`；`kubectl get secret,configmap -n <ns>` |
| `FailedCreatePodSandBox` | CNI 网络分配失败 | 检查 CNI 插件：`kubectl get pods -n kube-system -l k8s-app=tke-cni` |
| 长时间无 Events（卡在 Pulling） | 镜像拉取慢或卡住 | `kubectl get events -n <ns> -w` 观察；转至 [ImagePullBackOff 排障](../Pod%20一直处于%20ImagePullBackOff%20状态/tccli%20操作.md) |
| `ErrImagePull` | 镜像拉取失败 | 转至 [ImagePullBackOff 排障](../Pod%20一直处于%20ImagePullBackOff%20状态/tccli%20操作.md) |

### 步骤4：排查卷挂载问题（需 VPN/IOA）

```bash
kubectl get pvc -A
# 预期: STATUS 列 Bound

kubectl describe pvc <pvc-name> -n <ns>
# 预期: Events 中查看绑定状态
```

```text
NAME  STATUS  AGE
...
```

### 步骤5：排查 CNI 问题（需 VPN/IOA）

```bash
kubectl get pods -n kube-system -l k8s-app=tke-cni -o wide
# 预期: 每节点一个 CNI 代理，STATUS Running

kubectl logs -n kube-system -l k8s-app=tke-cni --tail=30
# 预期: 查看是否有 IP 分配失败日志
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
# 预期: STATUS 从 ContainerCreating 变为 Running
```

```text
NAME  STATUS  AGE
...
```

## 清理

```bash
# 问题 Pod 修复后可删除测试资源（需 VPN/IOA）
kubectl delete pod <pod-name> -n <ns>
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|---|---|---|---|
| Events 中 `FailedMount` + `waiting for a volume to be created` | `kubectl get pvc` 查看 PVC 状态 | StorageClass provisioner 未创建 PV | 确认 CBS-CSI provisioner 运行正常；检查 StorageClass 配置 |
| Events 中 `FailedMount` + `secret not found` | `kubectl get secret -n <ns>` 确认 | 引用的 Secret 不存在或命名空间错误 | 创建对应 Secret 到正确的命名空间 |

### 操作成功但异常

| 现象 | 诊断 | 根因 | 修复 |
|---|---|---|---|
| 大规模创建 Pod 时部分卡住 | 检查节点磁盘 IO 和网络带宽 | 并发镜像拉取和卷挂载导致节点资源瓶颈 | 分批创建 Pod；使用镜像预热 |
| Pod 变为 Running 但立即 CrashLoopBackOff | `kubectl logs <pod> --previous` 查看退出日志 | ContainerCreating 阶段通过但容器内应用启动失败 | 转至 [Pod 处于 CrashLoopBackOff 状态](../Pod%20处于%20CrashLoopBackOff%20状态/tccli%20操作.md) |

## 下一步

- [Pod 健康检查失败](../Pod%20健康检查失败/tccli%20操作.md) -- page_id `43129`
- [Pod 网络无法访问排查处理](../../Pod%20网络无法访问排查处理/tccli%20操作.md) -- page_id `40332`

## 控制台替代

[容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) -> 工作负载 -> Pod -> 事件 Tab。
