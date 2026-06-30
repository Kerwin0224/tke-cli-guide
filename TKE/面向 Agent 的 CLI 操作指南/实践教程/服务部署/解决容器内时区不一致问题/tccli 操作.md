# 解决容器内时区不一致问题（tccli）

> 对照官方：[解决容器内时区不一致问题](https://cloud.tencent.com/document/product/457/41877) · page_id `41877`

## 概述

容器默认时区为 UTC，解决容器内时区不一致的三种方式：挂载宿主机 `/etc/localtime`、设置 `TZ` 环境变量、在 Dockerfile 中设置时区。

## 前置条件

- [环境准备](../../../环境准备.md)

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 挂载宿主机时区 | `hostPath` `/etc/localtime` → `/etc/localtime` | 是 |
| 环境变量方式 | `env: [{name: TZ, value: Asia/Shanghai}]` | 是 |
| 查看容器时区 | `kubectl exec <pod> -- date` | 是 |

## 操作步骤

### 方式1：hostPath 挂载（推荐）

```yaml
spec:
  containers:
  - name: app
    image: nginx:alpine
    volumeMounts:
    - name: timezone
      mountPath: /etc/localtime
      readOnly: true
  volumes:
  - name: timezone
    hostPath:
      path: /etc/localtime
```

```bash
kubectl apply -f timezone-hostpath.yaml
```

### 方式2：环境变量

```yaml
env:
- name: TZ
  value: Asia/Shanghai
```

### 方式3：Dockerfile

```dockerfile
FROM alpine
RUN apk add tzdata && cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
```

### 验证

```bash
kubectl exec <pod> -- date
```

```output
Fri Jun 05 14:30:00 CST 2026
```

## 验证

```bash
kubectl exec <pod> -- date +%Z
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl` 连接超时/拒绝 | `tccli tke DescribeClusters` 返回集群 Running 但 kubectl 不可达 | CAM 策略 240463971 拒绝公网访问 APIServer 端点 | 通过 VPN/IOA 接入内网后执行 kubectl 命令 |
| TZ 环境变量无效 | `kubectl exec <pod> -- date` 仍显示 UTC | 镜像不读取 TZ 变量 | 改用挂载 /etc/localtime 方式 |
| /etc/localtime 挂载失败 | `kubectl describe pod <pod>` 查看 Events | 节点时区文件路径不同 | 检查宿主机 `/etc/localtime` 路径是否存在 |

## 清理

```bash
kubectl delete pod <pod>
```

## 下一步

- [docker run 参数适配](../docker%20run%20参数适配/tccli%20操作.md)
- [资源合理分配](../资源合理分配/tccli%20操作.md)

## 控制台替代

控制台：工作负载 → 环境变量 → 添加 TZ=Asia/Shanghai。
