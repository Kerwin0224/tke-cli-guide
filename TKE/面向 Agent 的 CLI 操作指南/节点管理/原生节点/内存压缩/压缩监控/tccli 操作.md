# 压缩监控

> 对照官方：[压缩监控](https://cloud.tencent.com/document/product/457/102625) · page_id `102625` · tccli ≥3.1.107.1 · API 2018-07-24

## 概述

QosAgent 是原生节点 QoS 特性的监控组件，以 DaemonSet 形式部署在每台原生节点上，在端口 `8084` 上暴露 Prometheus 兼容格式的指标。通过查询这些指标，可以监控节点和 Pod 的内存压缩情况、内存与 CPU 的压力状态。

核心监控目标：

| 监控目标 | 说明 | 关键指标 |
|---------|------|---------|
| 哪些业务可以被压缩 | 识别可压缩的匿名内存和文件页 | `pod_memory_info`（Inactive anon、Inactive File） |
| 节约了多少内存量 | 每个 Pod 的 zram 压缩收益 | `pod_memory_info`（zram 压缩前大小 - 压缩后大小） |
| 内存回收是否准确 | 缺页和 PSI 压力是否在合理范围 | `node_memory_page_fault_major`、`node_pressure_total`（Memory PSI） |
| 业务是否稳定 | OOM 次数、PSI 变化、Zram 设备 IO | `pod_memory_oom_kill`、`node_pressure_total`、`node_disk_io_time_seconds_total`（zram0） |

**访问方式**：QosAgent 暴露的 Prometheus 指标可通过以下方式获取：
- `kubectl exec` 进入 QosAgent Pod，执行 `curl localhost:8084/metrics`
- `kubectl port-forward` 将 QosAgent Pod 端口转发到本地
- 从同 VPC 内网直接访问 `http://<NodeIP>:8084/metrics`
- 对接腾讯云 Prometheus 监控服务自动化采集，通过 Grafana 面板可视化

> **注意**：内存压缩仅在**原生节点（Native Node）**上可用。普通节点池不支持此特性。

## 前置条件

- [环境准备](../../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置，region 为目标地域

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:DescribeClusterNodePools
#    tke:DescribeClusterNodePoolDetail
#    tke:DescribeClusterInstances
#    monitor:DescribePrometheusInstancesOverview        # 步骤 7 需要
#    monitor:CreatePrometheusClusterAgent               # 步骤 7 需要
#    monitor:DescribePrometheusClusterAgents            # 步骤 7 需要
#    monitor:DeletePrometheusClusterAgent               # 清理需要
# 验证：执行 DescribeClusterNodePools 确认权限
tccli tke DescribeClusterNodePools --ClusterId <ClusterId> --region <Region>
# expected: exit 0，返回节点池列表（可为空）

# 4. 检查 kubectl（数据面操作需要）
kubectl version --client
# expected: Client Version: v1.xx.x
```

### 资源检查

```bash
# 5. 确认集群存在且包含原生节点池
tccli tke DescribeClusterNodePools --ClusterId <ClusterId> --region <Region>
# expected: 至少返回 1 个节点池，检查 Annotations 中是否包含
#           node.tke.cloud.tencent.com/in-place-upgrade 注解
```

### 网络可达性前提

kubectl 和 QosAgent 端口（`8084`）属于数据面操作，需要满足以下条件之一才能访问：

- **IOA / VPN / 专线**：接入集群所在 VPC
- **同 VPC CVM 跳板机**：在集群 VPC 内启动一台 CVM，从跳板机执行数据面命令
- **kubectl proxy**：通过集群内网端点代理访问

> ⚠️ 以下命令中，标记为「数据面」的操作均需满足上述网络前提。若当前环境不满足，命令将以注释形式标注，读者需在可达环境下自行执行。

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看节点池列表 | `DescribeClusterNodePools` | 是 |
| 查看原生节点池详情 | `DescribeClusterNodePoolDetail` | 是 |
| 查看节点实例信息 | `DescribeClusterInstances` | 是 |
| 查看集群访问端点 | `DescribeClusterEndpoints` | 是 |

**数据面操作（kubectl / curl）**：

| 控制台操作 | 数据面命令 | 说明 |
|-----------|-----------|------|
| 查看节点状态 | `kubectl get nodes -o wide` | 确认原生节点正常运行 |
| 访问 QosAgent 指标 | `kubectl exec` / `port-forward` / `curl http://<NodeIP>:8084/metrics` | 查看压缩监控指标 |

## 操作步骤

### 步骤 1：识别原生节点池

内存压缩仅在原生节点上可用。通过 `DescribeClusterNodePools` 查询所有节点池，检查 `Annotations` 中是否包含 `node.tke.cloud.tencent.com/in-place-upgrade` 注解来区分原生节点池与普通节点池。

```bash
tccli tke DescribeClusterNodePools --ClusterId <ClusterId> \
    --region <Region>
# expected: exit 0，返回节点池列表
```

**预期输出**：

```json
{
    "NodePoolSet": [
        {
            "NodePoolId": "np-example-sn",
            "Name": "<普通节点池名称>",
            "LifeState": "normal",
            "NodeCountSummary": {
                "AutoscalingAdded": {"Normal": 1, "Total": 1},
                "ManuallyAdded": {"Total": 0}
            },
            "RuntimeConfig": {
                "RuntimeType": "containerd",
                "RuntimeVersion": "1.6.9"
            },
            "NodePoolOs": "ubuntu22.04x86_64"
        },
        {
            "NodePoolId": "np-example-native",
            "Name": "<原生节点池名称>",
            "LifeState": "normal",
            "NodeCountSummary": {
                "AutoscalingAdded": {"Normal": 1, "Total": 1},
                "ManuallyAdded": {"Total": 0}
            },
            "RuntimeConfig": {
                "RuntimeType": "containerd",
                "RuntimeVersion": "1.6.9"
            },
            "NodePoolOs": "tlinux3.1x86_64",
            "Annotations": [
                {
                    "Name": "node.tke.cloud.tencent.com/in-place-upgrade",
                    "Value": "true"
                }
            ]
        }
    ],
    "TotalCount": 2,
    "RequestId": "..."
}
```

| 字段 | 说明 |
|------|------|
| `Annotations[].Name` | 标识原生节点池的关键注解：`node.tke.cloud.tencent.com/in-place-upgrade` |
| `LifeState` | `normal` 表示节点池处于正常状态 |
| `NodePoolOs` | 原生节点池通常使用 tlinux 系列操作系统 |

**识别原生节点池的快捷命令**：

```bash
tccli tke DescribeClusterNodePools --ClusterId <ClusterId> \
    --region <Region> \
    | jq '.NodePoolSet[] | select(.Annotations != null) | {NodePoolId, Name, Annotations}'
# expected: 仅输出包含 Annotations 的节点池（即原生节点池）
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<ClusterId>` | 集群 ID | 格式 `cls-xxxxxxxx` | `tccli tke DescribeClusters --region <Region>` |
| `<Region>` | 地域 | 如 `ap-guangzhou` | `tccli configure list` 查看已配置的 region |

### 步骤 2：查询原生节点池详情

确认原生节点池后，查询其详细配置，包括标签、注解、期望 Pod 数等。

```bash
tccli tke DescribeClusterNodePoolDetail --ClusterId <ClusterId> \
    --NodePoolId <NodePoolId> \
    --region <Region>
# expected: exit 0，返回节点池完整配置
```

**预期输出**：

```json
{
    "NodePool": {
        "NodePoolId": "np-example-native",
        "Name": "<原生节点池名称>",
        "LifeState": "normal",
        "Labels": [
            {"Name": "pool-name", "Value": "<LabelValue>"},
            {"Name": "env", "Value": "rewrite"}
        ],
        "Annotations": [
            {"Name": "node.tke.cloud.tencent.com/in-place-upgrade", "Value": "true"}
        ],
        "NodeCountSummary": {
            "AutoscalingAdded": {"Normal": 1, "Total": 1}
        },
        "RuntimeConfig": {
            "RuntimeType": "containerd",
            "RuntimeVersion": "1.6.9"
        },
        "NodePoolOs": "tlinux3.1x86_64",
        "DeletionProtection": false,
        "DesiredPodNum": 64
    },
    "RequestId": "..."
}
```

| 字段 | 说明 |
|------|------|
| `DesiredPodNum` | 该节点池上预期调度的 Pod 数量上限 |
| `DeletionProtection` | 节点池删除保护状态 |
| `Labels` | 节点标签，可用于 Pod 调度策略 |

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<NodePoolId>` | 原生节点池 ID | 格式 `np-xxxxxxxx` | 步骤 1 输出中 `Annotations` 不为空的 `NodePoolId` |

### 步骤 3：查询节点实例信息

获取原生节点池中节点的实例 ID 和内网 IP，方便后续访问 QosAgent 端口。

```bash
tccli tke DescribeClusterInstances --ClusterId <ClusterId> \
    --region <Region>
# expected: exit 0，返回集群内所有节点实例
```

**预期输出**：

```json
{
    "InstanceSet": [
        {
            "InstanceId": "ins-example-01",
            "InstanceRole": "WORKER",
            "InstanceState": "running",
            "LanIP": "<普通节点 IP>",
            "NodePoolId": "np-example-sn"
        },
        {
            "InstanceId": "ins-example-02",
            "InstanceRole": "WORKER",
            "InstanceState": "running",
            "LanIP": "<NodeIP>",
            "NodePoolId": "np-example-native"
        }
    ],
    "TotalCount": 2,
    "RequestId": "..."
}
```

| 字段 | 说明 |
|------|------|
| `LanIP` | 节点内网 IP，QosAgent 监听地址 |
| `NodePoolId` | 所属节点池 ID，匹配步骤 1 中识别的原生节点池 |
| `InstanceState` | `running` 表示节点正常运行 |

**提取原生节点 IP**：

```bash
tccli tke DescribeClusterInstances --ClusterId <ClusterId> \
    --region <Region> \
    | jq '.InstanceSet[] | select(.NodePoolId == "<NodePoolId>") | {InstanceId, LanIP, InstanceState}'
# expected: 仅返回目标原生节点池中的节点信息
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<ClusterId>` | 集群 ID | 格式 `cls-xxxxxxxx` | `tccli tke DescribeClusters --region <Region>` |
| `<Region>` | 地域 | 如 `ap-guangzhou` | `tccli configure list` 查看已配置的 region |
| `<NodePoolId>` | 原生节点池 ID | 格式 `np-xxxxxxxx` | 步骤 1 输出中 `Annotations` 不为空的 `NodePoolId` |

### 步骤 4：查看节点状态（数据面）

> ⚠️ 此步骤需要 kubectl 连通集群。若当前环境不满足网络可达性前提（见[前置条件](#前置条件)），请从可达环境执行此命令。

```bash
kubectl get nodes -o wide
# expected: 各节点 STATUS 为 Ready，原生节点出现在列表中
```

**预期输出**（示例）：

```text
NAME           STATUS   ROLES    AGE   VERSION   INTERNAL-IP    EXTERNAL-IP   OS-IMAGE            KERNEL-VERSION
<NODE_NAME>    Ready    <none>   30d   v1.32.2   <NodeIP>       <none>        TencentOS Server    5.4.241-1-tlinux4
```

### 步骤 5：访问 QosAgent 监控指标

QosAgent 以 DaemonSet 形式运行在每台原生节点上，监听端口 `8084`，提供 Prometheus 兼容格式的指标。

#### 选择依据

*为什么选择特定访问方式：*

- **`kubectl exec` 进入 Pod**：最直接的方式，无需额外网络配置。适用于在节点内部查询。
- **`kubectl port-forward` 端口转发**：将 QosAgent Pod 的 8084 端口映射到本地，便于本地浏览器或 curl 访问。
- **同 VPC 直接 curl**：从同 VPC 的 CVM 跳板机直接访问 `<NodeIP>:8084/metrics`，延迟最低。
- **Prometheus 自动采集**：推荐生产环境使用，对接腾讯云 Prometheus 监控服务，配合 Grafana 面板实现持续监控（详见步骤 7）。

#### 方法一：kubectl exec 进入 Pod 查询

> ⚠️ 此步骤需要 kubectl 连通集群。

```bash
# 查找 QosAgent Pod
kubectl get pods -n kube-system -l app=qosagent -o wide
# expected: 每台原生节点上有一个 QosAgent Pod，STATUS 为 Running

# 进入 QosAgent Pod 查询指标
kubectl exec -it <QOSAGENT_POD> -n kube-system -- curl -s http://localhost:8084/metrics | head -50
# expected: 返回 Prometheus 格式的指标数据
```

#### 方法二：kubectl port-forward 端口转发

> ⚠️ 此步骤需要 kubectl 连通集群。

```bash
# 将 QosAgent Pod 的 8084 端口转发到本地 8084
kubectl port-forward -n kube-system <QOSAGENT_POD> 8084:8084
# expected: Forwarding from 127.0.0.1:8084 -> 8084

# 在另一个终端访问
curl -s http://localhost:8084/metrics | head -50
```

#### 方法三：同 VPC 内网直连

> ⚠️ 此步骤要求执行机器在集群 VPC 内网可达（IOA/VPN/专线 或同 VPC CVM）。

```bash
curl -s http://<NodeIP>:8084/metrics | head -50
# expected: 返回 Prometheus 格式的指标数据
```

**预期输出**（Prometheus 指标示例）：

```text
# HELP pod_memory_info Pod memory information including RSS, anonymous, file and active/inactive pages
# TYPE pod_memory_info gauge
pod_memory_info{pod="<POD_NAME>",namespace="<NAMESPACE>",container="<CONTAINER>"} 1.048576e+08

# HELP pod_pressure_total Pod PSI showing stall time due to lack of CPU/memory/IO
# TYPE pod_pressure_total counter
pod_pressure_total{pod="<POD_NAME>",namespace="<NAMESPACE>",resource="memory"} 1234

# HELP node_memory_page_fault_major Major page faults requiring disk reads
# TYPE node_memory_page_fault_major counter
node_memory_page_fault_major 5678

# HELP node_disk_io_time_seconds_total Total IO time for zram0 device
# TYPE node_disk_io_time_seconds_total counter
node_disk_io_time_seconds_total{device="zram0"} 42.5

# HELP node_pressure_total Node-level CPU/IO/Memory PSI
# TYPE node_pressure_total gauge
node_pressure_total{resource="memory"} 0.05
```

### 步骤 6：关键指标解读

QosAgent 在端口 `8084` 上暴露的指标分为 Pod 级别和 Node 级别两类，覆盖内存压缩、压力检测、缺页统计和 OOM 监控。

#### Pod 级别指标

| 指标名 | 类型 | 说明 | 使用场景 |
|--------|------|------|---------|
| `pod_pressure_total` | Counter | Pod 级 PSI（Pressure Stall Information），显示因缺少 CPU / 内存 / IO 的等待时长 | 评估 Pod 资源压力趋势，`resource` 标签区分 CPU、memory、IO |
| `pod_memory_info` | Gauge | Pod 内存详情：RSS、匿名内存（Anon）、文件页（File）、活跃内存（Active）、非活跃内存（Inactive） | 判断哪些 Pod 的内存可被压缩。**Workingset Saved** = Inactive Anon（kubelet 视角）；**Memory Saved** = Inactive Anon + Inactive File（监控视角） |
| `pod_memory_page_fault_info` | Counter | Pod 缺页次数统计 | 观察内存压缩导致的缺页频率变化 |
| `pod_memory_oom_kill` | Counter | Pod OOM Kill 次数 | 判断压缩是否导致业务被 OOM Kill |

**内存节约量计算**：每个 Pod 节约的内存 = zram 压缩前大小 - zram 压缩后大小（通过 `pod_memory_info` 中 Anon 字段的压缩前后对比得出）。

#### Node 级别指标

| 指标名 | 类型 | 说明 | 使用场景 |
|--------|------|------|---------|
| `node_pressure_total` | Gauge | 节点级别 CPU / IO / Memory PSI | 评估整机资源压力，Memory PSI 过高说明内存回收频繁 |
| `node_memory_page_fault_distance` | Counter | Refault 频率，表明「热」页面被换出后再次被访问的次数 | Refault 值过高说明压缩策略过于激进，「热」页面被错误换出 |
| `node_memory_page_fault_major` | Counter | 发生磁盘读取的缺页次数（Major Page Fault） | 数值突增表明内存回收导致大量磁盘 IO，业务性能可能受影响 |
| `node_disk_io_time_seconds_total` | Counter | 节点磁盘 IO 总时间 | 关注 `device="zram0"` 标签，观察换出/换入的 IO 耗时 |
| `node_disk_read_bytes_total` | Counter | 磁盘/zram0 设备 IO 读带宽 | zram0 设备读带宽过高说明缺页后换入频繁 |
| `node_disk_reads_completed_total` | Counter | IO 读取完成次数 | 间接反映内存压缩导致匿名内存缺页的频率 |
| `node_disk_writes_completed_total` | Counter | zram0 写入完成次数 | 反映内存压缩写入（换出）的频率 |
| `node_disk_write_time_seconds_total` | Counter | zram0 写入总耗时 | 与 `node_disk_writes_completed_total` 结合计算平均写入延迟 |
| `node_memory_oom_kill` | Counter | 节点级别 OOM Kill 次数 | 评估整机 OOM 风险 |

#### 业务稳定性判断方法

| 判断维度 | 观察指标 | 异常信号 |
|---------|---------|---------|
| 内存回收是否准确 | `node_memory_page_fault_major`、`node_pressure_total`（Memory PSI） | Major Page Fault 持续增长 + Memory PSI 偏高 → 回收过于激进 |
| 业务是否稳定 | `pod_memory_oom_kill`、`node_pressure_total`、`node_disk_io_time_seconds_total`（zram0） | OOM Kill 增多 / PSI 持续高位 / zram0 IO 突增 → 业务受影响 |

### 步骤 7：对接 Prometheus 监控服务（可选）

推荐生产环境将 QosAgent 指标对接腾讯云 Prometheus 监控服务，通过 Grafana 面板实现长期监控和告警。

#### 选择依据

*为什么要对接 Prometheus 监控服务：*

- **持久化存储**：Prometheus 服务端存储历史指标数据，支持任意时间范围查询
- **Grafana 可视化**：导入预置面板，直观展示压缩效果和压力趋势
- **告警能力**：基于指标阈值配置告警规则，及时发现异常

#### 控制面操作

**步骤 7.1：查看可用的 Prometheus 实例**

```bash
tccli monitor DescribePrometheusInstancesOverview --region <Region>
# expected: exit 0，返回 Prometheus 实例列表
```

**预期输出**：

```json
{
    "Instances": [
        {
            "InstanceId": "prom-example",
            "InstanceName": "<实例名称>",
            "VpcId": "vpc-example",
            "SubnetId": "subnet-example",
            "InstanceStatus": 2,
            "ChargeStatus": 1,
            "EnableGrafana": 0,
            "GrafanaURL": "",
            "InstanceChargeType": 3,
            "SpecName": "共享版",
            "DataRetentionTime": 15,
            "BoundTotal": 3,
            "BoundNormal": 1
        }
    ],
    "TotalCount": 10,
    "RequestId": "..."
}
```

| 字段 | 说明 |
|------|------|
| `InstanceId` | Prometheus 实例 ID，步骤 7.2 的 `--InstanceId` 参数来源 |
| `InstanceStatus` | `2` 表示实例运行中 |
| `BoundTotal` | 已关联集群数，关联前检查是否已达上限 |
| `SpecName` | 实例规格（共享版/基础版/专业版），共享版有集群关联数量限制 |

**实例选择依据**：

*在有多个 Prometheus 实例时，按以下优先级选择：*

- **同 VPC 优先**：选择与目标集群相同 VPC 的实例（通过 `DescribeClusters` 查询集群 `ClusterNetworkSettings.VpcId`），同 VPC 采集延迟最低且无需额外网络配置
- **同地域必须**：Prometheus 实例必须与集群在同一地域（`--region` 参数），跨地域采集数据面不可达
- **实例状态检查**：仅 `InstanceStatus: 2` 的实例可用
- **容量检查**：共享版实例有关联集群数量上限，确认 `BoundTotal` 未超过限制

**步骤 7.2：关联集群与 Prometheus 实例**

> ⚠️ **副作用警告**：
> - 关联后 Prometheus 会开始采集该集群的监控数据，可能产生**指标数据存储费用**（按数据保留时长和采集量计费）
> - 关联操作预计耗时 1-3 分钟，期间集群状态显示为「绑定中」，「绑定中」状态不可执行解绑操作
> - **同一集群不可重复关联同一 Prometheus 实例**，重复关联将返回 `FailedOperation.ResourceExist` 错误

```bash
# 创建关联（将 <InstanceId> 替换为步骤 7.1 输出中的 Prometheus 实例 ID）
tccli monitor CreatePrometheusClusterAgent --region <Region> \
    --InstanceId <InstanceId> \
    --Agents '[{"Region":"<Region>","ClusterId":"<ClusterId>","ClusterType":"tke"}]'
# expected: exit 0，返回 RequestId
```

**`--Agents` 参数字段说明**：

| 字段 | 必填 | 类型 | 说明 | 错误后果 |
|------|:--:|------|------|---------|
| `Region` | 是 | String | 集群所在地域，如 `ap-guangzhou` | 缺少此字段报 `InvalidParameter: Region为必填字段` |
| `ClusterId` | 是 | String | 集群 ID，格式 `cls-xxxxxxxx` | 格式错误或集群不存在报 `InvalidParameter.ClusterId` |
| `ClusterType` | 是 | String | 集群类型，可选 `tke`（标准集群）、`eks`（弹性集群）、`tkeedge`（边缘集群）、`tdcc`（注册集群）、`external`（外部集群） | 类型与集群不匹配导致绑定异常 |
| `EnableExternal` | 否 | Boolean | 是否开启公网 CLB，默认 `false`（仅内网） | 公网端点 CAM 拒绝时不可设为 `true` |

**预期输出**：

```json
{
    "RequestId": "..."
}
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<InstanceId>` | Prometheus 实例 ID | 格式 `prom-xxxxxxxx` | 步骤 7.1 输出的 `Instances[].InstanceId` |
| `<ClusterId>` | 集群 ID | 格式 `cls-xxxxxxxx` | `tccli tke DescribeClusters --region <Region>` |
| `<Region>` | 地域 | 如 `ap-guangzhou` | `tccli configure list` 查看已配置的 region |

**步骤 7.3：查看已关联的集群**

```bash
tccli monitor DescribePrometheusClusterAgents --region <Region> \
    --InstanceId <InstanceId>
# expected: 返回已关联的集群列表，确认 <ClusterId> 在内
```

**预期输出**：

```json
{
    "Agents": [
        {
            "ClusterType": "tke",
            "ClusterId": "cls-example",
            "Status": "normal",
            "ClusterName": "<集群名称>",
            "Region": "ap-guangzhou",
            "VpcId": "vpc-example",
            "EnableExternal": false
        }
    ],
    "Total": 1,
    "RequestId": "..."
}
```

| 字段 | 说明 |
|------|------|
| `Status` | 关联状态：`binding`（绑定中，不可解绑）、`normal`（正常，可解绑）、`abnormal`（异常） |
| `ClusterId` | 已关联的集群 ID，确认目标集群出现在列表中 |
| `EnableExternal` | 是否开启公网 CLB 访问 |

> **控制台补充操作**：关联集群后，登录 [Prometheus 控制台](https://console.cloud.tencent.com/monitor/prometheus)，在「数据采集配置」中新建自定义监控，配置 QosAgent 的 ServiceMonitor 或 PodMonitor 自动发现规则，再导入 Grafana 面板进行可视化。

## 验证

### 控制面（tccli）

```bash
# 1. 验证原生节点池识别
tccli tke DescribeClusterNodePools --ClusterId <ClusterId> \
    --region <Region> \
    | jq '.NodePoolSet[] | select(.Annotations != null) | .NodePoolId'
# expected: 返回原生节点池 ID，非空

# 2. 验证节点实例正常
tccli tke DescribeClusterInstances --ClusterId <ClusterId> \
    --region <Region> \
    | jq '.InstanceSet[] | select(.NodePoolId == "<NodePoolId>") | {InstanceId, LanIP, InstanceState}'
# expected: InstanceState 为 "running"，LanIP 非空
```

| 验证维度 | 命令 | 预期 |
|---------|------|------|
| 原生节点池注解 | `DescribeClusterNodePools` → 检查 `Annotations` | 含 `node.tke.cloud.tencent.com/in-place-upgrade` |
| 节点池生命周期 | `DescribeClusterNodePoolDetail` → `LifeState` | `normal` |
| 节点运行状态 | `DescribeClusterInstances` → `InstanceState` | `running` |
| Prometheus 关联（如已配置） | `DescribePrometheusClusterAgents` | 返回关联的 ClusterId |

### 数据面（kubectl）

> ⚠️ 以下验证需要 kubectl 连通集群。若当前环境不满足网络可达性前提，请从可达环境执行。

```bash
# 3. 验证 QosAgent DaemonSet 运行正常
kubectl get pods -n kube-system -l app=qosagent -o wide
# expected: 每台原生节点上有一个 Running 状态的 QosAgent Pod

# 4. 验证 QosAgent 指标端点可达
kubectl exec -it <QOSAGENT_POD> -n kube-system -- curl -s http://localhost:8084/metrics | head -1
# expected: 返回 Prometheus 指标格式输出（以 # HELP 或指标名开头）
```

## 清理

本页控制面操作（步骤 1-3 的查询命令）不产生需清理的资源。**若在步骤 7 中执行了 Prometheus 关联（`CreatePrometheusClusterAgent`），必须解绑以释放 Prometheus 实例的关联配额**：

```bash
# 解绑集群与 Prometheus 实例
tccli monitor DeletePrometheusClusterAgent --region <Region> \
    --InstanceId <InstanceId> \
    --Agents '[{"ClusterId":"<ClusterId>","ClusterType":"tke"}]'
# ⚠️ 解绑后 Prometheus 将停止采集该集群的监控数据
# expected: exit 0，返回 RequestId
```

> **注意**：
> - 仅解绑 Prometheus 关联，不删除 Prometheus 实例本身。Prometheus 实例的创建/销毁请参见 Prometheus 相关文档
> - 解绑前确认 `Status` 不为 `binding`（通过 `DescribePrometheusClusterAgents` 查看），`binding` 状态下解绑会返回 `FailedOperation.ResourceOperating`，需等待绑定完成后再解绑

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DescribeClusterNodePools` 返回 `InvalidParameter.ClusterId` | 检查 `--ClusterId` 格式是否为 `cls-xxxxxxxx` | ClusterId 格式错误或不属于当前账号/地域 | `tccli tke DescribeClusters --region <Region>` 列出全部集群，确认正确的 ClusterId |
| `DescribeClusterNodePools` 返回的 `NodePoolSet` 中所有节点池均无 `Annotations` 字段 | `tccli tke DescribeClusterNodePools --ClusterId <ClusterId> --region <Region> \| jq '.NodePoolSet[].Annotations'` | 集群中没有原生节点池，所有节点池都是普通类型 | 先创建原生节点池（参照[新建原生节点](../新建原生节点/tccli%20操作.md)），再执行本页监控操作 |
| `CreatePrometheusClusterAgent` 返回 `InvalidParameter: Region为必填字段` | 检查 `--Agents` JSON 数组内每个元素是否包含 `"Region"` 字段 | `Agents` 数组内每个关联集群必须指定 `Region` 字段 | 在 `--Agents` 数组中添加 `"Region":"<Region>"`，如 `'[{"Region":"<Region>","ClusterId":"<ClusterId>","ClusterType":"tke"}]'` |
| `CreatePrometheusClusterAgent` 返回 `FailedOperation.ResourceOperating` | 通过 `DescribePrometheusClusterAgents` 查看 `Status` | 关联的绑定操作仍在进行中（`Status: binding`），此时不可解绑或重新绑定 | 等待绑定完成后重试（通常 1-3 分钟），可通过状态轮询确认 |
| `curl` 直连 `<NodeIP>:8084/metrics` 返回 `Connection timeout / no route to host` | `ping <NodeIP>` 确认网络连通性 | 本地机器不在 VPC 内网 `172.24.0.0` 段，无法直接访问节点端口 | 改用 `kubectl exec` 进入 QosAgent Pod 内部查询 `curl localhost:8084/metrics`；或从同 VPC 跳板机访问（详见步骤 5 选择依据） |
| `kubectl` 返回 `dial tcp: lookup cls-xxx.ccs.tencent-cloud.com: no such host` | `kubectl cluster-info` 确认集群可达性 | 集群仅配内网端点，本地 DNS 无法解析内网域名 | 需要通过 IOA / VPN / 专线 接入 VPC 内网；或在 VPC 内启动 CVM 跳板机后执行 `kubectl` |
| `CreateClusterEndpoint --IsExtranet true` 返回 `InvalidParameter: SecurityGroup can not be empty` | 检查请求参数是否缺少 `--SecurityGroup` | 创建公网端点必须指定安全组 | 补充 `--SecurityGroup <SecurityGroupId>` 参数。安全组可通过 `tccli vpc DescribeSecurityGroups --region <Region>` 查询 |
| `CreateClusterEndpoint --IsExtranet true` 返回 `InvalidParameter.Param: ACTION_NO_AUTH (tke:CreateClusterEndpoint)` | `tccli configure list` 确认当前账号凭据。错误消息含 `strategyId:240463971`、`condition: tke:clusterExtranetEndpoint=true` | 组织级 CAM 策略（strategyId:240463971）以 `tke:clusterExtranetEndpoint=true` 条件硬拒绝当前账号创建公网端点。此为环境限制，非命令错误 | 联系管理员将策略 condition 中 `tke:clusterExtranetEndpoint` 改为 `false` 或添加账号例外；或改用内网端点（`--IsExtranet false`），通过 IOA/VPN/专线 或同 VPC CVM 访问集群 |

### 指标数据异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `node_memory_page_fault_major` 持续增长 | 观察 `node_pressure_total{resource="memory"}` 和 `node_disk_io_time_seconds_total{device="zram0"}` | 内存压缩过于激进，导致「热」页面被频繁换出，引起大量磁盘 IO | 调整节点池的内存压缩配置，降低压缩水位线或关闭部分 Pod 的压缩 |
| `pod_memory_oom_kill` 或 `node_memory_oom_kill` 增加 | 检查 `pod_memory_info` 中各 Pod 的 Inactive Anon 变化趋势 | 压缩后的内存仍不足，触发 OOM Kill | 增加 Pod 内存 Limit 或节点规格；减少单节点 Pod 密度 |
| `node_memory_page_fault_distance`（refault 频率）过高 | 对比 `node_disk_reads_completed_total` 和 `node_disk_writes_completed_total` 趋势 | 大量「热」页面被错误换出后又被换入，压缩策略不适合当前负载 | 提高换出阈值，或在业务低峰期执行压缩 |

## 下一步

- [原生节点概述](../原生节点概述/tccli%20操作.md) — 了解原生节点的功能特性与适用场景
- [新建原生节点](../新建原生节点/tccli%20操作.md) — 通过 CLI 创建支持内存压缩的原生节点
- [腾讯云 Prometheus 监控服务](https://cloud.tencent.com/document/product/1416) — 对接 Prometheus 实现长期监控和告警

## 控制台替代

[TKE 控制台 → 集群 → 节点管理 → 原生节点 → 内存压缩 → 压缩监控](https://console.cloud.tencent.com/tke2/cluster)：在控制台查看 QosAgent 指标图表和监控面板。
