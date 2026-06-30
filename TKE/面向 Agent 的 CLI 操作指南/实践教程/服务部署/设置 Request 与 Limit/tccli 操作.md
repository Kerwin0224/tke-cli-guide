# 设置 Request 与 Limit（tccli）

> 对照官方：[设置 Request 与 Limit](https://cloud.tencent.com/document/product/457/45634) · page_id `45634`

## 概述

通过 `resources.requests` 和 `resources.limits` 定义容器资源需求。配合 LimitRange 和 ResourceQuota 在命名空间级别实现资源管控。

## 前置条件

- [环境准备](../../../环境准备.md)

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 设置容器资源 | Deployment YAML `resources.requests/limits` | 是 |
| 命名空间级默认值 | `kubectl apply -f limitrange.yaml` | 是 |
| 命名空间级配额 | `kubectl apply -f resourcequota.yaml` | 是 |
| 查看实际用量 | `kubectl top pod` | 是 |

## 操作步骤

### 1. 容器级设置

```yaml
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi
```

### 2. LimitRange

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
spec:
  limits:
  - default:
      cpu: 500m
      memory: 512Mi
    defaultRequest:
      cpu: 200m
      memory: 256Mi
    type: Container
```

```bash
kubectl apply -f limitrange.yaml -n default
```

```text
# command executed successfully
```

```output
limitrange/default-limits created
```

### 3. ResourceQuota

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
```

## 验证

```bash
kubectl describe limitrange default-limits -n default
kubectl get resourcequota compute-quota -n default
```

```text
NAME  STATUS  AGE
...
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl` 连接超时/拒绝 | `tccli tke DescribeClusters` 返回集群 Running 但 kubectl 不可达 | CAM 策略 240463971 拒绝公网访问 APIServer 端点 | 通过 VPN/IOA 接入内网后执行 kubectl 命令 |
| Pod 创建报 `Exceeded quota` | `kubectl describe resourcequota compute-quota -n default` 查看配额使用量 | 命名空间 ResourceQuota 已达上限 | 提高 ResourceQuota 的 hard 值或清理不需要的资源 |
| Pod 被赋予非预期默认 limit | `kubectl describe limitrange default-limits -n default` | LimitRange 的 default 值自动注入到未设置 limit 的容器 | 调整 LimitRange default 值或在 Pod spec 中显式设置 resources |

## 清理

```bash
kubectl delete limitrange default-limits -n default
kubectl delete resourcequota compute-quota -n default
```

## 下一步

- [资源合理分配](../资源合理分配/tccli%20操作.md)
- [弹性伸缩](../../弹性伸缩/在%20TKE%20上利用%20HPA%20实现业务的弹性伸缩/tccli%20操作.md)

## 控制台替代

控制台：工作负载 → 资源限制 → 设置 Request/Limit。
