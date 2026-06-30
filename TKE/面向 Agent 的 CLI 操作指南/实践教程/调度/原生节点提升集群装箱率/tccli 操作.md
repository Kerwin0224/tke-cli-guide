# 原生节点提升集群装箱率（tccli）

> 对照官方：[原生节点提升集群装箱率](https://cloud.tencent.com/document/product/457/97925) · page_id `97925`

## 概述

原生节点（Native Node）是 TKE 推出的 CVM 原生节点池，配合 QoSAgent 组件实现动态资源超卖、CPUBurst、内存压缩和 PageCache 回收等能力，在保障业务稳定性的前提下大幅提升集群资源利用率和 Pod 装箱率（Bin-Packing Ratio）。

核心优化手段：
- **动态超卖**：根据节点实时负载动态调整超卖比，充分利用闲时资源
- **CPUBurst**：允许 Pod 在 CPU 空闲时使用超出 limit 的 CPU 资源
- **内存压缩与回收**：自动压缩冷内存页，回收可释放的 PageCache
- **QoSAgent 智能调度**：结合节点负载感知调度，避免热点节点

预期收益：装箱率提升 20-50%，集群节点数量减少 30%+（取决于工作负载特征）。

> **kubectl 数据面不可达**：CAM拒绝公网端点(strategyId:240463971)，内网需VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

## 前置条件

### 环境检查

```bash
tccli --version
# expected: tccli version >= 1.0.0

tccli configure list
# expected: secretId/secretKey 已配置，region: ap-guangzhou

tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]'
# expected: TotalCount >= 1, ClusterStatus "Running", ClusterType "MANAGED_CLUSTER"

# 检查集群是否已有原生节点池
tccli tke DescribeClusterNodePools --region <Region> \
    --ClusterId CLUSTER_ID
# expected: NodePoolSet 列表，标识原生节点池（Type "Native" 或含 Native 标注）

# 检查 QoSAgent 组件状态
tccli tke DescribeAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName qosagent 2>&1
# expected: Status "Succeeded"（已安装）或 ResourceNotFound（待安装）
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
# 查看节点列表和资源使用概况
tccli tke DescribeClusterInstances --region <Region> \
    --ClusterId CLUSTER_ID \
    --Limit 50
# expected: InstanceSet 含 InstanceId, InstanceRole, InstanceState

# 查看节点规格信息
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId CLUSTER_ID \
    --NodePoolId NODE_POOL_ID
# expected: NodeCountSummary, AutoscalingGroupSpec 含实例规格
```

```json
{
  "TotalCount": "<TotalCount>",
  "InstanceSet": "<InstanceSet>",
  "InstanceId": "<InstanceId>",
  "InstanceRole": "<InstanceRole>",
  "FailedReason": "<FailedReason>",
  "InstanceState": "<InstanceState>"
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli/kubectl 命令 | 幂等 |
|-----------|-------------------|:--:|
| 查看集群节点池 | `tccli tke DescribeClusterNodePools --region <Region> --ClusterId CLUSTER_ID` | 是 |
| 查看节点池详情 | `tccli tke DescribeClusterNodePoolDetail --region <Region> --ClusterId CLUSTER_ID --NodePoolId NODE_POOL_ID` | 是 |
| 创建原生节点池 | `tccli tke CreateClusterNodePool --region <Region> --cli-input-json file://create-nodepool.json` | 否 |
| 修改节点池配置 | `tccli tke ModifyClusterNodePool --region <Region> --cli-input-json file://modify-nodepool.json` | 否 |
| 安装 QoSAgent | `tccli tke InstallAddon --region <Region> --AddonName qosagent --ClusterId CLUSTER_ID` | 否 |
| 查看节点资源使用 | `kubectl top nodes`（需 VPN/IOA） | 是 |
| 查看节点分配详情 | `kubectl describe node NODE_NAME`（需 VPN/IOA） | 是 |
| 查看 Pod 资源使用 | `kubectl top pods -A --sort-by=cpu`（需 VPN/IOA） | 是 |
| 查看装箱率分析 | `kubectl get pods -A -o json \| jq` (需 VPN/IOA) | 是 |

## 操作步骤

### 步骤 1：分析当前装箱率

#### 选择依据

装箱率低的主要表现：
- 节点 CPU Request 利用率 < 50%（大量预留资源未使用，但无法调度新 Pod）
- 节点内存碎片化（总空闲内存足以调度新 Pod，但无单个节点能满足大内存 Pod）
- 大量小 Pod 散布在多个节点上（每个 Pod 有系统 Overhead：pause 容器、kubelet、CNI 等）

诊断指标：
- **Request 装箱率** = Sum(Pod Requests) / Node Allocatable：低于 60% 需优化
- **实际利用率** = Sum(Pod Actual Usage) / Node Allocatable：远低于 Request 装箱率说明 Pod 过度预留
- **节点碎片率** = 空闲资源片段数 / 总空闲资源：碎片率 > 20% 说明存在调度碎片

```bash
# 控制面：获取所有节点资源容量
tccli tke DescribeClusterInstances --region <Region> \
    --ClusterId CLUSTER_ID \
    | jq -r '.InstanceSet[] | "\(.InstanceId) \(.InstanceState)"'
# expected: 节点列表和运行状态

# 数据面：查看节点资源分配和使用
kubectl describe node NODE_NAME | grep -A25 "Allocated resources"
# expected:
#   Resource           Requests      Limits
#   cpu                3200m (80%)   6000m (150%)
#   memory             12Gi (75%)    20Gi (125%)

kubectl top nodes
# expected: NAME CPU(cores) CPU% MEMORY(bytes) MEMORY%
```

```json
{
  "TotalCount": "<TotalCount>",
  "InstanceSet": "<InstanceSet>",
  "InstanceId": "<InstanceId>",
  "InstanceRole": "<InstanceRole>",
  "FailedReason": "<FailedReason>",
  "InstanceState": "<InstanceState>"
}
```

计算装箱率（数据面）：

```bash
# 各节点 Request CPU 装箱率
kubectl get nodes -o json \
    | jq -r '.items[] | "\(.metadata.name) " + (
        ([.status.allocatable.cpu | gsub("m";"") | tonumber] | .[0]) as $total |
        ([.status.allocatable.memory | gsub("Ki";"") | tonumber] | .[0]) as $memTotal |
        "\(.status.capacity.cpu) allocatable")'
# 使用 kubectl resource-capacity 或 kube-resource-report 工具更精确
```

```text
NAME  STATUS  AGE
...
```

### 步骤 2：启用 QoSAgent 资源优化

#### 选择依据

- **QoSAgent** vs **仅调整 Request/Limit**：Request/Limit 调优是基础手段，QoSAgent 提供运行时动态超卖、CPUBurst、内存压缩等内核级优化
- **超卖策略**：激进超卖（高密度部署，装箱率 > 80%）vs 保守超卖（留 30% 余量以防突发）
- **适用负载**：Web/API 服务（CPU 间歇使用）适合 CPUBurst；批处理任务适合内存超卖

```bash
# 查看 QoSAgent 可用版本
tccli tke DescribeAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    | jq '.Addons[] | select(.AddonName=="qosagent")'
# expected: AddonVersion 列表

cat > install-qosagent.json <<'EOF'
{
    "ClusterId": "CLUSTER_ID",
    "AddonName": "qosagent",
    "AddonVersion": "1.0.0"
}
EOF
tccli tke InstallAddon --region <Region> \
    --cli-input-json file://install-qosagent.json
# expected: exit 0

# 等待安装完成
for i in $(seq 1 30); do
    STATUS=$(tccli tke DescribeAddon --region <Region> \
        --ClusterId CLUSTER_ID \
        --AddonName qosagent 2>/dev/null \
        | jq -r '.Status')
    if [ "$STATUS" = "Succeeded" ]; then break; fi
    sleep 10
done
# expected: QoSAgent installed successfully
```

### 步骤 3：调整 Pod 资源 Request/Limit

#### 最小创建

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: optimized-app
  namespace: NAMESPACE
spec:
  replicas: 3
  selector:
    matchLabels:
      app: optimized-demo
  template:
    metadata:
      labels:
        app: optimized-demo
    spec:
      containers:
      - name: app
        image: APP_IMAGE
        resources:
          requests:
            cpu: "200m"
            memory: "256Mi"
          limits:
            cpu: "1000m"
            memory: "512Mi"
```

```bash
kubectl apply -f deployment-optimized.yaml
# expected: deployment.apps/optimized-app configured
```

#### 增强配置（CPU Burst 注解）

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: burst-app
  namespace: NAMESPACE
  annotations:
    tke.cloud.tencent.com/qos-agent: "enable"
spec:
  replicas: 5
  selector:
    matchLabels:
      app: burst-demo
  template:
    metadata:
      labels:
        app: burst-demo
      annotations:
        tke.cloud.tencent.com/cpu-burst: "enabled"
        tke.cloud.tencent.com/cpu-burst-limit: "2"
    spec:
      containers:
      - name: burst-app
        image: APP_IMAGE
        resources:
          requests:
            cpu: "200m"
            memory: "256Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
```

```bash
kubectl apply -f deployment-burst.yaml
# expected: deployment.apps/burst-app created
```

### 步骤 4：创建/调整原生节点池配置

#### 选择依据

- **原生节点** vs **普通节点**：原生节点支持 QoSAgent 内核级优化、动态超卖、CVM 规格灵活
- **节点规格选择**：使用 CPU:Memory 比例与工作负载匹配的实例（如 1:4 适合内存密集型）
- **节点池数量**：少量大规格节点池 > 大量小规格节点池（减少碎片）

参数说明：

| 参数 | 说明 | 示例值 |
|------|------|--------|
| ClusterId | 集群 ID | `CLUSTER_ID` |
| Name | 节点池名称 | `native-pool-optimized` |
| InstanceType | CVM 实例规格 | `S5.2XLARGE16` |
| DesiredNodesNum | 期望节点数 | `3` |
| NodePoolOs | 操作系统 | `tlinux3.1x86_64` |

```bash
cat > create-native-nodepool.json <<'EOF'
{
    "ClusterId": "CLUSTER_ID",
    "Name": "native-pool-optimized",
    "Type": "Native",
    "DeletionProtection": false,
    "Labels": [
        {
            "Name": "node-type",
            "Value": "native"
        }
    ],
    "Taints": [],
    "Annotations": [
        {
            "Name": "tke.cloud.tencent.com/qos-agent",
            "Value": "enabled"
        }
    ],
    "Native": {
        "InstanceChargeType": "POSTPAID_BY_HOUR",
        "SubnetIds": ["SUBNET_ID"],
        "InstanceTypes": ["S5.2XLARGE16"],
        "SystemDisk": {
            "DiskType": "CLOUD_PREMIUM",
            "DiskSize": 50
        },
        "DesiredReplicas": 3,
        "Scaling": {
            "MinReplicas": 2,
            "MaxReplicas": 10
        }
    }
}
EOF
tccli tke CreateClusterNodePool --region <Region> \
    --cli-input-json file://create-native-nodepool.json
# expected: 返回 NodePoolId
```

### 步骤 5：验证装箱率提升

```bash
# 优化前后对比
kubectl describe node NODE_NAME | grep -A25 "Allocated resources"
# expected: CPU Requests 百分比从 45% 上升到 65-75%

kubectl top nodes
# expected: 实际 CPU 使用率更接近 Requests 值，说明预留更合理

# 查看节点上 Pod 分布密度
kubectl get pods -A -o wide --field-selector spec.nodeName=NODE_NAME \
    --no-headers | wc -l
# expected: 单一节点 Pod 数量增加（装箱密度提升）

# 查看 QoSAgent 超卖统计
kubectl get configmap -n kube-system qosagent-stats -o json 2>/dev/null \
    | jq '.data'
# expected: CPU 超卖比、内存压缩量等统计
```

```text
NAME  STATUS  AGE
...
```

## 验证

### 控制面（tccli）

```bash
# 验证 QoSAgent 组件运行
tccli tke DescribeAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName qosagent
# expected: Status "Succeeded", AddonVersion "1.0.0"

# 验证原生节点池
tccli tke DescribeClusterNodePools --region <Region> \
    --ClusterId CLUSTER_ID \
    | jq '.NodePoolSet[] | select(.Name=="native-pool-optimized")'
# expected: NodeCountSummary, LifeState "normal"
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

### 数据面（需 VPN/IOA）

```bash
# 验证节点装箱率提升（优化后 Request 利用率为 65-80%）
kubectl describe node $(kubectl get nodes -l node-type=native -o name | head -1) \
    | grep -A25 "Allocated resources"
# expected: CPU Requests 65-80%, Memory Requests 60-75%

# 验证 QoSAgent Pod 运行
kubectl get pods -n kube-system -l app=qosagent --no-headers
# expected: 每个原生节点上有一个 qosagent Pod 在 Running

# 模拟 CPU Burst（发送突发流量后查看）
kubectl top pod -n NAMESPACE -l app=burst-demo
# expected: CPU 使用率可能短暂超过 limits（因 CPUBurst 允许）

# 对比两种部署的装箱率差异
echo "=== QoSAgent 优化前 ==="
kubectl get pods -A -o wide | grep -v "native" | awk '{print $NF}' | sort | uniq -c | sort -rn
echo "=== QoSAgent 优化后 ==="
kubectl get pods -A -o wide | grep "native" | awk '{print $NF}' | sort | uniq -c | sort -rn
# expected: 原生节点上 Pod 密度更高
```

```text
NAME  STATUS  AGE
...
```

## 清理

### 数据面（需 VPN/IOA）

```bash
# 删除测试 Deployment
kubectl delete deployment optimized-app burst-app -n NAMESPACE
# expected: deployments deleted
# ⚠️ 警告：生产环境请根据实际需求决定是否迁移 Pod 后再缩容节点池

# 驱逐原生节点上的 Pod（缩容前迁移工作负载）
kubectl cordon NODE_NAME
kubectl drain NODE_NAME --ignore-daemonsets --delete-emptydir-data
# expected: Pod 从节点迁移到其他节点
# ⚠️ 警告：drain 会驱逐所有 Pod，确认无状态 Pod 数据已持久化
```

### 控制面（tccli）

```bash
# 缩容原生节点池到 0
tccli tke ModifyClusterNodePool --region <Region> \
    --ClusterId CLUSTER_ID \
    --NodePoolId NODE_POOL_ID \
    --DesiredNodesNum 0
# expected: 节点池缩容中

# 删除节点池
tccli tke DeleteClusterNodePool --region <Region> \
    --ClusterId CLUSTER_ID \
    --NodePoolId NODE_POOL_ID
# expected: exit 0
# ⚠️ 警告：删除节点池会释放所有节点实例（CVM），实例上所有数据丢失
# ⚠️ 计费警告：POSTPAID_BY_HOUR 实例按秒计费，删除后立即停止计费

# 卸载 QoSAgent
tccli tke DeleteAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName qosagent
# expected: exit 0
```

```bash
# 验证清理
tccli tke DescribeClusterNodePools --region <Region> \
    --ClusterId CLUSTER_ID \
    | jq '.NodePoolSet[] | select(.Name=="native-pool-optimized")'
# expected: （无输出，节点池已删除）
```

```json
{
  "NodePoolSet": "<NodePoolSet>",
  "NodePoolId": "<NodePoolId>",
  "Name": "<Name>",
  "ClusterInstanceId": "<ClusterInstanceId>",
  "LifeState": "<LifeState>",
  "LaunchConfigurationId": "<LaunchConfigurationId>"
}
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 装箱率仍然低（<50%） | `kubectl describe node NODE_NAME \| grep "Allocated"` | Pod Request 设置过高，远大于实际使用量 | 调整 Request 到实际使用的 50-80%，避免过度预留 |
| 节点内存碎片化严重 | `kubectl describe node NODE_NAME \| grep "Allocated"` 对比 Requests 和 Allocatable | 大量小 Pod + 少量大内存 Pod 混合造成碎片 | 使用节点亲和性将大内存 Pod 集中调度，或使用 Pod Priority/Preemption |
| QoSAgent 组件安装后节点未启用 | `kubectl get nodes -o json \| jq '.items[] \| select(.metadata.annotations["tke.cloud.tencent.com/qos-agent"])'` | 节点类型不是原生节点或不支持 QoSAgent | 确认节点为原生节点（Native Node），非普通 CVM 节点池 |
| CPUBurst 未生效 | `kubectl top pod POD_NAME -n NAMESPACE` 观察 CPU 是否超限 | Pod 缺少 CPUBurst 注解或 QoSAgent 未运行 | 添加 Pod 注解 `tke.cloud.tencent.com/cpu-burst: enabled`，确认 QoSAgent Pod Running |

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 超卖导致节点 OOMKill | `kubectl describe pod POD_NAME -n NAMESPACE \| grep -i "oom"` | 超卖比设置过高，内存超卖导致实际内存不足 | 降低超卖比、增加 Pod memory limit、或使用 Guaranteed QoS（request=limit） |
| 原生节点池创建失败 | `tccli tke DescribeClusterNodePoolDetail --ClusterId CLUSTER_ID --NodePoolId NODE_POOL_ID` | 子网 IP 不足或实例规格售罄 | 选择其他可用区子网或更换实例规格 |
| QoSAgent Pod CrashLoopBackOff | `kubectl logs -n kube-system qosagent-POD_NAME` | 内核模块不兼容或节点 OS 版本过旧 | 升级节点 OS 到 TLinux 3.1+ 或确认内核版本 >= 4.14 |
| Request 调低后 Pod 被 QoS Eviction | `kubectl describe pod POD_NAME -n NAMESPACE \| grep "Evict"` | Request 过低导致 Pod 在资源紧张时被驱逐 | 设置 Request 不低于实测使用量的 50%，或使用 PriorityClass 保护关键 Pod |

## 下一步

- [安装 CoScheduling 实现批调度](../安装%20CoScheduling%20实现批调度/tccli%20操作.md) -- page_id `106150`
- [调度组件概述](../../../调度配置/调度组件概述/tccli%20操作.md) -- page_id `111862`
- [资源利用率优化调度](../../../调度配置/资源利用率优化调度/tccli%20操作.md) -- page_id `118259`
- [HPA 弹性伸缩](../../../弹性伸缩/HPA%20弹性伸缩/tccli%20操作.md) -- page_id `37835`

## 控制台替代

[容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) -> 选择集群 -> 节点管理 -> 节点池 -> 新建节点池（类型选「原生节点」）。
控制台支持创建/删除节点池、安装组件，资源分析和装箱率详情需通过 kubectl 查询。
