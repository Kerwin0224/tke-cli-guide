# 自动规整集群资源（tccli）

> 对照官方：[自动规整集群资源](https://cloud.tencent.com/document/product/457/122378) · page_id `122378`

## 概述

TKE 通过 DeScheduler 组件（AddonName=`DeScheduler`）实现集群资源自动规整。DeScheduler 定期扫描集群，识别低利用率和资源碎片化节点，自动驱逐 Pod 并触发重新调度，使 Pod 分布趋于均衡，释放碎片化资源。与手动节点排水（drain）相比，DeScheduler 策略可定制、自动化执行，无需人工介入。

## 前置条件

- [环境准备](../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置

# 3. 检查 kubectl 版本
kubectl version --client
# expected: Client Version >= 1.30.0

# 4. 检查 CAM 权限 — 需要以下 Action 名
#    tke:DescribeClusters, tke:InstallAddon, tke:DescribeAddon
#    tke:DeleteAddon, tke:DescribeClusterKubeconfig
# 验证：执行 DescribeClusters 确认权限
tccli tke DescribeClusters --region ap-guangzhou
# expected: exit 0，返回集群列表
```

```json
{
  "TotalCount": "<TotalCount>",
  "Clusters": "<Clusters>",
  "ClusterId": "<ClusterId>",
  "ClusterName": "<ClusterName>",
  "ClusterDescription": "<ClusterDescription>",
  "ClusterVersion": "<ClusterVersion>"
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
            "ClusterStatus": "Running"
        }
    ]
}
```

```bash
# 6. 确认集群至少有 3 个节点（规整需要有足够节点可调度）
tccli tke DescribeClusterKubeconfig --region ap-guangzhou --ClusterId cls-xxxxxxxx \
    --output json | jq -r '.Kubeconfig' > ~/.kube/config-cls-xxxxxxxx
export KUBECONFIG=~/.kube/config-cls-xxxxxxxx
kubectl get nodes
# expected: READY 节点数 >= 3
```

**预期输出**：

```text
NAME          STATUS   ROLES    AGE   VERSION
node-ex-001   Ready    <none>   1d    v1.30.0
node-ex-002   Ready    <none>   1d    v1.30.0
node-ex-003   Ready    <none>   1d    v1.30.0
```

```bash
# 7. 检查现有组件，确认未重复安装
tccli tke DescribeAddon --region ap-guangzhou \
    --ClusterId cls-xxxxxxxx \
    --AddonName DeScheduler
# expected: 如已安装返回组件详情；未安装返回 ResourceNotFound 或空
```

**预期输出**（未安装时）：

```json
{
    "Error": {
        "Code": "ResourceNotFound",
        "Message": "Addon not found"
    }
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 安装 DeScheduler | `InstallAddon --AddonName DeScheduler` | 是 |
| 查看组件状态 | `DescribeAddon --AddonName DeScheduler` | 是 |
| 升级组件版本 | `UpdateAddon --AddonName DeScheduler` | 否 |
| 卸载组件 | `DeleteAddon --AddonName DeScheduler` | 否 |

## Key parameters

`InstallAddon` 的主要参数：

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `ClusterId` | String | 是 | 目标集群 ID，格式 `cls-xxxxxxxx` | 集群不存在 → `InvalidParameter.ClusterId` |
| `AddonName` | String | 是 | `DeScheduler`，大小写敏感 | 名称错误 → `InvalidParameter.AddonName` |
| `AddonVersion` | String | 否 | 组件版本号。不指定则安装最新版 | 版本不存在 → `InvalidParameter.AddonVersion` |

## 操作步骤

### 步骤 1：安装 DeScheduler

#### 选择依据

- **DeScheduler** 适用于以下场景：集群中存在碎片化资源、部分节点负载过高而其他节点空闲、需要自动均衡 Pod 分布但不想手动 drain。
- 不适用场景：有状态应用（StatefulSet）且 Pod 驱逐会影响数据一致性、对 Pod 中断敏感的关键任务、集群节点数少于 3 个（驱逐后无处可调度）。
- **安装时机**：建议在集群已有工作负载、出现资源不均衡后安装；空集群安装无实际效果。

#### 最小安装（仅必填字段）

```bash
tccli tke InstallAddon --region ap-guangzhou \
    --ClusterId cls-xxxxxxxx \
    --AddonName DeScheduler
# expected: exit 0，返回 RequestId，组件开始安装
```

**预期输出**：

```json
{
    "RequestId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
}
```

#### 增强配置（指定版本）

```bash
tccli tke InstallAddon --region ap-guangzhou \
    --ClusterId cls-xxxxxxxx \
    --AddonName DeScheduler \
    --AddonVersion ADDON_VERSION
# expected: exit 0，返回 RequestId
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `ap-guangzhou` | 地域 | 如 `ap-guangzhou` | `tccli configure list` |
| `cls-xxxxxxxx` | 目标集群 ID | 格式 `cls-xxxxxxxx` | `tccli tke DescribeClusters --region ap-guangzhou` |
| `ADDON_VERSION` | 组件版本 | 如 `v1.0.0` | `tccli tke DescribeAddonValues --region ap-guangzhou --ClusterId cls-xxxxxxxx --AddonName DeScheduler` |

### 步骤 2：轮询组件安装状态

```bash
tccli tke DescribeAddon --region ap-guangzhou \
    --ClusterId cls-xxxxxxxx \
    --AddonName DeScheduler
# expected: Status: "Succeeded"
```

```json
{
  "Addons": "<Addons>",
  "AddonName": "<AddonName>",
  "AddonVersion": "<AddonVersion>",
  "RawValues": "<RawValues>",
  "Phase": "<Phase>",
  "Reason": "<Reason>"
}
```

### 步骤 3：创建 DeSchedulerPolicy

#### 选择依据

- **DeSchedulerPolicy** 定义规整策略，核心参数：
  - `interval`：扫描间隔（默认 60s）。
  - `strategies`：启用哪些规整策略（`RemoveDuplicates`、`LowNodeUtilization`、`HighNodeUtilization` 等）。
- **RemoveDuplicates**：移除同一节点上运行多个副本的 Pod（适合提高可用性）。
- **LowNodeUtilization**：将低利用率节点的 Pod 迁移到其他节点，释放空节点（适合缩容场景）。
- **HighNodeUtilization**：将高负载节点的 Pod 疏散到其他节点（适合均衡场景）。

#### 最小配置（使用默认策略）

DeScheduler 安装后默认开启 `LowNodeUtilization` 策略，无需额外配置即可生效。等待一个扫描周期（约 60s）后，DeScheduler 自动开始识别低利用率节点并驱逐 Pod。

```bash
# 需 kubectl 可达环境 — 确认 DeScheduler Pod 运行
kubectl get pods -n kube-system -l app=descheduler
# expected: descheduler-xxx Pod 状态为 Running
```

**预期输出**：

```text
NAME                          READY   STATUS    RESTARTS   AGE
descheduler-xxxxxxxxxx-xxxxx   1/1     Running   0          2m
```

#### 增强配置（自定义策略）

```bash
# 需 kubectl 可达环境 — 创建自定义 DeSchedulerPolicy
cat > descheduler-policy.yaml <<'EOF'
apiVersion: descheduler.crane.io/v1alpha1
kind: DeSchedulerPolicy
metadata:
  name: custom-defrag-policy
spec:
  interval: 120
  strategies:
    LowNodeUtilization:
      enabled: true
      params:
        thresholds:
          cpu: 20
          memory: 20
          pods: 20
        targetThresholds:
          cpu: 50
          memory: 50
          pods: 50
    RemoveDuplicates:
      enabled: true
    HighNodeUtilization:
      enabled: true
      params:
        thresholds:
          cpu: 80
          memory: 80
          pods: 80
EOF

kubectl apply -f descheduler-policy.yaml
# expected: deschedulerpolicy.descheduler.crane.io/custom-defrag-policy created
```

**预期输出**：

```text
deschedulerpolicy.descheduler.crane.io/custom-defrag-policy created
```

| 参数 | 说明 | 推荐值 |
|------|------|--------|
| `interval` | 扫描间隔（秒） | `120`（生产环境） |
| `thresholds.cpu` | 低利用率 CPU 阈值（%）| `20` |
| `thresholds.memory` | 低利用率内存阈值（%）| `20` |
| `targetThresholds.cpu` | 驱逐后目标 CPU 利用率（%）| `50` |
| `targetThresholds.memory` | 驱逐后目标内存利用率（%）| `50` |

## 验证

### 控制面（tccli）

```bash
# 维度 1：确认组件状态
tccli tke DescribeAddon --region ap-guangzhou \
    --ClusterId cls-xxxxxxxx \
    --AddonName DeScheduler
# expected: Status: "Succeeded"
```

**预期输出**：

```json
{
    "AddonName": "DeScheduler",
    "Status": "Succeeded"
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
    "ClusterId": "cls-xxxxxxxx",
    "ClusterStatus": "Running"
}
```

### 数据面

```bash
# 需 kubectl 可达环境 — 维度 3：确认 DeScheduler Pod 运行
kubectl get pods -n kube-system -l app=descheduler
# expected: STATUS 为 Running，READY 1/1
```

**预期输出**：

```text
NAME                          READY   STATUS    RESTARTS   AGE
descheduler-xxxxxxxxxx-xxxxx   1/1     Running   0          2m
```

```bash
# 需 kubectl 可达环境 — 维度 4：确认策略已创建
kubectl get deschedulerpolicy
# expected: custom-defrag-policy 在列表中
```

**预期输出**：

```text
NAME                   AGE
custom-defrag-policy   5m
```

```bash
# 需 kubectl 可达环境 — 维度 5：查看 DeScheduler 日志，确认正在扫描
kubectl logs -n kube-system -l app=descheduler --tail=20
# expected: 含 "scanning" 或 "LowNodeUtilization" 日志，无 FATAL/ERROR
```

**预期输出**：

```text
I0101 00:00:00.000000       1 descheduler.go:100] Starting descheduler
I0101 00:00:00.000000       1 descheduler.go:105] scanning cluster for LowNodeUtilization
```

```bash
# 需 kubectl 可达环境 — 维度 6：观察节点 Pod 分布变化
kubectl get pods -o wide --all-namespaces --field-selector spec.nodeName=NODE_NAME
# 对比规整前后节点 Pod 数量变化
```

**预期输出**：

```text
NAMESPACE     NAME                              READY   STATUS    NODE
default       nginx-deploy-xxxxxxxxxx-aaaa      1/1     Running   node-ex-001
kube-system   coredns-xxxxxxxxxx-bbbb           1/1     Running   node-ex-001
```

| 维度 | 命令 | 预期 |
|------|------|------|
| 组件状态 | `DescribeAddon --AddonName DeScheduler` | `Status: "Succeeded"` |
| Pod 运行 | `kubectl get pods -n kube-system -l app=descheduler` | Running |
| 策略就绪 | `kubectl get deschedulerpolicy` | 策略已创建 |
| 日志 | `kubectl logs -n kube-system -l app=descheduler --tail=20` | 正常扫描 |
| 资源分布 | 观察多节点 Pod 数变化 | 逐步均衡 |

## 清理

> **警告**：卸载 DeScheduler 后，集群不再自动规整资源。已驱逐的 Pod 不会自动回迁。卸载前确认当前无正在执行的驱逐操作（检查 DeScheduler 日志）。如有依赖 DeScheduler 维持节点均衡的业务，卸载后需手动监控。

### 数据面

数据面资源清理在控制面之前。

```bash
# 需 kubectl 可达环境 — 清理前确认 DeSchedulerPolicy
kubectl get deschedulerpolicy
# 确认目标策略
```

**预期输出**：

```text
NAME                   AGE
custom-defrag-policy   10m
```

```bash
# 需 kubectl 可达环境 — 删除 DeSchedulerPolicy
kubectl delete deschedulerpolicy custom-defrag-policy
# expected: deschedulerpolicy.descheduler.crane.io "custom-defrag-policy" deleted
```

```bash
# 需 kubectl 可达环境 — 验证策略已删除
kubectl get deschedulerpolicy custom-defrag-policy
# expected: Error from server (NotFound)
```

**预期输出**：

```text
Error from server (NotFound): deschedulerpolicies.descheduler.crane.io "custom-defrag-policy" not found
```

### 控制面（tccli）

```bash
# 清理前状态检查
tccli tke DescribeAddon --region ap-guangzhou \
    --ClusterId cls-xxxxxxxx \
    --AddonName DeScheduler
# expected: 确认组件当前状态
```

```json
{
  "Addons": "<Addons>",
  "AddonName": "<AddonName>",
  "AddonVersion": "<AddonVersion>",
  "RawValues": "<RawValues>",
  "Phase": "<Phase>",
  "Reason": "<Reason>"
}
```

```bash
# 卸载组件
tccli tke DeleteAddon --region ap-guangzhou \
    --ClusterId cls-xxxxxxxx \
    --AddonName DeScheduler
# expected: exit 0，返回 RequestId
```

```bash
# 验证已卸载
tccli tke DescribeAddon --region ap-guangzhou \
    --ClusterId cls-xxxxxxxx \
    --AddonName DeScheduler
# expected: ResourceNotFound 或 AddonName 不在列表中
```

**预期输出**：

```json
{
    "Error": {
        "Code": "ResourceNotFound",
        "Message": "Addon not found"
    }
}
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `InstallAddon` 返回 `InvalidParameter.AddonName` | 检查 AddonName 大小写：`DeScheduler` 而非 `descheduler` | AddonName 大小写不匹配 | 使用精确名称 `DeScheduler`（D、S 大写） |
| `InstallAddon` 返回 `FailedOperation.AddonAlreadyExists` | `tccli tke DescribeAddon --region ap-guangzhou --ClusterId cls-xxxxxxxx --AddonName DeScheduler` | 组件已安装 | 如状态异常，先 `DeleteAddon` 卸载后重新安装 |
| `InstallAddon` 返回 `LimitExceeded` | `tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]'` 检查集群状态 | 集群资源不足或配额达上限（此为环境限制，非命令错误） | 扩容集群或等待其他安装任务完成 |
| `kubectl apply -f` DeSchedulerPolicy 返回资源类型未找到 | `kubectl get crd \| grep descheduler` | DeScheduler CRD 未注册，组件安装未完成 | 等待安装完成（`DescribeAddon` 状态为 `Succeeded`）后重试 |

### 安装成功但规整未生效

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| DeScheduler Pod 运行但无驱逐事件 | `kubectl logs -n kube-system -l app=descheduler --tail=50` 查看日志 | 策略未启用或节点不满足阈值条件 | 确认 DeSchedulerPolicy 中目标策略 `enabled: true`；检查节点利用率是否在阈值范围内 |
| 驱逐事件过多，Pod 频繁迁移 | `kubectl get events --all-namespaces --field-selector reason=Descheduled \| tail -20` 查看驱逐事件 | 阈值设置过低，大部分节点持续触发策略 | 调高 `thresholds` 值或调低 `targetThresholds` 值；增加 `interval` 扫描间隔 |
| 某些 Pod 被驱逐后无法重新调度 | `kubectl describe pod EVICTED_POD_NAME` 查看调度失败原因 | 集群资源已满，或 Pod 的调度约束过于严格 | 扩容集群节点；放宽 Pod 的 nodeSelector/affinity 约束 |
| DeScheduler 状态为 `Failed` | `tccli tke DescribeAddon --region ap-guangzhou --ClusterId cls-xxxxxxxx --AddonName DeScheduler` | 安装过程异常 | 记录 RequestId → `DeleteAddon` 卸载 → 重新安装 |

## 下一步

- [节点放大](../节点放大/tccli%20操作.md) — 通过水分放大节点可调度资源
- [Crane 调度器介绍](../../调度组件概述/Crane%20调度器/Crane%20调度器介绍/tccli%20操作.md) — 智能调度基础组件
- [调度策略配置](../../调度组件概述/Crane%20调度器/调度策略配置/tccli%20操作.md) — 配置绑定调度策略
- [集群扩缩容](https://cloud.tencent.com/document/product/457/32194) — 手动扩缩节点

## 控制台替代

登录 [容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) → 选择集群 → **组件管理** → **新建** → 搜索 `DeScheduler` → 选择版本并安装 → 在 **DeScheduler** 详情页配置规整策略。
