# GPU 监控指标获取

> 对照官方：[GPU 监控指标获取](https://cloud.tencent.com/document/product/457/90912) · page_id `90912`

## 概述

TKE 的 `gpu-exporter` 组件自动随 qGPU 或 nvidia-gpu 组件安装，提供 Pod/容器维度的 GPU 监控指标。通过节点 5678 端口的 `/metrics` 路径获取 Prometheus 格式指标。

## 前置条件

- 已安装 qGPU 或 nvidia-gpu 组件
- kubectl 可达集群以查看 Pod

> **kubectl 可达性说明**：当前共享集群 kubectl 不可达。以下命令完整，但无法真跑验证。

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 安装 nvidia-gpu 组件 | `tccli tke InstallAddon --AddonName nvidia-gpu` | 否 |
| 查看 gpu-exporter DaemonSet | `kubectl get daemonset -n kube-system` | 是 |
| 获取 GPU 监控指标 | `curl <NodeIP>:5678/metrics` | 是 |

## 操作步骤

### 确认 gpu-exporter 运行状态

```bash
# qGPU 安装的 exporter
kubectl get daemonset elastic-gpu-exporter -n kube-system
# expected: DaemonSet 存在，DESIRED == READY

# nvidia-gpu 安装的 exporter
kubectl get daemonset nvidia-gpu-exporter -n kube-system
# expected: DaemonSet 存在，DESIRED == READY
```

```text
NAME  STATUS  AGE
...
```

### 获取 GPU 监控指标

```bash
curl <NodeIP>:5678/metrics
```

### GPU 卡相关指标

| 指标 | 说明 | 标签 |
|------|------|------|
| `gpu_core_usage` | GPU 实际使用算力（%） | card, node |
| `gpu_mem_usage` | GPU 实际使用显存（MiB） | card, node |
| `gpu_core_utilization_percentage` | GPU 算力使用率 | card, node |
| `gpu_mem_utilization_percentage` | GPU 显存使用率 | card, node |
| `gpu_core_allocatable` | 可分配算力（仅 qGPU） | card, node |
| `gpu_core_allocated` | 已分配算力（仅 qGPU） | card, container_name, namespace, pod_name |
| `gpu_core_available` | 剩余算力（仅 qGPU） | card, node |
| `gpu_mem_allocatable` | 可分配显存 GiB（仅 qGPU） | card, node |
| `gpu_mem_allocated` | 已分配显存（仅 qGPU） | card, container_name, namespace, pod_name |
| `gpu_mem_available` | 剩余显存（仅 qGPU） | card, node |
| `gpu_enc_utilization_percentage` | 视频编码使用率 | card, node |
| `gpu_dec_utilization_percentage` | 视频解码使用率 | card, node |

### Pod 相关指标

| 指标 | 说明 | 标签 |
|------|------|------|
| `pod_core_usage` | Pod 实际使用算力（%） | namespace, node, pod |
| `pod_mem_usage` | Pod 实际使用显存（MiB） | namespace, node, pod |
| `pod_core_utilization_percentage` | Pod 实际使用算力占申请算力百分比 | namespace, node, pod |
| `pod_mem_utilization_percentage` | Pod 实际使用显存占申请显存百分比 | namespace, node, pod |
| `pod_core_occupy_node_percentage` | Pod 实际使用算力占节点总算力百分比 | namespace, node, pod |
| `pod_mem_occupy_node_percentage` | Pod 实际使用显存占节点总显存百分比 | namespace, node, pod |
| `pod_core_request` | Pod 申请算力（%） | namespace, node, pod |
| `pod_mem_request` | Pod 申请显存（MiB） | namespace, node, pod |

### 容器相关指标

| 指标 | 说明 |
|------|------|
| `container_assigned_card` | 容器分配到的卡序号 |
| `container_gpu_utilization` | 容器实际使用算力（%） |
| `container_gpu_memory_total` | 容器实际使用显存（MiB） |
| `container_core_utilization_percentage` | 容器实际使用算力占申请算力百分比 |
| `container_mem_utilization_percentage` | 容器实际使用显存占申请显存百分比 |
| `container_request_gpu_memory` | 容器申请显存（MiB） |
| `container_request_gpu_utilization` | 容器申请算力（%） |

### GPU 硬件相关指标

| 指标 | 说明 |
|------|------|
| `gpu_count` | 节点显卡数量 |
| `gpu_info` | 显卡信息（类型、驱动版本、UUID） |
| `gpu_mem_each_card` | 每张卡显存（MiB） |
| `gpu_power_usage` | 功率（W） |
| `gpu_ecc_mode` | 是否开启 ECC（1/0） |
| `gpu_ecc_error_count` | ECC 错误数量 |
| `gpu_persistence_mode` | 持久模式（1/0） |

## 验证

```bash
# 确认 exporter 运行
kubectl get pods -n kube-system | grep exporter
# expected: elastic-gpu-exporter 或 nvidia-gpu-exporter Pod Running

# 获取指标样本
kubectl exec -n kube-system <exporter-pod> -- curl -s localhost:5678/metrics | head -20
```

```text
NAME  STATUS  AGE
...
```

## 清理

本页为监控指标说明页，无资源需清理。

## 下一步

- [qGPU 概述](../GPU 共享/qGPU概述/tccli 操作.md)
- [使用 qGPU](../GPU 共享/使用 qGPU/tccli 操作.md)

## 控制台替代

[TKE 控制台 → 集群详情 → 组件管理](https://console.cloud.tencent.com/tke2/cluster)：安装 nvidia-gpu 或 qGPU 组件后，gpu-exporter 自动部署。监控数据可在云监控控制台查看。
