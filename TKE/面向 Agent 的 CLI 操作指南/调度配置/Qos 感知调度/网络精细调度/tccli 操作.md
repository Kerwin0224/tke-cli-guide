# 网络精细调度（tccli）

> 对照官方：[网络精细调度](https://cloud.tencent.com/document/product/457/79780) · page_id `79780`

## 概述

网络精细调度（Network Fine-Grained Scheduling）是 QoS 感知调度中的一项特性，TKE Crane 通过 CNRP（Container Network Resource Policy）CRD 实现 Pod 级网络精细控制。支持配置 Pod 的出站/入站带宽限制、网络优先级，对网络敏感型业务进行精准控制。

**工作原理**：
- **CNRP CRD**：定义网络 QoS 策略，包括 `netPriority`（网络优先级）和 `netBandwidth`（入/出站带宽上限）。
- **Pod 匹配**：Pod 的 label 与 CNRP `selector.matchLabels` 匹配时，QoSAgent 对该 Pod 施加网络限制。
- **QoSAgent 执行**：通过 Linux tc（Traffic Control）和 net_cls cgroup 实现带宽限速和优先级排队。

**tccli 能力范围**：网络精细调度为 CRD/kubectl 配置型特性，tccli 主要负责检查集群状态和 QoSAgent 组件状态。以下 kubectl 命令为数据面参考操作。

## 前置条件

- [环境准备](../../../环境准备.md)
- 集群中已安装 [QoSAgent](../QoSAgent/tccli%20操作.md) 组件（依赖 craned）
- 节点内核需支持 tc（Traffic Control）和 net_cls cgroup
- 演示集群 `cls-xxxxxxxx`（ap-guangzhou，v1.30.0，Running），kubectl 因 CAM 策略限制不可达，需通过 VPN/IOA 内网或数据面集群操作。以下 kubectl 命令为参考格式。

### 控制面检查

```bash
# 确认集群状态正常
tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]'
```

```json
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "cls-xxxxxxxx",
            "ClusterName": "demo-cluster",
            "ClusterType": "MANAGED_CLUSTER",
            "ClusterStatus": "Running",
            "ClusterVersion": "1.30.0",
            "ClusterNodeNum": 3,
            "ClusterOs": "tlinux3.1",
            "VpcId": "vpc-xxxxxxxx",
            "CreatedTime": "2024-06-15T10:30:00Z"
        }
    ],
    "RequestId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
}
```

```bash
# 确认 QoSAgent 组件已安装且运行正常
tccli tke DescribeAddon --region ap-guangzhou \
    --ClusterId cls-xxxxxxxx \
    --AddonName QoSAgent
```

```json
{
    "Addons": [
        {
            "AddonName": "QoSAgent",
            "AddonVersion": "v1.2.0",
            "Status": "Succeeded",
            "Phase": "Running"
        }
    ],
    "RequestId": "b2c3d4e5-f6a7-8901-bcde-f12345678901"
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 查看集群状态 | `tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]'` | 是 |
| 查看 QoSAgent 组件状态 | `tccli tke DescribeAddon --region ap-guangzhou --ClusterId cls-xxxxxxxx --AddonName QoSAgent` | 是 |
| 创建 CNRP 策略 | `kubectl apply -f cnrp.yaml` | 是 |
| 查看 CNRP 策略 | `kubectl get cnrp -o yaml` | 是 |
| 为 Pod 指定网络策略 | Pod label 匹配 CNRP selector | 是 |
| 验证带宽限制 | `kubectl exec` 执行 iperf3 测速对比 | 是 |

## 操作步骤

### 步骤 1：确认控制面状态

```bash
tccli tke DescribeAddon --region ap-guangzhou \
    --ClusterId cls-xxxxxxxx \
    --AddonName QoSAgent
```

```json
{
    "Addons": [
        {
            "AddonName": "QoSAgent",
            "AddonVersion": "v1.2.0",
            "Status": "Succeeded",
            "Phase": "Running"
        }
    ],
    "RequestId": "c3d4e5f6-a7b8-9012-cdef-123456789012"
}
```

### 步骤 2：创建 CNRP 网络策略（需 kubectl 可达环境）

`cnrp-net-qos.yaml`：

```yaml
apiVersion: scheduling.crane.io/v1alpha1
kind: CNRP
metadata:
  name: net-qos-policy
spec:
  resourceQOS:
    networkQOS:
      netPriority: 0
      netBandwidth:
        ingressBandwidth: "100M"
        egressBandwidth: "100M"
  selector:
    matchLabels:
      net-qos: high
```

```bash
# 需 kubectl 可达环境
kubectl apply -f cnrp-net-qos.yaml
# expected: cnrp.scheduling.crane.io/net-qos-policy created
```

### 步骤 3：标记 Pod 应用网络策略（需 kubectl 可达环境）

```bash
# 需 kubectl 可达环境
# 为已有 Pod 添加 label
kubectl label pod <pod-name> net-qos=high
# expected: pod/<pod-name> labeled
```

或在 Pod 创建时指定：

`net-qos-pod.yaml`：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: net-qos-app
  labels:
    net-qos: high
spec:
  containers:
  - name: app
    image: nginx:alpine
    ports:
    - containerPort: 80
```

```bash
# 需 kubectl 可达环境
kubectl apply -f net-qos-pod.yaml
# expected: pod/net-qos-app created
```

### CNRP 字段说明

| 字段 | 类型 | 说明 |
|------|------|------|
| `netPriority` | int | 网络优先级。0=独占（最高），3=低。值越小优先级越高 |
| `ingressBandwidth` | string | 入站带宽上限。支持 M（Mbps）、G（Gbps），如 `"100M"`、`"1G"` |
| `egressBandwidth` | string | 出站带宽上限。格式同 ingressBandwidth |

### netPriority 等级

| netPriority | 名称 | 行为 |
|:----------:|------|------|
| 0 | 独占（Exclusive） | 最高网络优先级，带宽保障不受低优 Pod 干扰 |
| 1 | 高优先级（High） | 网络争抢时优先获得带宽 |
| 2 | 中优先级（Medium） | 默认级别 |
| 3 | 低优先级（Low） | 网络拥塞时优先被限速 |

### 工作原理

1. **策略定义**：通过 CNRP CRD 定义网络 QoS 策略，包括优先级和带宽限制。
2. **Pod 匹配**：CNRP `selector.matchLabels` 匹配 Pod label，QoSAgent 对该 Pod 施加限制。
3. **tc qdisc 配置**：QoSAgent 在节点上通过 tc（Traffic Control）创建 qdisc（排队规则）：
   - 使用 HTB（Hierarchical Token Bucket）实现带宽限速
   - 使用 prio qdisc 实现多优先级队列
4. **net_cls cgroup**：将 Pod 的虚拟网卡（veth pair）veth 接口关联到对应的 tc class，实现流出流量限速。
5. **实时调整**：Pod 创建/销毁或策略变更时，QoSAgent 实时更新 tc 配置。

## 验证

### 控制面

```bash
tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]'
```

```json
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "cls-xxxxxxxx",
            "ClusterName": "demo-cluster",
            "ClusterStatus": "Running",
            "ClusterVersion": "1.30.0",
            "ClusterNodeNum": 3
        }
    ],
    "RequestId": "d4e5f6a7-b8c9-0123-def0-234567890123"
}
```

### 数据面（需 kubectl 可达环境）

```bash
# 确认 CNRP 策略已创建
kubectl get cnrp net-qos-policy -o yaml | grep -A5 networkQOS
# expected: 包含 netPriority 和 netBandwidth 字段

# 确认 Pod 已匹配策略
kubectl get pod -l net-qos=high
# expected: net-qos-app   1/1   Running

# 在节点上确认 tc qdisc 配置（需 ssh 到节点）
tc qdisc show dev <veth-interface>
# expected: 包含 HTB qdisc 和对应的 class/rate 限制

# 验证带宽限制效果（使用 iperf3）
# 在另一个 Pod 中启动 iperf3 server
kubectl exec net-qos-app -- iperf3 -s &
# 从其他 Pod 测试带宽
kubectl exec <test-pod> -- iperf3 -c <net-qos-app-ip> -t 10
# expected: 带宽不超过 CNRP 中配置的 ingressBandwidth/egressBandwidth 值
```

```text
NAME  STATUS  AGE
...
```

## 清理

### 数据面（需 kubectl 可达环境）

```bash
# 删除测试 Pod
kubectl delete pod net-qos-app
# expected: pod "net-qos-app" deleted

# 删除 CNRP 策略
kubectl delete cnrp net-qos-policy
# expected: cnrp.scheduling.crane.io "net-qos-policy" deleted

# 确认已清理
kubectl get cnrp
# expected: No resources found 或列表中不含 net-qos-policy
```

```text
NAME  STATUS  AGE
...
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl apply -f cnrp-net-qos.yaml` 返回 "no matches for kind CNRP" | `tccli tke DescribeAddon --region ap-guangzhou --ClusterId cls-xxxxxxxx --AddonName QoSAgent` | QoSAgent 未安装，CNRP CRD 未注册 | 先参见 [QoSAgent](../QoSAgent/tccli%20操作.md) 安装 |
| `kubectl get cnrp` 返回 "the server doesn't have a resource type" | 同上 | craned 未安装，CRD 未注册 | 确认 craned 组件已安装 |

### 带宽限制不生效

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 策略已创建但 Pod 带宽未被限制 | `tc qdisc show` 检查节点 tc 配置；内核 `uname -r` | 内核不支持 tc HTB/prio 或 QoSAgent 未正确配置 tc | 确认节点内核支持；检查 qos-agent 日志 |
| bandwidth 值设置为 0 或极大 | 检查 CNRP `netBandwidth` 字段值 | 配置错误 | 使用有效带宽值（如 `"100M"`） |
| Pod 未匹配 CNRP selector | `kubectl get pod --show-labels` | Pod label 不匹配 | 修正 label |
| 部分节点限制不生效 | `kubectl get pods -n crane-system -l app=qos-agent -o wide` | 部分节点 qos-agent 未运行 | 排查节点污点和 DaemonSet 调度状态 |
| 高速率时限制不精确 | 使用较低 bandwidth 值测试 | tc HTB 在高带宽下有一定精度偏差 | 正常现象，偏差通常在 5% 以内 |

## 下一步

- [磁盘 IO 精细调度](../磁盘%20IO%20精细调度/tccli%20操作.md) — 通过 PodQOS 实现磁盘 IOPS/BPS 限制
- [内存精细调度](../内存精细调度/tccli%20操作.md) — 通过 PodQOS 实现内存 OOM 优先级控制
- [CPU 使用优先级](../CPU%20使用优先级/tccli%20操作.md) — 通过 PodQOS 设置 CPU 优先级
- [Pod 带宽限速](../../../实践教程/网络/在%20TKE%20上对%20Pod%20进行带宽限速/tccli%20操作.md) — 控制台 Pod 带宽限速功能
- [QoSAgent 组件介绍](../组件介绍/QoSAgent/tccli%20操作.md) — 了解 QoSAgent 架构与工作原理

## 控制台替代

无独立控制台界面。网络精细调度通过 CNRP CRD 配置，在集群中 [安装 QoSAgent](https://console.cloud.tencent.com/tke2/cluster) 后，通过 kubectl 管理 CNRP 资源。
