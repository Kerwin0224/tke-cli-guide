# 在 TKE 上安装 metrics-server（tccli）

> 对照官方：[在 TKE 上安装 metrics-server](https://cloud.tencent.com/document/product/457/50074) · page_id `50074`

## 概述

metrics-server 是 HPA 和 `kubectl top` 的数据源。通过 TKE 组件管理一键安装，安装后 `kubectl top` 命令可用。

## 前置条件

- [环境准备](../../../环境准备.md)

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 安装 metrics-server | `tccli tke InstallAddon --AddonName metrics-server` | 是 |
| 验证 | `kubectl top nodes` | 是 |
| 卸载 | `tccli tke DeleteAddon --AddonName metrics-server` | 否 |

## 操作步骤

### 1. 安装

```bash
tccli tke InstallAddon --region ap-guangzhou --cli-input-json "{\"ClusterId\":\"<ClusterId>\",\"AddonName\":\"metrics-server\",\"AddonVersion\":\"<AddonVersion>\"}"
```

### 2. 验证

```bash
kubectl top nodes
```

```text
# command executed successfully
```

```output
NAME         CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
node-ex-1    150m         3%     1200Mi          15%
```

```bash
kubectl top pods -A
```

```text
# command executed successfully
```

## 验证

```bash
kubectl get deployment metrics-server -n kube-system
kubectl top nodes
```

```text
NAME  STATUS  AGE
...
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl` 命令报错/无响应（本环境实测） | `tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]' --filter "Clusters[0].ClusterStatus"` | CAM 拒绝公网端点（strategyId:240463971），数据面不可达 | 接入 VPN/IOA 内网后执行 kubectl；控制面 tccli 不受影响 |
| `kubectl top` 无数据 | `tccli tke DescribeAddon --region ap-guangzhou --ClusterId <ClusterId> --AddonName metrics-server` | metrics-server 未就绪或未安装 | 等待组件 Running（2-5 分钟初始化） |
| `metrics not available` | `kubectl get --raw /apis/metrics.k8s.io/v1beta1/nodes` | metrics API 未注册 | 等待 2-5 分钟初始化 |

## 清理

```bash
tccli tke DeleteAddon --region ap-guangzhou --cli-input-json "{\"ClusterId\":\"<ClusterId>\",\"AddonName\":\"metrics-server\"}"
```

## 下一步

- [利用 HPA 实现业务的弹性伸缩](../在%20TKE%20上利用%20HPA%20实现业务的弹性伸缩/tccli%20操作.md)
- [使用自定义指标进行弹性伸缩](../在%20TKE%20上使用自定义指标进行弹性伸缩/tccli%20操作.md)

## 控制台替代

控制台：组件管理 → 安装 → metrics-server。
