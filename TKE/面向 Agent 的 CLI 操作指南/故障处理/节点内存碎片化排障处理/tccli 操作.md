# 节点内存碎片化排障处理（tccli）

> 对照官方：[节点内存碎片化排障处理](https://cloud.tencent.com/document/product/457/43128) · page_id `43128`

## 概述

节点内存碎片化导致无法分配大块连续内存，表现为 Pod 调度失败或 OOM，即使节点总内存有余量。通过 tccli 检查集群和节点状态，SSH 到节点查看 `/proc/buddyinfo` 和内核参数来诊断和缓解。

> **kubectl 数据面不可达**：CAM拒绝公网端点(strategyId:240463971)，内网需VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

## 前置条件

- [环境准备](../../环境准备.md)
- 节点 SSH 可登录

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
| 查看内存碎片 | SSH 到节点 `cat /proc/buddyinfo` | 是 |
| 查看内存信息 | SSH 到节点 `cat /proc/meminfo` | 是 |
| 触发内存压缩 | SSH 到节点 `echo 1 > /proc/sys/vm/compact_memory` | 否 |

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

### 步骤2：数据面症状识别（需 VPN/IOA）

```bash
kubectl top nodes
# 预期: 内存使用率正常但 Pod 调度失败，提示"insufficient memory"

kubectl describe pod <pod> | grep -A5 Events
# 预期: FailedScheduling 显示内存不足，但节点实际有余量
```

```text
Name:         ...
Status:       Running
...
```

### 步骤3：SSH 到节点诊断内存碎片

```bash
ssh root@<node-ip>

# 查看 buddyinfo（内存伙伴系统信息）
cat /proc/buddyinfo
# 预期: 关注高阶（大块）内存的可用数量。例如 order 9（2^9=2MB页面）或 order 10（4MB页面）为 0 表示大块内存碎片化

# 查看内存使用概况
cat /proc/meminfo | grep -E "^(MemTotal|MemFree|MemAvailable|CmaTotal|CmaFree)"
# 预期: MemAvailable 远小于 MemFree + Cached 表示碎片化严重

# 查看透明大页状态
cat /sys/kernel/mm/transparent_hugepage/enabled
# 预期: [always] madvise never
```

### 步骤4：解读 buddyinfo

`/proc/buddyinfo` 输出格式：

```
Node 0, zone Normal 1234 567 234 89 45 12 3 2 1 0 0
```

每列代表 order N（2^N 个连续页面）的空闲块数。从右往左数字越大表示大块连续内存。
- 右边为 0：无法分配大块连续内存，碎片化严重
- 运行时内存碎片化是正常现象，但可采取措施缓解

### 步骤5：缓解方法

**手动触发内存压缩**

```bash
ssh root@<node-ip>
echo 1 > /proc/sys/vm/compact_memory
# 预期: 无输出，内核执行内存压缩
```

操作后重新检查 `/proc/buddyinfo`，高阶栏应有增加。

**启用透明大页（THP）**

```bash
echo always > /sys/kernel/mm/transparent_hugepage/enabled
# 预期: 启用透明大页，减少内存碎片
```

**调整内核参数**

在 `/etc/sysctl.conf` 中配置：

```
vm.min_free_kbytes=262144     # 预留更多空闲内存
vm.zone_reclaim_mode=0        # 禁用 zone reclaim
vm.extfrag_threshold=500      # 降低外部碎片阈值
```

执行 `sysctl -p` 生效。

### 步骤6：长期预防

1. **使用 HugePages**：为数据库等大内存应用预留 HugePages
2. **重启节点**：定期重启节点使内存回到初始状态
3. **升级内核**：较新的内核版本内存管理更优
4. **避免内存钉扎**：限制长期运行的 Pod 占用大量小页

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
ssh root@<node-ip> "cat /proc/buddyinfo"
# 预期: 高阶栏有空闲块（特别是 order 9、10）

kubectl top nodes
# 预期: 内存使用率在正常范围，Pod 可正常调度
```

```text
# command executed successfully
```

## 清理

无需特殊清理（排障页）。内存压缩操作无需回滚。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|---|---|---|---|
| `compact_memory` 执行后 buddyinfo 无变化 | 查看 `dmesg` 是否有内存分配失败日志 | 内存已严重碎片化，压缩无法释放连续块 | 重启节点是最彻底的解决方法 |
| `echo always > .../enabled` Permission denied | 检查文件权限 | 该接口需要 root 权限 | 使用 `sudo` 或以 root 用户执行 |

### 操作成功但异常

| 现象 | 诊断 | 根因 | 修复 |
|---|---|---|---|
| 内存压缩后 Pod 仍无法调度 | 检查 Pod 的 memory requests 是否超过单次可分配块 | 应用请求的内存超过单个 NUMA 节点可供分配的最大连续块 | 降低 memory requests 或将 Pod 拆分为多个小内存 Pod |
| 内存碎片周期性出现 | 监控 buddyinfo 随时间变化 | 长期运行的应用导致内存碎片累积 | 配置定期内存压缩 cron job；定期滚动重启节点 |

## 下一步

- [节点高负载排障处理](../节点高负载排障处理/tccli%20操作.md) -- page_id `43127`
- [节点常见报错与处理](../节点常见报错与处理/tccli%20操作.md) -- page_id `89869`

## 控制台替代

无控制台界面，需 SSH 登录节点操作。
