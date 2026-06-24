# TKE 集群中节点移出再移入操作指引（tccli）

> 对照官方：[TKE 集群中节点移出再移入操作指引](https://cloud.tencent.com/document/product/457/40771) · page_id `40771`
## 概述

将 TKE 集群中的节点移出后再重新加入集群的操作流程，包括通过 tccli RemoveNodeFromCluster 和 CreateClusterInstances 实现。

> **kubectl 数据面不可达**：CAM拒绝公网端点(strategyId:240463971)，内网需VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

## 前置条件

- [环境准备](../../../环境准备.md)：`tccli` 已配置，`kubectl` >= v1.28
- 已获取集群 kubeconfig：

```bash
tccli tke DescribeClusterKubeconfig --region <Region> --ClusterId <ClusterId> \
    --filter "Kubeconfig" > kubeconfig.yaml
export KUBECONFIG=$(pwd)/kubeconfig.yaml
# expected: 生成 kubeconfig.yaml
```

```json
{
  "Kubeconfig": "<Kubeconfig>",
  "RequestId": "<RequestId>"
}
```

- 需要 CAM 权限：`tke:DescribeClusters`、`tke:DescribeClusterKubeconfig`

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 获取集群凭证 | `tccli tke DescribeClusterKubeconfig` | 是 |
| 查看集群状态 | `tccli tke DescribeClusterStatus --region <Region> --ClusterIds '["<ClusterId>"]'` | 是 |
| 相关 kubectl 操作 | 见 Procedure | -- |

## 操作步骤

### 1. 控制面：查看当前节点

```bash
tccli tke DescribeClusterInstances --region <Region> --ClusterId <ClusterId>
# expected: InstanceSet 列出所有节点
```

```json
{
  "TotalCount": "<TotalCount>",
  "InstanceSet": "<InstanceSet>",
  "InstanceId": "<InstanceId>",
  "InstanceRole": "<InstanceRole>",
  "FailedReason": "<FailedReason>",
  "InstanceState": "<InstanceState>"
}
```

### 2. 移出节点

```bash
# 通过 tccli 移出节点
tccli tke DeleteClusterInstances --region <Region> \
    --ClusterId <ClusterId> \
    --InstanceIds '["<InstanceId>"]'
# expected: RequestId 返回，节点移出中
```

注意：移出前需在数据面（VPN/IOA 环境）执行：

```bash
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
kubectl delete node <node-name>
```

### 3. 重新加入节点

```bash
# 通过 tccli 添加已有 CVM 实例到集群
tccli tke CreateClusterInstances --region <Region> \
    --ClusterId <ClusterId> \
    --RunInstancePara '{"InstanceIds":["<InstanceId>"]}'
# expected: RequestId 返回，节点加入中
```

### 4. 验证节点加入

```bash
tccli tke DescribeClusterInstances --region <Region> --ClusterId <ClusterId>
# expected: 节点恢复 Running

kubectl get nodes
# expected: 节点 Ready
```

```json
{
  "TotalCount": "<TotalCount>",
  "InstanceSet": "<InstanceSet>",
  "InstanceId": "<InstanceId>",
  "InstanceRole": "<InstanceRole>",
  "FailedReason": "<FailedReason>",
  "InstanceState": "<InstanceState>"
}
```

### 注意事项

- 移出前先 drain 节点，避免影响业务
- 移出后 PVC 数据可能丢失（取决于存储类型）
- 重新加入的节点需要重新初始化
- 节点标签和污点需重新配置

## 验证

### 控制面

```bash
tccli tke DescribeClusterInstances --region <Region> --ClusterId <ClusterId> \
    --filter "InstanceSet[].InstanceState"
# expected: running
```

```json
{
  "TotalCount": "<TotalCount>",
  "InstanceSet": "<InstanceSet>",
  "InstanceId": "<InstanceId>",
  "InstanceRole": "<InstanceRole>",
  "FailedReason": "<FailedReason>",
  "InstanceState": "<InstanceState>"
}
```

### 数据面（需 VPN/IOA）

```bash
kubectl get nodes
# expected: 所有节点 Ready
```

```text
NAME  STATUS  AGE
...
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl drain` 报 `Unable to connect to the server` | `tccli tke DescribeClusters --region ap-guangzhou --ClusterId cls-xxxxxxxx --filter "Clusters[].ClusterStatus"` | CAM 拒绝公网端点（strategyId 240463971），数据面不可达 | 接入 VPN/IOA 后在内网执行 drain/delete |
| `DeleteClusterInstances` 返回 `OperationDenied` | `tccli tke DescribeClusterInstances --region ap-guangzhou --ClusterId cls-xxxxxxxx --filter "InstanceSet[].{Id:InstanceId,State:InstanceState}"` | 节点有未驱逐的 Pod 或 CVM 状态异常 | 先 `kubectl drain`（VPN/IOA）再移出；确认 CVM 处于 Running |
| 节点重新加入后 `NotReady` | `kubectl get nodes`（VPN/IOA） | CNI/容器运行时初始化未完成 | 等待 1-2 分钟；`kubectl describe node <name>` 查看事件 |

## 清理

<!-- 节点操作无需特殊清理 -->

## 下一步

- [节点常见报错与处理](../../../故障处理/节点常见报错与处理/tccli%20操作.md) -- page_id `89869`
- [应用高可用部署](../../服务部署/应用高可用部署/tccli%20操作.md) -- page_id `40212`

## 控制台替代

[容器服务控制台](https://console.cloud.tencent.com/tke2/cluster)
