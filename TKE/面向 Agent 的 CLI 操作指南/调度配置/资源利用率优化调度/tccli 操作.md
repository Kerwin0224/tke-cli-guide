# 资源利用率优化调度（tccli）

> 对照官方：[资源利用率优化调度](https://cloud.tencent.com/document/product/457/122378) · page_id `122378`

## 概述

TKE 资源利用率优化调度通过多维度的资源管理策略，在保障业务 QoS 的前提下提升集群整体资源利用效率。核心策略包括：节点放大（node amplification，通过水分系数让调度器感知更多可分配资源）、自动规整（auto defragmentation，通过 DeScheduler 将分散 Pod 重新调度到更少节点，释放低利用率节点）。配合 CPU Burst、内存精细调度等能力，实现安全可控的资源超售。

## 前置条件

- 演示集群 `cls-xxxxxxxx`（ap-guangzhou, v1.30.0, Running），kubectl 因 CAM 策略限制不可达，需通过 VPN/IOA 内网或数据面集群操作
- 已安装 [Crane 调度器](../调度组件概述/tccli%20操作.md)
- 如需使用自动规整，需安装 [DeScheduler](../调度组件概述/tccli%20操作.md#安装-crane-调度器含-qosagent)
- 节点放大功能依赖 craned 组件参数配置

### 环境检查

```bash
# 1. 确认集群状态
tccli tke DescribeClusters --ClusterIds '["cls-xxxxxxxx"]' --region ap-guangzhou --output json
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
# 2. 确认 Crane 调度器已安装
tccli tke DescribeAddon --ClusterId cls-xxxxxxxx --AddonName craned --region ap-guangzhou --output json
```

```json
{
    "AddonName": "craned",
    "AddonVersion": "v1.2.0",
    "Status": "Succeeded",
    "RequestId": "b2c3d4e5-f6a7-8901-bcde-f12345678901"
}
```

```bash
# 3. 确认 DeScheduler 安装状态（如需自动规整）
tccli tke DescribeAddon --ClusterId cls-xxxxxxxx --AddonName DeScheduler --region ap-guangzhou --output json
```

```json
{
    "Error": {
        "Code": "ResourceNotFound",
        "Message": "Addon not found"
    },
    "RequestId": "c3d4e5f6-a7b8-9012-cdef-123456789012"
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 查看集群状态 | `tccli tke DescribeClusters --ClusterIds '["cls-xxxxxxxx"]' --region ap-guangzhou` | 是 |
| 查看已安装组件 | `tccli tke DescribeAddon --ClusterId cls-xxxxxxxx --AddonName craned --region ap-guangzhou` | 是 |
| 启用节点放大 | `tccli tke UpdateAddon --ClusterId cls-xxxxxxxx --AddonName craned --region ap-guangzhou --RawValues '{"nodeResourceAmplificationRatio":1.5}'` | 否 |
| 设置放大系数 | craned 组件 `nodeResourceAmplificationRatio` 参数 | 是 |
| 查看节点放大后资源 | `kubectl describe node <node-name> \| grep -A5 "Allocatable"` | 是 |
| 安装 DeScheduler | `tccli tke InstallAddon --ClusterId cls-xxxxxxxx --AddonName DeScheduler --region ap-guangzhou` | 是 |
| 查看 DeScheduler 状态 | `tccli tke DescribeAddon --ClusterId cls-xxxxxxxx --AddonName DeScheduler --region ap-guangzhou` | 是 |
| 创建规整策略 | `kubectl apply -f descheduler-policy.yaml` | 是 |
| 查看规整策略 | `kubectl get deschedulerpolicy` | 是 |
| 查看驱逐事件 | `kubectl get events --all-namespaces --field-selector reason=Descheduled` | 是 |
| 卸载 DeScheduler | `tccli tke DeleteAddon --ClusterId cls-xxxxxxxx --AddonName DeScheduler --region ap-guangzhou` | 否 |
| 删除规整策略 | `kubectl delete deschedulerpolicy <name>` | 否 |

## 操作步骤

### 资源利用率优化维度

| 维度 | 策略 | 组件 | 效果 |
|------|------|------|------|
| **节点放大** | 通过水分系数虚增节点 Allocatable 资源 | craned 组件参数 | 调度器感知更多可用资源，提高装箱率 |
| **自动规整** | 周期性扫描，将分散 Pod 重新打包到更少节点 | DeScheduler | 释放低利用率节点，减少资源碎片 |
| **CPU Burst** | 容器空闲时积累 credit，突发时短时突破 Limit | PodQOS (QoSAgent) | 提升应用响应速度，提高 CPU 利用率 |
| **内存超售** | 内存水位控制 + PageCache 回收 + OOM 优先级 | NodeQOS + PodQOS (QoSAgent) | 安全超售内存，避免 OOM 影响关键服务 |

### 节点放大

节点放大（Node Amplification）通过调整 craned 组件的 `nodeResourceAmplificationRatio` 参数，虚增 kubelet 上报的 Allocatable 资源量。调度器感知到更多可用资源后，会将更多 Pod 调度到该节点，从而提高装箱率。

**工作原理**：

```
原始节点 Allocatable:  CPU=4, Memory=8192Mi
放大系数 1.5 后:       CPU=6, Memory=12288Mi (虚拟)
实际物理资源不变:      CPU=4, Memory=8192Mi (真实)
调度器决策基于:        CPU=6, Memory=12288Mi (虚拟)
```

**配置节点放大**：

```bash
tccli tke UpdateAddon --region ap-guangzhou \
    --ClusterId cls-xxxxxxxx \
    --AddonName craned \
    --AddonVersion v1.2.0 \
    --RawValues '{"nodeResourceAmplificationRatio":1.5}'
```

```json
{
    "RequestId": "e5f6a7b8-c9d0-1234-ef01-234567890123"
}
```

> **注意**：节点放大不改变物理资源，仅影响调度决策。过度放大（系数 > 2.0）会导致节点严重过载。建议配合 QoS 策略使用，通过 CPU 优先级和内存 OOM 优先级保障高优业务。推荐系数范围 1.2-1.5。

详细操作见 [节点放大](./节点放大/tccli%20操作.md) — page_id `79032`。

### 自动规整

自动规整（Auto Defragmentation）通过 DeScheduler 组件，定期扫描集群中的低利用率和资源碎片化节点，自动驱逐 Pod 并触发重新调度，减少资源碎片。

**规整策略对比**：

| 策略 | 行为 | 适用场景 | 注意事项 |
|------|------|---------|---------|
| `LowNodeUtilization` | 将低利用率节点的 Pod 迁移到其他节点 | 缩容场景，释放空闲节点 | 目标节点必须有足够资源容纳迁移 Pod |
| `HighNodeUtilization` | 疏散高负载节点的部分 Pod | 负载均衡，避免单点过载 | 可能引起临时服务中断 |
| `RemoveDuplicates` | 移除同一节点上多个同服务副本 | 提高可用性，分散风险 | 适用于副本数 > 1 的 Deployment |
| `RemovePodsViolatingInterPodAntiAffinity` | 移除违反 Pod 反亲和性的 Pod | 确保亲和性约束生效 | -- |

**安装 DeScheduler**：

```bash
tccli tke InstallAddon --region ap-guangzhou \
    --ClusterId cls-xxxxxxxx \
    --AddonName DeScheduler
```

```json
{
    "RequestId": "f6a7b8c9-d0e1-2345-f012-345678901234"
}
```

轮询安装状态：

```bash
tccli tke DescribeAddon --ClusterId cls-xxxxxxxx --AddonName DeScheduler --region ap-guangzhou --output json
```

```json
{
    "AddonName": "DeScheduler",
    "AddonVersion": "v1.0.0",
    "Status": "Succeeded",
    "RequestId": "a7b8c9d0-e1f2-3456-0123-456789012345"
}
```

**创建规整策略**：

```yaml
apiVersion: descheduler.crane.io/v1alpha1
kind: DeSchedulerPolicy
metadata:
  name: defrag-policy
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
```

```bash
kubectl apply -f descheduler-policy.yaml
```

```text
deschedulerpolicy.descheduler.crane.io/defrag-policy created
```

| 参数 | 说明 | 推荐值 |
|------|------|--------|
| `interval` | 扫描间隔（秒） | `120`（生产环境，避免频繁驱逐） |
| `thresholds` | 低利用率阈值，低于此值触发驱逐 | cpu: `20`, memory: `20`, pods: `20` |
| `targetThresholds` | 目标利用率，驱逐后节点应达到的利用率 | cpu: `50`, memory: `50`, pods: `50` |

### 策略组合建议

| 场景 | 推荐策略组合 | 说明 |
|------|------------|------|
| 提高装箱率 | 节点放大（系数 1.2-1.5） + CPU 优先级 | 安全超售，核心业务保障 CPU |
| 减少资源碎片 | 自动规整 + 节点放大 | 碎片整理同时提高单节点利用率 |
| 在线离线混部 | CPU 优先级 + 内存 OOM 优先级 + 可抢占 Job | 离线任务占用空闲资源，高优在线任务抢占 |
| 成本优化 | 节点放大 + 自动规整（释放低利用率节点） | 最大化单节点利用率，释放后可缩容降成本 |

## 验证

### 控制面

```bash
# 维度 1：确认集群状态
tccli tke DescribeClusters --ClusterIds '["cls-xxxxxxxx"]' --region ap-guangzhou --output json
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
# 维度 2：确认 Crane 组件状态
tccli tke DescribeAddon --ClusterId cls-xxxxxxxx --AddonName craned --region ap-guangzhou --output json
```

```json
{
    "AddonName": "craned",
    "AddonVersion": "v1.2.0",
    "Status": "Succeeded",
    "AddonConfig": {
        "NodeResourceAmplificationRatio": 1.0
    },
    "RequestId": "d4e5f6a7-b8c9-0123-def0-123456789012"
}
```

```bash
# 维度 3：确认 DeScheduler 状态（如已安装）
tccli tke DescribeAddon --ClusterId cls-xxxxxxxx --AddonName DeScheduler --region ap-guangzhou --output json
```

```json
{
    "AddonName": "DeScheduler",
    "AddonVersion": "v1.0.0",
    "Status": "Succeeded",
    "RequestId": "a7b8c9d0-e1f2-3456-0123-456789012345"
}
```

### 数据面（需 VPN/IOA 内网环境）

```bash
# 维度 4：查看节点资源变化（节点放大前后对比）
kubectl describe node <node-name> | grep -A5 "Allocatable"
```

```text
Name:         ...
Status:       Running
...
```

| 维度 | 命令 | 预期 |
|------|------|------|
| 集群状态 | `DescribeClusters --ClusterIds '["cls-xxxxxxxx"]'` | `ClusterStatus: "Running"` |
| Crane 组件 | `DescribeAddon --ClusterId cls-xxxxxxxx --AddonName craned` | `Status: "Succeeded"` |
| DeScheduler | `DescribeAddon --ClusterId cls-xxxxxxxx --AddonName DeScheduler` | `Status: "Succeeded"`（如已安装） |
| 节点 Allocatable | `kubectl describe node <name>` | 放大后 Allocatable > 物理资源 |

## 清理

无需清理（概述说明页）。如已进行测试操作：

```bash
# 恢复节点放大默认系数
tccli tke UpdateAddon --region ap-guangzhou \
    --ClusterId cls-xxxxxxxx \
    --AddonName craned \
    --AddonVersion v1.2.0 \
    --RawValues '{"nodeResourceAmplificationRatio":1.0}'

# 卸载 DeScheduler
tccli tke DeleteAddon --ClusterId cls-xxxxxxxx --AddonName DeScheduler --region ap-guangzhou
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 节点放大后 Pod 无法调度 | `kubectl describe pod <name>` 查看调度事件 | 虚拟资源超过物理资源，节点实际已满 | 降低放大系数并配合自动规整释放低利用率节点 |
| 节点放大后节点过载 | `kubectl top node` 查看实际利用率 | 放大系数过高，物理资源不足 | 降低 `nodeResourceAmplificationRatio` 至 1.2-1.5 |
| DeScheduler 未驱逐 Pod | `kubectl logs -n kube-system -l app=descheduler --tail=50` | 节点利用率不满足阈值条件，或无可用目标节点 | 确认节点利用率在 thresholds 范围内；确认目标节点有足够资源 |
| 规整导致服务中断 | `kubectl get events --field-selector reason=Descheduled` | 驱逐了有状态工作负载或有数据一致性的 Pod | 为有状态 Pod 添加 `descheduler.crane.io/evict: "false"` annotation |
| DeScheduler 频繁驱逐 | `kubectl get events \| grep Descheduled \| wc -l` | 扫描间隔过短或阈值配置不当 | 增加 `interval` 到 120s+；调整阈值 |
| 放大与规整同时使用导致调度震荡 | `kubectl get pods --all-namespaces -o wide` 观察 Pod 分布 | 放大过度配合激进规整 | 放大系数 ≤ 1.5；规整 `targetThresholds` ≤ 50 |

## 下一步

- [节点放大详细操作](./节点放大/tccli%20操作.md) — page_id `79032`
- [自动规整集群资源](./自动规整集群资源/tccli%20操作.md) — DeScheduler 安装与策略配置
- [Qos 感知调度](../Qos%20感知调度/tccli%20操作.md) — page_id `79775`（节点放大的 QoS 保障前提）
- [业务优先级保障调度](../业务优先级保障调度/tccli%20操作.md) — page_id `118259`
- [调度组件概述](../调度组件概述/tccli%20操作.md) — page_id `111862`

## 控制台替代

登录 [容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) -> 选择集群 `cls-xxxxxxxx` -> **组件管理** -> 安装/配置 `craned`（节点放大参数在组件详情中设置） -> 安装 `DeScheduler` -> 在 **DeScheduler** 详情页配置规整策略。
