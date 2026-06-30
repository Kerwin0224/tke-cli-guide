# 在 TKE 上对 Pod 进行带宽限速（tccli）

> 对照官方：[在 TKE 上对 Pod 进行带宽限速](https://cloud.tencent.com/document/product/457/48766) · page_id `48766`

## 概述

TKE 支持通过 Pod Annotation 对 Pod 的入站/出站流量进行带宽限速。底层通过宿主机 tc (traffic control) 实现，支持 `kubernetes.io/ingress-bandwidth` 和 `kubernetes.io/egress-bandwidth` 注解。

## 前置条件

- [环境准备](../../../环境准备.md)
- 集群网络模式为 GlobalRouter 或 VPC-CNI
- [`tke-eni-ipamd` 组件](https://cloud.tencent.com/document/product/457/50354) 已开启带宽限速功能

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 设置 Pod 入站带宽 | Annotation `kubernetes.io/ingress-bandwidth: 10M` | 是 |
| 设置 Pod 出站带宽 | Annotation `kubernetes.io/egress-bandwidth: 10M` | 是 |
| 查看 Pod 带宽配置 | `kubectl describe pod <name>` → Annotations | 是 |
| 验证带宽限制 | `iperf3` / `wget` 测速 | 是 |

## 操作步骤

### 1. 创建带带宽限制的 Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-bw-limited
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-bw
  template:
    metadata:
      labels:
        app: nginx-bw
      annotations:
        kubernetes.io/ingress-bandwidth: "10M"
        kubernetes.io/egress-bandwidth: "10M"
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
```

```bash
kubectl apply -f nginx-bw.yaml
```

```text
# command executed successfully
```

```output
deployment.apps/nginx-bw-limited created
```

### 2. 验证带宽配置

```bash
kubectl get pod -l app=nginx-bw -o jsonpath='{.items[0].metadata.annotations}'
```

```text
NAME  STATUS  AGE
...
```

```output
{"kubernetes.io/ingress-bandwidth":"10M","kubernetes.io/egress-bandwidth":"10M"}
```

### 3. 带宽单位说明

| 单位 | 含义 | 示例 |
|------|------|------|
| `K` | Kbps | `512K` |
| `M` | Mbps | `10M` |
| `G` | Gbps | `1G` |

## 验证

```bash
kubectl describe pod -l app=nginx-bw | grep -A2 Annotations
```

```text
Name:         ...
Status:       Running
...
```

## 清理

```bash
kubectl delete deployment nginx-bw-limited
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl` 命令超时或连接被拒绝 | `tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]'`（控制面返回 Running） | CAM 策略 strategyId:240463971 拒绝公网端点，数据面需内网访问 | 通过 VPN/IOA 接入 VPC 内网后执行 kubectl；或使用控制台替代 |
| 限速不生效 | `tccli tke DescribeAddon --ClusterId <ClusterId> --AddonName tke-eni-ipamd` | `tke-eni-ipamd` 组件未开启带宽限速功能 | 确认 `tke-eni-ipamd` 组件已开启带宽限速功能 |
| Annotation 格式错误 | `kubectl describe pod -l app=<name> \| grep -i bandwidth` | Annotation 值单位错误或含空格 | 单位必须为大写 `K`/`M`/`G`，不能有空格 |

## 下一步

- [TKE 基于弹性网卡直连 Pod 的网络负载均衡](../TKE%20基于弹性网卡直连%20Pod%20的网络负载均衡/tccli%20操作.md)
- [在 TKE 上使用负载均衡直连 Pod](../在%20TKE%20上使用负载均衡直连%20Pod/tccli%20操作.md)

## 控制台替代

带宽限速仅支持通过 Annotation 配置，无控制台 UI。
