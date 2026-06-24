# TKE 基于弹性网卡直连 Pod 的网络负载均衡（tccli）

> 对照官方：[TKE 基于弹性网卡直连 Pod 的网络负载均衡](https://cloud.tencent.com/document/product/457/48768) · page_id `48768`

## 概述

VPC-CNI 网络模式下，Pod 拥有独立的 VPC 弹性网卡（ENI）IP。利用此特性，CLB 可直接绑定 Pod IP 实现直连，绕过 NodePort 转发，消除 SNAT，保留客户端源 IP。

## 前置条件

- [环境准备](../../../环境准备.md)
- 集群已启用 [VPC-CNI 网络模式](../../../集群配置/网络管理/VPC-CNI%20模式/非固定%20IP%20模式使用说明/tccli%20操作.md)
- 集群中已创建 CLB 实例或使用自动创建

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 查看 VPC-CNI 状态 | `tccli tke DescribeClusters` → `Property.NetworkType` | 是 |
| 创建 CLB 直连 Service | `kubectl apply -f` (annotation `tke-direct-eni: "true"`) | 是 |
| 查看 CLB 后端 | `tccli clb DescribeTargets` | 是 |
| 验证源 IP | `kubectl logs <pod>` 查看访问日志 | 是 |

## 操作步骤

### 1. 确认 VPC-CNI 模式

```bash
tccli tke DescribeClusters --region ap-guangzhou --cli-input-json file://query.json
```

检查 `ClusterNetworkSettings.VpcCniType` 字段，应为 `tke-route-eni` 或 `tke-direct-eni`。

### 2. 创建直连 Pod 的 Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-direct
  annotations:
    service.cloud.tencent.com/direct-access: "true"
    service.cloud.tencent.com/tke-service-config-auto: "true"
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
```

```bash
kubectl apply -f nginx-direct-svc.yaml
```

```text
# command executed successfully
```

```output
service/nginx-direct created
```

### 3. 验证 CLB 直连

```bash
kubectl get svc nginx-direct -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
```

```text
NAME  STATUS  AGE
...
```

```output
10.0.0.100
```

CLB 后端将直接显示 Pod IP 而非 Node IP。

## 验证

```bash
kubectl get svc nginx-direct -o yaml | grep -A5 annotations
kubectl get endpoints nginx-direct
```

```text
NAME  STATUS  AGE
...
```

## 清理

```bash
kubectl delete svc nginx-direct
kubectl delete deployment nginx
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl` 命令超时或连接被拒绝 | `tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]'`（控制面返回 Running） | CAM 策略 strategyId:240463971 拒绝公网端点，数据面需内网访问 | 通过 VPN/IOA 接入 VPC 内网后执行 kubectl；或使用控制台替代 |
| CLB 后端仍为 Node IP | `kubectl get svc <svc> -o jsonpath='{.metadata.annotations}'` | `direct-access: "true"` 注解未设置或未生效 | 确认 `direct-access: "true"` 注解已设置，重建 Service |
| Service 创建失败 | `tccli tke DescribeCluster --ClusterId <ClusterId> --filter "Cluster.NetworkSettings.Cni"` | 集群为 GlobalRouter 模式，不支持直连 | 确认集群已是 VPC-CNI 模式，非 GlobalRouter |

## 下一步

- [在 TKE 上使用负载均衡直连 Pod](../在%20TKE%20上使用负载均衡直连%20Pod/tccli%20操作.md)
- [在 TKE 中获取客户端真实源 IP](../在%20TKE%20中获取客户端真实源%20IP/tccli%20操作.md)

## 控制台替代

控制台：Service → 新建 → 勾选"直连 Pod 模式"。
