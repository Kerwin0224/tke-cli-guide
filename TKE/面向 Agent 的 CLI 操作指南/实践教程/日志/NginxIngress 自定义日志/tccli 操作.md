# NginxIngress 自定义日志（tccli）

> 对照官方：[NginxIngress 自定义日志](https://cloud.tencent.com/document/product/457/77840) · page_id `77840`

## 概述

通过修改 Nginx Ingress ConfigMap 的 `log-format-upstream` 自定义访问日志格式，记录请求耗时、上游地址、状态码等详细字段。

## 前置条件

- NginxIngress 组件已安装

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 修改日志格式 | `kubectl edit configmap nginx-configuration -n kube-system` | 是 |
| 查看日志 | `kubectl logs -n kube-system <nginx-pod>` | 是 |
| 查看配置 | `kubectl describe configmap nginx-configuration -n kube-system` | 是 |

## 操作步骤

### 自定义日志格式

```bash
kubectl edit configmap nginx-configuration -n kube-system
```

添加：

```yaml
data:
  log-format-upstream: '$remote_addr - $remote_user [$time_local] "$request" $status $body_bytes_sent "$http_referer" "$http_user_agent" $request_time $upstream_response_time $upstream_addr'
```

```bash
kubectl rollout restart deployment nginx-ingress-controller -n kube-system
```

### 查看自定义日志

```bash
kubectl logs -n kube-system -l app=nginx-ingress --tail=5
```

```text
...log output...
```

```output
10.0.1.100 - - [05/Jun/2026:14:00:00 +0000] "GET /api HTTP/1.1" 200 1234 "-" "curl/8.0" 0.005 0.003 10.0.2.10:80
```

## 验证

```bash
kubectl get configmap nginx-configuration -n kube-system -o yaml | grep log-format
```

```text
NAME  STATUS  AGE
...
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl` 命令超时或连接被拒绝 | `tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]'`（控制面返回 Running） | CAM 策略 strategyId:240463971 拒绝公网端点，数据面需内网访问 | 通过 VPN/IOA 接入 VPC 内网后执行 kubectl；或使用控制台替代 |
| 日志格式未生效 | `kubectl get cm nginx-configuration -n kube-system -o yaml \| grep log-format-upstream` | ConfigMap 修改后未重启 Ingress Controller | `kubectl rollout restart deployment nginx-ingress-controller -n kube-system` |
| 日志字段缺失 | `kubectl logs -n kube-system <nginx-pod> --tail=5` | `log-format-upstream` 字段名拼写错误 | 核对字段名，使用 nginx 内置变量（`$request_time` 等） |
| ConfigMap 不存在 | `kubectl get cm -n kube-system \| grep nginx-configuration` | NginxIngress 组件未安装或命名不同 | 确认 NginxIngress 组件已安装；组件名可能为 `nginx-configuration` 或 `nginx-ingress-controller` |

## 清理

恢复默认格式。

## 下一步

- [TKE 日志采集最佳实践](../TKE%20日志采集最佳实践/tccli%20操作.md)
- [Nginx Ingress 最佳实践](../../网络/Nginx%20Ingress%20最佳实践/tccli%20操作.md)

## 控制台替代

控制台：组件管理 → NginxIngress → 更新配置。
