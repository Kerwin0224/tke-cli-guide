# QoSAgent（tccli）

> 对照官方：[QoSAgent](https://cloud.tencent.com/document/product/457/79774) · page_id `79774`

## 概述

QoSAgent 是 TKE Crane 调度器套件中的节点级 QoS 代理组件，以 DaemonSet 形式部署在集群每个节点上。它负责采集节点资源使用数据，并执行中心调度器 craned 下发的 QoS 策略（CPU Burst、CPU 优先级、内存回收、IO 隔离、超线程隔离等）。QoSAgent 是 QoS 感知调度体系中连接控制面（craned）与数据面（节点 cgroup）的关键组件。

**tccli 能力范围**：QoSAgent 的安装、升级、卸载和状态查询均通过 tccli 完成（InstallAddon、DescribeAddon、UpdateAddon、DeleteAddon）。安装后 QoSAgent 自动以 DaemonSet 运行，数据面状态需通过 kubectl 查看。

## 前置条件

- [环境准备](../../../环境准备.md)
- 集群中需已安装 `craned`（Crane 调度器）组件。QoSAgent 依赖 craned 下发 QoS 策略
- CAM 权限：`tke:InstallAddon`、`tke:DescribeAddon`、`tke:DeleteAddon`、`tke:DescribeClusters`
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
# 确认 craned 已安装（QoSAgent 的前置依赖）
tccli tke DescribeAddon --region ap-guangzhou \
    --ClusterId cls-xxxxxxxx \
    --AddonName craned
```

```json
{
    "Addons": [
        {
            "AddonName": "craned",
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
| 查看集群列表 | `tccli tke DescribeClusters --region ap-guangzhou` | 是 |
| 查看可用版本 | `tccli tke DescribeAddonValues --region ap-guangzhou --ClusterId cls-xxxxxxxx --AddonName QoSAgent` | 是 |
| 安装 QoSAgent | `tccli tke InstallAddon --region ap-guangzhou --ClusterId cls-xxxxxxxx --AddonName QoSAgent` | 否 |
| 查询安装状态 | `tccli tke DescribeAddon --region ap-guangzhou --ClusterId cls-xxxxxxxx --AddonName QoSAgent` | 是 |
| 升级 QoSAgent | `tccli tke UpdateAddon --region ap-guangzhou --ClusterId cls-xxxxxxxx --AddonName QoSAgent --AddonVersion <version>` | 否 |
| 卸载 QoSAgent | `tccli tke DeleteAddon --region ap-guangzhou --ClusterId cls-xxxxxxxx --AddonName QoSAgent` | 否 |

## 操作步骤

### 步骤 1：查询可用版本

```bash
tccli tke DescribeAddonValues --region ap-guangzhou \
    --ClusterId cls-xxxxxxxx \
    --AddonName QoSAgent
```

```json
{
    "Values": {
        "Versions": ["v1.0.0", "v1.1.0", "v1.2.0"]
    },
    "DefaultVersion": "v1.2.0",
    "RequestId": "c3d4e5f6-a7b8-9012-cdef-123456789012"
}
```

### 步骤 2：安装 QoSAgent 组件

**最小安装**（使用默认版本）：

```bash
tccli tke InstallAddon --region ap-guangzhou \
    --ClusterId cls-xxxxxxxx \
    --AddonName QoSAgent
```

```json
{
    "RequestId": "d4e5f6a7-b8c9-0123-def0-234567890123"
}
```

**增强配置**（指定版本）：

```bash
tccli tke InstallAddon --region ap-guangzhou \
    --ClusterId cls-xxxxxxxx \
    --AddonName QoSAgent \
    --AddonVersion v1.2.0
```

```json
{
    "RequestId": "e5f6a7b8-c9d0-1234-ef01-345678901234"
}
```

| 参数 | 说明 | 获取方式 |
|------|------|---------|
| `--AddonName` | 固定值 `QoSAgent`（注意大小写） | 不适用 |
| `--AddonVersion` | 组件版本号 | `DescribeAddonValues` 查询 |
| `--ClusterId` | 目标集群 ID | `DescribeClusters` 查询 |

### 步骤 3：轮询安装状态

组件安装为异步操作，轮询直到 `Phase` 为 `Succeeded`：

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
    "RequestId": "f6a7b8c9-d0e1-2345-f012-456789012345"
}
```

### 步骤 4：验证 DaemonSet 运行（需 kubectl 可达环境）

安装成功后，QoSAgent 以 DaemonSet 形式在每个节点运行：

```bash
# 需 kubectl 可达环境
kubectl get pods -n crane-system -l app=qos-agent
# expected:
# NAME              READY   STATUS    RESTARTS   AGE
# qos-agent-xxxxx   1/1     Running   0          30s
# qos-agent-yyyyy   1/1     Running   0          30s
# qos-agent-zzzzz   1/1     Running   0          30s
```

```text
NAME  STATUS  AGE
...
```

### 工作原理

1. **安装阶段**：tccli `InstallAddon` 在 TKE 控制面注册 QoSAgent 组件，集群内自动创建 `crane-system` 命名空间和 qos-agent DaemonSet。
2. **策略同步**：qos-agent 定期从 craned 拉取 PodQOS/NodeQOS CRD 变更，同步到本地策略缓存。
3. **指标采集**：qos-agent 通过 cAdvisor、eBPF 采集节点和 Pod 的 CPU、内存、IO、网络等实时指标。
4. **策略执行**：根据 Pod 的优先级标签/注解，qos-agent 通过 cgroup 子系统执行资源隔离（cpuset 绑定、memory limit 回收、blkio 限速等）。

## 验证

### 控制面

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
    "RequestId": "a7b8c9d0-e1f2-3456-0123-567890123456"
}
```

| 维度 | 预期值 |
|------|--------|
| `AddonName` | `QoSAgent` |
| `Phase` | `"Succeeded"` |
| `Status` | `"Running"` |

### 数据面（需 kubectl 可达环境）

```bash
# 确认 PodQOS CRD 已注册
kubectl get crd podqos.scheduling.crane.io
# expected: podqos.scheduling.crane.io   <age>

# 确认 qos-agent DaemonSet 全部就绪
kubectl get ds -n crane-system qos-agent
# expected: DESIRED = CURRENT = READY = UP-TO-DATE = AVAILABLE = 节点数

# 检查 qos-agent 日志无异常
kubectl logs -n crane-system -l app=qos-agent --tail=10
# expected: 无 ERROR 级别日志，正常包含 QoS 策略加载信息
```

```text
NAME  STATUS  AGE
...
```

## 清理

> **警告**：卸载 QoSAgent 将导致节点上所有 QoS 策略（CPU Burst、内存回收、IO 隔离、超线程隔离等）立即失效。请确认集群中不存在依赖这些策略的关键业务 Pod。

### 步骤 1：清理前状态检查

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
    "RequestId": "b8c9d0e1-f2a3-4567-1234-678901234567"
}
```

### 步骤 2：卸载 QoSAgent

```bash
tccli tke DeleteAddon --region ap-guangzhou \
    --ClusterId cls-xxxxxxxx \
    --AddonName QoSAgent
```

```json
{
    "RequestId": "c9d0e1f2-a3b4-5678-2345-789012345678"
}
```

### 步骤 3：验证已卸载

```bash
tccli tke DescribeAddon --region ap-guangzhou \
    --ClusterId cls-xxxxxxxx \
    --AddonName QoSAgent
```

```json
{
    "Error": {
        "Code": "ResourceNotFound",
        "Message": "addon QoSAgent not found in cluster cls-xxxxxxxx"
    },
    "RequestId": "d0e1f2a3-b4c5-6789-3456-890123456789"
}
```

## 排障

### 安装失败

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `InstallAddon` 返回 `InvalidParameter.AddonName` | 检查 `AddonName` 值 | 填写了非 `QoSAgent` 的值（如小写 `qosagent`） | 使用 `--AddonName QoSAgent`（注意大小写） |
| `InstallAddon` 返回 "addon version not found" | `DescribeAddonValues` 查询可用版本 | 指定的 `AddonVersion` 不存在 | 从 `DescribeAddonValues` 输出取正确版本号 |
| `InstallAddon` 返回 `ResourceNotFound.ClusterNotFound` | `tccli configure list` 确认地域 | 集群 ID 错误或不属于当前账号/地域 | 确认集群 ID 和地域正确 |
| `InstallAddon` 返回 `FailedOperation.AddonAlreadyInstalled` | `DescribeAddon` 确认当前状态 | QoSAgent 已安装 | 如需重新安装，先 `DeleteAddon` 再 `InstallAddon` |

### DaemonSet 异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| DescribeAddon `Phase: "Succeeded"` 但部分节点无 qos-agent Pod | `kubectl describe ds -n crane-system qos-agent` 查看调度状态 | 节点存在污点或资源不足 | 检查节点污点 `kubectl describe node`；扩容节点 |
| qos-agent Pod CrashLoopBackOff | `kubectl logs -n crane-system <pod>` | 节点内核版本不兼容 | 确认节点 OS 和内核版本满足组件要求 |
| qos-agent Pod Pending | `kubectl describe pod -n crane-system <pod>` | 镜像拉取失败或资源不足 | 检查镜像仓库连通性和节点可用资源 |

## 下一步

- [CPU 使用优先级](../CPU%20使用优先级/tccli%20操作.md) — 基于 QoSAgent 的 PodQOS CPU 优先级配置
- [CPU Burst](../CPU%20Burst/tccli%20操作.md) — 允许容器短时间窗口内突破 CPU limits
- [内存精细调度](../内存精细调度/tccli%20操作.md) — 通过 PodQOS 实现内存水位和 OOM 控制
- [QoSAgent 组件介绍](../组件介绍/QoSAgent/tccli%20操作.md) — 了解 QoSAgent 架构与工作原理

## 控制台替代

[TKE 控制台 → 集群 → 组件管理](https://console.cloud.tencent.com/tke2/cluster)，在组件列表中搜索 QoSAgent，点击「安装」并选择版本。
