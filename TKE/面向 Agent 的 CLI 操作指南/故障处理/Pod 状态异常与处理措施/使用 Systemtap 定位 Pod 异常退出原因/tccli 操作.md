# 使用 Systemtap 定位 Pod 异常退出原因（tccli）

> 对照官方：[使用 Systemtap 定位 Pod 异常退出原因](https://cloud.tencent.com/document/product/457/43111) · page_id `43111`

## 概述

Systemtap 是 Linux 内核级动态追踪工具，可在不修改代码的情况下追踪系统调用、内核函数、信号传递等。当 Pod 异常退出但日志信息不足以定位根因时（如 SIGKILL 来源不明、进程被谁杀死），可在节点上使用 Systemtap 追踪进程退出原因。

> **kubectl 数据面不可达**：CAM拒绝公网端点(strategyId:240463971)，内网需VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

## 前置条件

- [环境准备](../../../环境准备.md)：`tccli` 已配置
- 节点 SSH 可登录（需 VPN/IOA 内网环境）
- 节点操作系统为 Linux，支持安装 Systemtap
- 确认节点上已安装 kernel-devel 包匹配当前内核版本
- 集群为 MANAGED_CLUSTER 类型，K8s 1.30.0，ap-guangzhou

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 查看集群状态 | `tccli tke DescribeClusterStatus --region <Region> --ClusterIds '["CLUSTER_ID"]'` | 是 |
| 查看集群实例 | `tccli tke DescribeClusterInstances --region <Region> --ClusterId CLUSTER_ID` | 是 |
| 查看 Pod 退出码 | `kubectl describe pod POD_NAME -n NAMESPACE`（需 VPN/IOA） | 是 |
| 查看前一容器日志 | `kubectl logs POD_NAME -n NAMESPACE --previous`（需 VPN/IOA） | 是 |
| 安装 Systemtap | 登录节点 `yum install systemtap`（需 SSH） | 否 |
| 追踪进程信号 | `stap -e 'probe signal.send { ... }'`（需 SSH） | 否 |
| 查看 OOM 记录 | `dmesg \| grep -i oom`（需 SSH） | 是 |

## 操作步骤

### 1. 控制面：确认集群和节点状态

```bash
tccli tke DescribeClusterStatus --region <Region> --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterState "Running"
```

```json
{
  "ClusterStatusSet": "<ClusterStatusSet>",
  "ClusterId": "<ClusterId>",
  "ClusterState": "<ClusterState>",
  "ClusterInstanceState": "<ClusterInstanceState>",
  "ClusterBMonitor": "<ClusterBMonitor>",
  "ClusterInitNodeNum": "<ClusterInitNodeNum>"
}
```

```bash
tccli tke DescribeClusterInstances --region <Region> --ClusterId CLUSTER_ID \
    --filter "InstanceSet[].InstanceState"
# expected: 节点状态 running
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

### 2. 数据面：获取 Pod 信息确定目标节点（需 VPN/IOA）

```bash
# 查看 Pod 所在节点和退出码
kubectl get pod POD_NAME -n NAMESPACE -o wide
# expected: NODE 列显示 Pod 所在节点名

# 查看退出详情
kubectl describe pod POD_NAME -n NAMESPACE | grep -A10 "Last State"
# expected: Exit Code 和 Reason（如 OOMKilled, Error）
```

```text
NAME  STATUS  AGE
...
```

### 3. 数据面：SSH 登录目标节点并安装 Systemtap（需 VPN/IOA）

```bash
# 登录节点（IP 从 kubectl get pod -o wide 获取）
ssh root@NODE_ID
# expected: 成功登录

# 安装 Systemtap 及依赖
yum install -y systemtap systemtap-runtime kernel-devel-$(uname -r) kernel-debuginfo-$(uname -r)
# expected: 安装完成
```

```text
NAME  STATUS  AGE
...
```

**注意：** `kernel-debuginfo` 包可能需要从 debuginfo 仓库安装。若安装失败，可跳过，部分 probe 仍可工作。验证安装：

```bash
stap -V
# expected: Systemtap translator/driver (version x.x)

stap -e 'probe begin { printf("ready\n"); exit() }'
# expected: ready
```

### 4. 数据面：运行追踪脚本定位退出原因

**场景一：追踪 SIGKILL 信号来源（OOM/Pod 驱逐常见）**

```bash
# 追踪谁向特定 PID 发送了 SIGKILL（替换 PID）
stap -e '
probe signal.send {
  if (sig_name == "SIGKILL") {
    printf("[%s] %s(pid:%d uid:%d) sent SIGKILL to pid=%d (%s)\n",
           ctime(gettimeofday_s()), execname(), pid(), uid(), sig_pid, pid_name)
  }
}' 
# expected: 捕获 SIGKILL 信号发送记录，显示发送进程名、PID 和 UID
```

**场景二：追踪进程退出原因（exit、被信号杀死）**

```bash
# 追踪特定进程的退出（替换 PID）
stap -e '
probe syscall.exit_group {
  if (pid() == target()) {
    printf("process %d (%s) exited with code %d\n", pid(), execname(), $status)
  }
}
probe kernel.function("do_exit") {
  if (pid() == target()) {
    printf("process %d (%s) do_exit called\n", pid(), execname())
    print_backtrace()
  }
}' -x PID
# expected: 打印退出码和调用栈
```

**场景三：追踪容器 OOM Killer 事件**

```bash
# 监控内核 OOM Killer 事件
stap -e '
probe kernel.function("oom_kill_process") {
  printf("[%s] OOM kill: pid=%d comm=%s\n",
         ctime(gettimeofday_s()), $p->pid, kernel_string($p->comm))
  printf("  oom_score_adj: %d\n", $p->signal->oom_score_adj)
}
probe kernel.function("dump_header") {
  printf("[%s] OOM dump header\n", ctime(gettimeofday_s()))
}'
# expected: 捕获 OOM Killer 触发记录，显示被杀进程信息
```

**场景四：追踪段错误（SIGSEGV）**

```bash
# 追踪 SIGSEGV 信号
stap -e '
probe signal.send {
  if (sig_name == "SIGSEGV" && sig_pid == target()) {
    printf("[%s] SIGSEGV sent to %s(pid=%d)\n",
           ctime(gettimeofday_s()), pid_name, sig_pid)
    print_ubacktrace()
  }
}' -x PID
# expected: 打印发送 SIGSEGV 时的用户态调用栈
```

### 5. 数据面：辅助诊断命令（不需 Systemtap）

```bash
# 查看内核 OOM 记录
dmesg | grep -i "out of memory\|oom"
# expected: 若有 OOM，显示 oom_kill_process 记录和被杀进程信息

# 查看系统日志
journalctl -k | grep -i oom
# expected: 内核日志中 OOM 相关记录

# 查看容器 cgroup 内存限制
cat /sys/fs/cgroup/memory/kubepods/*/memory.limit_in_bytes 2>/dev/null
# expected: Pod 级 cgroup 内存限制值

# 使用 crictl 查看容器信息（容器运行时为 containerd）
crictl ps -a | grep <container-id-prefix>
# expected: 容器状态和创建时间
```

## 验证

### 控制面（tccli）

```bash
tccli tke DescribeClusterStatus --region <Region> --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterState "Running"
```

```json
{
  "ClusterStatusSet": "<ClusterStatusSet>",
  "ClusterId": "<ClusterId>",
  "ClusterState": "<ClusterState>",
  "ClusterInstanceState": "<ClusterInstanceState>",
  "ClusterBMonitor": "<ClusterBMonitor>",
  "ClusterInitNodeNum": "<ClusterInitNodeNum>"
}
```

### 数据面（需 VPN/IOA）

```bash
# 通过 kubectl describe 确认退出码与追踪结果一致
kubectl describe pod POD_NAME -n NAMESPACE | grep -E "Exit Code|Reason|OOMKilled"
# expected: 退出码与 Systemtap 追踪结果匹配

# 若已修复根因，观察 Pod 不再异常退出
kubectl get pod POD_NAME -n NAMESPACE -w
# expected: Pod 稳定 Running，RESTARTS 不增长
```

```text
NAME  STATUS  AGE
...
```

## 清理

排障页通常无需清理。如需移除 Systemtap（不必要时可卸载）：

```bash
# 卸载 Systemtap（保留 kernel-devel 供后续使用）
yum remove systemtap systemtap-runtime
```

**注意：** kernel-devel/kernel-debuginfo 保留不会影响系统运行，建议保留以备再次排查。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `stap -V` 报错 "command not found" | `rpm -qa \| grep systemtap` | Systemtap 未安装 | `yum install -y systemtap systemtap-runtime` |
| `stap` 运行报 "kernel-devel not found" | `rpm -qa \| grep kernel-devel` | kernel-devel 版本与当前内核不匹配 | `yum install -y kernel-devel-$(uname -r)`；若仓库无匹配版本，需升级内核或使用其他调试工具 |
| `stap` 报 "pass 4: compilation failed" | `uname -r` 对比 `ls /usr/src/kernels/` | kernel-debuginfo 缺失导致符号名偏移计算失败 | 从 [debuginfo.centos.org](http://debuginfo.centos.org/) 或发行版仓库安装 `kernel-debuginfo-$(uname -r)` |
| `ssh root@NODE_ID` 连接超时 | `ping NODE_ID` 测试网络 | 节点内网 IP 不可达，VPN/IOA 未连接 | 连接 VPN/IOA 后重试；确认节点安全组允许 SSH (22) 入站 |
| `stap -e` 运行时 "ERROR: probe ... registration error" | `sudo stap -v -e 'probe begin { exit() }'` 测试最小脚本 | 内核模块加载权限不足或 SELinux 限制 | 使用 `sudo` 运行；检查 SELinux：`getenforce`，临时 `setenforce 0`；系统层面 `setsebool -P stap-server 1` |

### 创建成功但状态异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| SIGKILL 来源追踪到 PID 为 crio/containerd 进程 | `ps aux \| grep <pid>` 确认发送 SIGKILL 的进程 | 容器运行时按 OOM 判定发送 SIGKILL（cgroup OOM） | 检查内存限制过小或应用内存泄漏；`kubectl describe pod` 确认 OOMKilled 标记；增加 memory limits |
| SIGKILL 来源追踪到 PID 为 kubelet 进程 | `kubectl describe pod POD_NAME -n NAMESPACE \| grep -A5 Conditions` | Pod 被 kubelet 驱逐（Eviction），如 DiskPressure/MemoryPressure | 排查节点资源压力：[节点排障页](../../节点常见报错与处理/tccli%20操作.md)；清理节点磁盘、扩容节点 |
| SIGSEGV 段错误追踪到非法内存访问 | Systemtap `print_ubacktrace()` 输出的调用栈 | 应用程序代码 bug：空指针解引用、栈溢出、缓冲区溢出 | 根据调用栈定位代码行修复；开启 core dump 辅助分析：`ulimit -c unlimited` |
| `dmesg` 和 Systemtap 均无 OOM 记录但 Pod 显示 OOMKilled | `kubectl describe pod POD_NAME -n NAMESPACE \| grep -B2 -A2 containerStatuses` | cgroup OOM（非全局 OOM）：容器内存超限但节点内存充足 | 增大 Pod memory limits 或优化应用内存使用；监控 `kubectl top pod` 内存趋势 |
| Systemtap 追踪无输出（Pod 已退出） | `kubectl logs POD_NAME -n NAMESPACE --previous --tail=50` | 进程已结束，无法实时追踪 | 改用离线分析：`kubectl logs --previous` + `dmesg` + [Exit Code 排障页](../通过%20Exit%20Code%20定位%20Pod%20异常退出和重启原因/tccli%20操作.md)；为新创建的 Pod 设置 Systemtap 追踪 |

## 下一步

- [通过 Exit Code 定位 Pod 异常退出和重启原因](../通过%20Exit%20Code%20定位%20Pod%20异常退出和重启原因/tccli%20操作.md) -- page_id `43125`
- [Pod 处于 CrashLoopBackOff 状态](../Pod%20处于%20CrashLoopBackOff%20状态/tccli%20操作.md) -- page_id `42948`
- [容器 coredump 持久化](../../../实践教程/服务部署/容器%20coredump%20持久化/tccli%20操作.md) -- page_id `43115`

## 控制台替代

无控制台界面。Systemtap 为节点级内核追踪工具，需 SSH 登录节点操作。
