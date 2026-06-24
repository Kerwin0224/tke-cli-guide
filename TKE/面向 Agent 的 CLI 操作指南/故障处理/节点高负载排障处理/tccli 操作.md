# 节点高负载排障处理（tccli）

> 对照官方：[节点高负载排障处理](https://cloud.tencent.com/document/product/457/43127) · page_id `43127`

## 概述

排查和处理 TKE 节点高负载问题（CPU、内存、IO 压力），定位负载来源并采取措施。通过 tccli 检查集群和节点状态，配合 kubectl 查看资源使用和 Pod 分布。

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
| 查看 Pod 资源使用 | `kubectl top pods -A --sort-by=cpu`（需 VPN/IOA） | 是 |
| 查看节点资源 | `kubectl top nodes`（需 VPN/IOA） | 是 |
| SSH 查看系统负载 | `ssh root@<node> "uptime; top -bn1 \| head -20"` | 是 |

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
      "InstanceState": "running",
      "LanIP": "10.0.1.10"
    }
  ]
}
```

### 步骤2：数据面定位高负载来源（需 VPN/IOA）

```bash
# 查看各节点资源使用
kubectl top nodes
# 预期: CPU(cores) 和 MEMORY(bytes) 列

# 按 CPU 排序 Pod
kubectl top pods -A --sort-by=cpu | head -20
# 预期: 定位 CPU 使用最高的 Pod

# 按内存排序 Pod
kubectl top pods -A --sort-by=memory | head -20
# 预期: 定位内存使用最高的 Pod

# 查看节点上 Pod 分布
kubectl get pods -A -o wide --field-selector spec.nodeName=<node>
# 预期: 该节点上所有 Pod 列表
```

```text
NAME  STATUS  AGE
...
```

### 步骤3：SSH 到节点深入诊断

```bash
ssh root@<node-ip>

# 系统负载
uptime
# 预期: load average 与 CPU 核数对比判断是否过载

# 实时进程
top -bn1 | head -20
# 预期: CPU 和 MEM 使用最高的进程

# IO 等待
iostat -x 1 3
# 预期: %iowait 判断是否 IO 瓶颈
```

### 步骤4：按负载类型处理

**CPU 高负载**

1. 定位高 CPU Pod：`kubectl top pods -A --sort-by=cpu`
2. 检查该 Pod 是否配置了 CPU limits
3. 临时措施：`kubectl scale deploy <name> --replicas=<n>` 减少负载
4. 长期方案：配置 HPA 自动伸缩；调整 CPU limits

**内存高负载**

1. 定位高内存 Pod：`kubectl top pods -A --sort-by=memory`
2. 检查是否有内存泄漏：观察 Pod 内存随时间持续增长
3. 临时措施：重启问题 Pod 释放内存
4. 长期方案：增加 memory limits；排查应用内存泄漏

**IO 高负载**

1. SSH 到节点执行 `iostat -x 1` 观察 `%util` 和 `await`
2. 定位高 IO 进程：`iotop` 或 `pidstat -d 1`
3. 方案：使用 SSD 云硬盘；配置 IO 隔离（blkio cgroup）

**系统 Load 高（CPU 等待队列过长）**

1. Load average 超过 CPU 核数的 2-3 倍即为过载
2. 原因：CPU 密集型任务过多或 IO 等待严重
3. 方案：减少节点 Pod 密度；迁移部分 Pod 到其他节点

### 步骤5：临时紧急处理

```bash
# 暂停问题工作负载
kubectl scale deploy <name> --replicas=0 -n <ns>

# 隔离问题节点
kubectl cordon <node>

# 或扩容节点
# 在控制台或通过 tccli 添加节点
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
kubectl top nodes
# 预期: 各节点 CPU/MEM 使用率在正常范围内

kubectl get nodes
# 预期: 所有节点 STATUS 为 Ready
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
| `kubectl top` 返回 `metrics not available` | `kubectl get pods -n kube-system -l k8s-app=metrics-server` 检查 | metrics-server 组件未安装或异常 | 安装或重启 metrics-server |
| 节点 CPU 使用低但 Load 高 | SSH 到节点 `ps aux` 查看 D 状态进程 | IO 等待导致 Load 虚高 | 排查磁盘 IO 性能问题 |

### 操作成功但异常

| 现象 | 诊断 | 根因 | 修复 |
|---|---|---|---|
| HPA 频繁扩缩 | 查看 HPA 事件和 metrics | 阈值设置过于敏感或负载波动大 | 调整 HPA 阈值和冷却时间；使用 `--horizontal-pod-autoscaler-tolerance` |
| 扩容节点后新 Pod 仍调度到高负载节点 | 检查新 Pod 的 nodeSelector/affinity | Pod 亲和性规则约束了调度范围 | 调整亲和性规则使 Pod 可分散到新节点 |

## 下一步

- [节点常见报错与处理](../节点常见报错与处理/tccli%20操作.md) -- page_id `89869`
- [节点内存碎片化排障处理](../节点内存碎片化排障处理/tccli%20操作.md) -- page_id `43128`

## 控制台替代

[容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) -> 节点管理 -> 节点 -> 监控 -> CPU/内存/磁盘使用率。
