# 在 TKE 上部署 AI 大模型（tccli）

> 对照官方：[在 TKE 上部署 AI 大模型](https://cloud.tencent.com/document/product/457/116253) · page_id `116253`

## 概述

在 TKE 集群上创建 GPU 节点池并部署大模型推理服务（vLLM/TGI）。控制面操作通过 tccli 管理节点池，数据面通过 kubectl 部署推理服务。

> **kubectl 数据面不可达**：CAM 拒绝公网端点（strategyId:240463971），内网需 VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

| 方案 | GPU 类型 | 适用场景 | 显存 | 成本 |
|------|---------|---------|------|------|
| vLLM + T4 | NVIDIA T4（16GB） | 7B 模型推理（量化） | 16 GB | 低 |
| vLLM + V100 | NVIDIA V100（32GB） | 13B 模型推理 | 32 GB | 中 |
| vLLM/TGI + A100 | NVIDIA A100（40GB/80GB） | 70B 模型推理 | 40/80 GB | 高 |

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
#    cvm:DescribeInstances, cvm:DescribeInstanceConfigInfos, cvm:DescribeImages
# 验证：执行 DescribeClusters 确认权限
tccli tke DescribeClusters --region <Region>
# expected: exit 0，返回集群列表（可为空）

# 验证 GPU Info 权限
tccli tke DescribeGPUInfo --region <Region> --Zone ZONE
# expected: exit 0，返回 GPU 类型列表
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
# 4. 确认目标 TKE 集群存在且状态正常
tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
# expected: TotalCount >= 1，ClusterStatus 为 Running

# 5. 查询可用 GPU 机型及库存
tccli tke DescribeGPUInfo --region <Region> --Zone ZONE
# expected: 返回可用 GPU 类型及规格列表

# 6. 查询 VPC 子网资源
tccli vpc DescribeSubnets --region <Region> --Filters '[{"Name":"vpc-id","Values":["VPC_ID"]}]'
# expected: 至少 1 个子网，AvailableIpCount ≥ 所需节点数
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

- K8s 版本：推荐 ≥ 1.28。确认可用版本：
  `tccli tke DescribeVersions --region <Region>`
- GPU 驱动版本：`470.182.03`（T4/V100）或 `525.105.17`（A100）
- 容器运行时：强制 `containerd`（托管集群）

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 查看集群列表 | `tccli tke DescribeClusters --region ap-guangzhou` | 是 |
| 创建 GPU 节点池 | `tccli tke CreateClusterNodePool --region ap-guangzhou` | 否 |
| 查看 GPU 信息 | `tccli tke DescribeGPUInfo --region ap-guangzhou` | 是 |

## 关键字段说明

以下说明 `CreateClusterNodePool` 中与 GPU 节点池相关的主要参数。完整参数定义见 `tccli tke CreateClusterNodePool --generate-cli-skeleton`。

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `ClusterId` | String | 是 | 目标托管集群 ID，格式 `cls-xxxxxxxx` | 集群不存在或非托管 → `InvalidParameter.ClusterId` |
| `Name` | String | 是 | 节点池名称，长度 1-60 字符 | 重名或格式错误 → 创建失败 |
| `AutoScalingGroupPara.InstanceType` | String | 是 | GPU 机型，如 `GN10Xp.2XLARGE40`。`tccli cvm DescribeInstanceConfigInfos` 查询 | 机型不支持或库存不足 → `ResourceInsufficient` |
| `AutoScalingGroupPara.GPUConfig` | Object | 是 | GPU 配置对象，含 GPUType/GPUCount/GPUDriverVersion 等 | 缺失或字段错误 → GPU 节点不可用 |
| `GPUConfig.GPUType` | String | 是 | `V100` / `A100` / `T4` 等。`tccli tke DescribeGPUInfo` 查询可用类型 | 填不可用类型 → 节点无法调度 GPU 工作负载 |
| `GPUConfig.GPUCount` | Integer | 是 | 每节点 GPU 卡数，如 `1`、`2`、`4`、`8` | 超过机型物理 GPU 数 → `InvalidParameter` |
| `GPUConfig.GPUDriverVersion` | String | 是 | 驱动版本如 `470.182.03`（T4/V100）或 `525.105.17`（A100） | 驱动与 GPU 不兼容 → 节点 GPU 不可用 |
| `AutoScalingGroupPara.DesiredNodesNum` | Integer | 是 | 初始节点数，≥ 1 | 为 0 → 节点池无可用节点 |
| `AutoScalingGroupPara.VpcId` | String | 是 | VPC ID，格式 `vpc-xxxxxxxx`，与集群同 VPC | VPC 不一致 → 节点无法加入集群 |
| `AutoScalingGroupPara.SubnetIds` | Array | 是 | 子网 ID 列表 | 子网不可用 → `InvalidParameter.SubnetId` |

## 操作步骤

### 步骤 1：创建 GPU 节点池

#### 选择依据

*为什么选这个值而不是其他：*

- **GPU 类型**：7B-13B 模型选用 T4（16GB，低成本）或 V100（32GB，推理速度快）；70B+ 模型必须用 A100（40GB/80GB）。可通过 `tccli tke DescribeGPUInfo --region <Region> --Zone ZONE` 查看当前可用 GPU 库存。
- **节点数**：推理场景推荐 ≥ 2 个节点以实现高可用和滚动更新。单节点仅适合开发测试。
- **GPU 驱动**：T4/V100 用 `470.182.03`（CUDA 11.4），A100 用 `525.105.17`（CUDA 12.0）。版本与容器镜像的 CUDA 版本必须兼容。
- **节点池规格**：`GN10Xp.2XLARGE40` 为 1 卡 V100，`GN7.2XLARGE32` 为 1 卡 T4。A100 用 `GN13.2XLARGE32`。

查询当前地域可用 GPU 类型：

```bash
tccli tke DescribeGPUInfo --region <Region> --Zone ZONE
# expected: 返回 GPU 类型、规格、库存信息列表
```

#### 最小创建（1 节点 T4 GPU 节点池，只含必填字段）

`gpu-nodepool-minimal.json`：

```json
{
  "ClusterId": "CLUSTER_ID",
  "Name": "gpu-t4-pool",
  "AutoScalingGroupPara": {
    "AutoScalingGroupName": "gpu-t4-asg",
    "InstanceType": "GN7.2XLARGE32",
    "GPUConfig": {
      "GPUType": "T4",
      "GPUCount": 1,
      "GPUDriverVersion": "470.182.03"
    },
    "DesiredNodesNum": 1,
    "VpcId": "VPC_ID",
    "SubnetIds": ["SUBNET_ID"],
    "SystemDisk": {
      "DiskType": "CLOUD_SSD",
      "DiskSize": 100
    },
    "InternetAccessible": {
      "InternetMaxBandwidthOut": 0,
      "PublicIpAssigned": false
    },
    "SecurityGroupIds": ["SECURITY_GROUP_ID"],
    "LoginSettings": {
      "KeyIds": ["KEY_ID"]
    }
  }
}
```

```bash
tccli tke CreateClusterNodePool --region <Region> --cli-input-json file://gpu-nodepool-minimal.json
# expected: exit 0，返回 NodePoolId
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `CLUSTER_ID` | TKE 集群 ID | 格式 `cls-xxxxxxxx` | `tccli tke DescribeClusters --region <Region>` |
| `VPC_ID` | VPC 实例 ID | 与集群同 VPC | `tccli vpc DescribeVpcs --region <Region>` |
| `SUBNET_ID` | 子网 ID | 有充足可用 IP | `tccli vpc DescribeSubnets --region <Region>` |
| `SECURITY_GROUP_ID` | 安全组 ID | 放行所需端口 | `tccli vpc DescribeSecurityGroups --region <Region>` |
| `REGION` | 地域 | 如 `ap-guangzhou` | `tccli configure list` |

#### 增强配置（多 GPU 节点池 + 自动伸缩 + A100）

`gpu-nodepool-enhanced.json`：

```json
{
  "ClusterId": "CLUSTER_ID",
  "Name": "gpu-a100-pool",
  "AutoScalingGroupPara": {
    "AutoScalingGroupName": "gpu-a100-asg",
    "InstanceType": "GN13.2XLARGE32",
    "GPUConfig": {
      "GPUType": "A100",
      "GPUCount": 1,
      "GPUDriverVersion": "525.105.17",
      "CUDAVersion": "12.0",
      "CUDNNVersion": "8.9.0"
    },
    "DesiredNodesNum": 2,
    "MinNodesNum": 1,
    "MaxNodesNum": 5,
    "VpcId": "VPC_ID",
    "SubnetIds": ["SUBNET_ID_1", "SUBNET_ID_2"],
    "SystemDisk": {
      "DiskType": "CLOUD_SSD",
      "DiskSize": 200
    },
    "DataDisks": [
      {
        "DiskType": "CLOUD_PREMIUM",
        "DiskSize": 500
      }
    ],
    "InternetAccessible": {
      "InternetMaxBandwidthOut": 10,
      "PublicIpAssigned": true
    },
    "SecurityGroupIds": ["SECURITY_GROUP_ID"],
    "LoginSettings": {
      "KeyIds": ["KEY_ID"]
    },
    "Tags": [
      {"Key": "env", "Value": "production"},
      {"Key": "gpu-type", "Value": "A100"}
    ]
  },
  "Labels": [
    {"Name": "nvidia.com/gpu.type", "Value": "a100"},
    {"Name": "workload-type", "Value": "llm-inference"}
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
tccli tke CreateClusterNodePool --region <Region> --cli-input-json file://gpu-nodepool-enhanced.json
# expected: exit 0，返回 NodePoolId
```

### 步骤 2：轮询节点池就绪

节点池创建是异步操作，轮询直到节点就绪：

```bash
tccli tke DescribeClusterNodePools --region <Region> \
    --ClusterId CLUSTER_ID
# expected: 目标节点池 LifeState 为 "normal"，NodeCount ≥ DesiredNodesNum
```

**预期输出**：

```json
{
    "NodePoolSet": [
        {
            "NodePoolId": "np-example",
            "Name": "gpu-a100-pool",
            "ClusterId": "cls-example",
            "LifeState": "normal",
            "NodeCountSummary": {
                "AutoscalingAdded": {
                    "Total": 2,
                    "Running": 2
                }
            }
        }
    ],
    "RequestId": "..."
}
```

验证维度：

| 维度 | 命令 | 预期 |
|------|------|------|
| 状态 | `DescribeClusterNodePools --ClusterId CLUSTER_ID` | `LifeState: "normal"` |
| 节点数 | 同上，检查 `NodeCountSummary.AutoscalingAdded` | `Running` 数 = `DesiredNodesNum` |
| GPU 标签 | `kubectl get nodes -l nvidia.com/gpu.type` | 返回预期节点列表 |
| GPU 资源 | `kubectl describe node NODE_NAME` | `nvidia.com/gpu` 显示预期卡数 |

### 步骤 3：部署大模型推理服务（kubectl，数据面）

> 以下 kubectl 命令需在内网/VPN 环境下执行。

#### 选择依据

- **推理框架**：vLLM 吞吐高、支持 PagedAttention（节省显存 50%+）；TGI（Text Generation Inference）HuggingFace 原生支持，生态好。中小团队推荐 vLLM，HuggingFace 深度用户推荐 TGI。
- **GPU 请求**：`nvidia.com/gpu: 1` 表示请求 1 张 GPU。确保与节点 GPU 型号匹配（通过 `nodeSelector` 或 `nodeAffinity` 指定 GPU 类型节点）。
- **模型选择**：Qwen2.5-7B、Llama-3.1-8B 适合 T4/V100（单卡即可推理）；Qwen2.5-72B、Llama-3.1-70B 需 A100（单卡 80GB 可跑量化版）。

#### 部署 vLLM 推理服务

`vllm-deploy.yaml`：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: llm-inference
  namespace: NAMESPACE
  labels:
    app: llm-inference
spec:
  replicas: 1
  selector:
    matchLabels:
      app: llm-inference
  template:
    metadata:
      labels:
        app: llm-inference
    spec:
      containers:
      - name: vllm
        image: vllm/vllm-openai:latest
        args:
        - --model
        - "Qwen/Qwen2.5-7B-Instruct"
        - --trust-remote-code
        - --max-model-len
        - "4096"
        - --gpu-memory-utilization
        - "0.90"
        ports:
        - containerPort: 8000
          name: http
        resources:
          limits:
            nvidia.com/gpu: 1
          requests:
            nvidia.com/gpu: 1
        env:
        - name: HF_HOME
          value: /model-cache
        volumeMounts:
        - name: model-cache
          mountPath: /model-cache
      volumes:
      - name: model-cache
        persistentVolumeClaim:
          claimName: model-cache-pvc
      tolerations:
      - key: nvidia.com/gpu
        operator: Exists
        effect: NoSchedule
      nodeSelector:
        nvidia.com/gpu.type: "t4"
---
apiVersion: v1
kind: Service
metadata:
  name: llm-inference-svc
  namespace: NAMESPACE
spec:
  selector:
    app: llm-inference
  ports:
  - port: 8000
    targetPort: 8000
    name: http
  type: ClusterIP
```

```bash
kubectl apply -f vllm-deploy.yaml
# expected: deployment.apps/llm-inference created, service/llm-inference-svc created
```

#### 验证推理服务就绪

```bash
kubectl get pods -n NAMESPACE -l app=llm-inference
# expected: STATUS Running, READY 1/1
```

**预期输出**：

```text
NAME                              READY   STATUS    RESTARTS   AGE
llm-inference-5c8b9f7d6-xk9m2    1/1     Running   0          3m
```

#### 测试推理

```bash
kubectl exec -n NAMESPACE deploy/llm-inference -- \
    curl -s http://localhost:8000/v1/chat/completions \
    -H "Content-Type: application/json" \
    -d '{"model":"Qwen/Qwen2.5-7B-Instruct","messages":[{"role":"user","content":"你好"}]}'
# expected: 返回 JSON 格式的推理结果
```

## 验证

### 控制面（tccli）

```bash
# 验证 GPU 节点池状态
tccli tke DescribeClusterNodePools --region <Region> \
    --ClusterId CLUSTER_ID
# expected: 节点池 LifeState 为 normal，所有节点 Running

# 验证 GPU 信息
tccli tke DescribeGPUInfo --region <Region> --Zone ZONE
# expected: 返回 GPU 类型、规格列表
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
# 验证 GPU 节点可用资源
kubectl get nodes -l nvidia.com/gpu.type
# expected: 返回 GPU 节点列表

kubectl describe node GPU_NODE_NAME | grep nvidia.com/gpu
# expected: nvidia.com/gpu: 1, nvidia.com/gpu: 1 (Capacity 和 Allocatable)

# 验证推理 Pod 运行状态
kubectl get pods -n NAMESPACE -l app=llm-inference
# expected: STATUS Running

# 验证推理接口可访问
kubectl port-forward -n NAMESPACE svc/llm-inference-svc 8000:8000 &
curl -s http://localhost:8000/v1/models
# expected: 返回模型列表 JSON
```

```text
NAME  STATUS  AGE
...
```

## 清理

> **警告**：删除 GPU 节点池会终止关联的 CVM 实例，数据盘数据将丢失。如果节点池配置了自动伸缩，请先关闭自动伸缩再删除。如果不删除这些资源，将持续产生 GPU CVM 费用（GPU 机型单价较高）。

### 数据面（需 VPN/IOA）

先清理 Kubernetes 资源（依赖倒序）：

```bash
kubectl delete service llm-inference-svc -n NAMESPACE
# expected: service "llm-inference-svc" deleted

kubectl delete deployment llm-inference -n NAMESPACE
# expected: deployment.apps "llm-inference" deleted

kubectl delete pvc model-cache-pvc -n NAMESPACE
# expected: persistentvolumeclaim "model-cache-pvc" deleted
```

### 控制面（tccli）

```bash
# 1. 清理前状态检查
tccli tke DescribeClusterNodePools --region <Region> \
    --ClusterId CLUSTER_ID
# expected: 确认待删除节点池的 NodePoolId 和名称

# 2. 删除 GPU 节点池
tccli tke DeleteClusterNodePool --region <Region> \
    --ClusterId CLUSTER_ID \
    --NodePoolIds '["NODE_POOL_ID"]'
# 警告：此操作会终止 GPU CVM 实例，所有本地数据丢失

# 3. 验证已删除
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
| `CreateClusterNodePool` 返回 `ResourceInsufficient` | `tccli tke DescribeGPUInfo --region <Region> --Zone ZONE` 查看可用 GPU 库存 | GPU 机型库存不足或配额耗尽（此为环境限制，非命令错误） | 换用其他 GPU 机型（如 V100 换 T4）或更换可用区，或提交工单申请配额提升 |
| `CreateClusterNodePool` 返回 `InvalidParameter.GPUConfig` | 检查 JSON 中 `GPUConfig.GPUType` 和 `GPUCount` | GPUType 不在 `DescribeGPUInfo` 返回的可用列表中，或 GPUCount 超过机型物理 GPU 数 | 用 `tccli tke DescribeGPUInfo --region <Region> --Zone ZONE` 确认可用 GPU 类型和规格，修改 JSON |
| `CreateClusterNodePool` 返回 `InvalidParameter.SubnetId` | `tccli vpc DescribeSubnets --region <Region> --SubnetIds '["SUBNET_ID"]'` 确认子网状态 | 子网不存在、不属于指定 VPC、或无可用 IP | 更换有效子网或使用 `tccli vpc DescribeSubnets` 重新查询 |
| `CreateClusterNodePool` 返回 `InvalidParameter.ClusterId` | `tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'` 确认集群存在 | 集群 ID 不存在或不是托管集群（`ClusterType` ≠ `MANAGED_CLUSTER`） | 确认集群 ID 正确且为托管集群。独立集群不支持节点池操作 |
| `CreateClusterNodePool` 返回 `LimitExceeded` | `tccli tke DescribeClusterNodePools --region <Region> --ClusterId CLUSTER_ID` 统计现有节点池数 | 集群节点池数量达到上限（此为环境限制，非命令错误） | 删除不再使用的节点池后重试 |

### 创建成功但 GPU 不可用

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 节点 Running 但 Pod 无法调度 GPU | `kubectl describe node NODE_NAME \| grep nvidia.com/gpu` | 节点 GPU 驱动未正确安装，nvidia.com/gpu 资源为 0 | 检查 GPUConfig 中 GPUDriverVersion 是否与机型兼容。保留 region、ClusterId、NodePoolId → 重新创建节点池并指定正确驱动版本 |
| Pod 处于 Pending 状态，events 显示 insufficient nvidia.com/gpu | `kubectl describe pod POD_NAME -n NAMESPACE \| grep -A5 Events` | 节点 GPU 资源被其他 Pod 占用或节点 GPU 数量不足 | 增加节点池节点数或减少 Pod 的 GPU 请求数 |
| 推理服务 OOMKilled | `kubectl describe pod POD_NAME -n NAMESPACE \| grep OOMKilled` | 模型显存需求超过 GPU 可用显存 | 降低 `--max-model-len`，启用量化（添加 `--quantization awq`），或换用更大显存的 GPU 机型 |
| vLLM 启动失败，CUDA error | `kubectl logs deploy/llm-inference -n NAMESPACE --tail=50` | 容器 CUDA 版本与 GPU 驱动不兼容 | 确认驱动版本（T4/V100: 470.182.03→CUDA 11.4，A100: 525.105.17→CUDA 12.0），使用匹配的 vLLM 镜像 tag |
| 模型下载超时 | `kubectl logs deploy/llm-inference -n NAMESPACE \| grep -i download` | HuggingFace 下载速度慢或网络不通 | 创建 PVC 缓存模型文件，使用本地模型路径而非在线下载；或配置 HuggingFace 镜像源（添加 env `HF_ENDPOINT=https://hf-mirror.com`） |

## 下一步

- [部署大模型常见问题（tccli）](https://cloud.tencent.com/document/product/457/116254) — GPU 部署排障指南
- [在 TKE 上部署满血版 DeepSeek-R1（SGLang）](https://cloud.tencent.com/document/product/457/116858) — 671B 大模型多卡部署
- [使用 TKE 部署 Stable Diffusion WebUI](https://cloud.tencent.com/document/product/457/116255) — AI 绘图部署方案
- [TKE GPU 节点池文档](https://cloud.tencent.com/document/product/457/32207) — 节点池 API 参考
- [vLLM 官方文档](https://docs.vllm.ai/) — vLLM 参数和优化参考

## 控制台替代

[TKE 控制台 → 节点池 → 新建](https://console.cloud.tencent.com/tke2/node-pool)：选择 GPU 机型、配置 GPU 参数后创建节点池，然后用 kubectl 部署推理服务。
