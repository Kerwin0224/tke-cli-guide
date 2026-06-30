# CPU Burst（tccli）

> 对照官方：[CPU Burst](https://cloud.tencent.com/document/product/457/79776) · page_id `79776`

## 概述

CPU Burst 是 QoS 感知调度中的一项特性，允许容器在空闲时积累 CPU credit，在突发负载时短时间超越 CPU Limit 使用更多 CPU 资源，提升应用响应速度。TKE 通过 Crane PodQOS CRD 的 `cpuQOS.cpuBurst` 字段配置。

**工作原理**：CFS（Completely Fair Scheduler）Burst 机制允许 CPU 使用率较低的容器将未使用的 CPU 时间片累积为 credit。当突发流量到来时，容器可消耗 credit 短时间超越 limits.cpu 上限。QoSAgent 负责在节点上执行该策略。

**tccli 能力范围**：CPU Burst 为 CRD/kubectl 配置型特性，tccli 主要负责检查集群状态和 QoSAgent 组件状态。以下 kubectl 命令为数据面参考操作。

## 前置条件

- [环境准备](../../../环境准备.md)
- 集群中已安装 [QoSAgent](../QoSAgent/tccli%20操作.md) 组件（依赖 craned）
- 节点内核版本 >= 4.18（CFS Burst 特性依赖）
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
| 启用 CPU Burst（CRD） | `kubectl apply -f podqos-burst.yaml`（PodQOS `cpuQOS.cpuBurst: true`） | 是 |
| 查看 Burst 策略（数据面） | `kubectl get podqos -o yaml` | 是 |
| 验证 Burst 效果（数据面） | `kubectl top pod` | 是 |

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

确认 `Phase: "Succeeded"` 且 `Status: "Running"` 后，QoSAgent 已具备执行 CPU Burst 策略的能力。

### 步骤 2：创建 CPU Burst 策略（需 kubectl 可达环境）

CPU Burst 策略通过 PodQOS CRD 定义。`cpuBurstPercent` 控制突发时可超越 CPU Limit 的百分比，取值范围 0-100。

`podqos-burst.yaml`：

```yaml
apiVersion: scheduling.crane.io/v1alpha1
kind: PodQOS
metadata:
  name: cpu-burst-policy
spec:
  resourceQOS:
    cpuQOS:
      cpuBurst: true
      cpuBurstPercent: 100
  selector:
    matchLabels:
      cpu-burst: enabled
```

```bash
# 需 kubectl 可达环境
kubectl apply -f podqos-burst.yaml
# expected: podqos.scheduling.crane.io/cpu-burst-policy created
```

### 步骤 3：创建 Burst 测试 Pod（需 kubectl 可达环境）

Pod 需带有 PodQOS selector 匹配的 label。

`burst-pod.yaml`：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: burst-test
  labels:
    cpu-burst: enabled
spec:
  containers:
  - name: stress
    image: alpine:latest
    command: ["sh", "-c", "apk add stress-ng && stress-ng --cpu 2 --timeout 60s"]
    resources:
      requests:
        cpu: 100m
      limits:
        cpu: 200m
```

```bash
# 需 kubectl 可达环境
kubectl apply -f burst-pod.yaml
# expected: pod/burst-test created
```

### 步骤 4：验证 Burst 生效（需 kubectl 可达环境）

```bash
# 需 kubectl 可达环境
kubectl top pod burst-test
```

```text
# command executed successfully
```

Pod 实际 CPU 使用可短暂超过 `limits.cpu: 200m`，说明 Burst 生效。

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
    "RequestId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
}
```

### 数据面（需 kubectl 可达环境）

```bash
# 确认 PodQOS 策略已创建且 cpuBurst 为 true
kubectl get podqos cpu-burst-policy -o yaml | grep cpuBurst
# expected: cpuBurst: true

# 确认 Pod 已匹配策略（label 匹配）
kubectl get pod -l cpu-burst=enabled
# expected: burst-test   1/1   Running
```

```text
NAME  STATUS  AGE
...
```

## 清理

### 数据面（需 kubectl 可达环境）

```bash
# 删除测试 Pod
kubectl delete pod burst-test
# expected: pod "burst-test" deleted

# 删除 CPU Burst 策略
kubectl delete podqos cpu-burst-policy
# expected: podqos.scheduling.crane.io "cpu-burst-policy" deleted
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl apply -f podqos-burst.yaml` 返回 "no matches for kind PodQOS" | `tccli tke DescribeAddon --region ap-guangzhou --ClusterId cls-xxxxxxxx --AddonName QoSAgent` 确认组件状态 | QoSAgent 组件未安装，PodQOS CRD 未注册到集群 | 先参见 [QoSAgent](../QoSAgent/tccli%20操作.md) 安装组件 |

### Burst 策略不生效

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Pod 已匹配策略但 CPU 使用未超越 Limit | 检查节点内核版本 `uname -r` | 节点内核版本 < 4.18，CFS Burst 不支持 | 升级节点内核至 4.18+ |
| Pod 未匹配策略 | `kubectl get pod --show-labels \| grep cpu-burst` 确认 Pod label | Pod label 未匹配 PodQOS selector | 添加与 selector 匹配的 label |
| `kubectl top pod` 无法显示数据 | `kubectl get pods -n kube-system -l k8s-app=metrics-server` | metrics-server 未安装或异常 | 安装或修复 metrics-server |

## 下一步

- [CPU 使用优先级](../CPU%20使用优先级/tccli%20操作.md) — 基于 PodQOS 设置 CPU 使用优先级
- [内存精细调度](../内存精细调度/tccli%20操作.md) — 通过 PodQOS 实现内存水位控制
- [QoSAgent 组件介绍](../组件介绍/QoSAgent/tccli%20操作.md) — 了解 QoSAgent 架构与工作原理

## 控制台替代

无独立控制台界面。CPU Burst 通过 CRD 配置，在集群中 [安装 QoSAgent](https://console.cloud.tencent.com/tke2/cluster) 后，通过 kubectl 管理 PodQOS 资源。
