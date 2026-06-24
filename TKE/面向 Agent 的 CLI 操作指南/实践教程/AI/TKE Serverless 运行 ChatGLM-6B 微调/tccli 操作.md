# TKE Serverless 运行 ChatGLM-6B 微调（tccli）

> 对照官方：[TKE Serverless 运行 ChatGLM-6B 微调](https://cloud.tencent.com/document/product/457/97823) · page_id `97823`

## 概述

在 TKE Serverless（EKS）集群上使用 LoRA 对 ChatGLM-6B 模型进行参数高效微调。EKS 无需预购 GPU 节点，按 Pod 实际运行时长计费，适合间歇性微调任务。控制面通过 tccli 管理 Serverless 集群，数据面通过 kubectl 提交带 EKS 注解的 GPU 微调 Job。

> **kubectl 数据面不可达**：CAM 拒绝公网端点（strategyId:240463971），内网需 VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

| 方案 | 集群类型 | GPU 获取方式 | 适用场景 | 成本模型 |
|------|---------|-------------|---------|---------|
| Serverless（EKS）+ LoRA 微调 | Serverless 集群 | EKS 注解自动分配 | 间歇性微调任务，小批量实验 | 按 Pod 运行秒数 + GPU 计费 |
| 标准集群 + 节点池 + 全量微调 | 托管集群 | 预购 GPU 节点池 | 长期持续训练，大规模微调 | 按 CVM 实例小时计费 |

**建议**：ChatGLM-6B LoRA 微调仅需单卡 V100（32GB），任务运行时间 1-4 小时。EKS Serverless 按需付费，比预购 GPU 节点池更经济。如需要全参数微调（需 4-8 卡 A100），建议使用标准集群 + GPU 节点池。

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
#    tke:DescribeClusters, tke:DescribeEKSClusters, tke:CreateEKSCluster
#    tke:DeleteEKSCluster
# 验证：执行 DescribeEKSClusters 确认权限
tccli tke DescribeEKSClusters --region <Region>
# expected: exit 0，返回 EKS 集群列表（可为空）

# 验证 kubectl
kubectl version --client
# expected: Client Version >= v1.28.0
```

```json
{
  "TotalCount": "<TotalCount>",
  "Clusters": "<Clusters>",
  "ClusterId": "<ClusterId>",
  "ClusterName": "<ClusterName>",
  "VpcId": "<VpcId>",
  "SubnetIds": "<SubnetIds>"
}
```

### 资源检查

```bash
# 4. 确认 EKS 集群存在（或创建新的）
tccli tke DescribeEKSClusters --region <Region>
# expected: 返回 EKS 集群列表。如为空，需先创建 Serverless 集群

# 5. 查询 VPC 子网资源（EKS Pod 需占用子网 IP）
tccli vpc DescribeSubnets --region <Region> --Filters '[{"Name":"vpc-id","Values":["VPC_ID"]}]'
# expected: AvailableIpCount ≥ 5

# 6. 确认 COS 存储桶（存放训练数据和模型输出，可选）
# 无 tccli COS 直接命令，可通过控制台或 COS SDK 确认
```

```json
{
  "TotalCount": "<TotalCount>",
  "Clusters": "<Clusters>",
  "ClusterId": "<ClusterId>",
  "ClusterName": "<ClusterName>",
  "VpcId": "<VpcId>",
  "SubnetIds": "<SubnetIds>"
}
```

### 版本与规格选择

- EKS 集群：K8s ≥ 1.28，Serverless 类型
- GPU 类型：V100（32GB），ChatGLM-6B LoRA 微调最低要求。可在 EKS 注解中指定 `V100` 或 `T4`
- 微调框架：`transformers` + `peft`（LoRA），Python 3.10+ / PyTorch 2.0+
- 数据存储：PVC（CBS）或 COS 挂载，推荐 CBS（IO 性能好）

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 查看集群列表 | `tccli tke DescribeClusters --region ap-guangzhou` | 是 |
| 创建 GPU 节点池 | `tccli tke CreateClusterNodePool --region ap-guangzhou` | 否 |
| 查看 GPU 信息 | `tccli tke DescribeGPUInfo --region ap-guangzhou` | 是 |

## 关键字段说明

以下说明 `CreateEKSCluster` 的主要参数，以及 EKS Pod 的 GPU 注解。

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `ClusterName` | String | 是 | Serverless 集群名称，长度 1-60 | 重名 → 创建失败 |
| `K8sVersion` | String | 是 | 如 `1.28.3`。`tccli tke DescribeVersions --region <Region>` 查询 | 版本不存在 → `InvalidParameter` |
| `VpcId` | String | 是 | VPC ID，格式 `vpc-xxxxxxxx` | VPC 不存在 → `InvalidParameter.VpcId` |
| `SubnetIds` | Array | 是 | 子网 ID 列表，Pod 将从此子网获取 IP | 子网 IP 不足 → Pod Pending |
| `ClusterDesc` | String | 否 | 集群描述 | — |
| `ServiceSubnetId` | String | 是 | Service 网段子网 ID | 无 Service 子网 → Service 无法分配 ClusterIP |
| `DnsServers` | Array | 否 | 自定义 DNS 列表 | — |
| EKS 注解 `eks.tke.cloud.tencent.com/gpu-type` | String | Pod 注解 | `V100` / `T4` / `A10` 等。ChatGLM-6B LoRA 选 `V100` | 填不可用 GPU → Pod 创建失败 |
| EKS 注解 `eks.tke.cloud.tencent.com/gpu-count` | String | Pod 注解 | GPU 卡数，如 `1` | 填 0 → 不使用 GPU，训练无法进行 |

## 操作步骤

### 步骤 1：确认或创建 EKS Serverless 集群

#### 选择依据

*为什么选 EKS 而不是标准节点池：*

- **EKS Serverless**：按 Pod 实际运行时间（秒级）计费。ChatGLM-6B LoRA 微调通常 1-4 小时完成，用 EKS 比预购 GPU 节点更经济。无需管理节点、无需预先购买 CVM。
- **标准集群 GPU 节点池**：适合长期持续训练任务（≥24 小时），或需要多机分布式训练的场合。需要预先购买 GPU CVM。
- **LoRA vs 全量微调**：LoRA 仅训练低秩适配器（约 1-5MB），显存需求 ~20GB（含基础模型），单卡 V100（32GB）即可。全量微调需 4-8x A100，LoRA 更适合 Serverless 场景。

#### 查询已有 EKS 集群

```bash
tccli tke DescribeEKSClusters --region <Region>
# expected: 返回已有 EKS 集群列表
```

**预期输出**：

```json
{
    "TotalCount": 0,
    "Clusters": [],
    "RequestId": "..."
}
```

#### 创建新 EKS 集群（如无已有集群）

`eks-cluster.json`：

```json
{
  "ClusterName": "eks-finetune-cluster",
  "K8sVersion": "1.28.3",
  "VpcId": "VPC_ID",
  "SubnetIds": ["SUBNET_ID"],
  "ClusterDesc": "ChatGLM-6B LoRA finetune cluster",
  "ServiceSubnetId": "SERVICE_SUBNET_ID"
}
```

```bash
tccli tke CreateEKSCluster --region <Region> --cli-input-json file://eks-cluster.json
# expected: exit 0，返回 ClusterId
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `VPC_ID` | VPC 实例 ID | 与 EKS 集群同地域 | `tccli vpc DescribeVpcs --region <Region>` |
| `SUBNET_ID` | Pod 子网 ID | AvailableIpCount ≥ 5 | `tccli vpc DescribeSubnets --region <Region>` |
| `SERVICE_SUBNET_ID` | Service 子网 ID | 可与 Pod 子网相同 | `tccli vpc DescribeSubnets --region <Region>` |
| `REGION` | 地域 | 如 `ap-guangzhou` | `tccli configure list` |

### 步骤 2：获取 EKS 集群 kubeconfig 并验证连通性（数据面，需 VPN/IOA）

```bash
# 获取 kubeconfig
tccli tke DescribeEKSClusterCredential --region <Region> \
    --ClusterId CLUSTER_ID \
    | jq -r '.Kubeconfig' > ~/.kube/eks-config

export KUBECONFIG=~/.kube/eks-config

# 验证连通性
kubectl cluster-info
# expected: Kubernetes control plane 可访问

kubectl get ns
# expected: 返回 default 等系统命名空间
```

### 步骤 3：准备微调数据和模型存储

创建 PVC 存放训练数据和模型输出：

`finetune-pvc.yaml`：

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: finetune-data
  namespace: NAMESPACE
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: cbs-ssd
  resources:
    requests:
      storage: 100Gi
```

```bash
kubectl apply -f finetune-pvc.yaml
# expected: persistentvolumeclaim/finetune-data created
```

### 步骤 4：部署 LoRA 微调 Job（数据面，需 VPN/IOA）

#### 选择依据

- **LoRA rank (r=8)**：ChatGLM-6B LoRA 建议 r=8-16。r=8 平衡可训练参数量和显存消耗，适配 V100 32GB。
- **batch_size=1**：单卡 V100 微调 batch_size 设为 1（含梯度累积），确保不 OOM。
- **gradient_accumulation_steps=8**：通过梯度累积模拟更大 batch size（等效 batch_size=8），提升训练稳定性。
- **EKS GPU 注解**：`eks.tke.cloud.tencent.com/gpu-type: 'V100'` 和 `eks.tke.cloud.tencent.com/gpu-count: '1'` 告知 EKS 调度器为 Pod 分配 GPU 资源。

`chatglm-lora-job.yaml`：

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: chatglm-lora-finetune
  namespace: NAMESPACE
spec:
  ttlSecondsAfterFinished: 86400
  template:
    metadata:
      annotations:
        eks.tke.cloud.tencent.com/gpu-type: 'V100'
        eks.tke.cloud.tencent.com/gpu-count: '1'
        eks.tke.cloud.tencent.com/cpu: '8'
        eks.tke.cloud.tencent.com/mem: '64Gi'
        eks.tke.cloud.tencent.com/root-cbs-size: '100'
    spec:
      restartPolicy: Never
      containers:
      - name: trainer
        image: nvidia/cuda:12.1.0-devel-ubuntu22.04
        command:
        - /bin/bash
        - -c
        - |
          set -e
          # 安装依赖
          pip install torch transformers datasets peft accelerate \
            sentencepiece protobuf wandb -i https://pypi.tuna.tsinghua.edu.cn/simple
          # 执行 LoRA 微调脚本
          python3 /scripts/finetune_chatglm_lora.py \
            --model_name_or_path THUDM/chatglm3-6b \
            --dataset_path /data/train_dataset.jsonl \
            --output_dir /data/finetuned-model \
            --lora_r 8 \
            --lora_alpha 32 \
            --lora_dropout 0.1 \
            --per_device_train_batch_size 1 \
            --gradient_accumulation_steps 8 \
            --num_train_epochs 3 \
            --learning_rate 2e-5 \
            --fp16 True \
            --logging_steps 10 \
            --save_steps 500
        resources:
          limits:
            nvidia.com/gpu: 1
            cpu: 8
            memory: 64Gi
          requests:
            nvidia.com/gpu: 1
            cpu: 8
            memory: 64Gi
        volumeMounts:
        - name: data-volume
          mountPath: /data
        - name: scripts-volume
          mountPath: /scripts
        env:
        - name: HF_HOME
          value: /data/.cache
        - name: TRANSFORMERS_CACHE
          value: /data/.cache
        - name: WANDB_MODE
          value: "offline"
      volumes:
      - name: data-volume
        persistentVolumeClaim:
          claimName: finetune-data
      - name: scripts-volume
        configMap:
          name: finetune-script
```

```bash
kubectl apply -f chatglm-lora-job.yaml
# expected: job.batch/chatglm-lora-finetune created
```

### 步骤 5：监控微调进度（数据面，需 VPN/IOA）

```bash
# 查看 Job 状态
kubectl get jobs -n NAMESPACE chatglm-lora-finetune
# expected: COMPLETIONS 逐渐趋于 1/1

# 查看 Pod 状态
kubectl get pods -n NAMESPACE -l job-name=chatglm-lora-finetune
# expected: STATUS Running → Succeeded

# 实时查看训练日志
kubectl logs -n NAMESPACE -l job-name=chatglm-lora-finetune --follow
# expected: 显示 loss 下降、step 进度等训练指标
```

**预期输出**（训练日志）：

```text
{'loss': 2.4531, 'learning_rate': 1.8e-05, 'epoch': 0.5}
{'loss': 1.8923, 'learning_rate': 1.6e-05, 'epoch': 1.0}
{'loss': 1.5234, 'learning_rate': 1.2e-05, 'epoch': 1.5}
{'loss': 1.2145, 'learning_rate': 8.0e-06, 'epoch': 2.0}
...
Training completed. Model saved to /data/finetuned-model
```

### 步骤 6：验证微调结果（数据面，需 VPN/IOA）

```bash
# 列出输出目录中的 LoRA 权重
kubectl exec -n NAMESPACE -l job-name=chatglm-lora-finetune -- \
    ls -la /data/finetuned-model/
# expected: adapter_config.json, adapter_model.bin 等 LoRA 权重文件

# 验证模型文件大小
kubectl exec -n NAMESPACE -l job-name=chatglm-lora-finetune -- \
    du -sh /data/finetuned-model/
# expected: 约 10-50MB（LoRA 适配器权重）
```

## 验证

### 控制面（tccli）

```bash
# 确认 EKS 集群状态正常
tccli tke DescribeEKSClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterStatus 为 Running

# 确认集群节点资源
tccli tke DescribeEKSClusters --region <Region>
# expected: 返回 EKS 集群列表，目标集群就绪
```

```json
{
  "TotalCount": "<TotalCount>",
  "Clusters": "<Clusters>",
  "ClusterId": "<ClusterId>",
  "ClusterName": "<ClusterName>",
  "VpcId": "<VpcId>",
  "SubnetIds": "<SubnetIds>"
}
```

### 数据面（需 VPN/IOA）

```bash
# 验证 Job 已完成
kubectl get jobs -n NAMESPACE chatglm-lora-finetune
# expected: COMPLETIONS 1/1, DURATION 显示运行时长

# 验证 Pod 成功退出
kubectl get pods -n NAMESPACE -l job-name=chatglm-lora-finetune
# expected: STATUS Succeeded

# 验证 LoRA 权重文件完整性
kubectl exec -n NAMESPACE -l job-name=chatglm-lora-finetune -- \
    cat /data/finetuned-model/adapter_config.json
# expected: JSON 格式 LoRA 配置，含 r, lora_alpha, target_modules 等字段

# 验证 Pod 未被 OOMKilled
kubectl describe pod -n NAMESPACE -l job-name=chatglm-lora-finetune | grep -i "oom\|killed"
# expected: 无输出（无 OOM 事件）
```

```text
NAME  STATUS  AGE
...
```

## 清理

> **警告**：EKS 集群按 Pod 运行时间计费，Job 完成后 Pod 会退出（不再产生 GPU 费用），但 PVC 持续产生存储费用。如果不再需要保留微调模型权重和 EKS 集群，请按依赖倒序清理。

### 数据面（需 VPN/IOA）

```bash
# 1. 删除微调 Job（Pod 已 Succeeded，Job 仅清理元数据）
kubectl delete job chatglm-lora-finetune -n NAMESPACE
# expected: job.batch "chatglm-lora-finetune" deleted

# 2. 删除 ConfigMap（如创建了微调脚本 ConfigMap）
kubectl delete configmap finetune-script -n NAMESPACE
# expected: configmap "finetune-script" deleted

# 3. 删除 PVC（如不再需要保留微调后的模型权重）
kubectl delete pvc finetune-data -n NAMESPACE
# expected: persistentvolumeclaim "finetune-data" deleted
# 注意：删除 PVC 后，微调模型权重和训练数据将永久丢失
```

### 控制面（tccli）

```bash
# 4. 清理前状态检查
tccli tke DescribeEKSClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
# expected: 确认待删除集群的 ID 和名称

# 5. 删除 EKS 集群（如仅为此微调任务创建）
tccli tke DeleteEKSCluster --region <Region> \
    --ClusterId CLUSTER_ID
# 注意：确认集群中无其他运行中的工作负载

# 6. 验证已删除
tccli tke DescribeEKSClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
# expected: ResourceNotFound 或空列表
```

```json
{
  "TotalCount": "<TotalCount>",
  "Clusters": "<Clusters>",
  "ClusterId": "<ClusterId>",
  "ClusterName": "<ClusterName>",
  "VpcId": "<VpcId>",
  "SubnetIds": "<SubnetIds>"
}
```

> **计费提示**：EKS Pod 退出后不再产生 GPU 计费。PVC 按 GB/小时计费（CBS-SSD），删除 PVC 可停止存储计费。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `CreateEKSCluster` 返回 `InvalidParameter.VpcId` | `tccli vpc DescribeVpcs --region <Region> --VpcIds '["VPC_ID"]'` 确认 VPC 存在 | VPC 不存在或不属于当前地域/账号 | 用 `tccli vpc DescribeVpcs --region <Region>` 列出可用 VPC，使用正确的 VPC ID |
| `CreateEKSCluster` 返回 `InvalidParameter.SubnetId` | `tccli vpc DescribeSubnets --region <Region> --SubnetIds '["SUBNET_ID"]'` 确认子网存在且 AvailableIpCount > 0 | 子网不存在或 IP 已耗尽 | 更换有可用 IP 的子网。`tccli vpc DescribeSubnets --region <Region> --Filters '[{"Name":"vpc-id","Values":["VPC_ID"]}]'` 查询同 VPC 子网 |
| `CreateEKSCluster` 返回 `LimitExceeded` | `tccli tke DescribeEKSClusters --region <Region>` 统计已有 EKS 集群数 | EKS 集群数量达到地域配额上限（此为环境限制，非命令错误） | 删除不再使用的 EKS 集群后重试，或提交工单申请提升配额 |
| `DescribeEKSClusterCredential` 返回 `UnauthorizedOperation` | `tccli configure list` 检查当前凭据和 region | 子账号缺少 `tke:DescribeEKSClusterCredential` 权限（此为环境限制，非命令错误） | 联系主账号授予权限或切换到有权限的账号 |

### Job 提交成功但训练异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Pod 长时间 Pending | `kubectl describe pod -n NAMESPACE -l job-name=chatglm-lora-finetune \| grep -A5 Events` | EKS 调度 GPU 资源缓慢或子网 IP 不足、GPU 库存不足 | 检查 Events：如显示 "insufficient resource" 则 GPU 库存不足，等待或切换 GPU 类型（V100 换 T4）。如显示 "no available ip" 则子网 IP 耗尽，更换子网 |
| 训练 Pod OOMKilled | `kubectl describe pod -n NAMESPACE -l job-name=chatglm-lora-finetune \| grep OOMKilled`；`kubectl logs -n NAMESPACE -l job-name=chatglm-lora-finetune --tail=50` | 显存或内存不足。可能原因：`batch_size` 过大、未启用 `--fp16`、模型加载占用过多显存 | 确认 `--fp16 True` 已启用；将 `per_device_train_batch_size` 降为 1；减少 `gradient_accumulation_steps`；降低 `lora_r` 从 8 到 4 |
| 模型下载超时 | `kubectl logs -n NAMESPACE -l job-name=chatglm-lora-finetune \| grep -i "download\|timeout"` | HuggingFace 网络访问慢或超时 | 使用镜像源：安装脚本中添加 `HF_ENDPOINT=https://hf-mirror.com`；或将模型预先下载到 PVC 中 |
| Dataset 格式化错误 | `kubectl logs -n NAMESPACE -l job-name=chatglm-lora-finetune \| grep -i "error\|traceback"` | 训练数据格式与脚本期望不匹配（需 JSONL 格式，每行含 `instruction`/`input`/`output` 字段） | 检查 `/data/train_dataset.jsonl` 格式：每行为 JSON 对象，必须含 `instruction` 字段。示例：`{"instruction": "你好", "output": "你好！有什么可以帮助你的？"}` |
| LoRA 权重文件未生成 | `kubectl exec -n NAMESPACE -l job-name=chatglm-lora-finetune -- ls /data/finetuned-model/` | 训练过程中断（OOM/超时/脚本错误），未执行到最终保存步骤 | 检查 Job 日志最后 50 行：`kubectl logs -n NAMESPACE -l job-name=chatglm-lora-finetune --tail=50`。确认 `--save_steps` 频率是否合理，中间 checkpoint 是否存在 |
| GPU 利用率低（<50%） | `kubectl exec -n NAMESPACE -l job-name=chatglm-lora-finetune -- nvidia-smi`（训练期间） | `dataloader` 成为瓶颈（数据加载速度慢于 GPU 计算），或 `batch_size` 过小 | 增加 `dataloader_num_workers`（如设为 4），提高 `gradient_accumulation_steps` 等效增大 batch size |

## 下一步

- [在 TKE 上部署 AI 大模型（tccli）](https://cloud.tencent.com/document/product/457/116253) — GPU 模型推理部署
- [部署大模型常见问题（tccli）](https://cloud.tencent.com/document/product/457/116254) — GPU 部署排障
- [使用 TKE 快速部署 ChatGLM](https://cloud.tencent.com/document/product/457/116257) — ChatGLM 推理部署方案
- [TKE Serverless 产品概述](https://cloud.tencent.com/document/product/457/78327) — EKS 集群功能与限制
- [PEFT (LoRA) 官方文档](https://huggingface.co/docs/peft/) — LoRA 参数调优参考

## 控制台替代

[TKE 控制台 → Serverless 集群](https://console.cloud.tencent.com/tke2/ecluster)：创建 EKS 集群。微调 Job 需通过 kubectl 提交，控制台不直接支持创建训练 Job。
