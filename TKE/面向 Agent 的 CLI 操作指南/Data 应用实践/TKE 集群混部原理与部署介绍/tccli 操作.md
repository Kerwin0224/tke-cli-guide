# 部署 TKE 集群混部（tccli）

> 对照官方：[TKE 集群混部原理与部署介绍](https://cloud.tencent.com/document/product/457/125430) · page_id `125430`

## 概述

在 TKE 原生节点集群上部署混部（Colocation）能力：通过 Craned 和 QoSAgent 组件，将在线业务（延迟敏感）和离线任务（延迟不敏感）调度到同一批节点，利用在线业务的闲置资源运行离线任务，提升集群资源利用率。

混部资源模型基于 Crane 调度器：QoSAgent 动态计算每节点可用离线资源量（`gocrane.io/cpu`、`gocrane.io/memory` 扩展资源），离线 Pod 通过声明这些扩展资源被调度到有空闲资源的节点。当在线业务负载上升时，离线 Pod 自动被压制或驱逐。

> **kubectl 数据面不可达**：CAM 拒绝公网端点（strategyId:240463971），内网需 VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。控制面操作（集群管理、组件安装）通过 tccli 完成。

## 前置条件

- [环境准备](../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId、secretKey、region 均已配置

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:DescribeClusters, tke:InstallAddon, tke:DescribeAddon
#    tke:DescribeClusterInstances, tke:DescribeClusterKubeconfig
#    tke:UpdateAddon
# 验证
tccli tke DescribeClusters --region <Region>
# expected: exit 0

# 4. 检查 kubectl（须 VPN/IOA）
kubectl version --client
# expected: Client Version v1.32+
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
# 5. 查询目标集群，确认是原生节点集群
tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterStatus "Running"

# 6. 检查是否有原生节点（混部仅支持原生节点）
tccli tke DescribeClusterInstances --region <Region> \
    --ClusterId CLUSTER_ID
# expected: 至少 1 个原生节点（非普通节点）

# 7. 检查 Craned 组件状态
tccli tke DescribeAddon --region <Region> \
    --ClusterId CLUSTER_ID --AddonName craned
# expected: 组件状态已安装或待安装

# 8. 检查 QOSAgent 组件状态
tccli tke DescribeAddon --region <Region> \
    --ClusterId CLUSTER_ID --AddonName qosagent
# expected: 组件状态已安装或待安装
```

### 版本与规格选择

| 组件 | 要求 | 说明 |
|------|------|------|
| TKE 集群 | 原生节点集群（Native Node） | 混部组件仅支持部署在原生节点上 |
| Craned | 任意版本 | 处理数据采集、干扰检测、资源预测 |
| QoSAgent | 需开启 CPU 使用优先级、CPU 超线程隔离、网络 QoS 增强 | DaemonSet，配置离线 Pod 资源限制 |

## 关键字段说明

以下说明 `InstallAddon` 和 `UpdateAddon` 的主要参数。

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `ClusterId` | String | 是 | 原生节点集群 ID，格式 `cls-xxxxxxxx` | 非原生节点集群 → 组件安装成功但无法生效 |
| `AddonName` | String | 是 | `"craned"` 或 `"qosagent"` | 填错名称 → `UnknownParameter` |
| `AddonVersion` | String | 否 | 不填使用默认最新版本 | 指定不存在的版本 → 安装失败 |
| `RawValues` | String | 否 | JSON 格式的组件配置值，用于 QoSAgent 启用高级能力 | 格式错误 → 组件安装失败 |

## 操作步骤

### 步骤 1：创建或确认原生节点集群

#### 选择依据

- **原生节点 vs 普通节点**：混部依赖内核级隔离能力（CPU、内存、IO、网络），仅 TKE 原生节点支持。普通节点无法使用 `gocrane.io/cpu` 等扩展资源。
- 如果已有原生节点集群，直接跳到步骤 2。

创建原生节点集群，最小配置：

`cluster-native-minimal.json`：

```json
{
  "ClusterType": "MANAGED_CLUSTER",
  "ClusterBasicSettings": {
    "ClusterName": "CLUSTER_NAME",
    "ClusterVersion": "1.30.0",
    "VpcId": "VPC_ID",
    "SubnetId": "SUBNET_ID"
  },
  "ClusterCIDRSettings": {
    "ClusterCIDR": "10.1.0.0/16",
    "ServiceCIDR": "10.2.0.0/16"
  },
  "ClusterAdvancedSettings": {
    "ContainerRuntime": "containerd"
  }
}
```

```bash
tccli tke CreateCluster --region <Region> --cli-input-json file://cluster-native-minimal.json
# expected: exit 0，返回 ClusterId
```

**预期输出**：

```json
{
    "ClusterId": "cls-example",
    "RequestId": "xxx-xxx-xxx"
}
```

> 集群创建完成后，需在控制台或通过 API 创建原生节点池。`CreateClusterNodePool` 暂不支持原生节点类型，需通过控制台创建。

轮询集群就绪：

```bash
tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterStatus "Running"
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

### 步骤 2：安装混部组件（控制面，tccli）

#### 选择依据

- **Craned**：处理数据采集、干扰检测和资源预测，是混部能力的中枢。
- **QoSAgent**：部署为 DaemonSet 运行在每个原生节点上，配置离线 Pod 的 CPU、内存、IO 约束。需要开启 CPU 使用优先级、CPU 超线程隔离、网络 QoS 增强。
- **安装顺序**：先安装 Craned，再安装 QoSAgent。

```bash
# 1. 安装 Craned 调度器
tccli tke InstallAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName craned
# expected: exit 0，返回 RequestId

# 2. 安装 QoSAgent
tccli tke InstallAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName qosagent
# expected: exit 0，返回 RequestId

# 3. 轮询 Craned 状态
tccli tke DescribeAddon --region <Region> \
    --ClusterId CLUSTER_ID --AddonName craned
# expected: Status "Running"

# 4. 轮询 QoSAgent 状态
tccli tke DescribeAddon --region <Region> \
    --ClusterId CLUSTER_ID --AddonName qosagent
# expected: Status "Running"
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

### 步骤 3：配置 QoSAgent 高级能力（控制面，tccli）

#### 选择依据

- **CPU 使用优先级**：让在线 Pod 获得更高 CPU 调度优先级，离线 Pod 在 CPU 争抢时被自动压制。
- **CPU 超线程隔离**：在线和离线任务分配不同的 CPU 超线程，避免共享 L1/L2 缓存导致的干扰。
- **网络 QoS 增强**：保证在线业务的网络带宽不被离线任务抢占。

```bash
tccli tke UpdateAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName qosagent \
    --RawValues '{"cpuPriority":{"enabled":true},"hyperThreadIsolation":{"enabled":true},"netQos":{"enabled":true}}'
# expected: exit 0，返回 RequestId
```

> 以上 JSON 为示意格式，实际的 `RawValues` 格式需通过 `tccli tke DescribeAddonValues --region <Region> --ClusterId CLUSTER_ID --AddonName qosagent` 获取准确的 schemas。

### 步骤 4：确认节点离线可用资源（须 VPN/IOA）

获取 kubeconfig：

```bash
tccli tke DescribeClusterKubeconfig --region <Region> --ClusterId CLUSTER_ID \
    | jq -r '.Kubeconfig' > ~/.kube/config
# expected: kubeconfig 写入成功
```

```json
{
  "Kubeconfig": "<Kubeconfig>",
  "RequestId": "<RequestId>"
}
```

确认节点出现离线资源：

```bash
kubectl get nodes -o json | jq '.items[] | {
    name: .metadata.name,
    offline_cpu: .status.capacity["gocrane.io/cpu"],
    offline_mem: .status.capacity["gocrane.io/memory"]
}'
# expected: 原生节点显示 offline_cpu 和 offline_mem 有非零值
```

### 步骤 5：部署在线业务（模拟）（须 VPN/IOA）

`online-nginx.yaml`：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    k8s-app: online-server
  name: online-server
spec:
  replicas: 1
  selector:
    matchLabels:
      k8s-app: online-server
  template:
    metadata:
      labels:
        k8s-app: online-server
    spec:
      containers:
        - name: server
          image: nginx
          imagePullPolicy: Always
          resources:
            limits:
              cpu: "3"
              memory: 3072Mi
            requests:
              cpu: "2"
              memory: 256Mi
```

```bash
kubectl apply -f online-nginx.yaml
# expected: deployment.apps/online-server created
```

### 步骤 6：部署离线避让规则

#### 选择依据

离线避让由三个 CRD 协同工作：

1. **AvoidanceAction**：定义避让行为（驱逐/压制），设置冷却间隔避免频繁操作。
2. **NodeQOS**：定义节点级离线水位线和触发阈值。当在线业务 CPU/内存超过阈值时触发避让。
3. **PodQOS**：用 labelSelector 匹配离线 Pod，设置最低 CPU 优先级。

`colocation-rules.yaml`：

```yaml
# 1. 定义避让行为：驱逐离线 Pod
apiVersion: ensurance.crane.io/v1alpha1
kind: AvoidanceAction
metadata:
  name: eviction
spec:
  coolDownSeconds: 300
  description: "evict low priority pods when online business is under pressure"
  eviction:
    terminationGracePeriodSeconds: null
---
# 2. 节点级水位线：在线 CPU/内存超过 60% 触发驱逐
kind: NodeQOS
metadata:
  name: offline
spec:
  elasticCpuLimit:
    elasticNodeCpuLimit:
      percent: 60
  rules:
  - name: "cpu_total_utilization"
    avoidanceThreshold: 1
    actionName: "eviction"
    metricRule:
      name: "cpu_total_utilization"
      value: 60
  - name: "memory_total_utilization"
    avoidanceThreshold: 1
    actionName: "eviction"
    metricRule:
      name: "memory_total_utilization"
      value: 60
---
# 3. Pod 级 QoS：匹配离线 Pod，设最低 CPU 优先级
apiVersion: ensurance.crane.io/v1alpha1
kind: PodQOS
metadata:
  name: podqos-offline
spec:
  allowedActions:
  - eviction
  labelSelector:
    matchLabels:
      pod-type: offline
  resourceQOS:
    cpuQOS:
      cpuPriority: 7
```

```bash
kubectl apply -f colocation-rules.yaml
# expected: avoidanceaction.ensurance.crane.io/eviction created
#          nodeqos.ensurance.crane.io/offline created
#          podqos.ensurance.crane.io/podqos-offline created
```

### 步骤 7：部署离线任务（须 VPN/IOA）

#### 选择依据

- **扩展资源**：离线 Pod 必须使用 `gocrane.io/cpu` 和 `gocrane.io/memory` 作为资源声明，否则无法调度到原生节点的离线资源池。
- **标签匹配**：设置 `pod-type: offline` 标签以匹配 PodQOS 规则，确保获得正确的 CPU 优先级和避让策略。

`offline-job.yaml`：

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: offline
spec:
  completions: 2
  parallelism: 2
  activeDeadlineSeconds: 600
  template:
    metadata:
      labels:
        pod-type: offline
    spec:
      containers:
      - name: offline
        image: polinux/stress-ng
        command:
        - stress-ng
        - -c
        - "4"
        - --timeout
        - "600"
        resources:
          limits:
            gocrane.io/cpu: "4"
            gocrane.io/memory: 200Mi
          requests:
            gocrane.io/cpu: "4"
            gocrane.io/memory: 200Mi
      restartPolicy: Never
```

```bash
kubectl apply -f offline-job.yaml
# expected: job.batch/offline created
```

## 验证

### 控制面（tccli）

```bash
# 1. 确认集群状态
tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterStatus "Running"

# 2. 确认混部组件运行
tccli tke DescribeAddon --region <Region> \
    --ClusterId CLUSTER_ID --AddonName craned
# expected: Status "Running"

tccli tke DescribeAddon --region <Region> \
    --ClusterId CLUSTER_ID --AddonName qosagent
# expected: Status "Running"
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

### 数据面（须 VPN/IOA）

```bash
# 3. 确认节点有离线扩展资源
kubectl describe node NODE_NAME | grep -A 2 gocrane
# expected: gocrane.io/cpu 和 gocrane.io/memory 有非零值

# 4. 确认在线 Nginx Pod 运行
kubectl get pods -l k8s-app=online-server
# expected: STATUS Running

# 5. 确认离线 Job Pod 状态
kubectl get pods -l pod-type=offline
# expected: STATUS Running 或 Completed

# 6. 确认离线 Pod 使用扩展资源
kubectl get pod OFFLINE_POD_NAME -o json | \
    jq '.spec.containers[0].resources'
# expected: requests 和 limits 包含 gocrane.io/cpu 和 gocrane.io/memory

# 7. 确认离线 Pod 有 QoS 注解
kubectl describe pod OFFLINE_POD_NAME | grep gocrane
# expected: 包含 gocrane.io/cpu-qos 和 gocrane.io/qos 注解
```

```text
NAME  STATUS  AGE
...
```

### 验证离线避让（当在线负载上升时）

```bash
# 8. 模拟在线负载上升：修改在线 Deployment 资源
kubectl patch deployment online-server -p '{"spec":{"template":{"spec":{"containers":[{"name":"server","resources":{"limits":{"cpu":"8","memory":"8192Mi"}}}]}}}}'
# expected: deployment.apps/online-server patched

# 9. 观察离线 Pod 被驱逐
kubectl get pods -l pod-type=offline -w
# expected: 离线 Pod 在在线 CPU 超阈值后被驱逐
```

```text
NAME  STATUS  AGE
...
```

## 清理

> **警告**：删除混部组件不会删除已部署的在线/离线 Pod。`DeleteCluster` 配合 `InstanceDeleteMode: "terminate"` 会级联删除所有关联 CVM 和磁盘。在线业务数据需提前备份。

### 数据面清理（须 VPN/IOA，在控制面之前）

```bash
# 1. 清理前状态检查
kubectl get all,job -A | grep -E "online-server|offline|podqos|nodeqos|avoidance"
# 确认是待删除的目标资源

# 2. 删除离线任务
kubectl delete job offline
# expected: job.batch "offline" deleted

# 3. 删除在线业务
kubectl delete deployment online-server
# expected: deployment.apps "online-server" deleted

# 4. 删除避让规则
kubectl delete -f colocation-rules.yaml
# expected: 三个 CRD 资源已删除

# 5. 验证已删除
kubectl get pods -l pod-type=offline
# expected: No resources found
```

```text
NAME  STATUS  AGE
...
```

### 控制面清理（tccli）

```bash
# 6. 卸载混部组件（如不再使用）
tccli tke DeleteAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName qosagent
# expected: exit 0

tccli tke DeleteAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName craned
# expected: exit 0

# 7. 验证组件已卸载
tccli tke DescribeAddon --region <Region> \
    --ClusterId CLUSTER_ID --AddonName craned
# expected: 返回组件不存在或空

# 8. 删除集群（如为本页新建专用集群）
tccli tke DeleteCluster --region <Region> \
    --ClusterId CLUSTER_ID \
    --InstanceDeleteMode terminate
# ⚠️ 将删除集群及所有关联 CVM、磁盘
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

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `InstallAddon craned` 返回错误 | `tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'` 确认集群类型 | 非原生节点集群（混部仅支持原生节点） | 创建原生节点集群或为现有集群添加原生节点池 |
| `InstallAddon qosagent` 返回 `UnknownParameter` | 检查 AddonName 拼写 | 组件名拼写错误（如 `QOSAgent` vs `qosagent`） | 使用 `qosagent`（全小写） |
| `UpdateAddon` 返回 `InvalidParameter.RawValues` | `tccli tke DescribeAddonValues --region <Region> --ClusterId CLUSTER_ID --AddonName qosagent` | RawValues JSON 格式不符合组件 schema | 用 DescribeAddonValues 获取正确的配置格式后重试 |
| `kubectl apply` 返回 `no matches for kind` | `kubectl api-resources \| grep crane` | 混部 CRD 未安装（Craned 未成功安装） | 确认 `tccli tke DescribeAddon --region <Region> --ClusterId CLUSTER_ID --AddonName craned` 返回 Running |

### 安装成功但功能异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 节点无 `gocrane.io/cpu` 扩展资源 | `kubectl describe node NODE_NAME \| grep gocrane` | 节点非原生节点或 QoSAgent 未运行 | 确认为原生节点；`kubectl get pods -n crane-system` 确认 QoSAgent Pod Running |
| 离线 Pod 一直 Pending | `kubectl describe pod OFFLINE_POD_NAME` 查看 Events | 节点的 `gocrane.io/cpu` 可分配量不足 | 降低离线 Job 的资源请求或减少并发数 |
| 离线 Pod 调度到非原生节点 | `kubectl get pod OFFLINE_POD_NAME -o yaml \| grep nodeName` | 没有配置 nodeAffinity 限制调度范围 | 在离线 Pod 中添加 nodeSelector 或 affinity 匹配原生节点 |
| 在线负载升高但离线 Pod 未被驱逐 | `kubectl describe nodeqos offline` 检查阈值配置 | NodeQOS 阈值设置过高（如 90%） | 降低 `metricRule.value` 阈值（如 60%） |
| 离线 Pod 频繁被驱逐 | 观察驱逐日志 | 阈值设置过低或冷却间隔太短 | 提高阈值或增加 `coolDownSeconds` |

## 下一步

- [TKE 集群混部 Spark 任务实践](../TKE%20集群混部%20Spark%20任务实践/tccli%20操作.md) -- page_id `125431`
- [Crane 调度器介绍](../../调度配置/Crane%20调度器介绍/tccli%20操作.md) -- page_id `75472`
- [业务优先级保障调度](../../调度配置/业务优先级保障调度/tccli%20操作.md)
- [QoS 感知调度](../../调度配置/Qos%20感知调度/tccli%20操作.md)
- [Crane 官方文档](https://github.com/gocrane/crane)

## 控制台替代

[容器服务控制台 - 组件管理](https://console.cloud.tencent.com/tke2/cluster)
