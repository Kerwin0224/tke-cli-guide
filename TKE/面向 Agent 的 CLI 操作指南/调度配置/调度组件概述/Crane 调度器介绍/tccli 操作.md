# Crane 调度器介绍（tccli）

> 对照官方：[Crane 调度器介绍](https://cloud.tencent.com/document/product/457/75472) · page_id `75472`

## 概述

Crane 是腾讯云开源的云原生资源优化平台。TKE 集成 Crane 调度器（AddonName=`CraneScheduler`）作为 kube-scheduler 的增强替代，核心优势在于基于节点**真实资源利用率**（而非 Pod request 声明值）进行调度决策。

Crane 调度器包含三个关键子组件：

| 子组件 | 角色 | 说明 |
|-------|------|------|
| **CraneScheduler** | 调度内核 | 替代 kube-scheduler 的二次调度器，基于 Prometheus 实时指标决定 Pod 放置 |
| **craned** | 控制面 | 管理 QoS 分类、资源画像采集、Pod 资源推荐 |
| **CraneDescheduler** | 重调度器 | 监控节点负载，将 Pod 从过载节点迁移至低负载节点 |

安装后，用户通过 SchedulingPolicy CRD 定义调度策略，无需重启任何组件即可生效。

## 前置条件

- [环境准备](../../../环境准备.md)

演示集群信息：**cls-xxxxxxxx**（ap-guangzhou，v1.30.0，Running，3 节点）。注意：kubectl 因 CAM 策略限制不可达，需通过 VPN/IOA 内网或数据面集群操作。以下 kubectl 命令为参考格式。

### 环境检查

```bash
# 1. 检查 tccli 和 kubectl 版本
tccli --version
# expected: tccli version >= 1.0.0

kubectl version --client
# expected: Client Version >= 1.30.0

# 2. 检查 CAM 权限
#    需要: tke:DescribeClusters, tke:InstallAddon, tke:DescribeAddon
tccli tke DescribeClusters --region ap-guangzhou
# expected: exit 0，返回集群列表
```

**预期输出**：

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

### 资源检查

```bash
# 3. 确认目标集群存在
tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]'
# expected: TotalCount >= 1，ClusterStatus: "Running"
```

**预期输出**：

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
# 4. 检查是否已安装 CraneScheduler
tccli tke DescribeAddon --region ap-guangzhou \
    --ClusterId cls-xxxxxxxx \
    --AddonName CraneScheduler
# expected: 如未安装，返回 ResourceNotFound
```

**预期输出**（未安装时）：

```json
{
    "Error": {
        "Code": "ResourceNotFound",
        "Message": "Addon not found"
    },
    "RequestId": "d4e5f6a7-b8c9-0123-defa-234567890123"
}
```

```bash
# 5. 查询可用版本
tccli tke DescribeAddonValues --region ap-guangzhou \
    --ClusterId cls-xxxxxxxx \
    --AddonName CraneScheduler
# expected: 返回可用版本列表
```

**预期输出**：

```json
{
    "Values": [
        {
            "Name": "CraneScheduler",
            "DefaultVersion": "v1.2.0",
            "Versions": ["v1.0.0", "v1.1.0", "v1.2.0"]
        }
    ],
    "RequestId": "e5f6a7b8-c9d0-1234-efab-345678901234"
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 安装 Crane 调度器 | `InstallAddon --AddonName CraneScheduler` | 是 |
| 查看组件状态 | `DescribeAddon --AddonName CraneScheduler` | 是 |
| 查询可用版本 | `DescribeAddonValues --AddonName CraneScheduler` | 是 |
| 升级组件 | `UpdateAddon --AddonName CraneScheduler` | 否 |
| 卸载组件 | `DeleteAddon --AddonName CraneScheduler` | 否 |
| 配置调度策略 | kubectl `apply -f schedulingpolicy.yaml`（CRD） | 否 |

### 关键字段说明

| 字段 | 类型 | 必填 | 取值与约束 |
|------|------|:--:|------|
| `ClusterId` | String | 是 | 目标集群 ID，如 `cls-xxxxxxxx` |
| `AddonName` | String | 是 | `CraneScheduler`，C 和 S 必须大写 |
| `AddonVersion` | String | 否 | 版本号如 `v1.2.0`，不指定则安装最新版 |

## 操作步骤

### 步骤 1：安装 CraneScheduler 组件

```bash
tccli tke InstallAddon --region ap-guangzhou \
    --ClusterId cls-xxxxxxxx \
    --AddonName CraneScheduler \
    --AddonVersion v1.2.0
# expected: exit 0，组件开始安装
```

**预期输出**：

```json
{
    "RequestId": "c3d4e5f6-a7b8-9012-cdef-123456789012"
}
```

### 步骤 2：轮询安装状态

```bash
tccli tke DescribeAddon --region ap-guangzhou \
    --ClusterId cls-xxxxxxxx \
    --AddonName CraneScheduler
# expected: Status: "Succeeded", Phase: "Running"
```

**预期输出**：

```json
{
    "Addons": [
        {
            "AddonName": "CraneScheduler",
            "AddonVersion": "v1.2.0",
            "Status": "Succeeded",
            "Phase": "Running"
        }
    ],
    "RequestId": "b2c3d4e5-f6a7-8901-bcde-f12345678901"
}
```

### 步骤 3：验证数据面

```bash
# 获取 kubeconfig（需 VPN/IOA 内网使 kubectl 可达）
tccli tke DescribeClusterKubeconfig --region ap-guangzhou --ClusterId cls-xxxxxxxx \
    --output json | jq -r '.Kubeconfig' > ~/.kube/config-cls-xxxxxxxx
export KUBECONFIG=~/.kube/config-cls-xxxxxxxx

# 验证 Pod
kubectl get pods -n crane-system
# expected: crane-scheduler Pod 全部 Running
```

**预期输出**：

```text
NAME                                            READY   STATUS    RESTARTS   AGE
crane-scheduler-xxxxx                            1/1     Running   0          2m
crane-scheduler-controller-xxxxxxxxxx-xxxxx      1/1     Running   0          2m
```

```bash
# 验证 CRD
kubectl get crd | grep scheduling.crane.io
# expected: podqos、schedulingpolicies、nodeqos 等已注册
```

**预期输出**：

```text
podqos.scheduling.crane.io              2024-06-15T10:32:00Z
schedulingpolicies.scheduling.crane.io  2024-06-15T10:32:00Z
nodeqos.scheduling.crane.io             2024-06-15T10:32:00Z
```

### 步骤 4：工作原理 — 真实利用率调度

传统 kube-scheduler 基于 Pod 声明的 `resources.requests` 做调度决策。例如节点有 4 核 CPU，两个 Pod 各 request 1 核，kube-scheduler 认为还剩 2 核可用。但实际上这两个 Pod 可能已经用满了 4 核 — 新 Pod 调度上去会导致 CPU 争抢。

Crane 调度器的不同之处：

1. **数据采集**：craned 持续采集节点的真实 CPU/内存使用率（通过 Prometheus/Node Exporter）
2. **画像生成**：craned 为每个 Pod 生成历史资源使用画像
3. **调度决策**：CraneScheduler 查询实际负载指标，选择负载最低的节点放置 Pod
4. **策略配置**：用户通过 SchedulingPolicy CRD 定义打分插件权重、过滤规则等

### 步骤 5：SchedulingPolicy CRD 示例

```yaml
apiVersion: scheduling.crane.io/v1alpha1
kind: SchedulingPolicy
metadata:
  name: hot-value-avoid
  namespace: crane-system
spec:
  scoringPlugins:
  - name: RealtimeLoadScoring
    weight: 10
    parameters:
      metric: cpu_usage_avg_1m
  - name: BalancedAllocationScoring
    weight: 5
  filters:
  - name: NumaTopologyFilter
```

```bash
# 需 kubectl 可达环境 — 应用策略
kubectl apply -f scheduling-policy.yaml
# expected: schedulingpolicy.scheduling.crane.io/hot-value-avoid created
```

**预期输出**：

```text
schedulingpolicy.scheduling.crane.io/hot-value-avoid created
```

### 步骤 6：功能全景

| 功能 | 依赖子组件 | 说明 |
|------|----------|------|
| QoS 感知调度 | CraneScheduler + craned | 根据节点真实负载选择 Pod 放置节点 |
| CPU Burst | craned + QoSAgent | 允许容器短时超用 CPU |
| 内存精细调度 | craned | NUMA 拓扑感知的内存分配 |
| 可抢占式 Job | CraneScheduler | 基于 PriorityClass 的抢占调度 |
| 节点放大 | craned | 通过水分率放大节点可调度资源 |

## 验证

### tccli 验证

```bash
# 维度 1：组件状态
tccli tke DescribeAddon --region ap-guangzhou \
    --ClusterId cls-xxxxxxxx \
    --AddonName CraneScheduler
# expected: Status: "Succeeded"
```

**预期输出**：

```json
{
    "Addons": [
        {
            "AddonName": "CraneScheduler",
            "AddonVersion": "v1.2.0",
            "Status": "Succeeded",
            "Phase": "Running"
        }
    ],
    "RequestId": "b2c3d4e5-f6a7-8901-bcde-f12345678901"
}
```

```bash
# 维度 2：集群状态
tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]'
# expected: ClusterStatus: "Running"
```

**预期输出**：

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

### kubectl 验证

```bash
# 需 kubectl 可达环境 — 维度 3：Pod 状态
kubectl get pods -n crane-system
# expected: 全部 Running

# 需 kubectl 可达环境 — 维度 4：CRD 状态
kubectl get crd | grep scheduling.crane.io
# expected: podqos, schedulingpolicies, nodeqos 已注册

# 需 kubectl 可达环境 — 维度 5：日志
kubectl logs -n crane-system -l app=crane-scheduler --tail=10
# expected: 无 FATAL/ERROR
```

**预期输出**：

```text
I0101 00:00:00.000000       1 scheduler.go:100] Starting crane scheduler
I0101 00:00:00.000000       1 scheduler.go:101] Watching for PodQOS changes
```

| 维度 | 命令 | 预期 |
|------|------|------|
| 组件状态 | `DescribeAddon --AddonName CraneScheduler` | `Status: "Succeeded"` |
| 集群状态 | `DescribeClusters --ClusterIds '["cls-xxxxxxxx"]'` | `ClusterStatus: "Running"` |
| Pod 运行 | `kubectl get pods -n crane-system` | 全部 Running |
| CRD 就绪 | `kubectl get crd \| grep scheduling.crane.io` | 已注册 |
| 日志正常 | `kubectl logs -n crane-system -l app=crane-scheduler --tail=10` | 无异常 |

## 清理

> **警告**：卸载 CraneScheduler 后，依赖其调度策略的工作负载将回退到默认 kube-scheduler。已运行的 Pod 不受影响，但新 Pod 不再按 Crane 策略调度。卸载不可逆。

### 数据面清理（kubectl，需可达环境）

```bash
# 1. 查看并删除 Crane CRD 实例
kubectl get podqos,schedulingpolicies --all-namespaces
kubectl delete podqos --all --all-namespaces
kubectl delete schedulingpolicies --all --all-namespaces
```

```text
NAME  STATUS  AGE
...
```

### 控制面清理（tccli）

```bash
# 2. 卸载组件
tccli tke DeleteAddon --region ap-guangzhou \
    --ClusterId cls-xxxxxxxx \
    --AddonName CraneScheduler
# expected: exit 0，返回 RequestId
```

**预期输出**：

```json
{
    "RequestId": "a7b8c9d0-e1f2-3456-abcd-567890123456"
}
```

```bash
# 3. 验证已卸载
tccli tke DescribeAddon --region ap-guangzhou \
    --ClusterId cls-xxxxxxxx \
    --AddonName CraneScheduler
# expected: ResourceNotFound
```

**预期输出**：

```json
{
    "Error": {
        "Code": "ResourceNotFound",
        "Message": "Addon not found"
    },
    "RequestId": "b8c9d0e1-f2a3-4567-bcde-678901234567"
}
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `InvalidParameter.AddonName` | 检查大小写：`CraneScheduler`（C、S 大写） | 名称拼写错误 | 使用正确名称 `CraneScheduler` |
| `InvalidParameter.ClusterId` | `DescribeClusters` 确认集群 ID | 集群不存在或 ID 错误 | 使用 `cls-xxxxxxxx` 或从 `DescribeClusters` 获取 |
| `FailedOperation.AddonAlreadyExists` | `DescribeAddon` 查看状态 | 已安装 | 如状态异常先卸载再重装 |
| `Status: "Failed"` | `kubectl describe pod -n crane-system` | 资源不足或镜像问题 | 检查节点资源，查看 Pod Events |
| Pod CrashLoopBackOff | `kubectl logs -n crane-system` | 启动参数错误 | 卸载后重装 |
| CRD 未注册（超 5 分钟） | `kubectl get crd \| grep crane.io` | 安装未完成 | 卸载后重新安装 |
| kubectl 不可达 | 检查 VPN/IOA 连接 | CAM 策略限制 | 通过内网或数据面集群操作 |

## 下一步

- [QoSAgent](../../Qos%20感知调度/QoSAgent/tccli%20操作.md) — 节点 QoS Agent 安装
- [CPU Burst](../../Qos%20感知调度/CPU%20Burst/tccli%20操作.md) — 容器 CPU 突发
- [应用启动时 CPU 突增](../../Qos%20感知调度/应用启动时%20CPU%20突增/tccli%20操作.md) — 加速启动
- [自定义资源优先级](../../业务优先级保障调度/自定义资源优先级/tccli%20操作.md) — PriorityClass
- [DeScheduler](../DeScheduler/tccli%20操作.md) — 重调度器
- [原生调度器](../原生调度器/Default-scheduler%20调度策略配置/tccli%20操作.md) — kube-scheduler 策略

## 控制台替代

登录 [容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) → 选择集群 **cls-xxxxxxxx** → **组件管理** → **新建** → 搜索 `CraneScheduler` → 选择版本完成安装。
