# Nginx Ingress 偶现 Connection Refused（tccli）

> 对照官方：[Nginx Ingress 偶现 Connection Refused](https://cloud.tencent.com/document/product/457/71216) · page_id `71216`

## 概述

Nginx Ingress 偶现 `Connection Refused` 的排查流程：检查 Ingress Controller Pod 是否重启中、连接队列溢出、upstream 后端健康检查失败。通过 tccli 检查集群状态，配合 kubectl 查看 Controller 状态和日志进行诊断。

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
| 查看 Controller Pod 状态 | `kubectl get pods -n kube-system -l app=nginx-ingress`（需 VPN/IOA） | 是 |
| 查看 Nginx 日志 | `kubectl logs -n kube-system <nginx-pod>`（需 VPN/IOA） | 是 |
| 修改 Nginx 配置 | `kubectl edit configmap nginx-configuration -n kube-system`（需 VPN/IOA） | 是 |

## 操作步骤

### 步骤1：控制面检查集群

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

### 步骤2：数据面检查 Controller 状态（需 VPN/IOA）

```bash
kubectl get pods -n kube-system -l app=nginx-ingress -o wide
# 预期: 所有 Pod Running，RESTARTS 计数不频繁增长

kubectl logs -n kube-system -l app=nginx-ingress --tail=100 | grep -i "refused\|error\|timeout"
# 预期: 定位 Connection Refused 出现的时间点和上下文
```

```text
NAME  STATUS  AGE
...
```

### 步骤3：常见原因排查与修复（需 VPN/IOA）

| 可能原因 | 诊断方法 | 根因 | 修复 |
|---|---|---|---|
| Worker 进程数不足 | 高并发时连接队列满 | CPU 核数多但 worker-processes 默认为 auto | ConfigMap 设置 `worker-processes: "8"` |
| 连接队列溢出 | `netstat -s \| grep overflow` 计数增长 | `somaxconn` 默认值过小 | `backlog-net-somaxconn: "32768"` |
| upstream 响应超时 | 后端服务处理慢 | 默认 `proxy-read-timeout` 过短 | `proxy-read-timeout: "60"` |
| 连接未复用 | 大量短连接 | `keepalive` 未开启或值过小 | `keepalive: "200"` |
| Controller 频繁重启 | `kubectl describe pod` 显示探针失败 | OOM 或配置冲突 | 增加 memory limits；检查日志 |

### 步骤4：优化 ConfigMap 配置（需 VPN/IOA）

```bash
kubectl edit configmap nginx-configuration -n kube-system
```

添加以下配置项：

```yaml
data:
  worker-processes: "8"
  worker-connections: "65535"
  keepalive: "200"
  proxy-connect-timeout: "30"
  proxy-read-timeout: "60"
  backlog-net-somaxconn: "32768"
```

修改后重启 Controller Pod 使配置生效。

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
# 压测验证
for i in $(seq 1 100); do
  curl -s -o /dev/null -w "%{http_code}\n" http://<Ingress-IP>/healthz
done | sort | uniq -c
# 预期: 100 次请求全部返回 200，无 Connection Refused
```

## 清理

无需特殊清理（排障页）。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|---|---|---|---|
| ConfigMap 修改不生效 | `kubectl describe configmap` 确认配置已更新 | Controller 未重载配置 | 删除 Controller Pod 触发重启：`kubectl delete pod -n kube-system -l app=nginx-ingress` |
| Controller Pod 无法启动 | `kubectl describe pod` 显示配置语法错误 | ConfigMap 中 YAML 格式错误 | 检查缩进和类型（数字不加引号，字符串加引号） |

### 操作成功但异常

| 现象 | 诊断 | 根因 | 修复 |
|---|---|---|---|
| 优化后仍有偶发 Refused | 持续压测观察 Nginx 错误日志 | 后端服务自身偶发故障 | 检查后端 Pod 就绪状态和资源限制 |
| 高峰期 Refused 增多 | 监控 Nginx 连接数 | 并发连接数超过 worker-connections | 扩容 Controller 副本：`kubectl scale deploy -n kube-system nginx-ingress-controller --replicas=3` |

## 下一步

- [Nginx Ingress 高并发实践](../../实践教程/网络/Nginx%20Ingress%20高并发实践/tccli%20操作.md)
- [Nginx Ingress 最佳实践](../../实践教程/网络/Nginx%20Ingress%20最佳实践/tccli%20操作.md)

## 控制台替代

[容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) -> 组件管理 -> NginxIngress -> 查看日志。
