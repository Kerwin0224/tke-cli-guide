# 资源利用率分析和提升最佳实践（tccli）

> 对照官方：[资源利用率分析和提升最佳实践](https://cloud.tencent.com/document/product/457/80787) · page_id `80787`

## 概述

通过 `kubectl top`、VPA、Crane 等工具分析和提升集群资源利用率。涵盖节点装箱率、Pod 资源使用趋势、闲置资源识别等。

## 前置条件

- metrics-server 已安装

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 查看节点用量 | `kubectl top nodes` | 是 |
| 查看 Pod 用量 | `kubectl top pods -A` | 是 |
| 查看节点分配 | `kubectl describe node <name> \| grep Allocated` | 是 |
| 安装 Crane | `tccli tke InstallAddon --AddonName craned` | 是 |

## 操作步骤

### 1. 分析当前用量

```bash
kubectl top nodes
kubectl top pods -A --sort-by=cpu
```

```text
# command executed successfully
```

### 2. 检查请求与实际用量的差距

```bash
kubectl describe node <node-name> | grep -A5 "Allocated resources"
```

```text
Name:         ...
Status:       Running
...
```

### 3. 安装 Crane 优化器

```bash
tccli tke InstallAddon --region ap-guangzhou --cli-input-json "{\"ClusterId\":\"<ClusterId>\",\"AddonName\":\"craned\"}"
```

### 4. 优化建议

- 设置合理的 Request（非远大于实际用量）
- 使用 LimitRange 防止过度配置
- 启用节点放大提升装箱率

## 验证

```bash
kubectl top nodes
kubectl top pods -A --sort-by=memory | head -10
```

```text
# command executed successfully
```

## 清理

> 本文操作为只读/非持久化操作，无需清理。

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl` 连接超时/拒绝 | `tccli tke DescribeClusters` 返回集群 Running 但 kubectl 不可达 | CAM 策略 240463971 拒绝公网访问 APIServer 端点 | 通过 VPN/IOA 接入内网后执行 kubectl 命令 |
| `kubectl top nodes` 无数据 | `kubectl get apiservice | grep metrics-server` 检查服务状态 | metrics-server 未安装或未就绪 | 安装 metrics-server 组件：`tccli tke InstallAddon --AddonName metrics-server` |
| Crane 安装失败 | `tccli tke DescribeAddon --ClusterId <ClusterId>` 查看 craned 状态 | 集群版本不满足 Crane 组件最低要求 | 升级集群版本或参考 Crane 组件文档 |

## 下一步

- [Crane 调度器介绍](../../../调度配置/调度组件概述/Crane%20调度器介绍/tccli%20操作.md)
- [节点放大](../../../调度配置/资源利用率优化调度/节点放大/tccli%20操作.md)

## 控制台替代

控制台：集群 → 资源监控 → 资源使用趋势。
