# 在 TKE Serverless 上运行深度学习（tccli）
> 对照官方：[在 TKE Serverless 上运行深度学习](https://cloud.tencent.com/document/product/457/60221) · page_id `60221`
## 概述
在 TKE Serverless（EKS）集群的 GPU 超级节点上运行深度学习训练任务。通过 EKS API 管理集群生命周期，通过 `kubectl` 提交 GPU Job 声明式工作负载，利用超级节点实现 GPU 资源按需分配、训练结束后自动释放，避免传统 GPU 节点包年包月的资源闲置成本。

**调度流程**: GitHub/本地代码 -> docker build & push (TCR) -> kubectl apply GPU Job -> EKS 超级节点 (Pod) 运行

**核心 API**:
- 控制面: `tccli tke CreateEKSCluster`, `tccli tke DescribeEKSClusters`, `tccli tke DeleteEKSCluster`
- 工作负载: `kubectl apply -f train-job.yaml`, `kubectl get jobs`, `kubectl logs`, `kubectl describe`
- 镜像管理: `tccli tcr DescribeImages` (验证镜像可用性)

## 前置条件

### 环境检查

```bash
# 1. 验证 tccli 版本
tccli --version
# expected: tccli version 1.x.x

# 2. 验证 kubectl 版本
kubectl version --client --short
# expected: Client Version: v1.27.x or later

# 3. 验证凭证有效性
tccli sts GetCallerIdentity
# expected:
# {
#     "AccountId": "1xxxxxxxxx",
#     "Arn": "arn:aws:sts::1xxxxxxxxx:assumed-role/...",
#     "UserId": "1xxxxxxxxx",
#     "RequestId": "xxx"
# }

# 4. 验证 CAM Action — EKS 读/写权限
tccli tke DescribeEKSClusters \
    --region <Region> \
    --Limit 1
# expected: (含有 RequestId，无 UnauthorizedOperation)

# 5. 验证 CAM Action — TCR 读权限（拉取镜像需要）
tccli tcr DescribeInstances \
    --region <Region> \
    --Limit 1
# expected: (含有 RequestId，无 UnauthorizedOperation)
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
# 6. 查询已有 EKS 集群
tccli tke DescribeEKSClusters \
    --region <Region> \
    --Limit 20
# expected:
# {
#     "Clusters": [
#         {
#             "ClusterId": "CLUSTER_ID",
#             "ClusterName": "eks-demo",
#             "VpcId": "VPC_ID",
#             "SubnetIds": ["SUBNET_ID"],
#             "K8SVersion": "1.30.0",
#             "Status": "Running",
#             "CreatedTime": "..."
#         }
#     ],
#     "TotalCount": 1,
#     "RequestId": "xxx"
# }

# 7. 如果无 EKS 集群，创建最小集群
cat > create-eks-cluster.json <<'EOF'
{
    "ClusterName": "eks-dl-training",
    "K8SVersion": "1.30.0",
    "VpcId": "VPC_ID",
    "SubnetIds": ["SUBNET_ID"],
    "ClusterDesc": "EKS cluster for deep learning training",
    "ServiceSubnetId": "SUBNET_ID",
    "DnsServers": [
        {"Domain": "cluster.local",
         "DnsServers": ["10.0.0.2"]}
    ],
    "ClusterLevel": "L5",
    "EnableVpcCoreDNS": true
}
EOF
tccli tke CreateEKSCluster \
    --region <Region> \
    --cli-input-json file://create-eks-cluster.json
# expected:
# {
#     "ClusterId": "CLUSTER_ID",
#     "RequestId": "xxx"
# }

# 8. 验证 TCR 训练镜像可用
tccli tcr DescribeImages \
    --region <Region> \
    --RegistryId REGISTRY_ID \
    --NamespaceName NAMESPACE \
    --RepositoryName "dl-training"
# expected:
# {
#     "ImageInfoList": [
#         {
#             "ImageVersion": "pytorch-2.1.0-cuda12.1",
#             "Digest": "sha256:xxxxxxxx",
#             "Size": 8589934592
#         }
#     ],
#     "TotalCount": 1,
#     "RequestId": "xxx"
# }

# 9. 检查 GPU 资源可用性（查看超级节点状态）
# 需 VPN/IOA — 以下为参考格式
kubectl get nodes -l node.kubernetes.io/instance-type=eklet
# expected:
# NAME           STATUS   ROLES    AGE   VERSION
# eklet-xxxxx    Ready    <none>   ...   v1.30.0-tke.x
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

## 控制台与 CLI 参数映射

| 控制台参数 | CLI 参数 | Idempotency 行为 | 约束 |
|---|---|---|---|
| EKS 集群名 | `ClusterName` (CreateEKSCluster) | 同名冲突报错 | 1-63 字符，全局唯一 |
| K8s 版本 | `K8SVersion` | 创建后不可降级 | 1.30.0 |
| VPC/子网 | `VpcId` / `SubnetIds` | 创建时绑定，不可更改 | 需提前规划 |
| GPU 类型 | `nvidia.com/gpu` (资源请求) | Pod spec 声明式，可随时修改 | T4/V100/A100/A800 |
| GPU 数量 | `resources.limits.nvidia.com/gpu` | apply 即为期望状态 | 1/2/4/8 |
| 镜像 | `spec.containers[].image` | apply 即为期望状态 | TCR 完整路径 |
| Job 并行数 | `spec.completions` / `spec.parallelism` | apply 即为期望状态 | 1-N |

## 操作步骤

### 占位符说明

| 占位符 | 说明 | 约束 | 获取方式 |
|---|---|---|---|
| `CLUSTER_ID` | EKS 集群 ID | 格式 cls-xxxxxxxx | `DescribeEKSClusters` |
| `EKS_CLUSTER_ID` | 同 CLUSTER_ID | cls-xxxxxxxx | `DescribeEKSClusters` |
| `REGION` | 地域 | ap-guangzhou | 固定值 |
| `NAMESPACE` | K8s 命名空间 | 不超过 63 字符 | 自定义创建或使用 default |
| `INSTANCE_ID` | TCR 实例 ID | 格式 tcr-xxxxxxxx | `DescribeInstances` |
| `TCR_DOMAIN` | TCR 实例公网域名 | xxx.tencentcloudcr.com | `DescribeInstances.PublicDomain` |
| `REGISTRY_ID` | 同 INSTANCE_ID | tcr-xxxxxxxx | `DescribeInstances` |

#### 选择依据

在 EKS 上运行深度学习训练任务需考虑 GPU 型号、资源配置和数据接入方式:

1. **GPU 型号选择**:
   - T4 (16GB 显存): 中小模型推理、微调、入门训练
   - V100 (32GB 显存): 中大规模模型训练（BERT-Large, ResNet-152）
   - A100 (40GB/80GB 显存): 大模型训练（GPT, LLaMA 微调）
   - A800 (80GB 显存): A100 替代，同等性能
2. **超级节点 (eklet)**: 区别于传统 TKE 节点池，超级节点无需预购 GPU 机器，Pod 按需创建，GPU 资源即时分配，训练结束后 Pod 销毁即停止计费
3. **数据接入**: 训练数据可通过 NFS/CFS 挂载、COS 对象存储、或直接在镜像中打包小规模数据集
4. **TCR 镜像拉取策略**: 私有 TCR 镜像需配置 `imagePullSecrets`；如使用 TCR 内网域名需确认 EKS VPC 与 TCR 实例 VPC 互通
5. **Job vs Deployment**: 训练任务为一次性任务应使用 `Job`（自动重试、完成后退出）；在线推理使用 `Deployment`

#### 最小创建

```bash
# 步骤 1: 确认 EKS 集群状态 Running
tccli tke DescribeEKSClusters \
    --region <Region> \
    --ClusterIds '["CLUSTER_ID"]'
# expected:
# {
#     "Clusters": [
#         {
#             "ClusterId": "CLUSTER_ID",
#             "ClusterName": "eks-dl-training",
#             "Status": "Running",
#             "K8SVersion": "1.30.0",
#             "VpcId": "VPC_ID",
#             "SubnetIds": ["SUBNET_ID"]
#         }
#     ],
#     "TotalCount": 1,
#     "RequestId": "xxx"
# }

# 步骤 2: 创建命名空间（如不存在）
kubectl create namespace NAMESPACE --dry-run=client -o yaml | kubectl apply -f -
# expected: namespace/NAMESPACE created (or unchanged)
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

#### 增强配置

```bash
# 步骤 3: 创建 TCR 镜像拉取密钥（用于私有 TCR 镜像）
# 需 VPN/IOA — 以下为参考格式
tccli tcr CreateInstanceToken \
    --region <Region> \
    --RegistryId REGISTRY_ID \
    --TokenType permanent \
    --Desc "eks-image-pull-secret"
# expected:
# {
#     "Username": "1xxxxxxxxx",
#     "Token": "eyJhbGciOi...",
#     "RequestId": "xxx"
# }

kubectl create secret docker-registry tcr-secret \
    --namespace NAMESPACE \
    --docker-server=TCR_DOMAIN \
    --docker-username=1xxxxxxxxx \
    --docker-password=TOKEN_VALUE
# expected: secret/tcr-secret created

# 步骤 4: 编写 GPU 训练 Job YAML（PyTorch 单机单卡示例）
cat <<'EOF' > train-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
    name: dl-training
    namespace: NAMESPACE
    labels:
        app: dl-training
        framework: pytorch
spec:
    completions: 1
    parallelism: 1
    backoffLimit: 2
    ttlSecondsAfterFinished: 3600
    template:
        metadata:
            labels:
                app: dl-training
        spec:
            imagePullSecrets:
                - name: tcr-secret
            containers:
                - name: trainer
                  image: TCR_DOMAIN/NAMESPACE/dl-training:pytorch-2.1.0-cuda12.1
                  command:
                      - python3
                      - /workspace/train.py
                      - --epochs
                      - "10"
                      - --batch-size
                      - "64"
                      - --lr
                      - "0.001"
                  resources:
                      limits:
                          nvidia.com/gpu: "1"
                          cpu: "8"
                          memory: 32Gi
                      requests:
                          nvidia.com/gpu: "1"
                          cpu: "4"
                          memory: 16Gi
                  env:
                      - name: NVIDIA_VISIBLE_DEVICES
                        value: all
                      - name: CUDA_VISIBLE_DEVICES
                        value: "0"
                  volumeMounts:
                      - name: training-data
                        mountPath: /workspace/data
                      - name: output
                        mountPath: /workspace/output
            volumes:
                - name: training-data
                  nfs:
                      server: NFS_SERVER_IP
                      path: /dl-data
                - name: output
                  nfs:
                      server: NFS_SERVER_IP
                      path: /dl-output
            restartPolicy: Never
            terminationGracePeriodSeconds: 30
EOF

# 步骤 5: 提交训练 Job
kubectl apply -f train-job.yaml
# expected: job.batch/dl-training created

# 步骤 6: 查看 Job 状态
kubectl get job dl-training --namespace NAMESPACE
# expected:
# NAME          COMPLETIONS   DURATION   AGE
# dl-training   0/1           10s        10s

# 步骤 7: 查看 Pod 启动进度
kubectl get pods --namespace NAMESPACE -l app=dl-training
# expected:
# NAME                READY   STATUS    RESTARTS   AGE
# dl-training-xxxxx   1/1     Running   0          30s

# 步骤 8: 实时查看训练日志
kubectl logs -f job/dl-training --namespace NAMESPACE
# expected:
# Epoch 1/10 - loss: 2.3456 - accuracy: 0.4567
# Epoch 2/10 - loss: 1.8765 - accuracy: 0.5678
# ...
# Epoch 10/10 - loss: 0.1234 - accuracy: 0.9567
```

```text
NAME  STATUS  AGE
...
```

## 验证

### 控制面（tccli）

```bash
# 1. 验证 EKS 集群状态和 Pod 调度情况
tccli tke DescribeEKSClusters \
    --region <Region> \
    --ClusterIds '["CLUSTER_ID"]'
# expected:
# {
#     "Clusters": [
#         {
#             "ClusterId": "CLUSTER_ID",
#             "Status": "Running",
#             "K8SVersion": "1.30.0"
#         }
#     ],
#     "TotalCount": 1,
#     "RequestId": "xxx"
# }

# 2. 查询 EKS 容器实例（Pod 对应 EKS 实例）
tccli tke DescribeEKSContainerInstances \
    --region <Region> \
    --EksCiIds '["eksci-xxxxxxxx"]'
# expected:
# {
#     "EksCis": [
#         {
#             "EksCiId": "eksci-xxxxxxxx",
#             "Name": "dl-training-xxxxx",
#             "Status": "Running",
#             "Cpu": 8.0,
#             "Memory": 32.0,
#             "GpuCount": 1,
#             "CreationTime": "..."
#         }
#     ],
#     "TotalCount": 1,
#     "RequestId": "xxx"
# }
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

> **kubectl 数据面不可达**：CAM 拒绝公网端点 (strategyId:240463971)，内网需 VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

```bash
# 3. 验证 Job 状态为 Completed
kubectl get job dl-training --namespace NAMESPACE
# expected:
# NAME          COMPLETIONS   DURATION   AGE
# dl-training   1/1           15m        20m

# 4. 验证 Pod 已成功完成（无 GPU OOM 或崩溃）
kubectl describe job dl-training --namespace NAMESPACE
# expected:
# ...
# Pods Statuses:  0 Active / 1 Succeeded / 0 Failed

# 5. 查看训练输出文件（如保存到 NFS）
kubectl exec job/dl-training --namespace NAMESPACE -- \
    ls -lh /workspace/output/
# expected:
# total xxx
# -rw-r--r-- 1 root root xxxM model_best.pt
# -rw-r--r-- 1 root root xxxK training_metrics.json

# 6. 查看训练日志末尾确认无异常
kubectl logs job/dl-training --namespace NAMESPACE --tail=20
# expected:
# Epoch 10/10 - loss: 0.1234 - accuracy: 0.9567 - val_loss: 0.2345 - val_accuracy: 0.9234
# Training completed. Best model saved to /workspace/output/model_best.pt

# 7. 确认 GPU 使用情况（训练过程中）
kubectl exec job/dl-training --namespace NAMESPACE -- \
    nvidia-smi --query-gpu=utilization.gpu,memory.used --format=csv
# expected:
# utilization.gpu [%], memory.used [MiB]
# 95 %, 14336 MiB
```

```text
NAME  STATUS  AGE
...
```

## 清理

> **计费警告**: EKS 超级节点按 Pod 实际运行时长（含 GPU 规格）计费，Job 完成后 Pod 销毁即停止 GPU 计费。但 EKS 集群本身（控制面）持续产生管理费。NFS/CFS 存储卷持续计费。TCR 镜像存储持续计费。

> **副作用警告**: 删除 Job 会同时删除关联 Pod（由 `ttlSecondsAfterFinished` 控制延迟删除）。如训练输出仅保存在 Pod 本地存储，Job 删除后数据不可恢复。

### 控制面（tccli）

```bash
# 1. 确认 Job 已完成且无活跃 Pod（避免误删运行中任务）
kubectl get job dl-training --namespace NAMESPACE
# expected: COMPLETIONS 1/1

# 2. 删除训练 Job
kubectl delete job dl-training --namespace NAMESPACE
# expected: job.batch "dl-training" deleted

# 3. 确认 Pod 已清理
kubectl get pods --namespace NAMESPACE -l app=dl-training
# expected: No resources found in NAMESPACE namespace.

# 4. (可选) 删除 EKS 集群（如为专用测试集群）
tccli tke DeleteEKSCluster \
    --region <Region> \
    --ClusterId CLUSTER_ID
# expected:
# {
#     "RequestId": "xxx"
# }

# 5. 验证 EKS 集群已删除
tccli tke DescribeEKSClusters \
    --region <Region> \
    --ClusterIds '["CLUSTER_ID"]'
# expected:
# {
#     "Clusters": [],
#     "TotalCount": 0,
#     "RequestId": "xxx"
# }

# 6. 删除 TCR 镜像拉取密钥
kubectl delete secret tcr-secret --namespace NAMESPACE
# expected: secret "tcr-secret" deleted
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
# 7. 验证 NFS/CFS 上的训练输出已留存
ls -lh /mnt/nfs/dl-output/
# expected: (列出 model_best.pt, training_metrics.json 等输出文件)
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|---|---|---|---|
| `CreateEKSCluster` 返回 `InvalidParameterValue.SubnetId` | `DescribeSubnets` 确认子网 ID | 子网不存在或不属于指定 VPC | 确认 SubnetIds 中各子网 ID 正确且与 VpcId 匹配 |
| `CreateEKSCluster` 返回 `LimitExceeded` | `DescribeEKSClusters` 确认现有集群数量 | 每地域 EKS 集群数量配额已满 | 删除闲置集群或提交工单提升配额 |
| `kubectl apply` 返回 `Forbidden` 或 `connection refused` | 检查 kubeconfig 和 CAM 权限 | CAM 策略拒绝或 kubeconfig 过期 | 确认 kubeconfig 由 EKS 集群生成且未过期；确认 CAM 策略含 tke:Describe* |
| Pod 创建后立即 `ErrImagePull` | `kubectl describe pod POD_NAME` | TCR 私有镜像无拉取权限 | 确认 `imagePullSecrets` 配置正确，TCR Token 未过期 |
| `ErrImagePull` 且 image pull secret 正确 | `docker pull` 同地域 CVM 验证 | TCR 域名 DNS 不可达或 VPC 未打通 | 使用公网 TCR 域名（非内网域名），或配置 TCR VPC 内网访问链路 |

### 创建成功但状态异常

| 现象 | 诊断 | 根因 | 修复 |
|---|---|---|---|
| Pod 长时间 Pending (>3 分钟) | `kubectl describe pod POD_NAME` 查看 Events | GPU 资源不足或超级节点未就绪 | 换用其他 GPU 型号，或等待 GPU 资源释放；检查超级节点 (eklet) 状态 |
| 训练突然终止 `OOMKilled` | `kubectl describe pod POD_NAME` | memory limit 低于训练实际需求 | 上调 `resources.limits.memory` 到 64Gi 或更高 |
| 训练 CUDA Out of Memory | `kubectl logs POD` 含 `CUDA out of memory` | batch size 超出 GPU 显存容量 | 减小 `--batch-size` 或使用显存更大的 GPU（V100/A100） |
| Pod CrashLoopBackOff 训练脚本报错 | `kubectl logs POD` 查看 Python 堆栈 | 代码 bug 或数据格式不匹配 | 先在本地 CPU 环境调试训练脚本，确认可运行后再提交 |
| 训练极慢但 GPU 利用率正常 | `kubectl exec POD -- nvidia-smi` | NFS 数据读取成为瓶颈 | 将训练数据预加载到内存/本地盘，或升级为 CFS Turbo 高性能存储 |
| Job 完成后 Pod 未自动清理 | `ttlSecondsAfterFinished` 未设置或过大 | Pod 保留占用资源展示 | 设置 `ttlSecondsAfterFinished: 3600` 使 Pod 在完成后 1 小时自动清理 |
| 镜像拉取慢导致 Pod 启动超时 | `kubectl describe pod POD_NAME` 查看 Pulling 事件时长 | 训练镜像过大（>10GB） | 使用多阶段构建减小镜像体积；提前预热镜像到 TCR |

## 下一步

- [构建深度学习容器镜像](../构建深度学习容器镜像/tccli%20操作.md)：优化 Dockerfile，构建更小更快的训练镜像
- [通过 NAT 网关访问外网](../../通过%20NAT%20网关访问外网/tccli%20操作.md)：为训练 Pod 提供外网出口（如 wandb logging）
- [通过弹性公网 IP 访问外网](../../通过弹性公网%20IP%20访问外网/tccli%20操作.md)：为训练 Pod 分配固定公网 IP
- [EKS GPU Pod 调度最佳实践](https://cloud.tencent.com/document/product/457/xxxxx)：多卡训练、分布式训练配置

## 控制台替代
控制台完成本节操作需在 **容器服务 > Serverless 集群 > 工作负载 > Job** 页面手动填写 YAML 或通过表单配置 GPU 训练任务，然后切换至 Pod 列表页查看日志。tccli 通过 `CreateEKSCluster` 管理集群、`CreateInstanceToken` 管理镜像凭据、`kubectl` 声明式提交 Job，形成完整的训练任务自动化流水线，适合集成到 ML pipeline（如 Kubeflow、Argo Workflows）。
