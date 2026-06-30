# qGPU 离在线混部说明

> 对照官方：[qGPU 离在线混部说明](https://cloud.tencent.com/document/product/457/81107) · page_id `81107`

## 概述

qGPU 离在线混部支持在线（高优）任务和离线（低优）任务同时混合部署在同一张 GPU 卡上。在内核与驱动层面实现了"低优 100% 使用闲置算力及高优 100% 抢占"，可将 GPU 利用率提升到 100%。

| 维度 | 在线 Pod（Online） | 离线 Pod（Offline） | 普通 Pod |
|------|-------------------|---------------------|---------|
| Annotation | `app-class: online` | `app-class: offline` | 无 `app-class` |
| 算力资源 | 无需声明（仅显存） | `qgpu-core-greedy`（贪婪，1-100） | `qgpu-core`（1-100 或 >100） |
| 显存资源 | `qgpu-memory`（静态切分，不会被抢占） | `qgpu-memory` | `qgpu-memory` |
| 多卡支持 | 不支持 | 不支持（greedy <= 100） | 支持（core > 100） |
| 算力优先级 | 高优（不受离线 Pod 影响） | 低优（抢占在线空闲算力） | 无优先级区分 |
| 适用场景 | 实时推理、在线服务 | 批量处理、模型训练 | 开发测试、独占 GPU |

**选型建议**：
- 对延迟敏感的在线推理服务使用 `app-class: online`（高优），仅声明显存，算力由内核保障
- 对实时性要求低的离线任务使用 `app-class: offline`（低优），声明 `qgpu-core-greedy` 贪婪抢占空闲算力
- 需要独占计算资源或多卡互联的业务使用普通 Pod（无 `app-class`），声明 `qgpu-core`

## 前置条件

- [环境准备](../../../../../环境准备.md)
- 已安装 qGPU 组件（参见 [使用 qGPU](../../使用%20qGPU/tccli%20操作.md)）
- TKE 原生 GPU 节点
- kubectl 可达集群以 apply YAML

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置，region 为集群所在地域

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:DescribeClusters, tke:DescribeAddon
# 验证：执行 DescribeClusters 确认权限
tccli tke DescribeClusters --region <Region>
# expected: exit 0，返回集群列表（可为空）

# 4. 检查 kubectl 版本
kubectl version --client
# expected: Client Version: v1.28 或更高

# 5. 获取并验证 kubeconfig 连通性
tccli tke DescribeClusterKubeconfig --region <Region> \
    --ClusterId CLUSTER_ID | jq -r '.Kubeconfig' > ~/.kube/config
kubectl get ns
# expected: 返回 default 等系统命名空间
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
# 6. 确认集群 QGPU 已启用
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]'
# expected: TotalCount >= 1，ClusterStatus 为 Running，QGPUShareEnable 为 true

# 7. 确认 qgpu 组件已安装且状态正常
tccli tke DescribeAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName qgpu
# expected: Phase 为 running 或 Succeeded

# 8. 确认 GPU 节点存在
kubectl get nodes -l accelerator=nvidia
# expected: 至少 1 个 GPU 节点
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

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `REGION` | 集群所在地域 | 如 `ap-guangzhou` | `tccli configure list` 确认当前 region |
| `CLUSTER_ID` | 集群 ID | 格式 `cls-xxxxxxxx` | `tccli tke DescribeClusters --region <Region>` |

## 控制台与 CLI 参数映射

| 控制台操作 | tccli / kubectl 命令 | 幂等 |
|-----------|----------------------|:--:|
| 安装 qGPU 组件 | `tccli tke InstallAddon --AddonName qgpu` | 否 |
| 查看 qGPU 组件状态 | `tccli tke DescribeAddon --AddonName qgpu` | 是 |
| 创建在线 Pod | `kubectl apply -f`（annotation `app-class: online`） | 是 |
| 创建离线 Pod | `kubectl apply -f`（annotation `app-class: offline`） | 是 |
| 创建普通 Pod | `kubectl apply -f`（无 `app-class` annotation） | 是 |

## 功能原理

### 两个 "100%" 控制

| 能力 | 说明 |
|------|------|
| 100% 使用高优闲置算力 | 高优任务闲时，低优任务可 100% 使用 GPU 算力 |
| 100% 抢占低优占用算力 | 高优任务忙时，可 100% 抢占低优任务所占用的 GPU 算力 |

### 技术实现

1. **感知与响应**：高优 Pod 提交 GPU 计算任务时，qGPU 驱动在 1ms 内将算力全部提供给高优 Pod。高优任务结束时，100ms 后释放算力重新分配给低优 Pod。

2. **暂停与继续**：低优 Pod 在高优 Pod 有计算任务时立即暂停；高优任务结束后，低优 Pod 在中断点继续计算。

### 调度策略

| Pod 类型 | 调度行为 |
|---------|---------|
| 高优 Pod | 有计算任务后立即抢占 GPU，高优 Pod 之间为争抢模式，不受 policy 控制 |
| 低优 Pod | 高优休眠时按 policy 调度；高优唤醒时全部暂停；高优结束后按 policy 继续 |

### 典型场景

| 场景 | 高优任务 | 低优任务 |
|------|---------|---------|
| 在线推理 + 离线推理混部 | 线上推理（低延迟要求） | 数据预处理（实时性要求低） |
| 在线推理 + 离线训练混部 | 实时推理（可用性要求高） | 模型训练（资源使用多，延迟不敏感） |

## 关键字段说明

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `tke.cloud.tencent.com/app-class` | Annotation | 否 | `online`（在线高优）、`offline`（离线低优）。不设则为普通 Pod | 填错误值 → Pod 调度后行为与预期不符，无混部优先级 |
| `tke.cloud.tencent.com/qgpu-core-greedy` | 资源 | 是（离线） | 1-100，贪婪算力百分比。仅离线 Pod 使用，不支持多卡 | 在线 Pod 误用 greedy → 高优保障失效；值 > 100 → 调度失败 |
| `tke.cloud.tencent.com/qgpu-core` | 资源 | 是（普通） | 1-100 单卡，>100 多卡。普通 Pod 使用 | 离线 Pod 用 core 而非 greedy → 无法抢占空闲算力 |
| `tke.cloud.tencent.com/qgpu-memory` | 资源 | 是 | GiB 显存，整数，最小 1。在线 Pod 静态切分不被抢占 | 超过节点总显存 → Pod 无法调度 |

## 操作步骤

### 步骤 1：了解三种 Pod 类型

**qGPU 离在线混部通过 Pod annotation `tke.cloud.tencent.com/app-class` 区分三种类型**，无需额外 tccli 操作（只需 qGPU 集群和 qgpu 组件就绪）。

| Pod 类型 | app-class Annotation | 算力资源声明 | 显存声明 | 行为 |
|---------|---------------------|-------------|---------|------|
| 在线（高优） | `online` | 无需声明 | `qgpu-memory` | 独享显存保障，不受离线影响 |
| 离线（低优） | `offline` | `qgpu-core-greedy` | `qgpu-memory` | 贪婪抢占在线空闲算力 |
| 普通 | 无 | `qgpu-core` | `qgpu-memory` | 标准 vGPU 行为，不参与优先级 |

### 步骤 2：确认节点低优算力资源

qgpu 组件安装后需重建 manager/scheduler Pod 以注册 `qgpu-core-greedy` 扩展资源：

```bash
# 重建 qgpu-manager Pod
kubectl delete pod -n kube-system QGPU_MANAGER_POD
# expected: pod "QGPU_MANAGER_POD" deleted

# 重建 qgpu-scheduler Pod
kubectl delete pod -n kube-system -l app=qgpu-scheduler
# expected: 返回已删除的 Pod 名称

# 确认 Pod 已重建
kubectl get pods -n kube-system | grep qgpu
# expected: qgpu-manager 和 qgpu-scheduler Pod 状态均为 Running
```

```text
NAME  STATUS  AGE
...
```

验证低优算力资源已注册：

```bash
kubectl describe node GPU_NODE | grep qgpu-core-greedy
# expected: 存在低优算力资源字段（值可能为 0，属正常现象）
```

```text
NAME  STATUS  AGE
...
```

### 步骤 3：部署三种 Pod（示例）

详细的 YAML 文件和部署步骤参见 [使用 qGPU 离在线混部](../使用%20qGPU%20离在线混部/tccli%20操作.md)。

核心区分点：

- **在线 Pod**：annotation `app-class: online` + 仅声明 `qgpu-memory`
- **离线 Pod**：annotation `app-class: offline` + 声明 `qgpu-core-greedy` + `qgpu-memory`
- **普通 Pod**：无 `app-class` + 声明 `qgpu-core` + `qgpu-memory`

## 验证

### 控制面（tccli）

```bash
# 确认 qgpu 组件状态正常
tccli tke DescribeAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName qgpu
# expected: Phase 为 running 或 Succeeded
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

### 数据面（kubectl）

```bash
# 确认节点已注册低优算力资源
kubectl describe node GPU_NODE | grep qgpu-core-greedy
# expected: 返回 tke.cloud.tencent.com/qgpu-core-greedy 资源行

# 确认三种 Pod 均已 Running
kubectl get pods -o wide
# expected: 在线、离线、普通 Pod 状态均为 Running，调度到同一 GPU 节点

# 确认离线 Pod 使用低优算力
kubectl describe pod OFFLINE_POD | grep qgpu-core-greedy
# expected: 显示 qgpu-core-greedy 资源声明

# 确认在线 Pod 仅申请显存
kubectl describe pod ONLINE_POD | grep qgpu
# expected: 仅显示 qgpu-memory，无 qgpu-core 或 qgpu-core-greedy
```

```text
NAME  STATUS  AGE
...
```

| 验证维度 | 命令 | 预期 |
|---------|------|------|
| 组件状态 | `DescribeAddon --AddonName qgpu` | Phase 为 running 或 Succeeded |
| 低优算力 | `kubectl describe node GPU_NODE \| grep qgpu-core-greedy` | 节点含 `qgpu-core-greedy` 扩展资源 |
| 在线 Pod Annotation | `kubectl describe pod ONLINE_POD \| grep app-class` | `app-class: online` |
| 离线 Pod Annotation | `kubectl describe pod OFFLINE_POD \| grep app-class` | `app-class: offline` |
| GPU 共享 | `kubectl exec ONLINE_POD -- nvidia-smi` | 多 Pod 共享 GPU，显存按声明分配 |

## 清理

本页为概念说明页，无资源需清理。如通过 [使用 qGPU 离在线混部](../使用%20qGPU%20离在线混部/tccli%20操作.md) 创建了测试 Pod，参见该页的清理章节。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DescribeAddon --AddonName qgpu` 返回空列表或 `ResourceNotFound` | `tccli tke DescribeAddon --region <Region> --ClusterId CLUSTER_ID` 查看全部组件 | qgpu 组件未安装，或 `AddonName` 拼写错误 | 用 `qgpu`（全小写）作为 AddonName。如未安装，参见 [使用 qGPU](../../使用%20qGPU/tccli%20操作.md) 安装 |
| `kubectl apply` 离线 Pod 返回 `Insufficient qgpu-core-greedy` | `kubectl describe node GPU_NODE \| grep qgpu-core-greedy` 检查节点资源 | qgpu-manager/scheduler Pod 未重建，节点未注册 greedy 资源 | 执行步骤 2 重建 qgpu Pod，重建后确认 `qgpu-core-greedy` 出现 |

### 行为异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 在线 Pod 被离线业务影响 | `kubectl describe pod ONLINE_POD \| grep -A3 Annotations` 检查 `app-class` | 在线 Pod 错误使用了 `app-class: offline` 或未设 `app-class` | 确认 annotation 为 `tke.cloud.tencent.com/app-class: "online"`，仅声明 `qgpu-memory` |
| 离线 Pod 无法抢占更多算力 | `kubectl exec ONLINE_POD -- nvidia-smi` 检查在线 Pod GPU 使用率 | 在线 Pod 持续高负载，离线 Pod 无法抢占空闲算力（正常行为） | 降低在线 Pod 计算负载，或调整离线 Pod 的 `qgpu-core-greedy` 目标值 |
| 离线 Pod 调度失败 | `kubectl describe pod OFFLINE_POD \| grep -E "Events\|Warning"` 查看调度事件 | `qgpu-core-greedy` 值 > 100 或节点显存不足 | `qgpu-core-greedy` 控制在 1-100 范围内；降低 `qgpu-memory` 值 |
| 节点无 `qgpu-core-greedy` 资源 | `kubectl logs -n kube-system QGPU_MANAGER_POD` 查看新 Pod 日志 | 组件版本不支持混部或 manager 启动失败 | 检查 qgpu 组件版本：`tccli tke DescribeAddon --AddonName qgpu`；保留 RequestId 提交工单 |

## 下一步

- [使用 qGPU 离在线混部](../使用%20qGPU%20离在线混部/tccli%20操作.md) — 完整部署和验证三种 Pod
- [使用 qGPU](../../使用%20qGPU/tccli%20操作.md) — qGPU 基础安装和单卡 Pod 部署
- [qGPU 多卡互联](../../qGPU%20多卡互联/tccli%20操作.md) — Pod 跨多张物理 GPU 使用 vGPU 资源
- [GPU 监控指标获取](../../../GPU%20监控指标获取/tccli%20操作.md) — 获取 GPU 使用率、显存等监控指标
- [GPU 故障检测与自愈](../../../GPU%20故障检测与自愈/tccli%20操作.md) — GPU 故障自动检测与隔离

## 控制台替代

[TKE 控制台 → 工作负载](https://console.cloud.tencent.com/tke2/cluster)：在 qgpu 组件与 GPU 节点池就绪后，新建 Deployment，在 Pod template 的 annotation 中设置 `tke.cloud.tencent.com/app-class` 区分在线/离线/普通 Pod。
