# 从 kube-dns 切换到 CoreDNS（tccli）

> 对照官方：[从 kube-dns 切换到 CoreDNS](https://cloud.tencent.com/document/product/457/97873) · page_id `97873`

## 概述

CoreDNS 是 Kubernetes 1.11+ 的默认 DNS 服务，替代 kube-dns。CoreDNS 架构更简化（单进程）、内存占用更小、支持插件化扩展。TKE 新版本集群默认使用 CoreDNS，存量集群可从 kube-dns 切换。

## 前置条件

- [环境准备](../../../环境准备.md)
- 当前集群使用 kube-dns（检查：`kubectl get deployment kube-dns -n kube-system`）
- 备份 kube-dns ConfigMap：`kubectl get configmap kube-dns -n kube-system -o yaml > kube-dns-cm-backup.yaml`

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 检查当前 DNS | `kubectl get deployment -n kube-system | grep dns` | 是 |
| 缩放 kube-dns 为 0 | `kubectl scale deployment kube-dns -n kube-system --replicas=0` | 否 |
| 部署 CoreDNS | `kubectl apply -f coredns.yaml` | 是 |
| 验证 DNS 解析 | `kubectl run test --rm -it -- nslookup kubernetes.default` | 是 |

## 操作步骤

### 1. 验证当前 DNS 服务

```bash
kubectl get deployment -n kube-system | grep dns
```

```text
NAME  STATUS  AGE
...
```

```output
kube-dns    1/1     1            1           30d
```

### 2. 准备 CoreDNS 迁移

```bash
kubectl get svc kube-dns -n kube-system -o yaml
```

```text
NAME  STATUS  AGE
...
```

确认 Service ClusterIP（需保留）和 Selector。

### 3. 缩放 kube-dns

```bash
kubectl scale deployment kube-dns -n kube-system --replicas=0
```

```output
deployment.apps/kube-dns scaled
```

### 4. 部署 CoreDNS

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: coredns
  namespace: kube-system
  labels:
    k8s-app: kube-dns
spec:
  replicas: 2
  selector:
    matchLabels:
      k8s-app: kube-dns
  template:
    metadata:
      labels:
        k8s-app: kube-dns
    spec:
      containers:
      - name: coredns
        image: coredns/coredns:1.11.1
        args: ["-conf", "/etc/coredns/Corefile"]
        volumeMounts:
        - name: config-volume
          mountPath: /etc/coredns
        ports:
        - containerPort: 53
          protocol: UDP
        - containerPort: 53
          protocol: TCP
      volumes:
      - name: config-volume
        configMap:
          name: coredns
```

```bash
kubectl apply -f coredns-deploy.yaml
```

### 5. 创建 CoreDNS ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health
        kubernetes cluster.local in-addr.arpa ip6.arpa {
            pods insecure
            fallthrough in-addr.arpa ip6.arpa
        }
        prometheus :9153
        forward . /etc/resolv.conf
        cache 30
        loop
        reload
        loadbalance
    }
```

### 6. 验证 DNS 解析

```bash
kubectl run dns-test --image=busybox:1.28 --rm -it --restart=Never -- nslookup kubernetes.default
```

```output
Server:    10.0.0.10
Address:   10.0.0.10#53
Name:      kubernetes.default.svc.cluster.local
Address:   10.0.0.1
```

## 验证

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=10
```

```text
NAME  STATUS  AGE
...
```

## 清理

确认 CoreDNS 正常工作后，可删除 kube-dns：

```bash
kubectl delete deployment kube-dns -n kube-system
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl` 命令超时或连接被拒绝 | `tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]'`（控制面返回 Running） | CAM 策略 strategyId:240463971 拒绝公网端点，数据面需内网访问 | 通过 VPN/IOA 接入 VPC 内网后执行 kubectl；或使用控制台替代 |
| DNS 解析失败 | `kubectl logs -n kube-system -l k8s-app=kube-dns --tail=20` | CoreDNS Pod 异常或 Corefile 配置 loop | 检查 CoreDNS Pod 是否 Running，日志中是否有 `loop` 检测错误 |
| Service 选择器不匹配 | `kubectl get svc kube-dns -n kube-system -o yaml \| grep -A5 selector` | 新 Deployment labels 与原 kube-dns Service selector 不一致 | 确保新 Deployment labels 与原 kube-dns Service selector 一致 |

## 下一步

- [TKE DNS 最佳实践](../DNS%20相关/TKE%20DNS%20最佳实践/tccli%20操作.md)
- [在 TKE 中实现自定义域名解析](../DNS%20相关/在%20TKE%20中实现自定义域名解析/tccli%20操作.md)

## 控制台替代

控制台：集群 → 组件管理 → CoreDNS → 若使用 kube-dns 集群可在此处切换到 CoreDNS。
