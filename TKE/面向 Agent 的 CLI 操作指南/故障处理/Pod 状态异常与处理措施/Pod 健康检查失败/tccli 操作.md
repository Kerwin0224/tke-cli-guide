# Pod 健康检查失败（tccli）

> 对照官方：[Pod 健康检查失败](https://cloud.tencent.com/document/product/457/43129) · page_id `43129`

## 概述

Pod `readinessProbe` 失败导致 Service 不转发流量，`livenessProbe` 失败导致容器被重启。排查探针配置是否正确。通过 tccli 检查集群状态，配合 kubectl 查看探针事件和配置。

> **kubectl 数据面不可达**：CAM拒绝公网端点(strategyId:240463971)，内网需VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

## 前置条件

- [环境准备](../../../环境准备.md)
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
| 查看探针事件 | `kubectl describe pod <pod>`（需 VPN/IOA） | 是 |
| 查看探针配置 | `kubectl get pod <pod> -o yaml \| grep -A20 probe`（需 VPN/IOA） | 是 |
| 手动测试探针 | `kubectl exec <pod> -- curl localhost:<port>/<path>`（需 VPN/IOA） | 是 |

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

### 步骤2：数据面查看探针失败事件（需 VPN/IOA）

```bash
kubectl describe pod <pod-name>
# 预期: Events 中有 Unhealthy 警告

kubectl get pod <pod-name> -o jsonpath='{.status.containerStatuses[0].ready}'
# 预期: false 表示 readinessProbe 未通过
```

```text
NAME  STATUS  AGE
...
```

### 步骤3：查看和测试探针配置

```bash
# 查看探针配置
kubectl get pod <pod-name> -o yaml | grep -A20 "probe"
# 预期: 显示 livenessProbe 和/或 readinessProbe 的配置
```

```text
NAME  STATUS  AGE
...
```

按探针类型手动验证：

**httpGet 探针**

```bash
kubectl exec <pod> -- curl -s -o /dev/null -w "%{http_code}" http://localhost:<port>/<path>
# 预期: 返回 200-399
```

**tcpSocket 探针**

```bash
kubectl exec <pod> -- nc -z localhost <port>
# 预期: Connection succeeded
```

**exec 探针**

```bash
kubectl exec <pod> -- <command>
# 预期: 返回码为 0
```

### 步骤4：常见探针配置问题

| 问题 | 诊断方法 | 根因 | 修复 |
|---|---|---|---|
| `initialDelaySeconds` 太短 | 容器启动后才开始探针，但应用尚未就绪 | 容器启动到探针开始之间的等待时间不足 | 增大 `initialDelaySeconds`，建议 15-30 秒 |
| `periodSeconds` 太频繁 | 查看 Events 中 Unhealthy 频率 | 探测间隔过短导致偶发超时被误判 | 延长 `periodSeconds` 到 10 秒以上 |
| `failureThreshold` 太低 | Pod 因少量失败就被重启 | 偶发的暂时性故障导致不必要的重启 | 增大 `failureThreshold`（livenessProbe 建议 3-5 次） |
| `timeoutSeconds` 不足 | HTTP 请求耗时超过超时值 | 应用响应慢但未设置足够超时 | 增大 `timeoutSeconds` 到 5-10 秒 |

### 步骤5：推荐探针配置模板

```yaml
readinessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
  failureThreshold: 3
  timeoutSeconds: 5
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
  failureThreshold: 5
  timeoutSeconds: 5
```

修改后重建 Pod 使新探针配置生效。

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
kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[0].ready}'
# 预期: true

kubectl describe pod <pod> | grep Unhealthy
# 预期: 无新的 Unhealthy 事件
```

```text
NAME  STATUS  AGE
...
```

## 清理

无需特殊清理（排障页）。修改探针配置后重建 Pod 即可生效。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|---|---|---|---|
| `curl` 返回 `Connection Refused` | Pod 内 `netstat -tlnp` 检查监听端口 | 应用端口未启动或监听地址为 127.0.0.1 | 确保应用监听 `0.0.0.0:<port>` 而非 localhost |
| exec 探针返回 127 | `kubectl exec` 执行同样命令 | 容器内缺少命令依赖（如 curl、wget） | 在镜像中安装所需工具或改用 httpGet 探针 |
| 探针配置修改后未生效 | `kubectl get pod <pod> -o yaml` 对比新旧配置 | Deploy/StatefulSet 未触发 Pod 重建 | 手动删除 Pod 触发重建：`kubectl delete pod <pod>` |

### 操作成功但异常

| 现象 | 诊断 | 根因 | 修复 |
|---|---|---|---|
| readiness 在高峰期频繁波动 | 观察探针失败与 Pod 负载的关联 | 高负载下健康检查端点响应慢 | 健康检查端点实现应轻量（不查数据库），或提高 timeoutSeconds |
| livenessProbe 调整后 Pod 仍偶尔重启 | `kubectl describe pod` 确认重启原因是否仍为探针失败 | 应用存在间歇性崩溃，探针只是症状 | 排查应用本身稳定性（内存泄漏、死锁等） |

## 下一步

- [Pod 处于 CrashLoopBackOff 状态](../Pod%20处于%20CrashLoopBackOff%20状态/tccli%20操作.md) -- page_id `43130`
- [容器进程主动退出](../容器进程主动退出/tccli%20操作.md) -- page_id `43148`

## 控制台替代

[容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) -> 工作负载 -> Pod -> 更新 Pod 配置 -> 健康检查。
