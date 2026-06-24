# 原生节点装箱率分析和提升（tccli）

> 对照官方：[原生节点装箱率分析和提升](https://cloud.tencent.com/document/product/457/97925) · page_id `97925`

## 概述

通过分析原生节点（TKE 节点池）的资源碎片和装箱情况，使用节点放大、资源请求优化等手段提升装箱率，减少资源浪费。

## 前置条件

- metrics-server 已安装

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 查看节点分配 | `kubectl describe node <name> \| grep Allocated` | 是 |
| 查看 Pod 请求 | `kubectl get pods -A -o json \| jq` | 是 |
| 启用节点放大 | `tccli tke UpdateAddon --AddonName craned` | 否 |
| 调整 Request | `kubectl set resources deploy <name> --requests=cpu=100m` | 是 |

## 操作步骤

### 1. 分析装箱率

```bash
kubectl describe node <node-name> | grep -A8 "Allocated resources"
```

```text
Name:         ...
Status:       Running
...
```

```output
Allocated resources:
  Resource           Requests     Limits
  cpu                1800m (45%)  2200m (55%)
  memory             4Gi (50%)    6Gi (75%)
```

### 2. 优化策略

- Pod Request 设置为实际用量的 1.2x
- 使用拓扑分散避免热点
- 启用节点放大（系数 1.2-1.5）

## 验证

```bash
kubectl top nodes
kubectl describe node <node> | grep "Allocated"
```

```text
Name:         ...
Status:       Running
...
```

## 清理

> 本文操作为只读/非持久化操作，无需清理。

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl` 连接超时/拒绝 | `tccli tke DescribeClusters` 返回集群 Running 但 kubectl 不可达 | CAM 策略 240463971 拒绝公网访问 APIServer 端点 | 通过 VPN/IOA 接入内网后执行 kubectl 命令 |
| `kubectl top nodes` 无数据 | `kubectl get apiservice | grep metrics-server` 检查服务状态 | metrics-server 未安装或未就绪 | 安装 metrics-server 组件：`tccli tke InstallAddon --AddonName metrics-server` |
| 装箱率低 | `kubectl describe node <node> | grep "Allocated"` 查看 Requests 百分比 | Pod Request 设置过大，远超实际用量 | 将 Request 设置为实际用量的 1.2x；启用节点放大 |

## 下一步

- [Crane 调度器介绍](../../../调度配置/调度组件概述/Crane%20调度器介绍/tccli%20操作.md)
- [节点放大](../../../调度配置/资源利用率优化调度/节点放大/tccli%20操作.md)

## 控制台替代

控制台：节点管理 → 点击节点 → 查看资源分配。
