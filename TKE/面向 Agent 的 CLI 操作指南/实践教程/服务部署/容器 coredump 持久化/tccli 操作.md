# 容器 coredump 持久化（tccli）

> 对照官方：[容器 coredump 持久化](https://cloud.tencent.com/document/product/457/49745) · page_id `49745`

## 概述

配置容器 coredump 文件持久化到宿主机目录，避免容器退出后丢失。通过 hostPath + initContainer sysctl 设置 core_pattern。

## 前置条件

- [环境准备](../../../环境准备.md)

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 挂载 coredump 目录 | `hostPath` `/var/coredump` → 容器内 `/coredump` | 是 |
| 设置 core_pattern | initContainer `sysctl kernel.core_pattern` | 是 |
| 调整 ulimit | Pod `securityContext` 或无限制 | 是 |

## 操作步骤

### 完整配置

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: coredump-test
spec:
  initContainers:
  - name: sysctl
    image: busybox
    command: ["sh", "-c", "sysctl -w kernel.core_pattern=/var/coredump/core.%e.%p.%t"]
    securityContext:
      privileged: true
  containers:
  - name: app
    image: alpine:latest
    command: ["sh", "-c", "ulimit -c unlimited && sleep 10 && kill -SIGSEGV \$\$"]
    volumeMounts:
    - name: coredump
      mountPath: /var/coredump
  volumes:
  - name: coredump
    hostPath:
      path: /var/coredump
  restartPolicy: Never
```

```bash
kubectl apply -f coredump-pod.yaml
```

```text
# command executed successfully
```

```output
pod/coredump-test created
```

### 查看 coredump

```bash
kubectl exec coredump-test -c app -- ls /var/coredump/
# 或登录宿主机查看
```

## 验证

```bash
kubectl get pod coredump-test
kubectl logs coredump-test -c app
```

```text
NAME  STATUS  AGE
...
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl` 连接超时/拒绝 | `tccli tke DescribeClusters` 返回集群 Running 但 kubectl 不可达 | CAM 策略 240463971 拒绝公网访问 APIServer 端点 | 通过 VPN/IOA 接入内网后执行 kubectl 命令 |
| initContainer sysctl 报权限拒绝 | `kubectl logs <pod> -c sysctl` | initContainer 未设置 privileged 或宿主机内核参数只读 | 确认 `securityContext.privileged: true` 已设置 |
| /var/coredump 下无文件 | `kubectl exec <pod> -c app -- ls /var/coredump/` | core_pattern 未生效或 ulimit -c 仍为 0 | 确认 initContainer 执行成功；容器内执行 `ulimit -c unlimited` |
| hostPath 挂载失败 | `kubectl describe pod <pod>` 查看 Events | 节点 `/var/coredump` 目录不存在 | 登录节点 `mkdir -p /var/coredump` 或改用 PVC |

## 清理

```bash
kubectl delete pod coredump-test
```

## 下一步

- [工作负载平滑升级](../工作负载平滑升级/tccli%20操作.md)
- [使用 Systemtap 定位 Pod 异常退出原因](../../../故障处理/Pod%20状态异常与处理措施/使用%20Systemtap%20定位%20Pod%20异常退出原因/tccli%20操作.md)

## 控制台替代

控制台：工作负载 → 数据卷 → 添加 hostPath。
