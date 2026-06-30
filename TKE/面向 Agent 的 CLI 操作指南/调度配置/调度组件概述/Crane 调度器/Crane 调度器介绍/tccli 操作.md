# Crane 调度器介绍（tccli）

> 对照官方：[Crane 调度器介绍](https://cloud.tencent.com/document/product/457/75472) · page_id `75472`

## 概述

Crane 是腾讯云开源的云原生资源优化平台，TKE 通过集成 Crane 调度器（AddonName=`CraneScheduler`）提供智能调度能力。与传统基于 request 的调度不同，Crane 调度器基于节点真实资源利用率（real utilization）进行调度决策，避免节点因"请求低但实际使用高"而出现资源热点。

安装 CraneScheduler 后，它作为 kube-scheduler 的二次调度器（secondary scheduler）运行，通过 CRD（Custom Resource Definition）提供调度策略配置入口。其子组件包括：

| 子组件 | 功能 |
|-------|------|
| **CraneScheduler** | 基于真实利用率的调度器内核，替代 kube-scheduler 做 Pod 放置决策 |
| **craned** | Crane 控制面组件，管理 QoS 分类、Pod 资源画像采集与推荐 |
| **CraneDescheduler** | 重调度器，将 Pod 从资源过载节点驱逐到更合适的节点 |

## 前置条件

- [环境准备](../../../../环境准备.md)

演示集群信息：**cls-xxxxxxxx**（ap-guangzhou，v1.30.0，Running，3 节点）。注意：kubectl 因 CAM 策略限制不可达，需通过 VPN/IOA 内网或数据面集群操作。以下 kubectl 命令为参考格式。

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置

# 3. 检查 kubectl 版本（数据面操作需要，当前集群 kubectl 不可达）
kubectl version --client
# expected: Client Version >= 1.30.0

# 4. 检查 CAM 权限 — 需要以下 Action 名
#    tke:DescribeClusters, tke:InstallAddon, tke:DescribeAddon
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
# 5. 确认目标集群存在
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
# 6. 检查现有组件，确认未重复安装
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
# 7. 查询可用的 CraneScheduler 版本
tccli tke DescribeAddonValues --region ap-guangzhou \
    --ClusterId cls-xxxxxxxx \
    --AddonName CraneScheduler
# expected: 返回可用版本和配置项
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
| 升级组件版本 | `UpdateAddon --AddonName CraneScheduler` | 否 |
| 卸载组件 | `DeleteAddon --AddonName CraneScheduler` | 否 |

### 关键字段说明

`InstallAddon` 的主要参数：

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `ClusterId` | String | 是 | 目标集群 ID，如 `cls-xxxxxxxx` | 集群不存在 → `InvalidParameter.ClusterId` |
| `AddonName` | String | 是 | `CraneScheduler`，大小写敏感（C、S 大写） | 名称错误 → `InvalidParameter.AddonName` |
| `AddonVersion` | String | 否 | 组件版本号，如 `v1.2.0`。不指定则安装最新版 | 版本不存在 → `InvalidParameter.AddonVersion` |

## 操作步骤

### 步骤 1：安装 Crane 调度器

#### 选择依据

- **AddonName**：TKE 组件市场中 Crane 调度器名称为 `CraneScheduler`（注意大小写）。与其他 Crane 生态组件区分：`CraneScheduler` 是调度器本身，`craned` 是控制面组件，`CraneDescheduler` 是重调度器。
- **AddonVersion**：如不指定，tccli 自动安装最新稳定版。建议先通过 `DescribeAddonValues` 查询可用版本后指定，避免自动升级带来的不兼容。
- **安装时机**：建议在集群创建后、部署业务负载前安装，避免调度策略变更影响已有 Pod。

#### 最小安装（仅必填字段）

```bash
tccli tke InstallAddon --region ap-guangzhou \
    --ClusterId cls-xxxxxxxx \
    --AddonName CraneScheduler
# expected: exit 0，返回 RequestId，组件开始安装
```

**预期输出**：

```json
{
    "RequestId": "c3d4e5f6-a7b8-9012-cdef-123456789012"
}
```

#### 增强配置（指定版本）

```bash
tccli tke InstallAddon --region ap-guangzhou \
    --ClusterId cls-xxxxxxxx \
    --AddonName CraneScheduler \
    --AddonVersion v1.2.0
# expected: exit 0，返回 RequestId
```

**预期输出**：

```json
{
    "RequestId": "d4e5f6a7-b8c9-0123-defa-234567890123"
}
```

### 步骤 2：轮询组件安装状态

组件安装是异步操作。轮询直到状态为 `Succeeded`：

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

### 步骤 3：获取 kubeconfig 并验证数据面

```bash
# 获取 kubeconfig（kubectl 操作需 VPN/IOA 内网或数据面集群可达）
tccli tke DescribeClusterKubeconfig --region ap-guangzhou --ClusterId cls-xxxxxxxx \
    --output json | jq -r '.Kubeconfig' > ~/.kube/config-cls-xxxxxxxx
# expected: 文件创建成功，非空

export KUBECONFIG=~/.kube/config-cls-xxxxxxxx
```

```json
{
  "Kubeconfig": "<Kubeconfig>",
  "RequestId": "<RequestId>"
}
```

```bash
# 需 kubectl 可达环境 — 验证 Crane 调度器 Pod 运行状态
kubectl get pods -n crane-system
# expected: crane-scheduler-xxx 和 crane-scheduler-controller-xxx Pod 均为 Running
```

**预期输出**：

```text
NAME                                            READY   STATUS    RESTARTS   AGE
crane-scheduler-xxxxx                            1/1     Running   0          2m
crane-scheduler-xxxxx                            1/1     Running   0          2m
crane-scheduler-controller-xxxxxxxxxx-xxxxx      1/1     Running   0          2m
```

```bash
# 需 kubectl 可达环境 — 验证 CRD 已注册
kubectl get crd | grep scheduling.crane.io
# expected: 返回 Crane 调度相关 CRD
```

**预期输出**：

```text
podqos.scheduling.crane.io              2024-01-01T00:00:00Z
schedulingpolicies.scheduling.crane.io  2024-01-01T00:00:00Z
nodeqos.scheduling.crane.io             2024-01-01T00:00:00Z
```

### 步骤 4：Crane 调度器工作原理解析

Crane 调度器与原生 kube-scheduler 的核心区别在于调度决策依据：

| 对比维度 | 原生 kube-scheduler | Crane 调度器 |
|---------|-------------------|-------------|
| 调度依据 | Pod 的 `resources.requests` | 节点的真实资源利用率（Prometheus/Node Exporter 指标） |
| 资源感知 | 静态（只看 request 值） | 动态（实时 CPU/内存使用率） |
| 热点避免 | 依赖 request 准确设定 | 自动避让高负载节点，即使其 request 剩余较多 |
| 策略配置 | Scheduler Configuration（需重启） | SchedulingPolicy CRD（动态生效） |

### 步骤 5：配置 SchedulingPolicy（CRD 示例）

安装 CraneScheduler 后，可通过 SchedulingPolicy CRD 自定义调度策略。以下示例定义按节点真实负载排序的调度策略：

```yaml
apiVersion: scheduling.crane.io/v1alpha1
kind: SchedulingPolicy
metadata:
  name: real-load-policy
  namespace: crane-system
spec:
  scoringPlugins:
  - name: RealtimeLoadScoring
    weight: 10
  - name: BalancedAllocationScoring
    weight: 5
  filters:
  - name: NumaTopologyFilter
```

```bash
# 需 kubectl 可达环境 — 应用 SchedulingPolicy
kubectl apply -f scheduling-policy.yaml
# expected: schedulingpolicy.scheduling.crane.io/real-load-policy created
```

**预期输出**：

```text
schedulingpolicy.scheduling.crane.io/real-load-policy created
```

### 步骤 6：Crane 调度器功能清单

安装 CraneScheduler 后，可通过 CRD 配置以下调度功能：

| 功能 | 相关 CRD | 说明 |
|------|---------|------|
| QoS 感知调度 | PodQOS, NodeQOS | 根据节点资源使用情况动态调度 |
| CPU Burst | PodQOS | 允许容器短时超用 CPU |
| 内存精细调度 | PodQOS | 内存 NUMA 感知和限制 |
| 可抢占式 Job | PriorityClass | 高优先级工作负载可抢占低优先级 |
| 节点放大 | — | 通过水分放大节点可调度资源 |

## 验证

### 控制面（tccli）

```bash
# 维度 1：确认组件状态
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
    "RequestId": "f6a7b8c9-d0e1-2345-fabc-456789012345"
}
```

```bash
# 维度 2：确认集群状态正常
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

### 数据面

```bash
# 需 kubectl 可达环境 — 维度 3：验证 Pod 运行
kubectl get pods -n crane-system
# expected: 所有 crane-scheduler 相关 Pod 状态均为 Running
```

**预期输出**：

```text
NAME                                            READY   STATUS    RESTARTS   AGE
crane-scheduler-xxxxx                            1/1     Running   0          2m
crane-scheduler-controller-xxxxxxxxxx-xxxxx      1/1     Running   0          2m
```

```bash
# 需 kubectl 可达环境 — 维度 4：验证 CRD 可用
kubectl get crd | grep scheduling.crane.io
# expected: podqos、schedulingpolicies、nodeqos 等 CRD 已注册
```

**预期输出**：

```text
podqos.scheduling.crane.io              2024-01-01T00:00:00Z
schedulingpolicies.scheduling.crane.io  2024-01-01T00:00:00Z
nodeqos.scheduling.crane.io             2024-01-01T00:00:00Z
```

```bash
# 需 kubectl 可达环境 — 维度 5：验证 scheduler 日志
kubectl logs -n crane-system -l app=crane-scheduler --tail=10
# expected: 无 FATAL/ERROR 日志
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
| CRD 就绪 | `kubectl get crd \| grep scheduling.crane.io` | podqos、schedulingpolicies 等已注册 |
| 日志正常 | `kubectl logs -n crane-system -l app=crane-scheduler --tail=10` | 无异常 |

## 清理

> **警告**：卸载 CraneScheduler 将移除 Crane 调度器组件，依赖其调度策略（PodQOS、SchedulingPolicy 等 CRD）的工作负载将回退到默认 kube-scheduler 调度。正在运行的 Pod 不受影响，但新 Pod 将不再按 Crane 策略调度。卸载操作不可逆，需重新安装恢复。

### 数据面

数据面资源清理在控制面之前。

```bash
# 需 kubectl 可达环境 — 清理前确认 Crane 相关 CRD 实例
kubectl get podqos --all-namespaces
kubectl get schedulingpolicies --all-namespaces
# 确认是否需保留
```

**预期输出**：

```text
NAMESPACE   NAME               AGE
default     high-priority-qos   10m
```

```bash
# 需 kubectl 可达环境 — 删除 Crane 调度相关 CRD 实例（如有）
kubectl delete podqos --all --all-namespaces
kubectl delete schedulingpolicies --all --all-namespaces
# expected: 逐一删除或提示 No resources found
```

### 控制面（tccli）

```bash
# 清理前状态检查
tccli tke DescribeAddon --region ap-guangzhou \
    --ClusterId cls-xxxxxxxx \
    --AddonName CraneScheduler
# expected: 确认组件当前状态，记录 AddonVersion
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
    "RequestId": "f6a7b8c9-d0e1-2345-fabc-456789012345"
}
```

```bash
# 卸载组件
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
# 验证已卸载
tccli tke DescribeAddon --region ap-guangzhou \
    --ClusterId cls-xxxxxxxx \
    --AddonName CraneScheduler
# expected: ResourceNotFound 或 AddonName 不在列表中
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

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `InstallAddon` 返回 `InvalidParameter.AddonName` | 检查 AddonName 大小写：`CraneScheduler` 而非 `cranescheduler` | AddonName 大小写不匹配 | 使用精确名称 `CraneScheduler`（C、S 大写） |
| `InstallAddon` 返回 `InvalidParameter.ClusterId` | `tccli tke DescribeClusters --region ap-guangzhou` 检查集群 ID | 集群 ID 格式错误或集群不存在于当前地域 | 用 `DescribeClusters` 获取正确的集群 ID |
| `InstallAddon` 返回 `FailedOperation.AddonAlreadyExists` | `tccli tke DescribeAddon --region ap-guangzhou --ClusterId cls-xxxxxxxx --AddonName CraneScheduler` | 组件已安装（幂等操作） | 如状态非 Running，先 `DeleteAddon` 卸载后重新安装；或直接使用当前安装 |
| `InstallAddon` 返回 `LimitExceeded` | `tccli tke DescribeClusters --region ap-guangzhou` 检查集群状态 | 集群资源不足或配额达上限 | 扩容集群或等待当前安装任务完成 |

### 安装成功但组件未正常运行

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DescribeAddon` 显示 `Status: "Failed"` | 查看组件安装日志：`kubectl describe pod -n crane-system -l app=crane-scheduler` | 节点资源不足、镜像拉取失败或配置错误 | 检查节点资源：`kubectl top nodes`；检查镜像：`kubectl describe pod -n crane-system` 查看 Events；保留 RequestId → 重试安装或提交工单 |
| Pod 一直 CrashLoopBackOff | `kubectl logs -n crane-system -l app=crane-scheduler --tail=50` 查看错误日志 | 启动参数或配置错误 | 根据日志修复配置；如无法定位 → 卸载后重新安装 |
| CRD 未注册 | `kubectl get crd \| grep scheduling.crane.io` | 安装过程未完成或权限不足 | 等待安装完成；如超 5 分钟未注册 → `DeleteAddon` 后重新安装 |
| kubectl 不可达 | 检查是否通过 VPN/IOA 连接内网 | CAM 策略限制或网络不通 | 通过 VPN/IOA 内网连接集群，或在数据面集群上执行 kubectl 命令 |

## 下一步

- [QoS 感知调度](../../../Qos%20感知调度/QoSAgent/tccli%20操作.md) — 安装节点 QoSAgent 组件
- [CPU Burst](../../../Qos%20感知调度/CPU%20Burst/tccli%20操作.md) — 启用 CPU 突发能力
- [自定义资源优先级](../../../业务优先级保障调度/自定义资源优先级/tccli%20操作.md) — PriorityClass 优先级调度
- [应用启动时 CPU 突增](../../../Qos%20感知调度/应用启动时%20CPU%20突增/tccli%20操作.md) — 加速应用启动
- [DeScheduler](../../DeScheduler/tccli%20操作.md) — 重调度器配置

## 控制台替代

登录 [容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) → 选择集群 **cls-xxxxxxxx** → **组件管理** → **新建** → 搜索 `CraneScheduler` → 选择版本并安装。
