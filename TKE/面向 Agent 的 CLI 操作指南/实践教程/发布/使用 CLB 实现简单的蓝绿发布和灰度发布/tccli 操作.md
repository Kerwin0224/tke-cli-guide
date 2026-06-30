# 使用 CLB 实现简单的蓝绿发布和灰度发布（tccli）

> 对照官方：[使用 CLB 实现简单的蓝绿发布和灰度发布](https://cloud.tencent.com/document/product/457/48877) · page_id `48877`

## 概述

利用 CLB 权重和转发规则实现蓝绿/灰度发布。通过 Service selector 切换或 Ingress canary annotation 控制流量分配。

## 前置条件

- [环境准备](../../../环境准备.md)
- CLB 类型 Service 或 Ingress 已创建

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 蓝绿切换 | `kubectl patch svc <name> -p '{"spec":{"selector":{"version":"v2"}}}'` | 否 |
| 金丝雀发布 | Ingress `canary: "true"` + `canary-weight: "10"` | 是 |
| 查看流量 | `kubectl get endpoints` / `kubectl logs` | 是 |

## 操作步骤

### 蓝绿发布

```bash
# 部署 v2
kubectl apply -f deploy-v2.yaml
# 切换 Service selector
kubectl patch svc myapp -p '{"spec":{"selector":{"app":"myapp","version":"v2"}}}'
```

### 灰度（金丝雀）发布

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-canary
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "10"
spec:
  rules:
  - host: app.example.com
    http:
      paths:
      - backend:
          service:
            name: myapp-v2
            port:
              number: 80
        path: /
        pathType: Prefix
```

## 验证

```bash
kubectl get svc myapp -o jsonpath='{.spec.selector}'
kubectl describe ingress myapp-canary
```

```text
NAME  STATUS  AGE
...
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl` 命令超时或连接被拒绝 | `tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]'`（控制面返回 Running） | CAM 策略 strategyId:240463971 拒绝公网端点，数据面需内网访问 | 通过 VPN/IOA 接入 VPC 内网后执行 kubectl；或使用控制台替代 |
| 蓝绿切换后流量未切到新版本 | `kubectl get svc myapp -o jsonpath='{.spec.selector}'` | Service selector 未更新，仍指向旧版本 label | `kubectl patch svc myapp -p '{"spec":{"selector":{"version":"v2"}}}'` |
| 金丝雀流量比例异常 | `kubectl describe ingress myapp-canary` 查看 `canary-weight` | `canary-weight` 注解值错误或 Ingress 未 reload | 确认 `nginx.ingress.kubernetes.io/canary-weight` 值在 0-100 之间 |
| 回滚后 Service selector 仍为新版本 | `kubectl get svc myapp -o yaml \| grep -A5 selector` | 回滚时只删除了 Ingress，未还原 selector | `kubectl patch svc myapp -p '{"spec":{"selector":{"version":"v1"}}}'` |

## 清理

```bash
kubectl delete ingress myapp-canary
kubectl patch svc myapp -p '{"spec":{"selector":{"version":"v1"}}}'
```

## 下一步

- [工作负载平滑升级](../../服务部署/工作负载平滑升级/tccli%20操作.md)
- [Nginx 升级最佳实践](../../网络/Nginx%20升级最佳实践/tccli%20操作.md)

## 控制台替代

控制台：Service → 更新访问方式 → 修改选择器。
