# 使用 Systemtap 定位 Pod 异常退出原因（tccli）

> 对照官方：[使用 Systemtap 定位 Pod 异常退出原因](https://cloud.tencent.com/document/product/457/43111) · page_id `43111`

## 概述

SystemTap 是 Linux 内核动态追踪工具，可在节点层面追踪进程退出信号、内核函数调用，定位 Pod 异常退出根因（OOM、段错误、信号等）。

> **操作限制**：SystemTap 需节点 root 权限和内核调试符号。kubectl 命令需 VPN/IOA 环境。以下所有节点操作命令需 SSH 到目标节点执行。

## 前置条件

### 环境检查

```bash
tccli --version
# expected: tccli version >= 1.0.0

tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterStatus "Running"

tccli tke DescribeClusterInstances --region <Region> --ClusterId CLUSTER_ID
# expected: 获取目标节点 InstanceId

# 节点：检查 SystemTap 安装
ssh NODE_IP "stap --version"
# expected: SystemTap translator/driver version x.x
```

```json
{
  "TotalCount": "<TotalCount>",
  "Clusters": "<Clusters>",
  "ClusterId": "<ClusterId>",
  "ClusterName": "<ClusterName>",
  "ClusterDescription": "<ClusterDescription>",
  "ClusterVersion": "<ClusterVersion>"
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli/ssh 命令 | 幂等 |
|-----------|---------------|:--:|
| 查看节点实例 | `tccli tke DescribeClusterInstances` | 是 |
| 安装 SystemTap | `ssh NODE_IP "yum install -y systemtap systemtap-runtime kernel-devel-$(uname -r)"` | 是 |
| 运行 SystemTap 脚本 | `ssh NODE_IP "stap -v script.stp"` | 是 |
| 查看 Pod 所在节点 | `kubectl get pod POD_NAME -n NAMESPACE -o wide`（需 VPN/IOA） | 是 |

## 操作步骤

### 步骤 1：定位目标 Pod 所在节点

```bash
kubectl get pod POD_NAME -n NAMESPACE -o wide
# expected: NODE 列显示节点名称

# 获取节点 IP
tccli tke DescribeClusterInstances --region <Region> --ClusterId CLUSTER_ID \
    --filter "InstanceSet[?InstanceId=='INS_ID'].LanIP"
# expected: 节点内网 IP
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

### 步骤 2：安装 SystemTap（如未安装）

```bash
ssh NODE_IP "yum install -y systemtap systemtap-runtime kernel-devel-$(uname -r) kernel-debuginfo-$(uname -r)"
# expected: 安装成功
```

### 步骤 3：追踪进程退出信号

创建 SystemTap 脚本：

```systemtap
# exit-trace.stp
probe begin {
    printf("Tracing process exits...\n")
}

probe syscall.exit_group {
    if (pid() == target()) {
        printf("Process %d (%s) exiting with code %d\n", pid(), execname(), $status)
    }
}

probe signal.send {
    if (pid() == target()) {
        printf("Signal %d sent to process %d (%s)\n", $sig, $pid, execname())
    }
}
```

```bash
# SSH 到节点执行
ssh NODE_IP "stap -v exit-trace.stp -x CONTAINER_PID > /tmp/exit-trace.log"
# expected: 进程退出时记录信号和退出码
```

### 步骤 4：分析追踪结果

```bash
ssh NODE_IP "cat /tmp/exit-trace.log"
# expected: 输出类似：
#   Signal 9 sent to process 12345 (java)     # SIGKILL (OOM)
#   Process 12345 (java) exiting with code 137  # 128+9

# 交叉验证：检查 Pod 事件中的退出码
kubectl describe pod POD_NAME -n NAMESPACE | grep "Exit Code"
# expected: Exit Code: 137
```

```text
NAME  STATUS  AGE
...
```

## 验证

```bash
ssh NODE_IP "stap --version"
# expected: SystemTap 就绪

ssh NODE_IP "dmesg | grep -i oom | tail -5"
# expected: 如有 OOM 事件，含 killed process 信息
```

## 清理

```bash
ssh NODE_IP "rm -f /tmp/exit-trace.log exit-trace.stp"
# expected: 清理追踪文件
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| SystemTap 安装失败 | `ssh NODE_IP "rpm -q kernel-devel-$(uname -r)"` | 内核开发包版本不匹配 | `yum install kernel-devel-$(uname -r)` 匹配内核版本 |
| stap 运行报 semantic error | `ssh NODE_IP "stap -v -p2 script.stp"` | 内核符号缺失 | 安装 kernel-debuginfo 包 |
| 追踪不到目标进程 | `ssh NODE_IP "ps aux | grep CONTAINER_ID"` | 进程 PID 变化（容器重启） | 重新获取最新 PID |
| 权限不足 | `ssh NODE_IP "sudo stap ..."` | 非 root 用户 | 使用 sudo 或 root 运行 stap |

### 退出信号对照表

| 信号 | 值 | 退出码 | 含义 | 常见根因 |
|------|---|--------|------|---------|
| SIGKILL | 9 | 137 | 强制终止 | OOM Killer / `kubectl delete --force` |
| SIGSEGV | 11 | 139 | 段错误 | 内存访问越界 / null 引用 |
| SIGTERM | 15 | 143 | 优雅终止 | 正常关闭 / preStop Hook |
| SIGABRT | 6 | 134 | 异常终止 | 应用调用 abort() |

## 下一步

- [通过 Exit Code 定位 Pod 异常退出和重启原因](../通过%20Exit%20Code%20定位%20Pod%20异常退出和重启原因/tccli%20操作.md) -- page_id `43125`
- [Pod 异常排查概述](../../Pod%20异常排查概述/tccli%20操作.md) -- page_id `42945`
- [节点高负载排障处理](../../../节点高负载排障处理/tccli%20操作.md) -- page_id `43127`

## 控制台替代

[容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) -> 集群 -> 节点 -> 登录 -> 执行诊断命令。
