# 部署 Embedding 模型到 TKE（tccli）

> 对照官方：[Embedding 模型部署](https://cloud.tencent.com/document/product/457/124002) · page_id `124002`

## 概述

在 TKE 集群上使用 vllm 推理框架部署 Embedding 模型（如 E5-mistral-7b-instruct），通过 LoadBalancer Service 暴露 OpenAI 兼容的 `/v1/embeddings` API。

Embedding 模型将文本转换为低维稠密向量，是语义搜索、推荐系统和聚类分析的基础。TKE 提供 GPU 节点的弹性算力，结合 CFS 持久化存储模型权重，支持单机单卡、单机多卡、多机多卡部署模式。

> **kubectl 数据面不可达**：CAM 拒绝公网端点（strategyId:240463971），内网需 VPN/IOA。以下 kubectl 和 helm 命令为参考格式，需在内网/VPN 环境下执行。

## 前置条件

- [环境准备](../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId、secretKey、region 均已配置

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:DescribeClusters, tke:CreateCluster, tke:InstallAddon
#    tke:DescribeClusterKubeconfig
#    cvm:DescribeInstances, cvm:DescribeInstanceConfigInfos
#    cvm:DescribeImages
# 验证
tccli tke DescribeClusters --region <Region>
# expected: exit 0

# 4. 检查 kubectl（须 VPN/IOA）
kubectl version --client
# expected: Client Version v1.32+

# 5. 检查 helm（须 VPN/IOA）
helm version
# expected: Version v3.x.x

# 如缺失 helm，安装：
# curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
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
# 6. 查询可用 TKE 集群
tccli tke DescribeClusters --region <Region>
# expected: 至少 1 个 Running 集群，网络模式为 VPC-CNI

# 7. 检查 GPU 节点可用性
tccli tke DescribeClusterInstances --region <Region> \
    --ClusterId CLUSTER_ID \
    --InstanceRole WORKER
# expected: 至少 1 个 GPU 节点（推荐 PNV5b.8XLARGE96，单卡 48GB 显存）

# 8. 检查 CFS 存储组件
tccli tke DescribeAddon --region <Region> \
    --ClusterId CLUSTER_ID --AddonName cfs
# expected: 组件状态已安装，或准备安装
```

### 版本与规格选择

| 组件 | 推荐版本/规格 | 说明 |
|------|-------------|------|
| K8s | >= 1.30.0 | VPC-CNI 网络模式 |
| GPU 节点 | PNV5b.8XLARGE96 | 单卡 48GB 显存，满足 7B 模型推理 |
| 下载节点 | SA5.LARGE8 x 3 | 3 节点并发下载，系统盘 >= 300GB |
| vllm 镜像 | `ccr.ccs.tencentyun.com/tke-ai-playbook/vllm-openai:v0.10.1-20250801` | TKE 团队维护 |
| 模型 | intfloat/e5-mistral-7b-instruct | 支持中英文，长上下文 |

## 关键字段说明

以下说明 `CreateCluster`（VPC-CNI 模式）和 `InstallAddon`（CFS 组件）的主要参数。

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `ClusterType` | String | 是 | `MANAGED_CLUSTER` | 填 `GR` 等控制台概念名 → `InvalidParameter.ClusterType` |
| `ClusterCIDRSettings.ClusterCIDR` | String | 是 | Pod IP 范围，如 `10.1.0.0/16` | CIDR 冲突 → `InvalidParameter.ClusterCIDRSettings` |
| `ContainerRuntime` | String | 是 | `containerd`（托管集群强制） | 填 `docker` → `InvalidParameter.ContainerRuntime` |
| `NetworkType` | String | 否 | 不填默认 GlobalRouter；VPC-CNI 需额外调用 `EnableVpcCniNetworkType` | 默认 GR 模式下 GPU Pod 网络性能受限 |
| `AddonName`（CFS） | String | 是 | `"cfs"` | 填错组件名 → 安装失败 |
| `GPU 节点机型` | — | — | 推荐 `PNV5b.8XLARGE96`（单卡 48GB 显存） | 选低显存节点 → `CudaOutofMemory` |


## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 |
|-----------|----------|
| 参见官方文档 | `tccli tke DescribeClusters` |

## 操作步骤

### 步骤 1：创建或确认集群（VPC-CNI 模式）

#### 选择依据

- **VPC-CNI vs GlobalRouter**：VPC-CNI 模式下 Pod IP 直接来自 VPC 子网，网络性能更高，GPU 节点间通信延迟更低，适合大模型推理。
- **CFS 存储**：模型权重文件通常在几十到几百 GB，需要持久化共享存储。CFS（文件存储）支持多 Pod 并发读取，比 CBS（块存储）更适合模型文件存储。

如果已有 VPC-CNI 集群，跳到步骤 3。

**最小创建**（仅必填字段）：

`cluster-vpc-cni-minimal.json`：

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
  }
}
```

```bash
tccli tke CreateCluster --region <Region> --cli-input-json file://cluster-vpc-cni-minimal.json
# expected: exit 0，返回 ClusterId
```

### 步骤 2：开启 VPC-CNI 并安装 CFS 组件

#### 选择依据

- **VPC-CNI**：须在集群创建后开启，指定 Pod 子网。
- **CFS**：通过 `InstallAddon` 安装，提供共享文件存储能力。

```bash
# 1. 开启 VPC-CNI 网络模式
tccli tke EnableVpcCniNetworkType --region <Region> \
    --ClusterId CLUSTER_ID \
    --VpcCniType tke-route-eni \
    --SubnetId SUBNET_ID
# expected: exit 0

# 2. 轮询 VPC-CNI 开启进度
tccli tke DescribeEnableVpcCniProgress --region <Region> \
    --ClusterId CLUSTER_ID
# expected: Status "Succeeded"

# 3. 安装 CFS 存储组件
tccli tke InstallAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName cfs
# expected: exit 0，RequestId 已返回
```

```json
{
  "Status": "<Status>",
  "ErrorMessage": "<ErrorMessage>",
  "RequestId": "<RequestId>"
}
```

### 步骤 3：创建存储资源（StorageClass + PVC）（数据面，须 VPN/IOA）

```bash
# 1. 获取 kubeconfig
tccli tke DescribeClusterKubeconfig --region <Region> --ClusterId CLUSTER_ID \
    | jq -r '.Kubeconfig' > ~/.kube/config
# expected: kubeconfig 写入成功
```

创建 StorageClass 和 PVC：

`ai-model-storage.yaml`：

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: cfs-std
provisioner: com.tencent.cloud.csi.cfs
parameters:
  vpcid: VPC_ID
  subnetid: SUBNET_ID
  vers: "3"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ai-model
  namespace: default
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: cfs-std
  resources:
    requests:
      storage: 500Gi
```

```bash
kubectl apply -f ai-model-storage.yaml
# expected: storageclass.storage.k8s.io/cfs-std created, persistentvolumeclaim/ai-model created

# 确认 PVC 绑定
kubectl get pvc ai-model
# expected: STATUS 为 Bound
```

```text
NAME  STATUS  AGE
...
```

### 步骤 4：下载 Embedding 模型（数据面，须 VPN/IOA）

#### 选择依据

- **tke-ai-playbook**：TKE 团队维护的开源脚本集，支持多节点并发下载模型，避免单节点带宽瓶颈。
- **并发度**：`--completions 3 --parallelism 3` 使用 3 个节点并发下载，推荐与下载节点数量一致。

```bash
# 1. 安装依赖
yum install -y jq git

# 2. 克隆 tke-ai-playbook
git clone https://github.com/tkestack/tke-ai-playbook.git
cd tke-ai-playbook

# 3. 执行模型下载（E5-mistral-7b-instruct）
bash scripts/tke-llm-downloader.sh \
    --pvc ai-model \
    --completions 3 \
    --parallelism 3 \
    --model intfloat/e5-mistral-7b-instruct
# expected: 3 个 Pod 状态 Completed，模型文件已写入 PVC
```

验证下载完成：

```bash
kubectl get pods -l job-name=tke-llm-downloader
# expected: 所有 Pod STATUS 为 Completed
```

### 步骤 5：部署 vllm 推理服务（数据面，须 VPN/IOA）

#### 选择依据

- **vllm-inference-tke Helm Chart**：TKE 团队维护，预配置 vllm OpenAI 兼容 API，支持 Embedding 接口。
- **maxModelLen: 2048**：匹配 E5-mistral-7b-instruct 的最大上下文长度。
- **tpSize: 1**：张量并行度为 1（单 GPU 部署）。

`vllm-values.yaml`：

```yaml
model:
  name: "intfloat/e5-mistral-7b-instruct"
  pvc:
    enabled: true
    name: "ai-model"
    path: "intfloat/e5-mistral-7b-instruct"
  local:
    enabled: false
    path: ""

server:
  replicas: 1
  lwsGroupSize: 1
  image: "ccr.ccs.tencentyun.com/tke-ai-playbook/vllm-openai:v0.10.1-20250801"
  imagePullPolicy: IfNotPresent
  apiKey: ""
  resources:
    requests:
      nvidia.com/gpu: 1
    limits:
      nvidia.com/gpu: 1
  args:
    tpSize: 1
    ppSize: 1
    epEnabled: false
    maxModelLen: 2048
    maxBatchSize: 32
  extraArgs:
  - --disable-log-requests
  - --cuda-graph-sizes 1 2 4 8 16 24 32
  env:
  - name: VLLM_WORKER_MULTIPROC_METHOD
    value: "spawn"
  service:
    enabled: true
    type: LoadBalancer
    port: 60000

labels: {}
podAnnotations: {}
```

```bash
cd tke-ai-playbook/helm-charts/vllm-inference-tke
helm install vllm-service . -f vllm-values.yaml
# expected: NAME: vllm-service, STATUS: deployed
```

### 步骤 6：获取推理服务入口

```bash
# 等待 LoadBalancer IP 分配
kubectl get svc | grep vllm-service
# expected: EXTERNAL-IP 不为 <pending>

# 查看 Pod 状态
kubectl get pods -l app=vllm-service
# expected: STATUS Running，READY 1/1

# 查看日志确认模型加载完成
kubectl logs -f POD_NAME | grep "Uvicorn running"
# expected: "Uvicorn running on http://0.0.0.0:60000"
```

```text
NAME  STATUS  AGE
...
```

## 验证

### 控制面（tccli）

```bash
# 确认集群状态
tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterStatus "Running"
```

**预期输出**（以真实集群 `cls-xxxxxxxx` 为例）：

```json
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "cls-xxxxxxxx",
            "ClusterName": "my-tke-cluster",
            "ClusterStatus": "Running",
            "ClusterType": "MANAGED_CLUSTER",
            "ClusterVersion": "1.30.0",
            "ClusterNodeNum": 3
        }
    ],
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

```bash
# 确认 CFS 组件状态
tccli tke DescribeAddon --region <Region> \
    --ClusterId CLUSTER_ID --AddonName cfs
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

### 数据面（须 VPN/IOA）

```bash
# 1. 确认 vllm Pod 运行
kubectl get pods -l app=vllm-service
# expected: STATUS Running

# 2. 确认 Service IP
kubectl get svc vllm-service -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
# expected: 返回公网 IP

# 3. 测试 Embedding API
curl -X POST http://SERVICE_IP:60000/v1/embeddings \
  -H 'Content-Type: application/json' \
  -d '{
    "model": "intfloat/e5-mistral-7b-instruct",
    "input": "用Python实现快速排序算法"
  }'
# expected: HTTP 200，返回 embedding 向量（长数组）

# 4. 验证嵌入向量维度
curl -s -X POST http://SERVICE_IP:60000/v1/embeddings \
  -H 'Content-Type: application/json' \
  -d '{
    "model": "intfloat/e5-mistral-7b-instruct",
    "input": "Hello world"
  }' | jq '.data[0].embedding | length'
# expected: 4096（E5-mistral-7b 的嵌入维度）
```

```text
NAME  STATUS  AGE
...
```

## 清理

> **警告**：以下操作将删除 GPU 推理服务、模型存储和集群节点。GPU 节点按量计费，删除前确认不会影响其他业务。`DeleteCluster` 配合 `InstanceDeleteMode: "terminate"` 会级联删除所有关联 CVM 和磁盘。

### 数据面清理（须 VPN/IOA，在控制面之前）

```bash
# 1. 清理前状态检查
kubectl get all -l app=vllm-service
helm list
# 确认是待删除的目标资源

# 2. 卸载 Helm Release
helm uninstall vllm-service
# expected: release "vllm-service" uninstalled

# 3. 删除 PVC（释放 CFS 存储）
kubectl delete pvc ai-model
# expected: persistentvolumeclaim "ai-model" deleted

# 4. 验证已删除
kubectl get pods -l app=vllm-service
# expected: No resources found
```

```text
NAME  STATUS  AGE
...
```

### 控制面清理（tccli）

```bash
# 5. 删除集群（如为本页新建的专用集群）
# ⚠️ InstanceDeleteMode terminate 会删除所有关联 CVM
tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
# 确认是待删除的目标集群

tccli tke DeleteCluster --region <Region> \
    --ClusterId CLUSTER_ID \
    --InstanceDeleteMode terminate
# ⚠️ 将删除集群及所有关联 CVM、磁盘

# 6. 验证已删除
tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
# expected: ResourceNotFound 或空列表
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

> 如集群为共享使用，仅卸载 `vllm-service` Helm Release 和删除 `ai-model` PVC 即可。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `InstallAddon cfs` 返回错误 | `tccli tke DescribeAddon --region <Region> --ClusterId CLUSTER_ID --AddonName cfs` 检查安装状态 | 集群版本不兼容或组件已安装 | 确认 K8s >= 1.30；如已安装则跳过 |
| `EnableVpcCniNetworkType` 返回 `InvalidParameter` | 检查 SubnetId 是否属于集群 VPC | 子网 ID 不匹配或子网 IP 不足 | `tccli vpc DescribeSubnets --region <Region>` 确认子网存在且 IP 充足（>= 64） |
| Docker pull TCR 镜像超时 | `docker pull ccr.ccs.tencentyun.com/tke-ai-playbook/vllm-openai:v0.10.1-20250801` | 网络延迟或节点无公网 IP | 为节点配置 NAT 网关或使用 TCR 内网端点 |

### 部署成功但服务异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| vllm Pod 日志显示 `model not found` | `kubectl logs POD_NAME` 查看日志 | 模型下载未完成或 PVC 路径不匹配 | `kubectl get pods -l job-name=tke-llm-downloader` 确认下载 Pod 全部 Completed；检查 PVC path 与脚本一致 |
| vllm Pod 日志显示 `CudaOutofMemory` | `kubectl describe pod POD_NAME` 查看 GPU 分配 | GPU 显存不足（7B 模型需 >= 36GB） | 选择更大显存 GPU 节点（如 PNV5b.8XLARGE96，单卡 48GB） |
| Embedding API 返回 500 | `curl -v http://SERVICE_IP:60000/v1/embeddings` 查看详细错误 | 模型推理异常或请求格式错误 | 检查 `model` 字段值是否与 values.yaml 中 `model.name` 一致 |
| 下载 Pod 一直 Running | `kubectl logs POD_NAME` 查看下载进度 | 模型文件大，下载耗时长；或网络带宽不足 | 等待完成；确保下载节点带宽 >= 100Mbps |
| PVC 无法 Bound | `kubectl describe pvc ai-model` 查看 Events | CFS 组件未安装或 StorageClass 参数错误 | `tccli tke DescribeAddon --region <Region> --ClusterId CLUSTER_ID --AddonName cfs` 确认已安装 |

## 下一步

- [AI 网关部署](../../AI%20模型部署/AI%20网关部署/tccli%20操作.md) -- page_id `123918`
- [Agent 代码沙箱工具部署](../../Agent%20智能体应用/Agent%20代码沙箱工具部署/tccli%20操作.md) -- page_id `123916`
- [tke-ai-playbook 项目](https://github.com/tkestack/tke-ai-playbook) — 更多 AI 模型部署脚本
- [vllm 官方文档](https://docs.vllm.ai/)
- [ModelScope 模型库](https://www.modelscope.cn/models) — 模型名称参考

## 控制台替代

[容器服务控制台](https://console.cloud.tencent.com/tke2/cluster)
