# 节点常见报错与处理（tccli）

> 对照官方：[节点常见报错与处理](https://cloud.tencent.com/document/product/457/89869) · page_id `89869`

## 概述

梳理 TKE 节点常见异常状态和处理方法：NotReady、DiskPressure、MemoryPressure、PIDPressure、NetworkUnavailable。通过 tccli 检查集群和节点实例状态，配合 kubectl 查看节点 Condition 进行诊断和恢复。

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
| 查看节点 Condition | `kubectl describe node <node>`（需 VPN/IOA） | 是 |
| 隔离节点 | `kubectl cordon <node>`（需 VPN/IOA） | 是 |
| 驱逐 Pod | `kubectl drain <node> --ignore-daemonsets --delete-emptydir-data`（需 VPN/IOA） | 否 |

## 操作步骤

### 步骤1：控制面检查集群和节点实例

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
      "InstanceState": "running",
      "LanIP": "10.0.1.10"
    }
  ]
}
```

### 步骤2：数据面查看节点状态（需 VPN/IOA）

```bash
kubectl get nodes
# 预期: 关注 STATUS 列，正常为 Ready

kubectl describe node <node> | grep -A10 "Conditions:"
# 预期: 各 Condition 详细状态
```

```text
NAME  STATUS  AGE
...
```

### 步骤3：按 Condition 分类诊断

**NotReady**

节点加入集群但 kubelet 未正常上报心跳。

1. 检查节点是否正常：`tccli tke DescribeClusterInstances` 确认 InstanceState 为 running
2. 检查 kubelet 状态（SSH 到节点）：`systemctl status kubelet`
3. 检查容器运行时：`systemctl status containerd` 或 `docker`
4. 检查节点到 API Server 连通性
5. 检查证书是否过期

**DiskPressure**

磁盘使用超过 85% 阈值（kubelet 默认配置）。

1. 检查磁盘使用：`ssh root@<node> "df -h"`
2. 常见占用来源：容器镜像、容器日志、临时文件
3. 清理方法：`docker image prune -a -f`，限制日志大小
4. 长期方案：扩容 CBS 云硬盘

**MemoryPressure**

节点可用内存不足。

1. `kubectl top nodes` 查看内存使用
2. `kubectl top pods -A --sort-by=memory` 定位高内存 Pod
3. 临时措施：驱逐低优先级 Pod
4. 长期方案：扩容节点或增加节点数

**PIDPressure**

进程 ID 耗尽（默认最大 32768 或 4194304）。

1. SSH 到节点：`cat /proc/sys/kernel/pid_max`
2. 检查僵尸进程：`ps aux | grep defunct`
3. 排查 PID 泄漏应用

**NetworkUnavailable**

CNI 网络插件未就绪。

1. 检查 CNI 组件 Pod：`kubectl get pods -n kube-system -l k8s-app=tke-cni`
2. 可能原因：CNI 插件未安装、配置错误、节点网络异常

### 步骤4：节点恢复操作流程（需 VPN/IOA）

```bash
# 1. 停止调度新 Pod 到问题节点
kubectl cordon <node>
# 预期: node/<node> cordoned

# 2. 驱逐节点上现有 Pod
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
# 预期: Pod 被迁移到其他节点

# 3. 修复问题（根据具体 Condition 类型）

# 4. 恢复调度
kubectl uncordon <node>
# 预期: node/<node> uncordoned

# 5. 验证
kubectl get nodes
# 预期: STATUS 变为 Ready
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
kubectl get nodes
# 预期: 所有节点 STATUS 为 Ready，无异常 Condition

kubectl describe node <node> | grep -A8 "Conditions:"
# 预期: 所有 Condition Status 为 False，或 MemoryPressure/DiskPressure/PIDPressure 为 False
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
| `cordon` 后 Pod 未重新调度 | `kubectl describe pod` 查看调度失败原因 | 其他节点资源不足 | 扩容节点或降低 Pod requests |
| `drain` 报错 `Cannot evict pod` | Pod 有 PDB 保护或为 DaemonSet | PDB 阻止驱逐或 `--ignore-daemonsets` 未生效 | 使用 `--force` 但不推荐；手动调整 PDB |

### 操作成功但异常

| 现象 | 诊断 | 根因 | 修复 |
|---|---|---|---|
| 节点周期性 NotReady | 监控 NotReady 发生时间规律 | kubelet 内存泄漏或定时任务引起系统负载飙升 | 升级 kubelet 版本；检查 cron job 执行时间 |
| Cordon 后节点仍分配到新 Pod | 检查新 Pod 是否指定了 nodeName | Pod 直接指定 nodeName 跳过调度器 | 将 nodeName 改为使用 nodeSelector 或 affinity |

## 下一步

- [节点磁盘爆满排障处理](../节点磁盘爆满排障处理/tccli%20操作.md) -- page_id `43126`
- [节点高负载排障处理](../节点高负载排障处理/tccli%20操作.md) -- page_id `43127`

## 控制台替代

[容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) -> 节点管理 -> 节点 -> 查看监控 -> Cordon/Uncordon/Drain。
