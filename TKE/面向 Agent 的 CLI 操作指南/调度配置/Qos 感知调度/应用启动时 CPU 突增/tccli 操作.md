# 应用启动时 CPU 突增（tccli）

> 对照官方：[应用启动时 CPU 突增](https://cloud.tencent.com/document/product/457/100618) · page_id `100618`

## 概述

应用启动时 CPU 突增（Startup CPU Burst）允许容器在启动阶段短时间使用超过 limits 的 CPU 资源，加速应用初始化和就绪过程。与常规 CPU Burst 不同，该功能专注于容器启动窗口，帮助需要大量 CPU 完成初始化的应用（如 JVM 预热、模型加载）快速就绪。

**实现机制**：集群需预先安装 QoSAgent 和 craned 组件。通过 Pod 注解指定启动突增参数（突增倍数和时长），QoSAgent 在 Pod 启动阶段自动放行额外 CPU，窗口过期后自动回落到 limits 限制。

## 前置条件

- [环境准备](../../../环境准备.md)

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
# expected: Client Version >= v1.26.0

# 4. 检查 CAM 权限 — 需要以下 Action 名
#    tke:DescribeClusters, tke:DescribeAddon
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
# 5. 确认目标集群存在且状态正常
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
# 6. 确认 QoSAgent 组件已安装且正常运行
tccli tke DescribeAddon --region ap-guangzhou \
    --ClusterId cls-xxxxxxxx \
    --AddonName QoSAgent
# expected: Phase: "Succeeded"
```

**预期输出**：

```json
{
    "Addons": [
        {
            "AddonName": "QoSAgent",
            "AddonVersion": "v1.3.0",
            "Status": "Succeeded",
            "Phase": "Running"
        }
    ],
    "RequestId": "f6a7b8c9-d0e1-2345-fabc-456789012345"
}
```

### 版本与规格选择

- **QoSAgent 版本**：需支持 Startup CPU Burst 功能。通过 `tccli tke DescribeAddonValues --region ap-guangzhou --ClusterId cls-xxxxxxxx --AddonName QoSAgent` 确认。
- **突增倍数**：建议 2-4 倍。倍数过小启动加速不明显，过大浪费节点空闲 CPU。
- **突增时长**：建议 30s-120s。根据应用初始化耗时设置，可通过观察初始启动时间确定。

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 / kubectl 命令 | 幂等 |
|-----------|------------------------|:--:|
| 查看集群详情 | `tccli tke DescribeClusters --ClusterIds '["cls-xxxxxxxx"]' --region ap-guangzhou` | 是 |
| 查看 QoSAgent 状态 | `tccli tke DescribeAddon --AddonName QoSAgent --ClusterId cls-xxxxxxxx --region ap-guangzhou` | 是 |
| 启用启动 CPU 突增 | `kubectl annotate pod POD_NAME qos.crane.io/startup-cpu-burst=true` | 否 |
| 指定突增倍数 | `kubectl annotate pod POD_NAME qos.crane.io/startup-cpu-burst-multiplier=N` | 否 |
| 指定突增时长 | `kubectl annotate pod POD_NAME qos.crane.io/startup-cpu-burst-duration=N` | 否 |
| 移除突增配置 | `kubectl annotate pod POD_NAME qos.crane.io/startup-cpu-burst-` | 否 |

## 操作步骤

### 步骤 1：确认集群环境

```bash
tccli tke DescribeClusters --region ap-guangzhou \
    --ClusterIds '["cls-xxxxxxxx"]'
# expected: exit 0，ClusterStatus: "Running"
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

### 步骤 2：为 Pod 配置启动 CPU 突增

#### 选择依据

- **qos.crane.io/startup-cpu-burst**：启用或禁用启动 CPU 突增。值 `"true"` 表示启用。
- **qos.crane.io/startup-cpu-burst-multiplier**：启动阶段 CPU 倍数。如 `"2"` 表示容器启动时最多使用 2 倍 CPU limits。适合需要大量 CPU 编译或预热的场景。
- **qos.crane.io/startup-cpu-burst-duration**：启动突增窗口时长，单位秒。如 `"60"` 表示启动后 60 秒内允许突增。窗口过后自动回到 CPU limits 限制。
- **不启用场景**：轻量应用启动耗时 < 5s，不需要启动突增。

#### 最小配置（仅启用启动突增，使用默认参数）

```bash
# 需 kubectl 可达环境
kubectl annotate pod POD_NAME qos.crane.io/startup-cpu-burst=true -n NAMESPACE
# expected: pod/POD_NAME annotated
```

**预期输出**：

```text
pod/POD_NAME annotated
```

#### 增强配置（指定突增倍数和时长）

`startup-burst-pod.yaml`：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: burst-startup-app
  namespace: default
  annotations:
    qos.crane.io/startup-cpu-burst: "true"
    qos.crane.io/startup-cpu-burst-multiplier: "3"
    qos.crane.io/startup-cpu-burst-duration: "90"
spec:
  containers:
  - name: app
    image: openjdk:17-slim
    command: ["java", "-jar", "/app.jar"]
    resources:
      requests:
        cpu: "1000m"
        memory: "1Gi"
      limits:
        cpu: "2000m"
        memory: "2Gi"
```

```bash
# 需 kubectl 可达环境
kubectl apply -f startup-burst-pod.yaml
# expected: pod/burst-startup-app created
```

**预期输出**：

```text
pod/burst-startup-app created
```

| 占位符 | 说明 | 获取方式 |
|--------|------|---------|
| `POD_NAME` | 目标 Pod 名称 | `kubectl get pods -n NAMESPACE` |
| `NAMESPACE` | Pod 所在命名空间 | `kubectl get ns` |

## 验证

### 控制面（tccli）

```bash
# 维度 1：集群状态
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

```bash
# 维度 2：QoSAgent 状态
tccli tke DescribeAddon --region ap-guangzhou \
    --ClusterId cls-xxxxxxxx \
    --AddonName QoSAgent
# expected: Phase: "Succeeded"
```

**预期输出**：

```json
{
    "Addons": [
        {
            "AddonName": "QoSAgent",
            "AddonVersion": "v1.3.0",
            "Status": "Succeeded",
            "Phase": "Running"
        }
    ],
    "RequestId": "f6a7b8c9-d0e1-2345-fabc-456789012345"
}
```

### 数据面（kubectl）

```bash
# 需 kubectl 可达环境
# 维度 3：确认 Pod 启动突增注解已设置
kubectl describe pod POD_NAME -n NAMESPACE | grep startup-cpu-burst
# expected: qos.crane.io/startup-cpu-burst=true
# expected: qos.crane.io/startup-cpu-burst-multiplier=3（若指定）
# expected: qos.crane.io/startup-cpu-burst-duration=90（若指定）

# 维度 4：确认 Pod 已正常运行
kubectl get pod POD_NAME -n NAMESPACE
# expected: STATUS: Running，READY: 1/1

# 维度 5：观察启动阶段 CPU 使用（Pod 刚启动时执行）
kubectl top pod POD_NAME -n NAMESPACE
# expected: CPU 使用量可能超过 limits（启动突增窗口内）

# 维度 6：确认突增窗口过期后 CPU 恢复限制（等待 duration 秒后）
kubectl top pod POD_NAME -n NAMESPACE
# expected: CPU 使用量回到 limits 范围内
```

```text
NAME  STATUS  AGE
...
```

| 维度 | 命令 | 预期 |
|------|------|------|
| 集群状态 | `DescribeClusters --ClusterIds '["cls-xxxxxxxx"]'` | `ClusterStatus: "Running"` |
| QoSAgent 状态 | `DescribeAddon --AddonName QoSAgent` | `Phase: "Succeeded"` |
| 注解存在 | `kubectl describe pod \| grep startup-cpu-burst` | 注解已设置 |
| Pod 运行 | `kubectl get pod POD_NAME` | STATUS: Running |
| 启动突增 | `kubectl top pod POD_NAME`（启动阶段） | CPU 可能超过 limits |
| 窗口过期 | `kubectl top pod POD_NAME`（duration 秒后） | CPU 回到 limits 内 |

## 清理

> **警告**：移除启动突增注解后，Pod 重启时将失去加速启动能力。对于依赖快速初始化的应用（如 JVM 预热），可能导致启动时间显著增加。

### 数据面（kubectl）

#### 步骤 1：清理前状态检查

```bash
# 需 kubectl 可达环境
kubectl describe pod POD_NAME -n NAMESPACE | grep startup-cpu-burst
# 确认当前注解状态，记录突增参数值
```

```text
Name:         ...
Status:       Running
...
```

#### 步骤 2：移除启动突增注解

```bash
# 需 kubectl 可达环境
kubectl annotate pod POD_NAME qos.crane.io/startup-cpu-burst- -n NAMESPACE
kubectl annotate pod POD_NAME qos.crane.io/startup-cpu-burst-multiplier- -n NAMESPACE
kubectl annotate pod POD_NAME qos.crane.io/startup-cpu-burst-duration- -n NAMESPACE
# expected: pod/POD_NAME annotated（每条返回一行确认）
```

**预期输出**：

```text
pod/POD_NAME annotated
pod/POD_NAME annotated
pod/POD_NAME annotated
```

#### 步骤 3：验证已移除

```bash
# 需 kubectl 可达环境
kubectl describe pod POD_NAME -n NAMESPACE | grep startup-cpu-burst
# expected: 无输出（注解已全部移除）
```

```text
NAME  STATUS  AGE
...
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl annotate pod` 返回 NotFound | `kubectl get pods -n NAMESPACE` 确认 Pod 是否存在 | Pod 名称或命名空间错误 | 确认 Pod 名称和命名空间 |
| Pod 创建后注解未显示 | `kubectl describe pod POD_NAME -n NAMESPACE` 查看完整 Annotations | 注解名拼写错误或 YAML 中缩进不正确 | 确认注解 Key 为 `qos.crane.io/startup-cpu-burst`，value 为 `"true"` |
| `kubectl top pod` 返回 "metrics not available" | `kubectl get pods -n kube-system \| grep metrics-server` 检查 metrics-server | Metrics Server 未安装或异常 | 安装 metrics-server 组件 |
| kubectl 不可达 | 检查 VPN/IOA 连接 | CAM 策略限制或网络不通 | 通过 VPN/IOA 内网连接集群，或在数据面集群上执行 kubectl 命令 |

### 注解已设置但启动未加速

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Pod 已注解但 CPU 始终不超过 limits | `kubectl get pods -n crane-system \| grep qos-agent` 确认 qos-agent 状态 | QoSAgent 未在所有节点运行或版本不支持 Startup Burst | 确认 qos-agent DaemonSet 全部 Ready；检查 QoSAgent 版本是否支持该功能 |
| 启动突增后 CPU 未回落 | `kubectl describe pod POD_NAME -n NAMESPACE \| grep startup-cpu-burst-duration` 确认时长配置 | 突增窗口时长仍生效或配置过长 | 等待窗口过期后自动回落。如果 `startup-cpu-burst-duration` 设置过长（如 >300s），移除注解后重新设置合理值 |
| Pod 频繁重启，每次重启都触发突增 | `kubectl describe pod POD_NAME -n NAMESPACE \| grep "Restart Count"` 查看重启次数 | 应用自身崩溃导致频繁重启，节点 CPU 被反复突增占用 | 先修复应用 Crash 根因，再进行突增配置。频繁重启可能影响同节点其他 Pod |

## 下一步

- [CPU Burst](../../CPU%20Burst/tccli%20操作.md) — 运行时 CPU 突增，应对流量突发场景
- [QoSAgent](../../QoSAgent/tccli%20操作.md) — QoSAgent 组件安装与管理
- [Crane 调度器介绍](../../调度组件概述/Crane%20调度器/Crane%20调度器介绍/tccli%20操作.md) — 了解完整的 Crane 调度能力
- [内存精细调度](https://cloud.tencent.com/document/product/457/79777) — 基于 PageCache 的精细化内存管理

## 控制台替代

登录 [容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) → 选择集群 **cls-xxxxxxxx** → **工作负载** → 在 Pod 的 YAML 编辑中手动添加 `qos.crane.io/startup-cpu-burst` 注解。
