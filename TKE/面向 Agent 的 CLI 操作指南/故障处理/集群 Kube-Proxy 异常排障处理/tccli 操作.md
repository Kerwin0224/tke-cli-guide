# 集群 Kube-Proxy 异常排障处理（tccli）

> 对照官方：[集群 Kube-Proxy 异常排障处理](https://cloud.tencent.com/document/product/457/79797) · page_id `79797`

## 概述

排查 TKE 集群 Kube-Proxy 异常问题：Service 无法访问、iptables/IPVS 规则异常、kube-proxy Pod 异常等。通过 tccli 检查集群和节点状态，配合 kubectl 查看 kube-proxy 组件和日志。

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
| 查看节点实例 | `tccli tke DescribeClusterInstances --region ap-guangzhou --ClusterId cls-xxxxxxxx` | 是 |
| 查看 kube-proxy Pod | `kubectl get pods -n kube-system -l k8s-app=kube-proxy -o wide`（需 VPN/IOA） | 是 |
| 查看 kube-proxy 日志 | `kubectl logs -n kube-system -l k8s-app=kube-proxy --tail=50`（需 VPN/IOA） | 是 |
| 查看 Service 规则 | `kubectl get svc -A`（需 VPN/IOA） | 是 |

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

### 步骤2：数据面检查 kube-proxy（需 VPN/IOA）

```bash
# 检查每节点 kube-proxy Pod 状态
kubectl get pods -n kube-system -l k8s-app=kube-proxy -o wide
# 预期: 每节点一个 Pod，STATUS 为 Running

# 查看 kube-proxy 日志
kubectl logs -n kube-system -l k8s-app=kube-proxy --tail=50 | grep -i "error\|warn\|timeout"
# 预期: 正常同步日志，无持续错误

# 查看 kube-proxy 配置模式
kubectl logs -n kube-system -l k8s-app=kube-proxy --tail=5 | grep -i "Using"
# 预期: "Using iptables Proxier" 或 "Using ipvs Proxier"
```

```text
NAME  STATUS  AGE
...
```

### 步骤3：诊断 Service 不可达

**确认 Endpoints 正常**

```bash
kubectl get endpoints <svc-name> -n <ns>
# 预期: ENDPOINTS 列有后端 Pod IP
```

```text
NAME  STATUS  AGE
...
```

**集群内测试**

```bash
kubectl run test-proxy --image=busybox --rm -it --restart=Never -- wget -O- -T3 http://<svc-name>.<ns>.svc.cluster.local:<port>
# 预期: 返回预期内容
```

**检查 iptables 规则（需 SSH 到节点）**

```bash
ssh root@<node-ip>
iptables -t nat -L KUBE-SERVICES -n | grep <svc-cluster-ip>
# 预期: 存在对应 Service 的 DNAT 规则
```

### 步骤4：按代理模式修复

**iptables 模式**

1. 重启异常节点的 kube-proxy Pod
2. 若规则丢失，手动触发：`kubectl delete pod -n kube-system <kube-proxy-pod>`

**IPVS 模式**

1. 检查 IPVS 规则：`ipvsadm -Ln`（需 SSH 到节点）
2. 确认内核模块加载：`lsmod | grep ip_vs`
3. 重建规则：删除并重建 kube-proxy Pod

**通用修复**

1. 检查 kube-proxy ConfigMap：`kubectl describe configmap -n kube-system kube-proxy`
2. 确认 API Server 可达：kube-proxy 通过 API Server watch Service/Endpoints
3. 调整 conntrack 表大小（高 Service 数场景）

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
kubectl get pods -n kube-system -l k8s-app=kube-proxy
# 预期: 每节点一个 Running Pod

kubectl run test-proxy --image=busybox --rm -it --restart=Never -- wget -O- -T3 http://<svc-name>.<ns>.svc.cluster.local:<port>
# 预期: 返回预期内容
```

```text
NAME  STATUS  AGE
...
```

## 清理

无需特殊清理（排障页）。测试用 Pod 使用 `--rm` 会自动清理。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|---|---|---|---|
| kube-proxy 日志中 `Failed to list *v1.Service` | `kubectl logs` 查看具体错误 | kube-proxy 无法连接 API Server | 检查 kube-proxy kubeconfig；检查 API Server 端点 |
| IPVS 模式 Service 异常 | `ipvsadm -Ln` 检查规则 | IPVS 规则未正确同步或内核模块缺失 | `modprobe ip_vs`；重建 kube-proxy Pod |
| conntrack 表满 | `dmesg \| grep "table full"` | Service 和连接数过多 | 增大 `nf_conntrack_max`：`sysctl -w net.netfilter.nf_conntrack_max=1048576` |

### 操作成功但异常

| 现象 | 诊断 | 根因 | 修复 |
|---|---|---|---|
| 特定 Service 偶发不可达 | 在问题节点上测试 Service | 该节点 kube-proxy 规则滞后 | 重启该节点 kube-proxy Pod |
| 新建 Service 延迟可达 | 监控 Service 创建到可达的时间差 | kube-proxy 同步周期过长（默认 syncPeriod） | 调整 kube-proxy 配置缩短同步间隔 |

## 下一步

- [节点常见报错与处理](../节点常见报错与处理/tccli%20操作.md) -- page_id `89869`
- [Service&Ingress 常见报错和处理](../Service&Ingress%20常见报错和处理/tccli%20操作.md) -- page_id `80913`

## 控制台替代

[容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) -> 集群 -> 组件管理 -> kube-proxy -> 查看日志。
