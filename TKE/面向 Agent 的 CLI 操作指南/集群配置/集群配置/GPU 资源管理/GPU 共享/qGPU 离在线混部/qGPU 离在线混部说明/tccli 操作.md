# qGPU 离在线混部说明

> 对照官方：[qGPU 离在线混部说明](https://cloud.tencent.com/document/product/457/81107) · page_id `81107`

## 概述

qGPU 离在线混部支持在线（高优）任务和离线（低优）任务同时混合部署在同一张 GPU 卡上。在内核与驱动层面实现了"低优 100% 使用闲置算力及高优 100% 抢占"，可将 GPU 利用率提升到 100%。

## 前置条件

- 已安装 qGPU 组件
- TKE 原生 GPU 节点
- kubectl 可达集群以 apply YAML

## 功能原理

### 两个 "100%" 控制

| 能力 | 说明 |
|------|------|
| 100% 使用高优闲置算力 | 高优任务闲时，低优任务可 100% 使用 GPU 算力 |
| 100% 抢占低优占用算力 | 高优任务忙时，可 100% 抢占低优任务所使用 GPU 算力 |

### 技术实现

1. **感知与响应**：高优 Pod 提交 GPU 计算任务时，qGPU 驱动在 1ms 内将算力全部提供给高优 Pod。高优任务结束时，100ms 后释放算力重新分配给低优 Pod。

2. **暂停与继续**：低优 Pod 在高优 Pod 有计算任务时立即暂停；高优任务结束后，低优 Pod 在中断点继续计算。

### 调度策略

| Pod 类型 | 调度行为 |
|---------|---------|
| 高优 Pod | 有计算任务后立即抢占 GPU，高优 Pod 之间为争抢模式，不受 policy 控制 |
| 低优 Pod | 高优休眠时按 policy 调度；高优唤醒时全部暂停；高优结束后按 policy 继续 |

### 典型场景

| 场景 | 高优任务 | 低优任务 |
|------|---------|---------|
| 在线推理 + 离线推理混部 | 线上推理（低延迟要求） | 数据预处理（实时性要求低） |
| 在线推理 + 离线训练混部 | 实时推理（可用性要求高） | 模型训练（资源使用多，延迟不敏感） |

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 安装 qGPU 组件 | `tccli tke InstallAddon --AddonName qgpu` | 否 |
| 创建在线 Pod | `kubectl apply -f <yaml>`（annotation `app-class: online`） | 是 |
| 创建离线 Pod | `kubectl apply -f <yaml>`（annotation `app-class: offline`） | 是 |
| 创建普通 Pod | `kubectl apply -f <yaml>`（无 app-class annotation） | 是 |

## 验证

```bash
kubectl describe node <gpu-node> | grep qgpu-core-greedy
# expected: 存在低优算力资源字段
```

```text
NAME  STATUS  AGE
...
```

## 清理

本页为概念说明页，无资源需清理。

## 下一步

- [使用 qGPU 离在线混部](../使用 qGPU 离在线混部/tccli 操作.md)
- [使用 qGPU](../../使用 qGPU/tccli 操作.md)

## 控制台替代

[TKE 控制台 → 工作负载](https://console.cloud.tencent.com/tke2/cluster)：新建 Deployment，在 Pod annotation 中设置 `tke.cloud.tencent.com/app-class`。
