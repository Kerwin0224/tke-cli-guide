# 使用 TKE NPDPlus 插件增强节点的故障自愈能力（tccli）

> 对照官方：[使用 TKE NPDPlus 插件增强节点的故障自愈能力](https://cloud.tencent.com/document/product/457/49376) · page_id `49376`

## 概述

NPDPlus（Node Problem Detector Plus）在社区 NPD 基础上扩展了更多故障检测项和自愈能力。通过 `InstallAddon` 安装，自动检测内核死锁、磁盘只读、OOM 等节点故障。

## 前置条件

- [环境准备](../../../环境准备.md)

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 安装 NPDPlus | `tccli tke InstallAddon --AddonName npdplus` | 是 |
| 查看节点状态 | `kubectl describe node <name> \| grep Conditions` | 是 |
| 查看 NPD 日志 | `kubectl logs -n kube-system -l name=node-problem-detector` | 是 |

## 操作步骤

### 1. 安装

```bash
tccli tke InstallAddon --region ap-guangzhou --cli-input-json "{\"ClusterId\":\"<ClusterId>\",\"AddonName\":\"npdplus\"}"
```

### 2. 查看检测到的节点状态

```bash
kubectl describe node <node-name> | grep -A1 "Conditions:"
```

```text
Name:         ...
Status:       Running
...
```

```output
Conditions:
  KernelDeadlock          False
  ReadonlyFilesystem      False
  FrequentKubeletRestart  False
```

### 3. 自愈配置

NPDPlus 检测到故障后，可自动执行：
- 封锁节点（cordon）
- 驱逐 Pod（drain）
- 重启服务

## 验证

```bash
kubectl get pods -n kube-system -l name=node-problem-detector
kubectl logs -n kube-system <npd-pod> --tail=20
```

```text
NAME  STATUS  AGE
...
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl describe node` 报 `Unable to connect to the server` | `tccli tke DescribeClusters --region ap-guangzhou --ClusterId cls-xxxxxxxx --filter "Clusters[].ClusterStatus"` | CAM 拒绝公网端点（strategyId 240463971），数据面不可达 | 接入 VPN/IOA 后在内网执行 kubectl |
| `InstallAddon` 返回 `ResourceInsufficient` 或超时 | `tccli tke DescribeClusters --region ap-guangzhou --ClusterId cls-xxxxxxxx --filter "Clusters[].ClusterStatus"` | 集群未 Running 或节点资源不足 | 确认集群 Running；补充节点资源后重新安装 npdplus |
| `kubectl get pods -l name=node-problem-detector` 无结果 | `kubectl get pods -n kube-system`（VPN/IOA） | NPDPlus 未安装成功或 Pod 未调度 | 重新 `InstallAddon`；`kubectl describe pod` 查看调度失败原因 |

## 清理

```bash
tccli tke DeleteAddon --region ap-guangzhou --cli-input-json "{\"ClusterId\":\"<ClusterId>\",\"AddonName\":\"npdplus\"}"
```

## 下一步

- [节点常见报错与处理](../../../故障处理/节点常见报错与处理/tccli%20操作.md)
- [集群 API Server 网络无法访问排障处理](../../../故障处理/集群%20API%20Server%20网络无法访问排障处理/tccli%20操作.md)

## 控制台替代

控制台：组件管理 → 安装 npdplus。
