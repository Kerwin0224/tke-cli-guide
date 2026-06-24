# 根据不同业务场景调节 HPA 扩缩容灵敏度（tccli）

> 对照官方：[根据不同业务场景调节 HPA 扩缩容灵敏度](https://cloud.tencent.com/document/product/457/50660) · page_id `50660`

## 概述

通过 HPA v2 `behavior` 字段配置扩缩容策略：缩容稳定窗口（stabilizationWindowSeconds）、扩容/缩容速率（policies）和选择策略（selectPolicy）。

## 前置条件

- [环境准备](../../../环境准备.md)
- HPA v2 API 可用

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 快速扩容 | `scaleUp.policies: [{type: Percent, value: 100, periodSeconds: 15}]` | 是 |
| 慢速缩容 | `scaleDown.stabilizationWindowSeconds: 300` | 是 |
| 禁用缩容 | `scaleDown.selectPolicy: Disabled` | 是 |

## 操作步骤

### 场景 1：秒级扩容 + 慢速缩容

```yaml
behavior:
  scaleUp:
    stabilizationWindowSeconds: 0
    policies:
    - type: Percent
      value: 100
      periodSeconds: 15
    - type: Pods
      value: 4
      periodSeconds: 15
    selectPolicy: Max
  scaleDown:
    stabilizationWindowSeconds: 300
    policies:
    - type: Percent
      value: 10
      periodSeconds: 60
```

### 场景 2：稳定优先（保守）

```yaml
scaleUp:
  stabilizationWindowSeconds: 60
scaleDown:
  stabilizationWindowSeconds: 600
  policies:
  - type: Pods
    value: 1
    periodSeconds: 120
```

### 场景 3：禁用缩容

```yaml
scaleDown:
  selectPolicy: Disabled
```

## 验证

```bash
kubectl get hpa <name> -o yaml | grep -A20 behavior
```

```text
NAME  STATUS  AGE
...
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl` 命令报错/无响应（本环境实测） | `tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]' --filter "Clusters[0].ClusterStatus"` | CAM 拒绝公网端点（strategyId:240463971），数据面不可达 | 接入 VPN/IOA 内网后执行 kubectl；控制面 tccli 不受影响 |
| `behavior` 配置不生效 | `kubectl get hpa <name> -o yaml` 对比 spec | HPA 使用 autoscaling/v1（不支持 behavior） | 升级为 autoscaling/v2 API |
| 扩容仍过慢 | `kubectl get hpa -w` 观察 REPLICAS 变化 | `scaleUp.stabilizationWindowSeconds` 非零或 `selectPolicy: Min` | 设 `stabilizationWindowSeconds: 0`、`selectPolicy: Max` |

## 清理

```bash
kubectl delete hpa <name>
```

## 下一步

- [HPA 弹性伸缩](../在%20TKE%20上利用%20HPA%20实现业务的弹性伸缩/tccli%20操作.md)
- [使用自定义指标进行弹性伸缩](../在%20TKE%20上使用自定义指标进行弹性伸缩/tccli%20操作.md)

## 控制台替代

控制台：工作负载 → HPA → 高级设置 → 扩缩容策略。
