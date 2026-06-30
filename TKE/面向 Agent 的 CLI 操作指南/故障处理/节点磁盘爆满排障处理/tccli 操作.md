# 节点磁盘爆满排障处理（tccli）

> 对照官方：[节点磁盘爆满排障处理](https://cloud.tencent.com/document/product/457/43126) · page_id `43126`

## 概述

排查和处理 TKE 节点磁盘爆满（DiskPressure）问题，梳理常见磁盘占用来源和清理方法。通过 tccli 检查集群和节点实例状态，配合 kubectl 查看节点磁盘压力和资源分配情况。

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
| 查看磁盘压力 | `kubectl describe node <node> \| grep DiskPressure`（需 VPN/IOA） | 是 |
| 查看节点磁盘 | SSH 到节点 `df -h` | 是 |
| 清理镜像 | SSH 到节点 `crictl rmi --prune` | 否 |

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

### 步骤2：数据面识别 DiskPressure（需 VPN/IOA）

```bash
# 查看节点 Condition
kubectl get nodes
# 预期: 关注 STATUS 列是否有 DiskPressure 标记

kubectl describe node <node> | grep -A5 "DiskPressure\|Allocated resources"
# 预期: DiskPressure Condition 的 Status 为 True 或 False
```

```text
NAME  STATUS  AGE
...
```

### 步骤3：SSH 到节点定位磁盘占用

```bash
ssh root@<node-ip>
df -h
# 预期: 查看各分区使用率

# 定位大目录
du -sh /* 2>/dev/null | sort -rh | head -10
# 预期: 按大小排序列出占用最多的目录

# 常见占用来源
du -sh /var/lib/docker 2>/dev/null
du -sh /var/lib/containerd 2>/dev/null
du -sh /var/log 2>/dev/null
du -sh /tmp 2>/dev/null
```

### 步骤4：清理方法

**清理容器镜像（containerd 环境）**

```bash
# SSH 到节点执行
crictl rmi --prune
# 预期: 删除所有未被使用的镜像
```

**清理容器镜像（docker 环境）**

```bash
docker image prune -a -f
# 预期: 删除所有未被使用的镜像
```

**限制容器日志大小**

在容器日志配置中设置日志轮转参数，防止日志无限增长：

```yaml
# 在 /etc/docker/daemon.json（docker）或 kubelet 参数中配置
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "3"
  }
}
```

**清理临时文件**

```bash
ssh root@<node-ip>
find /tmp -type f -mtime +7 -delete
# 预期: 删除 7 天前的临时文件
```

**清理退出的容器**

```bash
crictl rm $(crictl ps -a -q --state exited)
# 预期: 清理已退出容器的残留文件
```

### 步骤5：长期方案

1. **扩容云硬盘**：通过 CBS 在线扩容数据盘（需重启节点生效）
2. **配置镜像垃圾回收**：调整 kubelet `imageGCHighThresholdPercent`（默认 85%）和 `imageGCLowThresholdPercent`（默认 80%）
3. **监控告警**：配置节点磁盘使用率告警，提前发现问题
4. **日志采集**：将容器日志接入 CLS，减少本地存储压力

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
kubectl describe node <node> | grep DiskPressure
# 预期: Status 为 False

ssh root@<node-ip> "df -h /"
# 预期: 使用率降至 85% 以下
```

```text
Name:         ...
Status:       Running
...
```

## 清理

无需特殊清理（排障页）。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|---|---|---|---|
| `crictl rmi --prune` 执行慢或无效果 | 检查镜像列表 `crictl images` | 所有镜像均有容器在使用 | 确认无冗余 Deployment 占用旧镜像；缩容后再清理 |
| 清理后磁盘使用率仍高 | `du -sh /var/lib/containerd` 仍很大 | containerd 内容存储未清理：`crictl rmi` 只清镜像，不清层数据 | 考虑重建节点或扩容磁盘 |

### 操作成功但异常

| 现象 | 诊断 | 根因 | 修复 |
|---|---|---|---|
| 磁盘使用率周期性上升 | 监控磁盘使用率变化曲线 | 日志未配置轮转或应用持续写大文件 | 配置 logrotate；限制应用日志输出 |
| 清理后 Pod 被驱逐未恢复 | `kubectl get pods` 查看驱逐 Pod | Pod 在 DiskPressure 时被 Evicted | 重新创建 Deployment 或手动恢复 Pod |

## 下一步

- [节点常见报错与处理](../节点常见报错与处理/tccli%20操作.md) -- page_id `89869`
- [节点高负载排障处理](../节点高负载排障处理/tccli%20操作.md) -- page_id `43127`

## 控制台替代

[容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) -> 节点管理 -> 节点 -> 监控 -> 磁盘使用率。
