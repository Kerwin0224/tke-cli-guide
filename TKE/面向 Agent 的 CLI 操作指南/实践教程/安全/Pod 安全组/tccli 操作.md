# Pod 安全组（tccli）

> 对照官方：[Pod 安全组](https://cloud.tencent.com/document/product/457/80587) · page_id `80587`

## 概述

TKE 支持为 Pod 绑定独立的安全组（区别于节点安全组），通过 Pod Annotation `tke.cloud.tencent.com/pod-security-groups` 指定。

## 前置条件

- 集群 VPC-CNI 模式

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 设置 Pod 安全组 | Annotation `tke.cloud.tencent.com/pod-security-groups: sg-xxx` | 是 |
| 查看安全组 | `kubectl describe pod <name>` | 是 |

## 操作步骤

```yaml
metadata:
  annotations:
    tke.cloud.tencent.com/pod-security-groups: sg-example
```

```bash
kubectl apply -f pod-with-sg.yaml
```

```text
# command executed successfully
```

## 验证

```bash
kubectl get pod <pod> -o jsonpath='{.metadata.annotations}'
```

```text
NAME  STATUS  AGE
...
```

```output
{"tke.cloud.tencent.com/pod-security-groups":"sg-example"}
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl` 连接超时/拒绝 | `tccli tke DescribeClusters` 返回集群 Running 但 kubectl 不可达 | CAM 策略 240463971 拒绝公网访问 APIServer 端点 | 通过 VPN/IOA 接入内网后执行 kubectl 命令 |
| Pod 安全组未生效 | `kubectl describe pod <pod>` 检查 annotation 是否存在 | Annotation `tke.cloud.tencent.com/pod-security-groups` 未设置或值错误 | 确认集群为 VPC-CNI 模式；Annotation 值为有效安全组 ID |
| Pod 无法调度 | `kubectl get pod <pod>` 查看 STATUS 和 Events | VPC-CNI 模式下 IP 资源不足或安全组限制 | 检查子网可用 IP 数量；确认安全组规则未阻断必要流量 |

## 清理

```bash
kubectl delete pod <pod>
```

## 下一步

- [容器镜像签名及验证](../容器镜像签名及验证/tccli%20操作.md)
- [在 TKE 上使用负载均衡直连 Pod](../../网络/在%20TKE%20上使用负载均衡直连%20Pod/tccli%20操作.md)

## 控制台替代

控制台：工作负载 → 高级设置 → 安全组。
