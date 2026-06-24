# 内存精细调度（tccli）

> 对照官方：[内存精细调度](https://cloud.tencent.com/document/product/457/79778) · page_id `79778`

## 概述

TKE 通过 Crane QoSAgent 实现内存精细调度，基于内存水位（Memory Watermark）和 NUMA 感知调度策略，在节点内存压力下主动回收 PageCache、识别冷热内存页并优先回收冷内存，提升集群整体内存利用效率。核心能力包括 PageCache 回收、内存水位控制、NUMA 感知调度和冷热内存识别。

## 前置条件

- 演示集群 `cls-xxxxxxxx`（ap-guangzhou, v1.30.0, Running），kubectl 因 CAM 策略限制不可达，需通过 VPN/IOA 内网或数据面集群操作
- 已安装 [Crane 调度器](../调度组件概述/tccli%20操作.md)（含 QoSAgent 组件）
- 节点内核版本 >= 3.10（PageCache 回收和 Memcg 特性依赖）
- 已了解 [QoSAgent](../Qos%20感知调度/QoSAgent/tccli%20操作.md) 基础概念

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
# 3. 检查 kubectl 版本（需 VPN/IOA 内网环境）
kubectl version --client
# expected: Client Version >= 1.30.0
```

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 查看集群状态 | `tccli tke DescribeClusters --ClusterIds '["cls-xxxxxxxx"]' --region ap-guangzhou` | 是 |
| 查看已安装组件 | `tccli tke DescribeAddon --ClusterId cls-xxxxxxxx --AddonName craned --region ap-guangzhou` | 是 |
| 创建 NodeQOS 内存策略 | `kubectl apply -f nodeqos-mem.yaml` | 是 |
| 创建 PodQOS 内存优先级 | `kubectl apply -f podqos-mem-priority.yaml` | 是 |
| 查看已有 QoS 策略 | `kubectl get podqos,nodeqos` | 是 |
| 查看节点内存水位 | `kubectl describe node <node-name>` | 是 |
| 查看 QoSAgent 日志 | `kubectl logs -n crane-system -l app=qos-agent --tail=50` | 是 |
| 删除 QoS 策略 | `kubectl delete podqos <name> && kubectl delete nodeqos <name>` | 否 |

## 操作步骤

### 核心能力矩阵

| 能力 | 实现组件 | 配置方式 | 适用场景 |
|------|---------|---------|---------|
| PageCache 回收 | QoSAgent (NodeQOS) | NodeQOS CRD `memoryQOS.memPageCacheLimit` | 节点内存压力时主动回收 PageCache |
| 内存水位控制 | QoSAgent (NodeQOS) | NodeQOS CRD `memoryQOS.memcgWaterMark` | 设定安全内存水位线，超限触发回收 |
| NUMA 感知调度 | Crane Scheduler | 调度器自动感知 NUMA 拓扑 | 避免跨 NUMA 内存访问，降低延迟 |
| 冷热内存识别 | QoSAgent | NodeQOS CRD `memoryQOS.memColdPageReclaim` | 优先回收冷内存页，减少业务影响 |
| Memcg OOM 优先级 | QoSAgent (PodQOS) | PodQOS CRD `memoryQOS.memPriority` | OOM 时按优先级依次 Kill 容器 |

### 工作流程

```
1. QoSAgent (DaemonSet) 部署在每个节点上
2. 持续采集节点内存指标（MemTotal、MemAvailable、PageCache 等）
3. 周期性评估内存水位：当可用内存 < 安全水位，触发回收
4. 回收策略：
   - 优先回收冷内存页（长期未访问的 PageCache）
   - 按 Memcg OOM 优先级，低优先级容器优先回收
5. NUMA 感知：调度时记录 Pod 的 NUMA 亲和性，避免跨 NUMA 迁移
```

### 配置示例

#### 1. 创建 NodeQOS（节点级内存控制）

```yaml
apiVersion: scheduling.crane.io/v1alpha1
kind: NodeQOS
metadata:
  name: mem-watermark
spec:
  nodeQualityProbe:
    rules:
    - name: memory-available
      nodeLocalGet:
        localCacheTTL: 60s
  resourceQOS:
    memoryQOS:
      memcgWaterMark:
        enable: true
        lowPercent: 80        # 低水位线：内存使用率达到 80% 开始回收
        highPercent: 90       # 高水位线：内存使用率达到 90% 加速回收
      memPageCacheLimit:
        enable: true
        maxRatio: 50          # PageCache 最大占比 50%
      memColdPageReclaim:
        enable: true
        reclaimThreshold: 30   # 回收阈值（秒），超过 30s 未访问的页视为冷页
```

```bash
kubectl apply -f nodeqos-mem.yaml
```

#### 2. 创建 PodQOS（Pod 级内存优先级）

```yaml
apiVersion: scheduling.crane.io/v1alpha1
kind: PodQOS
metadata:
  name: mem-high-priority
spec:
  resourceQOS:
    memoryQOS:
      memPriority: 0           # 0 = 最高优先级，OOM 时最后被 Kill
  selector:
    matchLabels:
      app: critical-service
---
apiVersion: scheduling.crane.io/v1alpha1
kind: PodQOS
metadata:
  name: mem-low-priority
spec:
  resourceQOS:
    memoryQOS:
      memPriority: 3           # 3 = 最低优先级，OOM 时首先被 Kill
  selector:
    matchLabels:
      app: batch-job
```

```bash
kubectl apply -f podqos-mem-priority.yaml
```

```text
# command executed successfully
```

### 内存优先级等级

| memPriority | 行为 | 适用负载 |
|------------|------|---------|
| 0 | 最高优先级，OOM 时最后被 Kill | 核心在线服务 |
| 1 | 次高优先级 | 重要离线任务 |
| 2 | 中等优先级 | 一般批处理任务 |
| 3 | 最低优先级，OOM 时首先被 Kill | 可抢占的临时任务 |

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

### 数据面（需 VPN/IOA 内网环境）

```bash
# 维度 2：确认 QoSAgent DaemonSet 运行
kubectl get ds -n crane-system qos-agent
```

```text
NAME         DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
qos-agent    3         3         3       3            3           <none>          1d
```

```bash
# 维度 3：确认 QoS 策略已创建
kubectl get podqos
kubectl get nodeqos
```

```text
NAME                AGE
mem-high-priority   10m
mem-low-priority    10m

NAME            AGE
mem-watermark   10m
```

```bash
# 维度 4：查看 QoSAgent 日志确认回收动作
kubectl logs -n crane-system -l app=qos-agent --tail=20 | grep -i "memory\|pagecache\|reclaim"
```

```text
...log output...
```

| 维度 | 命令 | 预期 |
|------|------|------|
| 集群状态 | `DescribeClusters --ClusterIds '["cls-xxxxxxxx"]'` | `ClusterStatus: "Running"` |
| 组件状态 | `DescribeAddon --ClusterId cls-xxxxxxxx --AddonName craned` | `Status: "Succeeded"` |
| DaemonSet | `kubectl get ds -n crane-system qos-agent` | `READY 3/3` |
| 策略就绪 | `kubectl get podqos,nodeqos` | 策略已创建 |
| 回收日志 | `kubectl logs -n crane-system -l app=qos-agent --tail=20` | 含 memory/reclaim 关键词 |

## 清理

无需清理（功能说明页）。如已创建测试策略：

```bash
kubectl delete podqos mem-high-priority mem-low-priority
kubectl delete nodeqos mem-watermark
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| PageCache 回收未生效 | `kubectl logs -n crane-system -l app=qos-agent \| grep "pagecache"` 搜索回收日志 | NodeQOS CRD 未创建或 selector 未匹配节点 | `kubectl get nodeqos -o yaml` 检查配置；确认节点 label 匹配 selector |
| 内存水位控制不工作 | `kubectl describe node <node-name>` 查看节点内存指标 | `memcgWaterMark.enable` 未设为 `true` | 检查 NodeQOS CRD 中 `memcgWaterMark.enable` 字段为 `true` |
| 冷热内存回收效果不佳 | `cat /proc/meminfo` 查看节点 PageCache 使用量 | 节点内存充足，未触发回收阈值 | 降低 `lowPercent` 阈值或减少 `memPageCacheLimit.maxRatio` |
| QoSAgent Pod 异常 | `kubectl get pods -n crane-system -l app=qos-agent` | 组件未安装或安装失败 | `tccli tke DescribeAddon --ClusterId cls-xxxxxxxx --AddonName craned --region ap-guangzhou` 检查组件状态 |
| NUMA 感知调度无效 | `kubectl describe node <node-name> \| grep "NUMA"` | 节点未启用 NUMA 或内核不支持 | 确认节点 BIOS 中 NUMA 已启用；内核 >= 3.10 |

## 下一步

- [Qos 感知调度](../Qos%20感知调度/tccli%20操作.md) — 整体 QoS 调度能力总览（page_id `79775`）
- [QoSAgent 操作指南](../Qos%20感知调度/QoSAgent/tccli%20操作.md) — 安装和配置 QoSAgent
- [磁盘 IO 精细调度](../Qos%20感知调度/磁盘%20IO%20精细调度/tccli%20操作.md) — IO 优先级与隔离
- [网络精细调度](../Qos%20感知调度/网络精细调度/tccli%20操作.md) — 网络带宽精细管理

## 控制台替代

登录 [容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) -> 选择集群 `cls-xxxxxxxx` -> **组件管理** -> 搜索 `craned` -> 查看组件详情 -> 在节点管理页面查看节点内存指标。
