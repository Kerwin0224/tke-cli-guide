# 工作负载平滑升级（tccli）

> 对照官方：[工作负载平滑升级](https://cloud.tencent.com/document/product/457/77969) · page_id `77969`

## 概述

通过 Kubernetes RollingUpdate 策略实现工作负载零停机升级。配置 maxSurge、maxUnavailable、readinessGates、preStop hook 确保流量无损迁移。

## 前置条件

- [环境准备](../../../环境准备.md)

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 滚动升级 | `kubectl set image deploy/<name> <container>=<new-image>` | 否 |
| 查看升级状态 | `kubectl rollout status deploy/<name>` | 是 |
| 查看历史版本 | `kubectl rollout history deploy/<name>` | 是 |
| 回滚 | `kubectl rollout undo deploy/<name>` | 否 |
| 暂停/恢复 | `kubectl rollout pause/resume deploy/<name>` | 否 |

## 操作步骤

### 1. 配置滚动升级策略

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  minReadySeconds: 10
```

### 2. 触发升级

```bash
kubectl set image deployment/myapp myapp=nginx:1.25.0 --record
```

```output
deployment.apps/myapp image updated
```

### 3. 监控升级

```bash
kubectl rollout status deployment/myapp
```

```output
Waiting for deployment "myapp" rollout to finish: 1 old replicas are pending termination...
deployment "myapp" successfully rolled out
```

### 4. 查看版本历史

```bash
kubectl rollout history deployment/myapp
```

```output
REVISION  CHANGE-CAUSE
1         <none>
2         kubectl set image deployment/myapp myapp=nginx:1.25.0
```

### 5. 回滚

```bash
kubectl rollout undo deployment/myapp --to-revision=1
```

## 验证

```bash
kubectl get deploy myapp -o jsonpath='{.spec.strategy}'
kubectl rollout history deploy myapp
```

```text
NAME  STATUS  AGE
...
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl` 连接超时/拒绝 | `tccli tke DescribeClusters` 返回集群 Running 但 kubectl 不可达 | CAM 策略 240463971 拒绝公网访问 APIServer 端点 | 通过 VPN/IOA 接入内网后执行 kubectl 命令 |
| Rollout 卡住不前进 | `kubectl rollout status deploy/<name>` + `kubectl get pods -l app=<name>` | maxUnavailable=0 且新 Pod 未就绪 | 检查 readinessProbe；临时调大 maxSurge 加速滚动 |
| 升级期间请求报 502 | `kubectl get endpoints <svc>` 查看 Ready Pod 数 | 新旧 Pod 切换时 preStop hook 未配置或宽限时间不足 | 配置 `preStop` hook + 合理的 `terminationGracePeriodSeconds` |

## 清理

无，保留已验证的版本。

## 下一步

- [应用高可用部署](../应用高可用部署/tccli%20操作.md)
- [弹性伸缩](../../弹性伸缩/在%20TKE%20上利用%20HPA%20实现业务的弹性伸缩/tccli%20操作.md)

## 控制台替代

控制台：工作负载 → 更新镜像 → 选择滚动更新策略。
