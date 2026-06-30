# 部署大模型常见问题（tccli）

> 对照官方：[部署大模型常见问题](https://cloud.tencent.com/document/product/457/116254) · page_id `116254`

## 概述

本文汇总在 TKE 上部署大模型推理/训练服务时的常见问题及排障路径。以诊断命令为核心，覆盖 GPU 资源、模型加载、推理性能、节点池等典型故障场景。

> **kubectl 数据面不可达**：CAM 拒绝公网端点（strategyId:240463971），内网需 VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

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

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:DescribeClusters, tke:DescribeClusterNodePools, tke:DescribeGPUInfo
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
# 4. 确认目标集群状态
tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
# expected: TotalCount >= 1，ClusterStatus 为 Running

# 5. 确认 GPU 节点池状态
tccli tke DescribeClusterNodePools --region <Region> \
    --ClusterId CLUSTER_ID
# expected: 目标节点池 LifeState 为 normal
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

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看集群列表 | `DescribeClusters` | 是 |
| 查看节点池列表 | `DescribeClusterNodePools` | 是 |
| 查询 GPU 信息 | `DescribeGPUInfo` | 是 |
| 查看集群详情 | `DescribeClusters --ClusterIds` | 是 |

## 操作步骤

本节提供诊断所需的基础查询命令。每个问题场景的具体诊断命令见 [排障](#排障)。

### 步骤 1：检查集群和 GPU 节点池状态

```bash
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterStatus 为 Running
```

**预期输出**：

```json
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "cls-example",
            "ClusterName": "CLUSTER_NAME",
            "ClusterStatus": "Running",
            "ClusterType": "MANAGED_CLUSTER",
            "ClusterVersion": "1.30.0"
        }
    ],
    "RequestId": "..."
}
```

### 步骤 2：查询 GPU 节点池详情

```bash
tccli tke DescribeClusterNodePools --region <Region> \
    --ClusterId CLUSTER_ID
# expected: 返回节点池列表，含 GPU 配置信息
```

**预期输出**：

```json
{
    "NodePoolSet": [
        {
            "NodePoolId": "np-example",
            "Name": "gpu-pool",
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

### 步骤 3：查询可用 GPU 类型和库存

```bash
tccli tke DescribeGPUInfo --region <Region> --Zone ZONE
# expected: 返回当前可用区的 GPU 类型、规格及库存信息
```

```json
{"RequestId": "..."}
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `CLUSTER_ID` | TKE 集群 ID | 格式 `cls-xxxxxxxx` | `tccli tke DescribeClusters --region <Region>` |
| `REGION` | 地域 | 如 `ap-guangzhou` | `tccli configure list` |
| `ZONE` | 可用区 | 如 `ap-guangzhou-6` | `tccli tke DescribeClusters --region <Region>` 查看集群所在可用区 |

### 步骤 4：检查 GPU 节点实际资源（数据面，需 VPN/IOA）

```bash
# 检查节点 GPU 可分配量
kubectl get nodes -o wide
# expected: 列出所有节点及状态

kubectl describe node GPU_NODE_NAME | grep -A5 "Capacity:" | grep nvidia
# expected: nvidia.com/gpu: N (N 为节点 GPU 卡数)
```

```text
NAME  STATUS  AGE
...
```

```bash
# 检查 GPU Pod 状态和日志
kubectl get pods -n NAMESPACE
# expected: 目标 Pod STATUS 及 RESTARTS 次数

kubectl describe pod POD_NAME -n NAMESPACE | grep -A10 Events
# expected: Pod 事件，关注 OOMKilled、FailedScheduling 等
```

```text
NAME  STATUS  AGE
...
```

## 验证

### 控制面（tccli）

```bash
# 确认集群运行正常
tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterStatus: "Running"

# 确认节点池正常
tccli tke DescribeClusterNodePools --region <Region> \
    --ClusterId CLUSTER_ID
# expected: LifeState: "normal", 节点 Running 数与预期一致
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

### 数据面（需 VPN/IOA）

```bash
# 确认 GPU 节点资源
kubectl get nodes -l nvidia.com/gpu.type
# expected: 返回预期数量的 GPU 节点

# 确认 Pod 运行正常
kubectl get pods -n NAMESPACE
# expected: 所有推理 Pod STATUS 为 Running，RESTARTS 为 0

# 确认 GPU 算力可用
kubectl exec -n NAMESPACE DEPLOYMENT_NAME -- nvidia-smi
# expected: 显示 GPU 型号、驱动版本、显存使用量和 GPU 利用率
```

```text
NAME  STATUS  AGE
...
```

## 清理

本页为问题排查指南，不涉及资源创建，无需清理。如需清理 GPU 节点池及相关资源，参见：

- [在 TKE 上部署 AI 大模型（tccli）→ 清理](https://cloud.tencent.com/document/product/457/116253)
- [在 TKE 上部署满血版 DeepSeek-R1（SGLang）（tccli）→ 清理](https://cloud.tencent.com/document/product/457/116858)

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `CreateClusterNodePool` 返回 `ResourceInsufficient` | `tccli tke DescribeGPUInfo --region <Region> --Zone ZONE` 查看 GPU 库存；`tccli cvm DescribeInstanceConfigInfos --region <Region> --Filters '[{"Name":"instance-type","Values":["GPU_INSTANCE_TYPE"]}]'` 确认机型 | GPU 机型在目标可用区库存不足或用户配额用完（此为环境限制，非命令错误） | 切换可用区或 GPU 机型（V100 换 T4/A100），或联系售前提升配额。保留 region、Zone、InstanceType、RequestId 提交工单 |
| `CreateClusterNodePool` 返回 `InvalidParameter.GPUConfig` | 检查 JSON 中 `GPUConfig.GPUType` 是否为 `DescribeGPUInfo` 返回的合法值 | GPUType 填写了不支持的 GPU 型号或拼写错误（如 `T4` 写成 `t4`） | 执行 `tccli tke DescribeGPUInfo --region <Region> --Zone ZONE` 确认可用 GPU 类型枚举值，修改 JSON 中 `GPUType` 为正确值（区分大小写） |
| `DescribeGPUInfo` 返回空列表 | `tccli configure list` 确认 region 配置；`tccli tke DescribeClusters --region <Region>` 确认该地域有 TKE 服务 | 该地域不支持 GPU 机型或 region 配置错误 | 切换到支持 GPU 的地域（推荐 ap-guangzhou、ap-shanghai、ap-beijing），重新配置 `tccli configure set region REGION` |
| `CreateClusterNodePool` 返回 `InvalidParameter.SubnetId` | `tccli vpc DescribeSubnets --region <Region> --SubnetIds '["SUBNET_ID"]'` 确认子网存在且属于目标 VPC | 子网 ID 不存在或子网不属于 `VpcId` 指定的 VPC | 用 `tccli vpc DescribeSubnets --region <Region> --Filters '[{"Name":"vpc-id","Values":["VPC_ID"]}]'` 查询同 VPC 下的子网 |

### 创建成功但 GPU 工作负载异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| GPU OOM：Pod 被 OOMKilled | `kubectl describe pod POD_NAME -n NAMESPACE \| grep -B2 -A5 OOMKilled`；`kubectl logs POD_NAME -n NAMESPACE --previous` 查看 OOM 前日志 | 模型显存需求超过单卡 GPU 可用显存。常见原因：`--max-model-len` 设置过大，或未启用量化 | 降低 `--max-model-len`（如从 8192 降至 4096），启用量化 `--quantization awq`，或增加 `--gpu-memory-utilization` 从 0.90 降至 0.80。70B+ 模型需多卡部署（tensor parallelism） |
| GPU 不可用：节点无 `nvidia.com/gpu` 资源 | `kubectl describe node GPU_NODE_NAME \| grep nvidia.com/gpu` | 节点创建时 GPU 驱动未安装或 GPUConfig 配置错误 | 检查创建节点池时的 `GPUConfig.GPUDriverVersion` 是否正确。驱动版本：T4/V100 → `470.182.03`，A100 → `525.105.17`。保留 NodePoolId 和创建 JSON，删除并重建节点池 |
| 模型下载超时：Pod 长时间 Init 或 CrashLoopBackOff | `kubectl describe pod POD_NAME -n NAMESPACE \| grep -A5 "State:"`；`kubectl logs POD_NAME -n NAMESPACE \| tail -30` | HuggingFace 模型下载因网络原因超时或失败 | 方案一：配置 HuggingFace 镜像 `HF_ENDPOINT=https://hf-mirror.com`。方案二：创建 PVC 预缓存模型文件，将 `HF_HOME` 指向持久化路径。方案三：使用 initContainer 模型下载脚本并配置重试 |
| vLLM 启动失败：CUDA 版本不匹配 | `kubectl logs DEPLOYMENT_NAME -n NAMESPACE --tail=50` | 容器镜像的 CUDA 版本与节点 GPU 驱动版本不兼容 | 确认驱动版本 → 选择匹配的镜像 tag。T4/V100（470.182.03/CUDA 11.4）→ `vllm/vllm-openai:v0.5.4`；A100（525.105.17/CUDA 12.0）→ `vllm/vllm-openai:latest`。可通过 `kubectl exec POD_NAME -n NAMESPACE -- nvidia-smi` 确认驱动版本 |
| Pod Pending：事件显示 insufficient nvidia.com/gpu | `kubectl describe pod POD_NAME -n NAMESPACE \| grep -A10 Events` | 集群中 GPU 节点资源不足，所有 GPU 已被其他 Pod 占用 | 扩容节点池 GPU 节点数（`tccli tke ModifyClusterNodePool` 增加 DesiredNodesNum），或释放不再使用的 GPU Pod |
| 推理延迟高、吞吐低 | `kubectl exec DEPLOYMENT_NAME -n NAMESPACE -- nvidia-smi` 查看 GPU 利用率；`kubectl top pod -n NAMESPACE POD_NAME` 查看资源使用 | GPU 利用率低（<50%）说明请求不饱和；GPU 利用率达 100% 为计算瓶颈 | GPU 空闲时增加请求并发数；GPU 满负荷时增加 Pod 副本数实现负载均衡。检查 `--max-num-seqs`（vLLM）是否成为瓶颈 |
| OOMKilled 发生在长上下文推理 | `kubectl logs POD_NAME -n NAMESPACE --previous \| grep -i "out of memory"`；`kubectl describe pod POD_NAME -n NAMESPACE \| grep OOMKilled` | 上下文窗口过长导致 KV Cache 超出显存。`--max-model-len` 定义了最大上下文长度，KV Cache 随上下文线性增长 | 降低 `--max-model-len` 或添加 `--kv-cache-dtype fp8`（需 vLLM v0.5.4+）。若需支持长上下文，增加 GPU 数并使用 tensor parallelism（`--tensor-parallel-size 2`） |
| 节点池创建成功但节点状态长时间 Creating | `tccli tke DescribeClusterNodePools --region <Region> --ClusterId CLUSTER_ID` 查看 LifeState；`tccli cvm DescribeInstances --region <Region> --Filters '[{"Name":"instance-name","Values":["NODE_POOL_NAME*"]}]'` 查看 CVM 状态 | CVM 实例创建缓慢或无库存（此为环境限制） | 等待 5-10 分钟；若超过 15 分钟仍 Creating，保留 region、ClusterId、NodePoolId、RequestId → 登录 [TKE 控制台](https://console.cloud.tencent.com/tke2/node-pool) 查看详细信息 → 仍无法解决则 [提交工单](https://console.cloud.tencent.com/workorder) |
| 推理返回空或乱码 | `kubectl logs DEPLOYMENT_NAME -n NAMESPACE --tail=50 \| grep -i "error\|warning"` | 模型权重文件损坏、模型路径错误、或 prompting 模板不匹配 | 检查 `--model` 参数路径是否正确（HuggingFace model ID 或本地路径）。vLLM 启动日志会显示 "Loading model weights"。验证模型完整性：`kubectl exec DEPLOYMENT_NAME -n NAMESPACE -- ls /model-cache/models--Qwen--Qwen2.5-7B-Instruct/snapshots/` |

## 下一步

- [在 TKE 上部署 AI 大模型（tccli）](https://cloud.tencent.com/document/product/457/116253) — GPU 模型部署完整指南
- [在 TKE 上部署满血版 DeepSeek-R1（SGLang）（tccli）](https://cloud.tencent.com/document/product/457/116858) — 671B 大模型多卡部署
- [TKE Serverless 运行 ChatGLM-6B 微调（tccli）](https://cloud.tencent.com/document/product/457/97823) — Serverless GPU 微调方案
- [vLLM 官方文档 - 引擎参数](https://docs.vllm.ai/en/latest/models/engine_args.html) — vLLM 参数调优
- [NVIDIA GPU Operator 在 TKE 上的部署](https://cloud.tencent.com/document/product/457/116250) — GPU 驱动和监控方案

## 控制台替代

[TKE 控制台 → 节点池](https://console.cloud.tencent.com/tke2/node-pool)：查看 GPU 节点池详情。Pod 事件和日志需在 [TKE 控制台 → 工作负载](https://console.cloud.tencent.com/tke2/deployment) 查看。
