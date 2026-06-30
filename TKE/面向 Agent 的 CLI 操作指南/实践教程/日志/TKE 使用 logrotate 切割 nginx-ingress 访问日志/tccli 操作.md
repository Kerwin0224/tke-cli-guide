# TKE 使用 logrotate 切割 nginx-ingress 访问日志（tccli）

> 对照官方：[TKE 使用 logrotate 切割 nginx-ingress 访问日志](https://cloud.tencent.com/document/product/457/97747) · page_id `97747`

## 概述

Nginx Ingress 访问日志长期不切割会导致磁盘占用过高。通过 hostPath 共享日志目录 + sidecar logrotate 实现日志自动切割和清理。

## 前置条件

- NginxIngress 组件已安装

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 挂载日志目录 | hostPath `/var/log/nginx` shared | 是 |
| 配置 logrotate | ConfigMap + sidecar cron | 是 |
| 查看日志大小 | `kubectl exec -c logrotate -- du -sh /var/log/nginx` | 是 |

## 操作步骤

### 1. 创建 logrotate 配置

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: logrotate-conf
data:
  nginx: |
    /var/log/nginx/access.log {
        daily
        rotate 7
        compress
        delaycompress
        missingok
        notifempty
        copytruncate
    }
```

### 2. 添加 sidecar 到 Deployment

```yaml
containers:
- name: logrotate
  image: alpine:latest
  command: ["sh", "-c", "apk add logrotate && while true; do logrotate /etc/logrotate.d/nginx; sleep 3600; done"]
  volumeMounts:
  - name: log
    mountPath: /var/log/nginx
  - name: logrotate-conf
    mountPath: /etc/logrotate.d
volumes:
- name: log
  hostPath:
    path: /var/log/nginx
- name: logrotate-conf
  configMap:
    name: logrotate-conf
```

## 验证

```bash
kubectl exec <pod> -c logrotate -- ls -lh /var/log/nginx/
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl` 命令超时或连接被拒绝 | `tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]'`（控制面返回 Running） | CAM 策略 strategyId:240463971 拒绝公网端点，数据面需内网访问 | 通过 VPN/IOA 接入 VPC 内网后执行 kubectl；或使用控制台替代 |
| 日志文件未切割 | `kubectl exec <pod> -c logrotate -- cat /etc/logrotate.d/nginx` | logrotate Cron 未配置或 ConfigMap 挂载失败 | 确认 ConfigMap 已挂载；检查 sidecar 中 crond 是否运行 |
| logrotate 容器 CrashLoopBackOff | `kubectl logs <pod> -c logrotate --tail=20` | ConfigMap 路径不存在或权限不足 | 确认挂载路径 `/var/log/nginx` 与 nginx 容器一致 |
| 切割后磁盘仍满 | `kubectl exec <pod> -- df -h /var/log/nginx` | `rotate` 数量过大或 `size` 阈值过小 | 调小 `rotate` 保留份数；增大 `size` 触发阈值 |

## 清理

```bash
kubectl delete configmap logrotate-conf
```

## 下一步

- [NginxIngress 自定义日志](../NginxIngress%20自定义日志/tccli%20操作.md)
- [Nginx Ingress 最佳实践](../../网络/Nginx%20Ingress%20最佳实践/tccli%20操作.md)

## 控制台替代

无控制台界面。
