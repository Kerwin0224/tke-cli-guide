# Nginx Ingress 高并发实践（tccli）

> 对照官方：[Nginx Ingress 高并发实践](https://cloud.tencent.com/document/product/457/48142) · page_id `48142`

## 概述

针对高并发场景，通过调整 Nginx Ingress Controller 的 worker 进程数、连接数、keepalive 等参数提升吞吐量。TKE 的 NginxIngress 组件支持通过 ConfigMap 和 initContainer 系统参数调优。

## 前置条件

- [环境准备](../../../环境准备.md)
- 已安装 [NginxIngress 组件](../../../应用配置/组件和应用管理/组件管理/NginxIngress%20说明/tccli%20操作.md)

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 查看 Ingress Controller Pod | `kubectl get pods -n kube-system -l app=nginx-ingress` | 是 |
| 修改 worker 配置 | `kubectl edit configmap nginx-configuration -n kube-system` | 是 |
| 添加 initContainer sysctl | `kubectl edit deployment nginx-ingress-controller -n kube-system` | 否 |
| 压测验证 | `ab -n 10000 -c 100 http://<Ingress-IP>/` | 是 |

## 操作步骤

### 1. 调整 Nginx Worker 配置

```bash
kubectl edit configmap nginx-configuration -n kube-system
```

添加或修改以下配置：

```yaml
data:
  worker-processes: "4"
  worker-connections: "65535"
  worker-rlimit-nofile: "1048576"
  keepalive: "200"
  keepalive-requests: "10000"
  backlog-net-somaxconn: "32768"
```

```bash
kubectl rollout restart deployment nginx-ingress-controller -n kube-system
```

```output
deployment.apps/nginx-ingress-controller restarted
```

### 2. 系统级参数调优

为 Ingress Controller Deployment 添加 initContainer：

```yaml
initContainers:
- name: init-sysctl
  image: busybox:1.28
  command: ['sh', '-c', 'sysctl -w net.core.somaxconn=32768 && sysctl -w net.ipv4.tcp_max_syn_backlog=8192']
  securityContext:
    privileged: true
```

```bash
kubectl apply -f ingress-controller-patch.yaml
```

```text
# command executed successfully
```

### 3. 压测验证

```bash
kubectl run ab-test --image=httpd:alpine --rm -it --restart=Never -- ab -n 10000 -c 100 http://<Ingress-Service-IP>/
```

```output
Requests per second:    5234.12 [#/sec]
Time per request:       19.105 [ms]
```

## 验证

```bash
kubectl get configmap nginx-configuration -n kube-system -o yaml | grep -E "worker-processes|worker-connections|keepalive:"
kubectl exec -n kube-system <nginx-ingress-pod> -- nginx -T | grep worker_processes
```

```text
NAME  STATUS  AGE
...
```

## 清理

恢复默认配置：

```bash
kubectl rollout undo deployment nginx-ingress-controller -n kube-system
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl` 命令超时或连接被拒绝 | `tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]'`（控制面返回 Running） | CAM 策略 strategyId:240463971 拒绝公网端点，数据面需内网访问 | 通过 VPN/IOA 接入 VPC 内网后执行 kubectl；或使用控制台替代 |
| Connection Refused 偶发 | `kubectl exec -n kube-system <nginx-pod> -- nginx -T \| grep worker_connections` | somaxconn 或 worker-connections 过低，高并发时连接队列溢出 | 提升 `backlog-net-somaxconn` 和 `worker-connections` |
| 延迟增加 | `kubectl exec -n kube-system <nginx-pod> -- nginx -T \| grep keepalive` | 连接频繁建立释放，keepalive 未启用或过短 | 调整 `keepalive` 参数减少连接建立开销 |

## 下一步

- [Nginx Ingress 最佳实践](../Nginx%20Ingress%20最佳实践/tccli%20操作.md)
- [Nginx 升级最佳实践](../Nginx%20升级最佳实践/tccli%20操作.md)

## 控制台替代

控制台：集群 → 组件管理 → NginxIngress → 更新配置。
