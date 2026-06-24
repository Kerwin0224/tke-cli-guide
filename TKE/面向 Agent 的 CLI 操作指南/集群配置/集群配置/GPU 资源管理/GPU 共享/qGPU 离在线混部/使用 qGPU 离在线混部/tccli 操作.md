# 使用 qGPU 离在线混部

> 对照官方：[使用 qGPU 离在线混部](https://cloud.tencent.com/document/product/457/81034) · page_id `81034`

## 概述

本文介绍如何在 TKE 集群中配置和使用 qGPU 离在线混部能力，使在线（高优）和离线（低优）任务共享同一张 GPU 卡。

## 前置条件

- 已安装 qGPU 组件
- 已开启集群 qGPU 共享
- 已准备 GPU 资源（原生节点池）
- kubectl 可达集群以 apply YAML

> **kubectl 可达性说明**：当前共享集群 kubectl 不可达，且无 GPU 节点。以下 YAML 和命令完整，但无法真跑验证。

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 重建 qgpu-manager Pod | `kubectl delete pod qgpu-manager-<id> -n kube-system` | 否（Pod 重建） |
| 创建在线 Pod | `kubectl apply -f <yaml>` | 是 |
| 创建离线 Pod | `kubectl apply -f <yaml>` | 是 |
| 创建普通 Pod | `kubectl apply -f <yaml>` | 是 |

## 操作步骤

### 重建组件（必须）

节点创建后必须重建 qGPU 相关 Pod，否则离在线混部无法正常工作。

```bash
# 获取 qgpu-manager Pod 名称
kubectl get pods -n kube-system | grep qgpu-manager

# 删除重建
kubectl delete pod <qgpu-manager-pod-name> -n kube-system
kubectl delete pod -n kube-system -l app=qgpu-scheduler

# 验证低优算力资源
kubectl describe node <gpu-node> | grep qgpu-core-greedy
# expected: 存在 qgpu-core-greedy 资源字段
```

### 创建离线 Pod

通过 `tke.cloud.tencent.com/app-class: offline` annotation 标识，使用 `qgpu-core-greedy` 申请离线算力。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: offline-pod
  annotations:
    tke.cloud.tencent.com/app-class: offline
spec:
  containers:
  - name: offline-container
    image: nvidia/cuda:11.0-base
    resources:
      requests:
        tke.cloud.tencent.com/qgpu-core-greedy: 10   # 离线算力
        tke.cloud.tencent.com/qgpu-memory: 5          # 显存 GB
      limits:
        tke.cloud.tencent.com/qgpu-core-greedy: 10
        tke.cloud.tencent.com/qgpu-memory: 5
```

离线 Pod 不支持多卡，需同时设置低优算力与显存。

```bash
kubectl apply -f offline-pod.yaml
# expected: pod/offline-pod created
```

### 创建在线 Pod

通过 `tke.cloud.tencent.com/app-class: online` annotation 标识，仅需申请显存。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: online-pod
  annotations:
    tke.cloud.tencent.com/app-class: online
spec:
  containers:
  - name: online-container
    image: nvidia/cuda:11.0-base
    resources:
      requests:
        tke.cloud.tencent.com/qgpu-memory: 5   # 仅需申请显存
```

高优 Pod 不抢占低优 Pod 的显存资源（显存为静态切分）。

```bash
kubectl apply -f online-pod.yaml
# expected: pod/online-pod created
```

### 创建普通 Pod

没有 `app-class` annotation，普通 Pod 不受离在线混部策略影响，支持多卡。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: common-pod
spec:
  containers:
  - name: common-container
    image: nvidia/cuda:11.0-base
    resources:
      requests:
        tke.cloud.tencent.com/qgpu-core: 25
        tke.cloud.tencent.com/qgpu-memory: 6
```

```bash
kubectl apply -f common-pod.yaml
# expected: pod/common-pod created
```

## 验证

```bash
kubectl get pods offline-pod online-pod common-pod
kubectl describe node <gpu-node> | grep -E "qgpu-core|qgpu-memory|qgpu-core-greedy"
# expected: 各 Pod Running，节点 qGPU 资源正确分配
```

```text
NAME  STATUS  AGE
...
```

## 清理

```bash
kubectl delete pod offline-pod online-pod common-pod
```

## 排障

| 现象 | 根因 | 修复 |
|------|------|------|
| 离线 Pod 无法调度 | 节点无 `qgpu-core-greedy` 资源 | 重建 qgpu-manager 和 qgpu-scheduler Pod |
| 在线 Pod 抢占不生效 | 未重建 qgpu-scheduler | 执行 `kubectl delete pod -n kube-system -l app=qgpu-scheduler` |
| 离在线混部功能异常 | 先创建工作负载后创建节点 | 创建节点池时勾选封锁节点，重建后再解封 |

## 下一步

- [qGPU 离在线混部说明](../qGPU 离在线混部说明/tccli 操作.md)
- [使用 qGPU](../../使用 qGPU/tccli 操作.md)
- [GPU 监控指标获取](../../../GPU 监控指标获取/tccli 操作.md)

## 控制台替代

[TKE 控制台 → 工作负载](https://console.cloud.tencent.com/tke2/cluster)：新建 Deployment，在容器高级设置中添加 annotation `tke.cloud.tencent.com/app-class` 和对应资源限制。
