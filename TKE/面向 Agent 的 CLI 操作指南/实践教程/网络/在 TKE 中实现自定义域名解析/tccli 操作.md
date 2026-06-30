# 在 TKE 中实现自定义域名解析（tccli）

> 对照官方：[在 TKE 中实现自定义域名解析](https://cloud.tencent.com/document/product/457/50865) · page_id `50865`

## 概述

通过修改 CoreDNS ConfigMap 实现集群内自定义域名解析。支持三种方式：hosts 插件添加静态解析、rewrite 插件域名重写、stubDomains 转发到指定上游 DNS。

## 前置条件

- [环境准备](../../../环境准备.md)
- CoreDNS 为集群默认 DNS 服务

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 查看 CoreDNS ConfigMap | `kubectl get configmap coredns -n kube-system -o yaml` | 是 |
| 添加 hosts 解析 | `kubectl edit configmap coredns -n kube-system` → hosts plugin | 否 |
| 添加域名重写 | Corefile 中添加 rewrite plugin | 否 |
| 添加上游 DNS 转发 | Corefile 中添加 forward plugin | 否 |
| 重载配置 | `kubectl rollout restart deployment coredns -n kube-system` | 否 |

## 操作步骤

### 方式1：Hosts 插件（静态解析）

```bash
kubectl edit configmap coredns -n kube-system
```

在 Corefile 中添加：

```
.:53 {
    errors
    health
    hosts {
        10.0.0.100 myapp.internal
        10.0.0.200 db.internal
        fallthrough
    }
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

```bash
kubectl rollout restart deployment coredns -n kube-system
```

```output
deployment.apps/coredns restarted
```

### 验证

```bash
kubectl run test --image=nginx:alpine --rm -it --restart=Never -- nslookup myapp.internal
```

```output
Server:    10.0.0.10
Address:   10.0.0.10#53
Name:      myapp.internal
Address:   10.0.0.100
```

### 方式2：Rewrite 插件（域名重写）

```
rewrite name suffix myapp.internal myapp.default.svc.cluster.local
```

所有 `*.myapp.internal` 查询重写为 `*.myapp.default.svc.cluster.local`。

### 方式3：Forward 插件（上游 DNS 转发）

```
mycompany.local:53 {
    errors
    forward . 10.0.0.53
    cache 30
}
```

将 `mycompany.local` 域名的 DNS 查询转发到 `10.0.0.53`。

## 验证

```bash
kubectl run dns-test --image=busybox:1.28 --rm -it --restart=Never -- nslookup myapp.internal
```

## 清理

```bash
kubectl rollout undo deployment coredns -n kube-system
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl` 命令超时或连接被拒绝 | `tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]'`（控制面返回 Running） | CAM 策略 strategyId:240463971 拒绝公网端点，数据面需内网访问 | 通过 VPN/IOA 接入 VPC 内网后执行 kubectl；或使用控制台替代 |
| 解析不生效 | `kubectl rollout status deployment coredns -n kube-system` | `kubectl rollout restart` 未完成，CoreDNS 仍用旧 Corefile | 确认 `kubectl rollout restart` 已完成 |
| 语法错误 | `kubectl logs -n kube-system -l k8s-app=kube-dns --tail=20` | Corefile 语法错误 | `kubectl logs -n kube-system -l k8s-app=kube-dns` 查看错误，修正后重新 apply |

## 下一步

- [TKE DNS 最佳实践](../DNS%20相关/TKE%20DNS%20最佳实践/tccli%20操作.md)
- [从 kube-dns 切换到 CoreDNS](../DNS%20相关/从%20kube-dns%20切换到%20CoreDNS/tccli%20操作.md)

## 控制台替代

无控制台界面，需直接编辑 CoreDNS ConfigMap。
