# Pod 一直处于 ContainerCreating 或 Waiting 状态（tccli）

> 对照官方：[Pod 一直处于 ContainerCreating 或 Waiting 状态](https://cloud.tencent.com/document/product/457/42946) · page_id `42946`

## 概述

排查 Pod 长时间处于 ContainerCreating 或 Waiting 状态的原因：镜像拉取慢、存储卷挂载异常、Secret/ConfigMap 不存在、CNI IP 分配失败等。通过 tccli 检查集群和节点状态，配合 kubectl 查看 Pod Events 定位卡住的环节。

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
| 查看 Pod Events | `kubectl describe pod <pod>`（需 VPN/IOA） | 是 |
| 查看节点状态 | `kubectl get nodes`（需 VPN/IOA） | 是 |

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

### 步骤2：数据面症状检查（需 VPN/IOA）

```bash
kubectl describe pod <pod> -n <ns> | grep -A30 Events
# 预期: 查看 Events 定位卡住的具体环节

kubectl get pod <pod> -n <ns> -o jsonpath='{.status.containerStatuses[*].state}'
# 预期: 显示 waiting 状态及原因
```

```text
NAME  STATUS  AGE
...
```

### 步骤3：按阶段诊断

ContainerCreating 过程分为四个阶段，通过 Events 确定卡在哪一步：

**阶段一：Pulling（拉取镜像）**

Events 中出现 `Pulling image` 后长时间无进展：
- 确认镜像名和 Tag 正确
- 检查是否私有仓库，需配置 imagePullSecrets
- 大镜像拉取慢属正常，关注超时报错

**阶段二：Mounting（挂载卷）**

Events 中出现 `FailedMount` 或长时间停留在挂载阶段：
- `kubectl get pvc` 确认 PVC 已 Bound
- `kubectl describe pvc <pvc-name>` 查看挂载事件
- 检查 Secret/ConfigMap 是否存在：`kubectl get secret,configmap -n <ns>`

**阶段三：Creating（创建 Sandbox）**

Events 中出现 `FailedCreatePodSandBox`：
- CNI 网络插件未分配 IP
- VPC-CNI 模式下子网 IP 耗尽
- 检查 CNI 组件状态：`kubectl get pods -n kube-system -l k8s-app=tke-cni`

**阶段四：Starting（启动容器）**

Sandbox 创建成功但容器未 Running：
- 检查容器启动命令是否正确
- `kubectl logs <pod> -n <ns>` 查看启动日志（如果容器曾短暂运行）

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
| Events 中 `FailedMount` + `secret not found` | `kubectl get secret -n <ns>` 确认 Secret 存在 | Secret 未创建或被误删 | 创建对应 Secret：`kubectl create secret generic <name> --from-literal=key=value -n <ns>` |
| Events 中 `FailedMount` + `unable to attach` | `kubectl describe pvc` 查看 PVC 状态 | CBS 磁盘挂载超时，可能节点与 CBS 不在同一可用区 | 确认 Pod 调度的节点可用区与 CBS 一致 |
| Events 中 `FailedCreatePodSandBox` | `kubectl get pods -n kube-system` 检查 CNI Pod | VPC-CNI 子网无可用 IP | 在 VPC 控制台扩容子网或更换子网 |

### 操作成功但异常

| 现象 | 诊断 | 根因 | 修复 |
|---|---|---|---|
| Pod Running 后立即 CrashLoopBackOff | `kubectl logs <pod> --previous` 查看退出日志 | 容器启动后立即退出（Exit Code 1） | 转至 [Pod 处于 CrashLoopBackOff 状态](../Pod%20状态异常与处理措施/Pod%20处于%20CrashLoopBackOff%20状态/tccli%20操作.md) |
| 大规模镜像拉取耗时极长 | 查看节点磁盘 IO 和带宽 | 多 Pod 同时拉取大镜像导致网络和 IO 拥塞 | 使用镜像预热或分层拉取；将镜像推送至 TCR 加速 |

## 下一步

- [节点常见报错与处理](../节点常见报错与处理/tccli%20操作.md) -- page_id `89869`
- [Pod 状态异常与处理措施](../Pod%20状态异常与处理措施/tccli%20操作.md) -- page_id `42945`
- [Pod 一直处于 ImagePullBackOff 状态](../Pod%20一直处于%20ImagePullBackOff%20状态/tccli%20操作.md) -- page_id `42947`

## 控制台替代

[容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) -> 工作负载 -> Pod -> 事件 Tab。
