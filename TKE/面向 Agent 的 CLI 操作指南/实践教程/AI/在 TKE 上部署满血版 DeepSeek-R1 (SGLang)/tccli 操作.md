# 在 TKE 上部署满血版 DeepSeek-R1（SGLang）（tccli）

> 对照官方：[在 TKE 上部署满血版 DeepSeek-R1（SGLang）](https://cloud.tencent.com/document/product/457/116858) · page_id `116858`

## 概述

在 TKE 集群上使用 SGLang 部署 DeepSeek-R1 671B 全量模型（FP8 量化）。DeepSeek-R1 在 FP16 下约 1.3TB，FP8 下约 700GB，需至少 8 张 H20（96GB）或 8 张 A100-80GB 显卡才能完整加载。控制面通过 tccli 创建 8-GPU 节点池，数据面通过 kubectl 部署 SGLang 推理服务。

> **kubectl 数据面不可达**：CAM 拒绝公网端点（strategyId:240463971），内网需 VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

> **注意**：DeepSeek-R1 在 FP8 下约 700GB，需 8+ GPU（每卡 ≥ 80GB 显存）。单卡或单机 GPU 数不足的集群无法部署完整模型。如 GPU 资源不足，考虑使用 DeepSeek-R1 的蒸馏版本（如 DeepSeek-R1-Distill-Qwen-32B）。

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
# 验证：执行 DescribeClusters 确认权限
tccli tke DescribeClusters --region <Region>
# expected: exit 0，返回集群列表（可为空）
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
# 4. 确认目标 TKE 集群存在且状态正常，至少 K8s 1.28
tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterStatus 为 Running，ClusterVersion ≥ 1.28

# 5. 查询可用 GPU 机型（需 8 卡 A100/H20）
tccli tke DescribeGPUInfo --region <Region> --Zone ZONE
# expected: 返回含 8 卡 A100 或 H20 的 GPU 机型

# 6. 查询 VPC 子网资源（需要 ≥ 8+ 个可用 IP）
tccli vpc DescribeSubnets --region <Region> --Filters '[{"Name":"vpc-id","Values":["VPC_ID"]}]'
# expected: AvailableIpCount ≥ 10
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

### 版本与规格选择

- GPU 型号：H20（96GB，推荐）或 A100-80GB。最低要求 8x80GB = 640GB 可用显存
- GPU 驱动：H20/A100 推荐 `535.161.08`（CUDA 12.2+）
- 系统盘：≥ 200GB SSD（OS + Docker 镜像）
- 数据盘：≥ 1TB（模型权重约 700GB + 缓存空间）
- K8s 版本：推荐 ≥ 1.28

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 查看集群列表 | `tccli tke DescribeClusters --region ap-guangzhou` | 是 |
| 创建 GPU 节点池 | `tccli tke CreateClusterNodePool --region ap-guangzhou` | 否 |
| 查看 GPU 信息 | `tccli tke DescribeGPUInfo --region ap-guangzhou` | 是 |

## 关键字段说明

以下说明 `CreateClusterNodePool` 中与 8-GPU 节点池相关的主要参数。

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `ClusterId` | String | 是 | 目标托管集群 ID | 集群不存在或非托管 → `InvalidParameter.ClusterId` |
| `Name` | String | 是 | 节点池名称，如 `deepseek-r1-8gpu-pool` | — |
| `AutoScalingGroupPara.InstanceType` | String | 是 | 8 卡机型，如 `GNV4.24XLARGE352`（H20 8 卡）。`tccli tke DescribeGPUInfo` 查询 | 机型不支持或库存不足 → `ResourceInsufficient` |
| `AutoScalingGroupPara.GPUConfig.GPUType` | String | 是 | `H20` 或 `A100`。需至少 8 卡机型 | 填 T4/V100 等小显存 GPU 无法加载 700GB 模型 |
| `AutoScalingGroupPara.GPUConfig.GPUCount` | Integer | 是 | `8`，必须为 8 卡（模型需 8 张 80GB+ GPU） | <8 → 模型加载 OOM |
| `AutoScalingGroupPara.GPUConfig.GPUDriverVersion` | String | 是 | `535.161.08`（推荐，CUDA 12.2+） | 驱动与 SGLang CUDA 版本不兼容 → 启动失败 |
| `AutoScalingGroupPara.DesiredNodesNum` | Integer | 是 | ≥ 1。1 个 8 卡节点即够部署 | 为 0 → 无可用节点 |
| `AutoScalingGroupPara.DataDisks` | Array | 是 | 至少 1 块 ≥ 1TB 数据盘存放模型权重 | 空间不足 → 模型下载失败 |
| `AutoScalingGroupPara.SystemDisk.DiskSize` | Integer | 是 | ≥ 200 GB | 系统盘满 → 镜像拉取失败 |

## 操作步骤

### 步骤 1：创建 8-GPU 节点池

#### 选择依据

*为什么选这个值而不是其他：*

- **SGLang vs vLLM**：DeepSeek-R1 使用 MoE（Mixture of Experts）架构，SGLang 对 MoE 模型的原生支持更优（RadixAttention 对稀疏激活有专门优化），且 SGLang 的 `--tp` 配置与 DeepSeek-R1 的 expert parallelism 更匹配。
- **GPU 数量 8**：FP8 量化的 DeepSeek-R1 约 700GB。H20 单卡 96GB，8 卡共 768GB，加载模型后仅余约 68GB 用做 KV Cache —— 刚好够用。A100-80GB 8 卡共 640GB，无法加载完整 700GB 模型，需要用 `--cpu-offload-gb` 参数卸载部分层到 CPU 内存（性能显著下降）。因此 **强烈推荐 H20**。
- **数据盘 ≥ 1TB**：模型权重约 700GB（safetensors 格式），还需额外空间存放 shard 文件和 Docker 镜像层。推荐 1.5TB 以上。
- **节点数**：1 个 8 卡节点即可部署完整模型。如需高可用可部署 2 个节点，但需注意每节点独立加载完整模型显存（无跨节点共享）。

查询 GPU 库存确认 8 卡机型可用：

```bash
tccli tke DescribeGPUInfo --region <Region> --Zone ZONE
# expected: 返回 GPU 类型列表，关注 GPUCount=8 的机型
```

#### 创建节点池

`deepseek-nodepool.json`：

```json
{
  "ClusterId": "CLUSTER_ID",
  "Name": "deepseek-r1-8gpu-pool",
  "AutoScalingGroupPara": {
    "AutoScalingGroupName": "deepseek-r1-8gpu-asg",
    "InstanceType": "GNV4.24XLARGE352",
    "GPUConfig": {
      "GPUType": "H20",
      "GPUCount": 8,
      "GPUDriverVersion": "535.161.08",
      "CUDAVersion": "12.2",
      "CUDNNVersion": "8.9.7"
    },
    "DesiredNodesNum": 1,
    "MinNodesNum": 1,
    "MaxNodesNum": 2,
    "VpcId": "VPC_ID",
    "SubnetIds": ["SUBNET_ID"],
    "SystemDisk": {
      "DiskType": "CLOUD_SSD",
      "DiskSize": 200
    },
    "DataDisks": [
      {
        "DiskType": "CLOUD_PREMIUM",
        "DiskSize": 1500
      }
    ],
    "InternetAccessible": {
      "InternetMaxBandwidthOut": 100,
      "PublicIpAssigned": true
    },
    "SecurityGroupIds": ["SECURITY_GROUP_ID"],
    "LoginSettings": {
      "KeyIds": ["KEY_ID"]
    },
    "Tags": [
      {"Key": "workload", "Value": "deepseek-r1"},
      {"Key": "gpu-type", "Value": "H20-8GPU"}
    ]
  },
  "Labels": [
    {"Name": "nvidia.com/gpu.product", "Value": "NVIDIA-H20"},
    {"Name": "nvidia.com/gpu.count", "Value": "8"},
    {"Name": "workload-type", "Value": "deepseek-r1-inference"}
  ],
  "Taints": [
    {
      "Key": "nvidia.com/gpu",
      "Value": "true",
      "Effect": "NoSchedule"
    }
  ]
}
```

```bash
tccli tke CreateClusterNodePool --region <Region> --cli-input-json file://deepseek-nodepool.json
# expected: exit 0，返回 NodePoolId
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `CLUSTER_ID` | TKE 集群 ID | 格式 `cls-xxxxxxxx` | `tccli tke DescribeClusters --region <Region>` |
| `VPC_ID` | VPC 实例 ID | 与集群同 VPC | `tccli vpc DescribeVpcs --region <Region>` |
| `SUBNET_ID` | 子网 ID | AvailableIpCount ≥ 10 | `tccli vpc DescribeSubnets --region <Region>` |
| `SECURITY_GROUP_ID` | 安全组 ID | 放行 30000 端口（SGLang 默认） | `tccli vpc DescribeSecurityGroups --region <Region>` |
| `REGION` | 地域 | 如 `ap-guangzhou` | `tccli configure list` |

### 步骤 2：轮询节点池就绪

```bash
tccli tke DescribeClusterNodePools --region <Region> \
    --ClusterId CLUSTER_ID
# expected: 节点池 LifeState 为 "normal"，NodeCount 为 1，Running 为 1
```

**预期输出**：

```json
{
    "NodePoolSet": [
        {
            "NodePoolId": "np-example",
            "Name": "deepseek-r1-8gpu-pool",
            "ClusterId": "cls-example",
            "LifeState": "normal",
            "NodeCountSummary": {
                "AutoscalingAdded": {
                    "Total": 1,
                    "Running": 1
                }
            }
        }
    ],
    "RequestId": "..."
}
```

### 步骤 3：创建 PVC 存储模型权重（kubectl，数据面）

> 以下 kubectl 命令需在内网/VPN 环境下执行。

`model-storage-pvc.yaml`：

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: deepseek-r1-model
  namespace: NAMESPACE
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: cbs-premium
  resources:
    requests:
      storage: 1500Gi
```

```bash
kubectl apply -f model-storage-pvc.yaml
# expected: persistentvolumeclaim/deepseek-r1-model created
```

### 步骤 4：下载模型权重（initContainer 模式）

`model-download-job.yaml`：

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: download-deepseek-r1
  namespace: NAMESPACE
spec:
  ttlSecondsAfterFinished: 86400
  template:
    spec:
      restartPolicy: OnFailure
      containers:
      - name: model-downloader
        image: huggingface/transformers-pytorch-gpu:latest
        command:
        - /bin/bash
        - -c
        - |
          pip install -U "huggingface_hub[cli]"
          export HF_HUB_ENABLE_HF_TRANSFER=1
          huggingface-cli download deepseek-ai/DeepSeek-R1 \
            --local-dir /models/DeepSeek-R1 \
            --local-dir-use-symlinks False \
            --resume-download
        env:
        - name: HF_ENDPOINT
          value: https://hf-mirror.com
        - name: HF_HOME
          value: /models/.cache
        volumeMounts:
        - name: model-storage
          mountPath: /models
        resources:
          requests:
            cpu: 8
            memory: 32Gi
          limits:
            cpu: 16
            memory: 64Gi
      volumes:
      - name: model-storage
        persistentVolumeClaim:
          claimName: deepseek-r1-model
      tolerations:
      - key: nvidia.com/gpu
        operator: Exists
        effect: NoSchedule
      nodeSelector:
        nvidia.com/gpu.product: "NVIDIA-H20"
```

```bash
kubectl apply -f model-download-job.yaml
# expected: job.batch/download-deepseek-r1 created

# 监控下载进度
kubectl logs -n NAMESPACE job/download-deepseek-r1 --follow
# expected: 显示下载进度，约 700GB 需 30-60 分钟
```

### 步骤 5：部署 SGLang 推理服务

#### 选择依据

- **`--tp 8`**：Tensor parallelism 设为 8，将模型切分到 8 张 GPU 上。SGLang + DeepSeek-R1 MoE 架构在此配置下能充分利用各 GPU 的显存和算力。
- **`--mem-fraction-static 0.85`**：为模型权重预留 85% 显存，余 15% 给 KV Cache。如遇 OOM 可降至 0.80。
- **`--context-length 131072`**：DeepSeek-R1 原生上下文窗口为 128K，设置稍大以留余量。
- **`--max-running-requests 32`**：控制并发推理请求数。8 卡 H20 可适度提升此值，但需关注显存。

`deepseek-sglang-deploy.yaml`：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deepseek-r1-sglang
  namespace: NAMESPACE
  labels:
    app: deepseek-r1-sglang
spec:
  replicas: 1
  selector:
    matchLabels:
      app: deepseek-r1-sglang
  template:
    metadata:
      labels:
        app: deepseek-r1-sglang
    spec:
      hostNetwork: true
      dnsPolicy: ClusterFirstWithHostNet
      containers:
      - name: sglang
        image: lmsysorg/sglang:latest
        command:
        - python3
        - -m
        - sglang.launch_server
        args:
        - --model-path
        - /models/DeepSeek-R1
        - --host
        - "0.0.0.0"
        - --port
        - "30000"
        - --tp
        - "8"
        - --trust-remote-code
        - --mem-fraction-static
        - "0.85"
        - --context-length
        - "131072"
        - --max-running-requests
        - "32"
        ports:
        - containerPort: 30000
          name: http
          hostPort: 30000
        resources:
          limits:
            nvidia.com/gpu: 8
            memory: 700Gi
          requests:
            nvidia.com/gpu: 8
            memory: 700Gi
        env:
        - name: CUDA_VISIBLE_DEVICES
          value: "0,1,2,3,4,5,6,7"
        - name: NCCL_SOCKET_IFNAME
          value: eth0
        - name: NCCL_IB_DISABLE
          value: "1"
        volumeMounts:
        - name: model-storage
          mountPath: /models
        readinessProbe:
          httpGet:
            path: /health
            port: 30000
          initialDelaySeconds: 300
          periodSeconds: 30
          timeoutSeconds: 10
          failureThreshold: 30
        livenessProbe:
          httpGet:
            path: /health
            port: 30000
          initialDelaySeconds: 600
          periodSeconds: 60
          timeoutSeconds: 10
          failureThreshold: 5
      volumes:
      - name: model-storage
        persistentVolumeClaim:
          claimName: deepseek-r1-model
      tolerations:
      - key: nvidia.com/gpu
        operator: Exists
        effect: NoSchedule
      nodeSelector:
        nvidia.com/gpu.product: NVIDIA-H20
        nvidia.com/gpu.count: "8"
---
apiVersion: v1
kind: Service
metadata:
  name: deepseek-r1-svc
  namespace: NAMESPACE
spec:
  selector:
    app: deepseek-r1-sglang
  ports:
  - port: 30000
    targetPort: 30000
    name: http
  type: ClusterIP
```

```bash
kubectl apply -f deepseek-sglang-deploy.yaml
# expected: deployment.apps/deepseek-r1-sglang created, service/deepseek-r1-svc created
```

### 步骤 6：监控模型加载并测试推理

模型加载约需 10-20 分钟（700GB 从磁盘到 GPU 显存）：

```bash
# 监控 Pod 状态
kubectl get pods -n NAMESPACE -l app=deepseek-r1-sglang -w
# expected: STATUS 从 ContainerCreating → Running，READY 1/1

# 监控 SGLang 加载日志
kubectl logs -n NAMESPACE deploy/deepseek-r1-sglang --tail=20 -f
# expected: 看到 "The server is fired up and ready to roll!"
```

**预期输出**（加载阶段日志）：

```text
[2024-01-01 00:00:00] Load weight begin. avail mem=740.50 GB
[2024-01-01 00:10:00] Load weight end. type=DeepseekForCausalLM, ...
[2024-01-01 00:10:05] The server is fired up and ready to roll!
```

```bash
# 测试推理
kubectl exec -n NAMESPACE deploy/deepseek-r1-sglang -- \
    curl -s http://localhost:30000/v1/chat/completions \
    -H "Content-Type: application/json" \
    -d '{
      "model": "deepseek-ai/DeepSeek-R1",
      "messages": [{"role": "user", "content": "请用 Python 实现快速排序算法"}],
      "temperature": 0.6,
      "max_tokens": 4096
    }'
# expected: 返回 JSON 格式推理结果，含 reasoning_content（思维链）和 content（最终回答）
```

## 验证

### 控制面（tccli）

```bash
# 验证节点池状态
tccli tke DescribeClusterNodePools --region <Region> \
    --ClusterId CLUSTER_ID
# expected: 节点池 LifeState 为 normal，NodeCount=1 Running=1

# 验证 GPU 信息
tccli tke DescribeGPUInfo --region <Region> --Zone ZONE
# expected: 含水 8 卡 H20/A100 的 GPU 机型列表
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
# 验证 GPU 节点 8 卡全部可用
kubectl get nodes -l nvidia.com/gpu.count=8
# expected: 返回 8-GPU 节点

kubectl describe node GPU_NODE_NAME | grep nvidia.com/gpu
# expected: nvidia.com/gpu: 8（Capacity 和 Allocatable）

# 验证 SGLang Pod 运行且已加载模型
kubectl get pods -n NAMESPACE -l app=deepseek-r1-sglang
# expected: STATUS Running, READY 1/1

# 验证 GPU 显存占用（约 700GB）
kubectl exec -n NAMESPACE deploy/deepseek-r1-sglang -- nvidia-smi
# expected: 8 张 GPU 显存占用各 ~87-93GB

# 验证推理接口
kubectl exec -n NAMESPACE deploy/deepseek-r1-sglang -- \
    curl -s http://localhost:30000/health
# expected: HTTP 200 OK
```

```text
NAME  STATUS  AGE
...
```

## 清理

> **警告**：`DeleteClusterNodePool` 会终止 8-GPU CVM 实例，**数据盘上的 700GB 模型权重将永久丢失**。GPU CVM 单价高（8 卡 H20 实例），不删除将持续产生高额费用。如果模型权重在 PVC 中且 PVC 未随节点池销毁，需单独删除 PVC。按依赖倒序清理。

### 数据面（需 VPN/IOA）

```bash
# 1. 删除推理 Service 和 Deployment
kubectl delete service deepseek-r1-svc -n NAMESPACE
# expected: service "deepseek-r1-svc" deleted

kubectl delete deployment deepseek-r1-sglang -n NAMESPACE
# expected: deployment.apps "deepseek-r1-sglang" deleted

# 2. 删除模型下载 Job
kubectl delete job download-deepseek-r1 -n NAMESPACE
# expected: job.batch "download-deepseek-r1" deleted

# 3. 删除模型存储 PVC（如需保留模型数据，跳过此步）
kubectl delete pvc deepseek-r1-model -n NAMESPACE
# expected: persistentvolumeclaim "deepseek-r1-model" deleted
```

### 控制面（tccli）

```bash
# 4. 清理前状态检查
tccli tke DescribeClusterNodePools --region <Region> \
    --ClusterId CLUSTER_ID
# expected: 确认待删除节点池的 NodePoolId

# 5. 删除 8-GPU 节点池
tccli tke DeleteClusterNodePool --region <Region> \
    --ClusterId CLUSTER_ID \
    --NodePoolIds '["NODE_POOL_ID"]'
# 警告：此操作会终止 8-GPU CVM 实例，所有本地数据和 700GB+ 模型缓存丢失

# 6. 验证已删除
tccli tke DescribeClusterNodePools --region <Region> \
    --ClusterId CLUSTER_ID
# expected: 已删除的节点池不再出现在列表中
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
| `CreateClusterNodePool` 返回 `ResourceInsufficient` — 8 卡 H20/A100 机型 | `tccli tke DescribeGPUInfo --region <Region> --Zone ZONE` 确认 8 卡机型库存 | 目标可用区 8 卡 H20 或 A100 机型库存不足（此为环境限制，非命令错误） | 切换地域或可用区（ap-guangzhou-6、ap-shanghai-5 库存较充足），或使用 2 个 4 卡节点替代（需多机分布式推理，SGLang 支持 `--dp` 参数） |
| `CreateClusterNodePool` 返回 `InvalidParameter.GPUConfig` — GPUCount 不为 8 | 检查 JSON 中 `GPUConfig.GPUCount` 值 | 为单卡机型（GNUCount=1）创建节点池，但 DeepSeek-R1 需 8 卡 | 修改 `GPUCount` 为 `8`，并确认 `InstanceType` 为真实 8 卡机型（如 `GNV4.24XLARGE352`） |
| 模型下载 Job 失败 — `No space left on device` | `kubectl logs -n NAMESPACE job/download-deepseek-r1 --tail=20` | PVC 分配的存储空间小于 700GB | 删除 PVC 并重新创建，`storage` 至少设为 `1500Gi` |

### 部署成功但推理异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| SGLang Pod 启动时 OOMKilled | `kubectl describe pod POD_NAME -n NAMESPACE \| grep OOMKilled`；`kubectl logs POD_NAME -n NAMESPACE \| grep "out of memory"` | 模型权重 + KV Cache 超出 8 卡总显存。可能原因：使用了 FP16 模型（1.3TB）而非 FP8（700GB），或 `mem-fraction-static` 设置过高 | 降低 `--mem-fraction-static` 从 0.85 到 0.75。如为 FP16 模型，改用 FP8 版本 `deepseek-ai/DeepSeek-R1-FP8`。对于 A100-80GB 集群，添加 `--cpu-offload-gb 128` 卸载部分层到 CPU |
| 模型加载卡在 "Load weight begin" 不动 | `kubectl logs -n NAMESPACE deploy/deepseek-r1-sglang --tail=20` | 磁盘 I/O 瓶颈 — 700GB 模型从 PVC 加载到 GPU 显存速度慢 | 确认 `storageClassName` 使用高性能存储（如 `cbs-ssd`）；等待 10-30 分钟（正常加载需要此时间） |
| 推理时 CUDA OOM | `kubectl logs -n NAMESPACE deploy/deepseek-r1-sglang --tail=30 \| grep "CUDA out of memory"` | 并发请求过多导致 KV Cache 超出预留显存。每个请求的 KV Cache 随 `context-length` 增长 | 降低 `--max-running-requests`（如从 32 降至 8），或减少 `--context-length`（如从 131072 降至 65536） |
| 请求返回 HTTP 503（Server not ready） | `kubectl logs -n NAMESPACE deploy/deepseek-r1-sglang --tail=5` | 模型仍在加载中，尚未进入 serving 状态 | 等待 readiness probe 通过。加载 700GB 模型需 10-20 分钟。`kubectl logs -n NAMESPACE deploy/deepseek-r1-sglang \| grep "ready to roll"` 确认就绪 |
| 推理输出质量差或乱码 | `kubectl logs -n NAMESPACE deploy/deepseek-r1-sglang --tail=30 \| grep -i "warning\|error"` | 可能加载了不完整的模型权重，或 `--trust-remote-code` 未启用（DeepSeek-R1 需此参数） | 确认 args 中包含 `--trust-remote-code`；重新下载模型权重：删除 PVC 和 Job，重建 |
| `nvidia-smi` 显示 GPU 利用率 0% | `kubectl exec -n NAMESPACE deploy/deepseek-r1-sglang -- nvidia-smi`；`kubectl logs -n NAMESPACE deploy/deepseek-r1-sglang --tail=10` | SGLang 进程未使用 GPU（可能启动失败或回退到 CPU） | 检查 `CUDA_VISIBLE_DEVICES` env 是否正确（应为 0-7）；`kubectl exec -n NAMESPACE deploy/deepseek-r1-sglang -- python3 -c "import torch; print(torch.cuda.is_available())"` 确认 CUDA 可用 |
| 节点池创建成功但节点长时间不 Ready | `tccli tke DescribeClusterNodePools --region <Region> --ClusterId CLUSTER_ID` 查看 LifeState；`kubectl get nodes` | 8 卡 GPU 驱动安装耗时，或 CVM 首次初始化延迟 | 等待 5-15 分钟（8 卡驱动安装较慢）。超过 15 分钟保留 region、ClusterId、NodePoolId、RequestId → 登录 [TKE 控制台](https://console.cloud.tencent.com/tke2/node-pool) 查看详情 → [提交工单](https://console.cloud.tencent.com/workorder) |

## 下一步

- [在 TKE 上部署 AI 大模型（tccli）](https://cloud.tencent.com/document/product/457/116253) — 小规模 GPU 模型部署
- [部署大模型常见问题（tccli）](https://cloud.tencent.com/document/product/457/116254) — GPU 部署故障排查
- [使用 TKE 完整部署生产级 Stable Diffusion](https://cloud.tencent.com/document/product/457/116255) — AI 绘图部署方案
- [SGLang 官方文档](https://sgl-project.github.io/) — SGLang 引擎参数和性能调优
- [DeepSeek-R1 官方模型页](https://huggingface.co/deepseek-ai/DeepSeek-R1) — 模型说明和使用限制

## 控制台替代

[TKE 控制台 → 节点池 → 新建](https://console.cloud.tencent.com/tke2/node-pool)：选择 8 卡 H20/A100 机型，配置 GPU 参数后创建节点池。然后通过 kubectl 部署 SGLang 推理服务。
