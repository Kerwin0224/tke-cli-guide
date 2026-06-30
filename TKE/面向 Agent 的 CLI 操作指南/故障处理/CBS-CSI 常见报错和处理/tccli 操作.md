# CBS-CSI 常见报错和处理（tccli）

> 对照官方：[CBS-CSI 常见报错和处理](https://cloud.tencent.com/document/product/457/81071) · page_id `81071`

## 概述

CBS-CSI 驱动负责管理 CBS 云硬盘的挂载。常见报错包括：磁盘未找到、跨可用区挂载、多 Pod 同时挂载冲突等。通过 tccli 检查集群和节点状态，配合 kubectl 查看 PVC/PV 和 CBS CSI 组件日志进行排障。

> **kubectl 数据面不可达**：CAM拒绝公网端点(strategyId:240463971)，内网需VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

## 前置条件

- [环境准备](../../环境准备.md)
- 已获取集群 kubeconfig（如需 kubectl 操作）

### 环境检查

```bash
tccli --version
# 预期输出: tccli version X.X.X
```

### 资源检查

```bash
tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]'
```

**预期输出**：

```json
{
  "TotalCount": 1,
  "Clusters": [
    {
      "ClusterId": "cls-xxxxxxxx",
      "ClusterName": "my-test-cluster",
      "ClusterStatus": "Running",
      "ClusterVersion": "1.30.0",
      "ClusterType": "MANAGED_CLUSTER"
    }
  ]
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli / CLI | 幂等 |
|---|---|---|
| 查看集群状态 | `tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]'` | 是 |
| 查看 PVC 状态 | `kubectl get pvc`（需 VPN/IOA） | 是 |
| 查看 PV 状态 | `kubectl get pv`（需 VPN/IOA） | 是 |
| 查看 CBS CSI Pod | `kubectl get pods -n kube-system -l app=csi-cbscontroller`（需 VPN/IOA） | 是 |
| 查看 PVC 事件 | `kubectl describe pvc <name>`（需 VPN/IOA） | 是 |

## 操作步骤

### 步骤1：控制面检查集群和节点

```bash
tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]'
```

**预期输出**：

```json
{
  "Clusters": [
    {
      "ClusterId": "cls-xxxxxxxx",
      "ClusterStatus": "Running",
      "ClusterVersion": "1.30.0"
    }
  ]
}
```

```bash
tccli tke DescribeClusterInstances --region ap-guangzhou --ClusterId cls-xxxxxxxx
```

**预期输出**：

```json
{
  "TotalCount": 3,
  "InstanceSet": [
    {
      "InstanceId": "ins-xxxxxxxx",
      "InstanceRole": "WORKER",
      "InstanceState": "running"
    }
  ]
}
```

### 步骤2：数据面排障（需 VPN/IOA）

```bash
# 查看 PVC 绑定状态
kubectl get pvc -A
# 预期: STATUS 为 Bound

# 查看 CBS CSI 控制器日志
kubectl logs -n kube-system -l app=csi-cbscontroller --tail=50
# 预期: 正常挂载/卸载日志，无持续错误
```

```text
NAME  STATUS  AGE
...
```

### 步骤3：常见报错诊断（需 VPN/IOA）

```bash
kubectl describe pvc <pvc-name>
# 关注 Events 区域的错误信息
```

```text
Name:         ...
Status:       Running
...
```

### 常见报错速查表

| 错误信息 | 根因 | 修复 |
|---|---|---|
| `disk is not found` | CBS 磁盘已被手动删除 | 删除并重新创建 PVC，让 CSI 自动创建新磁盘 |
| `disk is already attached` | 多个 Pod 共享 RWO 模式 PVC | 改用 RWX 模式或确保单副本运行 |
| `zone mismatch` | Pod 调度到与 CBS 不同可用区的节点 | 为 Pod 添加 nodeSelector 指定 CBS 所在可用区 |
| `attach timeout` | CBS 挂载超时，可能磁盘状态异常 | 在 CBS 控制台检查磁盘状态，必要时卸载后重试 |

## 验证

### 控制面（tccli）

```bash
tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]'
```

**预期输出**：

```json
{
  "Clusters": [
    {
      "ClusterId": "cls-xxxxxxxx",
      "ClusterStatus": "Running"
    }
  ]
}
```

### 数据面（需 VPN/IOA）

```bash
kubectl get pvc -A
# 预期: 所有 PVC STATUS 为 Bound

kubectl get pods -n kube-system -l app=csi-cbscontroller
# 预期: 所有 CBS CSI Pod Running
```

```text
NAME  STATUS  AGE
...
```

## 清理

```bash
# 删除测试 PVC（需 VPN/IOA）
kubectl delete pvc <pvc-name>
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|---|---|---|---|
| `kubectl get pvc` 显示 Pending | PVC 未找到匹配的 StorageClass | StorageClass 未定义或 provisioner 不匹配 | `kubectl get sc` 确认 StorageClass；检查 provisioner 是否为 `com.tencent.cloud.csi.cbs` |
| CBS CSI Pod CrashLoopBackOff | `kubectl logs` 显示连接 CBS API 失败 | 节点 CAM 角色缺少 CBS 权限 | 为节点 CAM 角色添加 `QcloudCBSFullAccess` 策略 |
| PVC 长时间 Pending | `kubectl describe pvc` 无 Events | CBS 创建接口超时或配额不足 | 检查 CBS 配额；`tccli cbs DescribeDiskConfigQuota` |

### 操作成功但异常

| 现象 | 诊断 | 根因 | 修复 |
|---|---|---|---|
| PVC Bound 但 Pod 挂载失败 | `kubectl describe pod` 显示 `FailedMount` | 节点 kubelet 与 CBS API 网络不通 | 检查节点安全组出站规则，确保可访问 CBS 服务端点 |
| 磁盘数据丢失 | Pod 删除后 PVC 数据丢失 | PVC 的 `persistentVolumeReclaimPolicy` 为 Delete | 改为 Retain 策略：`kubectl patch pv <pv-name> -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'` |

## 下一步

- [通过 CBS-CSI 避免云硬盘跨可用区挂载](../../应用配置/组件和应用管理/组件管理/CBS-CSI%20说明/通过%20CBS-CSI%20避免云硬盘跨可用区挂载/tccli%20操作.md)
- [节点磁盘爆满排障处理](../节点磁盘爆满排障处理/tccli%20操作.md) -- page_id `43126`

## 控制台替代

[容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) -> 存储 -> PersistentVolumeClaim -> 查看事件。
