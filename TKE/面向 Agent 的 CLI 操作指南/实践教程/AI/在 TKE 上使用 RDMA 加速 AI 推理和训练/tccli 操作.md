
# 在 TKE 上使用 RDMA 加速 AI 推理和训练（tccli）

> 对照官方：[在 TKE 上使用 RDMA 加速 AI 推理和训练](https://cloud.tencent.com/document/product/457/116720) · page_id `116720`

## 概述

在 TKE 集群中创建 HCC（高性能计算集群）GPU 节点池，利用 RDMA（Remote Direct Memory Access）网络加速分布式 AI 训练和推理。RDMA 绕过 CPU 和操作系统内核，直接在 GPU 显存之间传输数据，大幅降低通信延迟和 CPU 开销。

> **kubectl 数据面不可达**：CAM 拒绝公网端点（strategyId:240463971），内网需 VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

| 维度 | RDMA（RoCEv2） | TCP/IP |
|------|---------------|--------|
| 带宽 | 100-400 Gbps（HCC 节点） | 25-100 Gbps |
| 延迟 | 1-3 us | 10-50 us |
| CPU 开销 | 极低（硬件卸载） | 高（内核协议栈） |
| GPU Direct | 支持（GPU 显存直通网卡） | 需经 CPU 内存中转 |
| 适用场景 | 多机多卡分布式训练（NCCL allreduce）、分布式推理 | 单机多卡、小规模训练 |

**选择依据**：当训练需要跨 ≥2 个 GPU 节点进行 NCCL 集合通信时，RDMA 是必备条件。HCC 节点类型（如 HCCPNV4h、HCCPNV4s、HCCG5v）出厂配备 RDMA 网卡，无需额外配置物理网络。

## 前置条件

- [环境准备](../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置，region 为目标地域

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:DescribeClusters, tke:CreateClusterNodePool, tke:DescribeClusterNodePools
#    tke:DeleteClusterNodePool, tke:DescribeGPUInfo
#    cvm:DescribeInstances, cvm:DescribeInstanceConfigInfos
#    vpc:DescribeVpcs, vpc:DescribeSubnets
# 验证：执行 DescribeClusters 确认权限
tccli tke DescribeClusters --region <Region>
# expected: exit 0，返回集群列表

# 验证 CVM 权限
tccli cvm DescribeInstances --region <Region>
# expected: exit 0，返回 CVM 列表（可为空）
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
# 4. 确认目标 TKE 集群存在且状态 Running
tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
# expected: TotalCount >= 1，ClusterStatus: "Running"，ClusterVersion >= 1.28

# 5. 查询可用 GPU 实例类型 — 确认 HCC RDMA 节点在目标地域可用
tccli tke DescribeGPUInfo --region <Region> --ClusterId CLUSTER_ID
# expected: 返回 GPUInfo 列表，含 InstanceType、GPUInfo 等字段

# 6. 查询 CVM 实例配置 — 确认 HCC 实例族售价与库存
tccli cvm DescribeInstanceConfigInfos --region <Region> \
    --Filters '[{"Name":"instance-family","Values":["HCCPNV4h","HCCPNV4s"]}]'
# expected: 返回实例配置信息，确认规格可用
```

### 版本与规格要求

- K8s 版本：推荐 >= 1.28。TKE 托管集群默认支持 RDMA 设备插件。
- GPU 驱动：HCC 节点预装 NVIDIA 驱动，版本 >= 525.60.13（支持 GPUDirect RDMA）。
- 网络：HCC 节点需在同一 VPC 且同一可用区，确保 RDMA 网络低延迟。

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看集群列表 | `DescribeClusters` | 是 |
| 查看 GPU 实例类型 | `DescribeGPUInfo` | 是 |
| 创建节点池（HCC GPU） | `CreateClusterNodePool` | 否 |
| 查看节点池列表 | `DescribeClusterNodePools` | 是 |
| 删除节点池 | `DeleteClusterNodePool` | 是 |

## 关键字段说明

`CreateClusterNodePool` 创建 HCC RDMA 节点池的主要参数：

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `InstanceType` | String | 是 | HCC RDMA 类型如 `HCCPNV4h`、`HCCPNV4s`、`HCCG5v`。`DescribeGPUInfo` 查询可用类型 | 填非 HCC 实例类型 → 节点无 RDMA 网卡，无法使用 RDMA 通信 |
| `GPUConfig` | Object | 是 | 含 `GPUType`（如 `nvidia-tesla-h800`）、`GPUCount`（如 `8`）、`GPUDriverVersion`（如 `525.60.13`） | GPU 类型与节点不匹配 → `InvalidParameter.GPUConfig` |
| `DesiredNodesNum` | Integer | 是 | >= 2（RDMA 多节点通信至少需要 2 个节点）。建议训练用 2^n 个节点 | 只创建 1 个节点 → 无法验证 RDMA 跨节点通信 |
| `LaunchConfigurePara` | String | 是 | CVM 创建参数 JSON（含 VPC、子网、安全组、系统盘等）。格式与 `RunInstances` 一致 | JSON 格式错误 → `InvalidParameter.LaunchConfigurePara` |
| `SubnetIds` | Array | 是 | 同可用区子网列表。RDMA 网络要求节点在同一可用区 | 跨可用区 → RDMA 延迟增加，可能无法建立 RoCE 连接 |
| `Labels` | Array | 否 | 建议添加 `rdma-capable: "true"` 标签，便于调度 | 不加标签 → 无法通过 nodeSelector/affinity 精确调度 RDMA 工作负载 |

## 操作步骤

### 步骤 1：创建 HCC GPU 节点池（含 RDMA 网卡）

#### 选择依据

- **实例类型**：选择 `HCCPNV4h`（搭载 H800 GPU + 400G RoCE RDMA）。HCC 系列是腾讯云高性能计算集群专用实例，出厂预配 RDMA 网卡和 GPU Direct RDMA 驱动。`HCCPNV4s`（H100）和 `HCCG5v`（A800）同样支持 RDMA。通过 `DescribeGPUInfo` 查询目标地域可用类型。
- **节点数量**：创建 2 个节点。RDMA 的价值体现在跨节点通信，至少 2 个节点才能验证 NCCL allreduce。生产训练建议 4-8 节点（2^n 拓扑最优）。
- **网络**：选择与集群相同的 VPC 和子网，确保节点在同一可用区。RDMA RoCE 网络依赖底层网络就近通信，跨可用区会引入额外延迟。
- **标签**：添加 `rdma-capable: "true"` 标签，Workload 通过 nodeSelector 精准调度到 RDMA 节点。

#### 最小创建（2 节点 HCC GPU 池）

`hcc-rdma-nodepool-minimal.json`：

```json
{
  "ClusterId": "CLUSTER_ID",
  "Name": "hcc-rdma-pool",
  "InstanceType": "HCCPNV4h",
  "GPUConfig": {
    "GPUType": "nvidia-tesla-h800",
    "GPUCount": 8,
    "InstanceType": "HCCPNV4h"
  },
  "DesiredNodesNum": 2,
  "LaunchConfigurePara": "{\"InstanceChargeType\":\"POSTPAID_BY_HOUR\",\"Placement\":{\"Zone\":\"ap-guangzhou-6\"},\"VirtualPrivateCloud\":{\"VpcId\":\"VPC_ID\",\"SubnetId\":\"SUBNET_ID\"},\"SystemDisk\":{\"DiskType\":\"CLOUD_SSD\",\"DiskSize\":100}}",
  "Labels": [
    {"Name": "rdma-capable", "Value": "true"}
  ]
}
```

```bash
tccli tke CreateClusterNodePool --region <Region> \
    --cli-input-json file://hcc-rdma-nodepool-minimal.json
# expected: exit 0，返回 NodePoolId
```

**预期输出**：

```json
{
    "NodePoolId": "np-example",
    "RequestId": "..."
}
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `CLUSTER_ID` | TKE 集群 ID | 格式 `cls-xxxxxxxx`，状态 Running | `tccli tke DescribeClusters --region <Region>` |
| `VPC_ID` | VPC 实例 ID | HCC 节点需与集群同 VPC、同可用区 | `tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'` 查看 `ClusterNetworkSettings.VpcId` |
| `SUBNET_ID` | 子网 ID | 需有足够可用 IP（>= 2） | `tccli vpc DescribeSubnets --region <Region>` |
| `REGION` | 地域 | 如 `ap-guangzhou` | `tccli tke DescribeRegions` |

#### 增强配置（加数据盘、SSH 密钥、RDMA 标签注释）

`hcc-rdma-nodepool-enhanced.json`：

```json
{
  "ClusterId": "CLUSTER_ID",
  "Name": "hcc-rdma-pool-prod",
  "InstanceType": "HCCPNV4h",
  "GPUConfig": {
    "GPUType": "nvidia-tesla-h800",
    "GPUCount": 8,
    "InstanceType": "HCCPNV4h",
    "GPUDriverVersion": "525.60.13",
    "CUDNNVersion": "8.9",
    "CUDAPlatform": "TENCENTOS"
  },
  "DesiredNodesNum": 4,
  "LaunchConfigurePara": "{\"InstanceChargeType\":\"POSTPAID_BY_HOUR\",\"Placement\":{\"Zone\":\"ap-guangzhou-6\"},\"VirtualPrivateCloud\":{\"VpcId\":\"VPC_ID\",\"SubnetId\":\"SUBNET_ID\"},\"SystemDisk\":{\"DiskType\":\"CLOUD_SSD\",\"DiskSize\":100},\"DataDisks\":[{\"DiskType\":\"CLOUD_SSD\",\"DiskSize\":500}],\"LoginSettings\":{\"KeyIds\":[\"KEY_ID\"]}}",
  "Labels": [
    {"Name": "rdma-capable", "Value": "true"},
    {"Name": "gpu-type", "Value": "h800"},
    {"Name": "workload", "Value": "distributed-training"}
  ],
  "Taints": [
    {"Key": "rdma-capable", "Value": "true", "Effect": "NoSchedule"}
  ]
}
```

### 步骤 2：等待节点池就绪

节点池创建是异步操作，等待节点状态变为 `Running`：

```bash
tccli tke DescribeClusterNodePools --region <Region> \
    --ClusterId CLUSTER_ID
# expected: LifeState: "normal"，DesiredNodesNum 与 NodesNum 一致
```

**预期输出**：

```json
{
    "NodePoolSet": [
        {
            "NodePoolId": "np-example",
            "Name": "hcc-rdma-pool",
            "ClusterId": "CLUSTER_ID",
            "LifeState": "normal",
            "DesiredNodesNum": 2,
            "NodeCountSummary": {
                "AutoscalingAdded": {
                    "Total": 2,
                    "Normal": 2
                }
            }
        }
    ],
    "TotalCount": 1,
    "RequestId": "..."
}
```

确认节点在集群中 Ready：

```bash
kubectl get nodes -l rdma-capable=true
# expected: 2 个节点，STATUS: Ready，ROLES 含 <none> 或 worker
```

**预期输出**：

```text
NAME                STATUS   ROLES    AGE   VERSION
10.0.1.15           Ready    <none>   2m    v1.28.3-tke.5
10.0.1.16           Ready    <none>   2m    v1.28.3-tke.5
```

### 步骤 3：部署 RDMA 设备插件

#### 选择依据

TKE 集群需要部署 NVIDIA 设备插件和 RDMA 设备插件，使 K8s 能够识别和管理 GPU 及 RDMA 网卡资源。NVIDIA 设备插件将 GPU 注册为 `nvidia.com/gpu` 可调度资源；RDMA 设备插件将 RDMA 网卡注册为 `tke.cloud.tencent.com/rdma-hca` 资源，确保 NCCL 通信时节点具有可用 RDMA 设备。

`nvidia-device-plugin.yaml`：

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: nvidia-device-plugin-daemonset
  namespace: kube-system
spec:
  selector:
    matchLabels:
      name: nvidia-device-plugin-ds
  template:
    metadata:
      labels:
        name: nvidia-device-plugin-ds
    spec:
      tolerations:
      - key: rdma-capable
        operator: Exists
        effect: NoSchedule
      containers:
      - name: nvidia-device-plugin-ctr
        image: nvidia/k8s-device-plugin:v0.15.0
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop: ["ALL"]
        volumeMounts:
        - name: device-plugin
          mountPath: /var/lib/kubelet/device-plugins
      volumes:
      - name: device-plugin
        hostPath:
          path: /var/lib/kubelet/device-plugins
```

`rdma-device-plugin.yaml`：

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: rdma-device-plugin
  namespace: kube-system
spec:
  selector:
    matchLabels:
      name: rdma-device-plugin
  template:
    metadata:
      labels:
        name: rdma-device-plugin
    spec:
      nodeSelector:
        rdma-capable: "true"
      containers:
      - name: rdma-device-plugin
        image: ccr.ccs.tencentyun.com/tkeimages/rdma-device-plugin:v1.0.0
        imagePullPolicy: Always
        securityContext:
          privileged: true
        volumeMounts:
        - name: device-plugin
          mountPath: /var/lib/kubelet/device-plugins
        - name: dev
          mountPath: /dev
        - name: sys
          mountPath: /sys
      volumes:
      - name: device-plugin
        hostPath:
          path: /var/lib/kubelet/device-plugins
      - name: dev
        hostPath:
          path: /dev
      - name: sys
        hostPath:
          path: /sys
```

```bash
kubectl apply -f nvidia-device-plugin.yaml
# expected: daemonset.apps/nvidia-device-plugin-daemonset created

kubectl apply -f rdma-device-plugin.yaml
# expected: daemonset.apps/rdma-device-plugin created
```

验证设备插件就绪：

```bash
kubectl get pods -n kube-system -l name=nvidia-device-plugin-ds
# expected: 2 个 Pod Running（每个 GPU 节点一个）

kubectl get pods -n kube-system -l name=rdma-device-plugin
# expected: 2 个 Pod Running（每个 RDMA 节点一个）
```

```text
NAME  STATUS  AGE
...
```

### 步骤 4：验证 RDMA 设备

#### 选择依据

在运行 NCCL 测试之前，需确认每个节点的 RDMA 网卡设备已正确识别。`ibstat` 命令显示 InfiniBand/RoCE 设备状态，`ibv_devinfo` 显示详细设备信息。HCCPNV4h 节点通常有 8 个 RDMA 设备（每个 GPU 对应一个网卡）。

```bash
# 在第一个 RDMA 节点上检查 RDMA 设备
kubectl exec -it $(kubectl get pod -n kube-system -l name=rdma-device-plugin -o jsonpath='{.items[0].metadata.name}') -n kube-system -- ibstat
# expected: 返回 8 个 mlx5_* 设备，LinkUp 状态
```

**预期输出**：

```text
CA 'mlx5_0'
	CA type: MT4129
	Number of ports: 1
	Firmware version: 22.35.2000
	Hardware version: 0
	Node GUID: 0x0000...
	Port 1:
		State: Active
		Physical state: LinkUp
		Rate: 400
		Base lid: 0
...
```

查看详细设备信息：

```bash
kubectl exec -it $(kubectl get pod -n kube-system -l name=rdma-device-plugin -o jsonpath='{.items[0].metadata.name}') -n kube-system -- ibv_devinfo
# expected: 列出 8 个 mlx5_0 ~ mlx5_7 设备，transport: InfiniBand
```

```text
NAME  STATUS  AGE
...
```

查看节点上的 RDMA 资源注册：

```bash
kubectl describe node $(kubectl get nodes -l rdma-capable=true -o jsonpath='{.items[0].metadata.name}') | grep rdma
# expected: tke.cloud.tencent.com/rdma-hca: 8
```

### 步骤 5：运行 NCCL AllReduce 带宽测试

#### 选择依据

NCCL（NVIDIA Collective Communications Library）是 GPU 间集合通信的标准库。`nccl-tests` 提供 `all_reduce_perf` 测试工具，可测量 RDMA 网络下的 GPU 间通信带宽。测试需要 NCCL 环境变量配置 RDMA 传输协议（`NCCL_PROTO=Simple`）和 RoCE 接口（`NCCL_IB_HCA` 指定 RDMA 设备）。

`nccl-test-job.yaml`：

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: nccl-allreduce-test
spec:
  parallelism: 2
  completions: 2
  template:
    metadata:
      labels:
        app: nccl-test
    spec:
      nodeSelector:
        rdma-capable: "true"
      containers:
      - name: nccl-test
        image: nvcr.io/nvidia/k8s/nccl-test:v2.22.3
        command:
        - mpirun
        - --allow-run-as-root
        - -np
        - "16"
        - -H
        - "nccl-allreduce-test-0.nccl-test-svc:8,nccl-allreduce-test-1.nccl-test-svc:8"
        - -mca
        - btl_tcp_if_include
        - eth0
        - /workspace/all_reduce_perf
        - -b
        - "8M"
        - -e
        - "128M"
        - -f
        - "2"
        - -g
        - "1"
        - -n
        - "20"
        env:
        - name: NCCL_DEBUG
          value: "INFO"
        - name: NCCL_PROTO
          value: "Simple"
        - name: NCCL_IB_DISABLE
          value: "0"
        - name: NCCL_IB_HCA
          value: "mlx5"
        - name: NCCL_SOCKET_IFNAME
          value: "eth0"
        - name: NCCL_IB_GID_INDEX
          value: "3"
        - name: NCCL_IB_TIMEOUT
          value: "24"
        - name: NCCL_IB_QPS_PER_CONNECTION
          value: "4"
        resources:
          limits:
            nvidia.com/gpu: 8
            tke.cloud.tencent.com/rdma-hca: 8
      restartPolicy: Never
---
apiVersion: v1
kind: Service
metadata:
  name: nccl-test-svc
spec:
  clusterIP: None
  selector:
    app: nccl-test
```

```bash
kubectl apply -f nccl-test-job.yaml
# expected: job.batch/nccl-allreduce-test created
#          service/nccl-test-svc created
```

等待 Job 完成：

```bash
kubectl wait --for=condition=complete job/nccl-allreduce-test --timeout=300s
# expected: job.batch/nccl-allreduce-test condition met
```

### 步骤 6：验证 RDMA 带宽

```bash
kubectl logs job/nccl-allreduce-test | tail -30
# expected: 输出 NCCL 初始化日志 + allreduce 带宽测试结果，out-of-place 带宽 >= 300 GB/s（8 卡 × 400 Gbps / 8 ≈ 400 GB/s 理论带宽，实际 >= 75%）
```

**预期输出**：

```text
# nThread 1 nGpus 1 minBytes 8388608 maxBytes 134217728 step: 2(factor) warmup iters: 5 iters: 20 agg iters: 1 validation: 1 graph: 0
#
# Using devices
#   Rank  0 Pid ... on nccl-allreduce-test-0 device  0 [0x10] NVIDIA H800
#   ...
#       size         count      type   redop    root     time   algbw   busbw  #wrong     time   algbw   busbw  #wrong
#         8M            20     float    sum      -1    0.123  68.10  68.10  ...    0.045 186.33 186.33  ...
#        16M            20     float    sum      -1    0.156 107.52 107.52  ...    0.057 294.40 294.40  ...
#        32M            20     float    sum      -1    0.200 167.77 167.77  ...    0.080 419.43 419.43  ...
#        64M            20     float    sum      -1    0.290 231.42 231.42  ...    0.120 559.24 559.24  ...
#       128M            20     float    sum      -1    0.450 298.28 298.28  ...    0.190 706.77 706.77  ...
# Out of bounds values : 0 OK
# Avg bus bandwidth    : 370.42
```

| 指标 | 期望值 | 说明 |
|------|--------|------|
| `busbw`（bus bandwidth）| >= 300 GB/s（128M 级别）| 跨节点 GPU 间通信带宽 |
| `#wrong` | 0 | 无数据校验错误 |
| `Out of bounds` | 0 | 无异常值 |

## 验证

### 控制面（tccli）

确认节点池状态正常，RDMA 节点均已就绪：

```bash
tccli tke DescribeClusterNodePools --region <Region> \
    --ClusterId CLUSTER_ID \
    | jq '.NodePoolSet[] | {Name, LifeState, DesiredNodesNum, NodeCountSummary}'
# expected: LifeState: "normal"，DesiredNodesNum == 实际可用节点数
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

确认节点资源包含 RDMA 网卡：

```bash
tccli tke DescribeClusterNodePools --region <Region> \
    --ClusterId CLUSTER_ID \
    | jq '.NodePoolSet[0]'
# expected: InstanceType 为 HCCPNV4h，GPUConfig.GPUType 为 nvidia-tesla-h800
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

### 数据面（需 VPN/IOA）

```bash
# 验证 GPU 资源可调度
kubectl describe node $(kubectl get nodes -l rdma-capable=true -o jsonpath='{.items[0].metadata.name}') \
    | grep -A 5 "Allocatable:"
# expected: nvidia.com/gpu: 8, tke.cloud.tencent.com/rdma-hca: 8

# 验证 RDMA 设备插件健康
kubectl get pods -n kube-system -l name=rdma-device-plugin
# expected: 2 个 Pod，STATUS: Running

# 验证 NCCL 测试无错误
kubectl logs job/nccl-allreduce-test | grep -E "Out of bounds|#wrong"
# expected: "Out of bounds values : 0 OK", #wrong 列为 0
```

```text
NAME  STATUS  AGE
...
```

| 验证维度 | 命令 | 预期 |
|---------|------|------|
| 节点池生命周期 | `DescribeClusterNodePools` | `LifeState: "normal"` |
| 节点 Ready | `kubectl get nodes -l rdma-capable=true` | STATUS: Ready |
| GPU 资源注册 | `kubectl describe node \| grep nvidia.com/gpu` | `nvidia.com/gpu: 8` |
| RDMA 资源注册 | `kubectl describe node \| grep rdma-hca` | `tke.cloud.tencent.com/rdma-hca: 8` |
| NCCL 带宽 | `kubectl logs job/nccl-allreduce-test` | `Avg bus bandwidth >= 300 GB/s` |

## 清理

> **计费警告**：HCC GPU 节点按量计费，即使不运行任务也会持续产生费用。HCCPNV4h × 8 GPU 实例费用较高，建议验证完成后及时删除。
> **副作用警告**：`DeleteClusterNodePool` 会删除节点池内所有 CVM 实例、系统盘和数据盘，不可恢复。

### 数据面（需 VPN/IOA）

```bash
# 1. 删除 NCCL 测试 Job 和 Service
kubectl delete job nccl-allreduce-test
# expected: job.batch "nccl-allreduce-test" deleted

kubectl delete svc nccl-test-svc
# expected: service "nccl-test-svc" deleted

# 2. 删除 RDMA 设备插件
kubectl delete daemonset rdma-device-plugin -n kube-system
# expected: daemonset.apps "rdma-device-plugin" deleted

# 3. 删除 NVIDIA 设备插件
kubectl delete daemonset nvidia-device-plugin-daemonset -n kube-system
# expected: daemonset.apps "nvidia-device-plugin-daemonset" deleted
```

### 控制面（tccli）

```bash
# 4. 清理前状态检查
tccli tke DescribeClusterNodePools --region <Region> \
    --ClusterId CLUSTER_ID \
    | jq '.NodePoolSet[] | {NodePoolId, Name, DesiredNodesNum}'
# 确认是待删除的目标节点池

# 5. 删除节点池（级联删除所有 CVM 节点）
tccli tke DeleteClusterNodePool --region <Region> \
    --ClusterId CLUSTER_ID \
    --NodePoolIds '["NP_ID"]'
# expected: exit 0
# 此操作将删除节点池内所有 CVM 实例及关联磁盘
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

验证已删除：

```bash
tccli tke DescribeClusterNodePools --region <Region> \
    --ClusterId CLUSTER_ID
# expected: NodePoolSet 中不再包含已删除的节点池
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

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `CreateClusterNodePool` 返回 `InvalidParameter.InstanceType` | `tccli cvm DescribeInstanceConfigInfos --region <Region> --Filters '[{"Name":"instance-family","Values":["HCCPNV4h"]}]'` 查询实例可用性 | 实例类型在该地域不可用或已售罄 | 通过 `DescribeGPUInfo` 查询可用 HCC 实例类型，更换地域或备选实例（如 `HCCPNV4s`） |
| `CreateClusterNodePool` 返回 `InvalidParameter.GPUConfig` | 检查请求 JSON 中 `GPUConfig.GPUType` 是否匹配 `InstanceType` | GPU 类型与实例类型不匹配 | 按对应关系填写：HCCPNV4h → `nvidia-tesla-h800`，HCCPNV4s → `nvidia-tesla-h100` |
| `CreateClusterNodePool` 返回 `InvalidParameter.LaunchConfigurePara` | 将 `LaunchConfigurePara` 字段 decode 检查 JSON 结构 | CVM 创建参数 JSON 格式错误或缺少必填项 | 确认 JSON 含 `InstanceChargeType`、`Placement.Zone`、`VirtualPrivateCloud`、`SystemDisk`。用 `jq` 验证 JSON 有效性 |
| `DescribeGPUInfo` 返回空列表 | `tccli tke DescribeGPUInfo --region <Region> --ClusterId CLUSTER_ID` 检查输出 | 该地域无 GPU 实例或集群不支持 GPU 节点 | 切换至支持 GPU 的地域（如 `ap-guangzhou-6`），或检查集群是否为托管集群 |
| `CreateClusterNodePool` 返回 `LimitExceeded` | `tccli cvm DescribeAccountQuota --region <Region>` 查看配额 | GPU 实例配额不足（此为环境限制，非命令错误） | [申请配额提升](https://console.cloud.tencent.com/cvm/quota) 或清理不再使用的 GPU 节点 |

### 节点池创建成功但 RDMA 不可用

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 节点 Ready 但 RDMA 设备插件不工作 | `kubectl logs -n kube-system -l name=rdma-device-plugin` 查看插件日志 | 节点未安装 RDMA 驱动或驱动版本不兼容 | `kubectl exec` 进入节点执行 `ibstat` 检查 RDMA 设备；若设备不存在则确认实例类型是否为 HCC 系列 |
| `kubectl describe node` 无 `tke.cloud.tencent.com/rdma-hca` 资源 | `kubectl get pods -n kube-system -l name=rdma-device-plugin` 查看 Pod 状态 | RDMA 设备插件未部署到该节点或 Pod CrashLoopBackOff | 检查 nodeSelector 标签 `rdma-capable: "true"` 是否打在该节点上 |
| NCCL 测试中 `#wrong` 列非零 | `kubectl logs job/nccl-allreduce-test \| grep NCCL` 查看 NCCL 初始化日志 | RDMA 链路错误或 GPU 显存异常 | 检查 NCCL_IB_HCA 环境变量是否正确（应为 `mlx5`）；重新运行测试，若持续出错联系售后 |
| NCCL 带宽远低于预期（< 100 GB/s） | `kubectl logs job/nccl-allreduce-test` 查看通信协议，检查是否显示 `[send] NET/IB` | NCCL 未使用 RDMA IB 传输，退化为 TCP | 确认 `NCCL_IB_DISABLE: "0"`、`NCCL_PROTO: "Simple"`；检查节点间是否在同一可用区 |
| 2 个节点 NCCL 测试一直 pending | `kubectl describe pod -l app=nccl-test` 查看事件 | RDMA 资源请求超过节点可分配量 | 每个节点请求 `tke.cloud.tencent.com/rdma-hca: 8`，确认节点 allocatable >= 8；资源不足则减少 `parallelism` 或增加节点 |

## 下一步

- [在 TKE 上使用 AIBrix 进行多节点分布式推理](https://cloud.tencent.com/document/product/457/117827) — 基于 RDMA 网络的分布式推理框架
- [使用 TKE 部署 AI 大模型](https://cloud.tencent.com/document/product/457/103983) — GPU 大模型部署完整方案
- [GPU 资源管理](https://cloud.tencent.com/document/product/457/32205) — GPU 共享、GPU 调度策略
- [NCCL 官方文档](https://docs.nvidia.com/deeplearning/nccl/user-guide/docs/) — NCCL 环境变量与性能调优
- [TKE 节点池管理](https://cloud.tencent.com/document/product/457/32204) — 节点池创建、扩缩容、删除

## 控制台替代

[TKE 控制台 → 节点管理 → 节点池](https://console.cloud.tencent.com/tke2/nodepool)：新建节点池，选择「高性能计算集群 HCC」实例族，配置 GPU 参数。
