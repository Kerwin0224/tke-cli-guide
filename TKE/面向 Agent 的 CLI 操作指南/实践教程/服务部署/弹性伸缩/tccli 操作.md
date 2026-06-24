# 弹性伸缩（tccli）

> 对照官方：[弹性伸缩](https://cloud.tencent.com/document/product/457/45636) · page_id `45636`

## 概述

TKE 弹性伸缩涵盖两个维度：节点级（CA，Cluster Autoscaler，通过 `ModifyClusterNodePool` 配置）和工作负载级（HPA，通过 kubectl 创建）。

## 前置条件

- [环境准备](../../../环境准备.md)
- metrics-server 已安装

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 启用节点池自动伸缩 | `tccli tke ModifyClusterNodePool --EnableAutoscale true` | 否 |
| 设置伸缩范围 | `ModifyClusterNodePool` MinSize/MaxSize | 是 |
| 创建 HPA | `kubectl autoscale deploy <name> --min=2 --max=10 --cpu-percent=70` | 是 |
| 查看 HPA 状态 | `kubectl get hpa` | 是 |

## 操作步骤

### 1. 节点池自动伸缩

```bash
tccli tke ModifyClusterNodePool --region ap-guangzhou --cli-input-json file://autoscale.json
```

```json
{
    "ClusterId": "<ClusterId>",
    "NodePoolId": "<NodePoolId>",
    "EnableAutoscale": true,
    "MinSize": 1,
    "MaxSize": 10
}
```

### 2. HPA 工作负载伸缩

```bash
kubectl autoscale deployment nginx --min=2 --max=10 --cpu-percent=70
```

```output
horizontalpodautoscaler.autoscaling/nginx autoscaled
```

### 3. 查看 HPA

```bash
kubectl get hpa
```

```text
NAME  STATUS  AGE
...
```

```output
NAME    REFERENCE          TARGETS   MINPODS   MAXPODS   REPLICAS
nginx   Deployment/nginx   5%/70%    2         10        2
```

## 验证

```bash
kubectl get hpa nginx -o yaml
kubectl top pods
```

```text
NAME  STATUS  AGE
...
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl` 连接超时/拒绝 | `tccli tke DescribeClusters` 返回集群 Running 但 kubectl 不可达 | CAM 策略 240463971 拒绝公网访问 APIServer 端点 | 通过 VPN/IOA 接入内网后执行 kubectl 命令 |
| HPA TARGETS 显示 `<unknown>` | `kubectl describe hpa <name>` 查看 Conditions | metrics-server 未安装或 Pod 未设置 cpu Request | 安装 metrics-server；确保 Pod spec 中设置了 `resources.requests.cpu` |
| HPA 不触发扩容 | `kubectl get hpa` 查看 TARGETS 与目标百分比对比 | 当前 CPU 利用率低于目标百分比 | 对 Pod 施加负载触发扩容；确认 minReplicas 未已达上限 |
| 节点池不扩容 | `tccli tke DescribeClusterNodePools --ClusterId <ClusterId>` 查看 NodePool 状态 | EnableAutoscale 未开启或 MaxSize 已达上限 | 确认 `EnableAutoscale: true`；提高 MaxSize |

## 清理

```bash
kubectl delete hpa nginx
tccli tke ModifyClusterNodePool --region ap-guangzhou --cli-input-json file://disable-autoscale.json
```

## 下一步

- [利用 HPA 实现业务的弹性伸缩](../../弹性伸缩/在%20TKE%20上利用%20HPA%20实现业务的弹性伸缩/tccli%20操作.md)
- [集群弹性伸缩实践](../../弹性伸缩/集群弹性伸缩实践/tccli%20操作.md)

## 控制台替代

控制台：节点池 → 弹性伸缩 / 工作负载 → HPA。
