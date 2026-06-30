
# 在 TKE 上使用 AIBrix 进行多节点分布式推理（tccli）

> 对照官方：[在 TKE 上使用 AIBrix 进行多节点分布式推理](https://cloud.tencent.com/document/product/457/117827) · page_id `117827`

## 概述

在 TKE GPU 集群中部署 AIBrix 框架，将大语言模型（LLM）的推理负载拆分到多个 GPU 节点，通过流水线并行（Pipeline Parallelism）和张量并行（Tensor Parallelism）突破单节点显存和算力瓶颈，实现多节点分布式推理。

> **kubectl 数据面不可达**：CAM 拒绝公网端点（strategyId:240463971），内网需 VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

| 维度 | 单节点推理 | AIBrix 多节点分布式推理 |
|------|-----------|----------------------|
| 模型支持 | 受单节点显存限制（如 8×H800 = 640GB） | 跨节点聚合显存（N×8×H800），支持超大模型（如 DeepSeek-R1 671B） |
| 吞吐量 | 受单节点 GPU 计算限制 | 线性扩展（Pipeline Parallelism + 多副本） |
| 首 Token 延迟 | 较低（无跨节点通信） | 略高（流水线气泡效应），可接受范围 |
| 部署复杂度 | 简单，单 Deployment | 需配置 Ray 集群 + AIBrix 控制器 |
| 适用场景 | 模型 < 单节点显存，低 QPS | 模型 > 单节点显存，高并发推理服务 |

**选择依据**：当模型参数量超过单节点总显存（如 70B 以上模型在 FP16 精度下约需 140GB 以上），或被服务化后需要高吞吐时，选择 AIBrix 多节点分布式推理。AIBrix 基于 Ray 分布式框架，内置自动模型切分和请求路由。

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
#    tke:DescribeClusters, tke:DescribeClusterNodePools
#    cvm:DescribeInstances
# 验证：执行 DescribeClusters 确认权限
tccli tke DescribeClusters --region <Region>
# expected: exit 0，返回集群列表

# 4. 检查 kubectl 和 helm（需在内网环境下）
kubectl version --client
# expected: Client Version >= v1.28

helm version --short
# expected: v3.x.x
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
# 5. 确认目标集群存在且状态 Running
tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
# expected: TotalCount >= 1，ClusterStatus: "Running"，ClusterVersion >= 1.28

# 6. 确认 GPU 节点池有多个节点（多节点推理前提）
tccli tke DescribeClusterNodePools --region <Region> \
    --ClusterId CLUSTER_ID \
    | jq '.NodePoolSet[] | {Name, LifeState, DesiredNodesNum, InstanceType}'
# expected: 至少 1 个 GPU 节点池，DesiredNodesNum >= 2（建议 >= 4），InstanceType 为 GPU 实例
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

### 版本与规格要求

- K8s 版本：>= 1.28。TKE 托管集群。
- GPU 节点：每个节点 >= 4 张 GPU（推荐 8 张 NVIDIA H800/A800/H100），节点间建议 RDMA 网络（可选，影响 Tensor Parallel 跨节点效率）。
- 显存要求：根据模型大小估算。以 70B 模型 FP16 为例，单节点 8×H800（640GB）可容纳，但多节点分布式推理能提供更高吞吐。

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看集群列表 | `DescribeClusters` | 是 |
| 查看节点池列表 | `DescribeClusterNodePools` | 是 |
| 获取集群 kubeconfig | `DescribeClusterKubeconfig` | 是 |

## 操作步骤

### 步骤 1：验证 GPU 节点池配置

#### 选择依据

多节点分布式推理需要至少 2 个 GPU 节点（越多吞吐越高）。通过 `DescribeClusterNodePools` 确认节点池中的 GPU 节点数量、实例类型和状态。建议使用 HCC 系列实例以获得 RDMA 网络支持，但如果模型切分仅在 Pipeline Parallelism 层面（而非 Tensor Parallelism），普通 GPU 实例也可用。

```bash
tccli tke DescribeClusterNodePools --region <Region> \
    --ClusterId CLUSTER_ID
# expected: 返回 GPU 节点池列表，NodesNum >= 2
```

**预期输出**：

```json
{
    "NodePoolSet": [
        {
            "NodePoolId": "np-example",
            "Name": "gpu-pool",
            "ClusterId": "CLUSTER_ID",
            "LifeState": "normal",
            "DesiredNodesNum": 4,
            "InstanceType": "HCCPNV4h",
            "NodeCountSummary": {
                "AutoscalingAdded": {
                    "Total": 4,
                    "Normal": 4
                }
            }
        }
    ],
    "TotalCount": 1,
    "RequestId": "..."
}
```

确认 GPU 节点在集群中 Ready：

```bash
kubectl get nodes -l node.kubernetes.io/instance-type=HCCPNV4h
# expected: >= 2 个节点，STATUS: Ready
```

**预期输出**：

```text
NAME                STATUS   ROLES    AGE   VERSION
10.0.1.15           Ready    <none>   5m    v1.28.3-tke.5
10.0.1.16           Ready    <none>   5m    v1.28.3-tke.5
10.0.1.17           Ready    <none>   5m    v1.28.3-tke.5
10.0.1.18           Ready    <none>   5m    v1.28.3-tke.5
```

### 步骤 2：安装 AIBrix 控制器

#### 选择依据

AIBrix 提供 K8s Operator 管理分布式推理的生命周期。通过 Helm 安装 AIBrix，包括 CRD 定义、控制器、Ray 集群管理组件。AIBrix 封装了 Ray Serve 的复杂性，提供声明式 API 管理推理服务。

```bash
helm repo add aibrix https://aibrix.github.io/helm-charts
# expected: "aibrix" has been added to your repositories

helm repo update
# expected: Hang tight while we grab the latest from your chart repositories... Update Complete.

helm install aibrix aibrix/aibrix-operator \
    --namespace aibrix-system \
    --create-namespace \
    --set controller.replicas=1 \
    --set image.tag=v0.2.0
# expected: NAME: aibrix, STATUS: deployed
```

验证控制器部署：

```bash
kubectl get pods -n aibrix-system
# expected: aibrix-controller Pod Running

kubectl get crd | grep aibrix
# expected: 列出 AIBrix CRD（如 aibrixservingruntimes, aibrixrayclusters 等）
```

**预期输出**：

```text
NAME                              CREATED AT
aibrixrayclusters.aibrix.io       2024-01-01T00:00:00Z
aibrixservingruntimes.aibrix.io   2024-01-01T00:00:00Z
```

### 步骤 3：部署 Ray 分布式集群

#### 选择依据

AIBrix 底层依赖 Ray 集群管理分布式计算资源。Ray Head 节点负责调度和协调，Ray Worker 节点（对应每个 GPU 节点）负责执行推理计算。Worker 节点数量即为参与分布式推理的 GPU 节点数。配置 `numGpus` 告知 Ray 每节点可用 GPU 数。

`ray-cluster.yaml`：

```yaml
apiVersion: aibrix.io/v1
kind: AIBrixRayCluster
metadata:
  name: llm-ray-cluster
  namespace: default
spec:
  rayVersion: "2.9.0"
  head:
    replicas: 1
    image: rayproject/ray:2.9.0-py310
    resources:
      requests:
        cpu: 4
        memory: 16Gi
      limits:
        cpu: 8
        memory: 32Gi
    serviceType: ClusterIP
  worker:
    replicas: 4
    minReplicas: 2
    maxReplicas: 8
    image: rayproject/ray:2.9.0-py310-gpu
    resources:
      requests:
        cpu: 32
        memory: 256Gi
        nvidia.com/gpu: 8
      limits:
        cpu: 64
        memory: 512Gi
        nvidia.com/gpu: 8
    nodeSelector:
      node.kubernetes.io/instance-type: HCCPNV4h
    volumeMounts:
    - name: model-storage
      mountPath: /models
    volumes:
    - name: model-storage
      persistentVolumeClaim:
        claimName: model-pvc
```

```bash
kubectl apply -f ray-cluster.yaml
# expected: aibrixraycluster.aibrix.io/llm-ray-cluster created
```

等待 Ray 集群就绪：

```bash
kubectl get aibrixraycluster llm-ray-cluster
# expected: STATUS: Ready, WORKERS: 4/4

kubectl get pods -l ray.io/cluster=llm-ray-cluster
# expected: 1 个 head pod + 4 个 worker pod，全部 Running
```

**预期输出**：

```text
NAME                             READY   STATUS    RESTARTS   AGE
llm-ray-cluster-head-xxxxx       1/1     Running   0          2m
llm-ray-cluster-worker-xxxxx-0   1/1     Running   0          2m
llm-ray-cluster-worker-xxxxx-1   1/1     Running   0          2m
llm-ray-cluster-worker-xxxxx-2   1/1     Running   0          2m
llm-ray-cluster-worker-xxxxx-3   1/1     Running   0          2m
```

### 步骤 4：部署模型（vLLM + AIBrix Serving Runtime）

#### 选择依据

AIBrix Serving Runtime 封装 vLLM，支持流水线并行（Pipeline Parallelism）和张量并行（Tensor Parallelism）。`pipelineParallelSize` 控制模型拆分到几个节点（stage），每个阶段负责模型的一部分层。`tensorParallelSize` 控制每阶段内张量拆分的 GPU 数（需节点内有足够 GPU 互联带宽）。

**关键参数选择指南**：

- **Pipeline Parallelism**：按节点数拆分模型层。如 4 节点、80 层的 Llama-70B，每节点处理约 20 层。优点：跨节点通信少（仅前向/反向传递中间激活值），适合跨节点场景。
- **Tensor Parallelism**：按 GPU 拆分单层运算。8 GPU per node 设置为 8。需要高带宽 GPU 互联（NVLink + NVSwitch），**不建议跨节点 Tensor Parallel**，除非有 RDMA 网络。
- **建议配置**：pipelineParallelSize = 节点数，tensorParallelSize = 单节点 GPU 数。

`aibrix-serving.yaml`（以 Llama-3.1-70B 为例）：

```yaml
apiVersion: aibrix.io/v1
kind: AIBrixServingRuntime
metadata:
  name: llama-70b
  namespace: default
spec:
  rayClusterRef:
    name: llm-ray-cluster
  engine: vllm
  model:
    name: meta-llama/Meta-Llama-3.1-70B-Instruct
    storageUri: "oss://MODEL_BUCKET/llama-3.1-70b/"
    hfTokenSecretRef:
      name: hf-token
      key: token
  parallelism:
    pipelineParallelSize: 4
    tensorParallelSize: 8
  resources:
    perWorker:
      gpu: 8
      cpu: 32
      memory: 256Gi
  scaling:
    minReplicas: 1
    maxReplicas: 4
  serverArgs:
  - --max-model-len
  - "8192"
  - --max-num-seqs
  - "128"
  - --gpu-memory-utilization
  - "0.95"
  - --enable-prefix-caching
```

```bash
kubectl apply -f aibrix-serving.yaml
# expected: aibrixservingruntime.aibrix.io/llama-70b created
```

等待模型加载并就绪：

```bash
kubectl get aibrixservingruntime llama-70b
# expected: STATUS: Running, REPLICAS: 1/1

kubectl get svc llama-70b
# expected: ClusterIP service created for inference endpoint
```

**预期输出**：

```text
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
llama-70b    ClusterIP   10.100.100.100  <none>        8000/TCP   5m
```

### 步骤 5：测试分布式推理性能

#### 选择依据

使用简单的 curl 请求验证推理接口可访问后，运行基准测试测量 tokens/s 吞吐量。AIBrix 自动将请求路由到可用的模型副本。

```bash
# 获取推理服务 endpoint
kubectl get svc llama-70b -o jsonpath='{.spec.clusterIP}'
# expected: service ClusterIP

# 发送测试推理请求（在内网 Pod 中执行）
kubectl run inference-test --rm -it --image=python:3.10 --restart=Never -- \
    python -c "
import requests, time, json
url = 'http://llama-70b.default.svc.cluster.local:8000/v1/completions'
payload = {'model': 'meta-llama/Meta-Llama-3.1-70B-Instruct', 'prompt': 'Explain quantum computing in one paragraph.', 'max_tokens': 256}
start = time.time()
resp = requests.post(url, json=payload, timeout=300)
elapsed = time.time() - start
data = resp.json()
tokens = data['usage']['completion_tokens']
print(f'Tokens: {tokens}, Time: {elapsed:.2f}s, Throughput: {tokens/elapsed:.1f} tokens/s')
print(f'Response: {data[\"choices\"][0][\"text\"][:200]}...')
"
# expected: 返回推理结果和 tokens/s 指标
```

**预期输出**：

```text
Tokens: 256, Time: 3.45s, Throughput: 74.2 tokens/s
Response: Quantum computing is a revolutionary approach to computation that leverages the principle...
```

## 验证

### 控制面（tccli）

确认集群和节点池状态正常：

```bash
tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterStatus: "Running"

tccli tke DescribeClusterNodePools --region <Region> \
    --ClusterId CLUSTER_ID \
    | jq '.NodePoolSet[] | {Name, LifeState, DesiredNodesNum, NodesNum: .NodeCountSummary.AutoscalingAdded.Normal}'
# expected: LifeState: "normal"，NodesNum >= 2
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
# 验证 Ray 集群状态
kubectl get aibrixraycluster llm-ray-cluster -o json | jq '.status'
# expected: status.phase: "Ready"，status.workerReplicas: 4

# 验证 AIBrix Serving Runtime 状态
kubectl get aibrixservingruntime llama-70b -o json | jq '.status'
# expected: status.phase: "Running"，status.readyReplicas >= 1

# 验证推理端点可达
kubectl run curl-test --rm -it --image=curlimages/curl:latest --restart=Never -- \
    curl -s -o /dev/null -w "%{http_code}" http://llama-70b.default.svc.cluster.local:8000/health
# expected: 200
```

```text
NAME  STATUS  AGE
...
```

| 验证维度 | 命令 | 预期 |
|---------|------|------|
| 集群状态 | `DescribeClusters` | `ClusterStatus: "Running"` |
| GPU 节点数 | `kubectl get nodes` | >= 2，Ready |
| Ray 集群 | `kubectl get aibrixraycluster` | STATUS: Ready |
| AIBrix 控制器 | `kubectl get pods -n aibrix-system` | Running |
| 推理服务 | `kubectl get aibrixservingruntime` | STATUS: Running |
| 推理连通性 | curl `/health` endpoint | HTTP 200 |

## 清理

> **计费警告**：GPU 节点和 Ray 集群即使无推理请求也会持续占用 GPU 资源。GPU 节点按量计费，建议测试完成后及时删除或缩容。
> **副作用警告**：删除 Ray 集群会终止所有 Worker Pod，正在处理的推理请求将丢失。

### 数据面（需 VPN/IOA）

```bash
# 1. 删除 AIBrix Serving Runtime（先停止推理服务）
kubectl delete aibrixservingruntime llama-70b
# expected: aibrixservingruntime.aibrix.io "llama-70b" deleted

# 2. 删除 Ray 集群
kubectl delete aibrixraycluster llm-ray-cluster
# expected: aibrixraycluster.aibrix.io "llm-ray-cluster" deleted

# 3. 卸载 AIBrix 控制器
helm uninstall aibrix -n aibrix-system
# expected: release "aibrix" uninstalled

kubectl delete namespace aibrix-system
# expected: namespace "aibrix-system" deleted

# 4. 清理模型存储 PVC（确认数据不再需要）
kubectl delete pvc model-pvc
# expected: persistentvolumeclaim "model-pvc" deleted
```

### 控制面（tccli）

GPU 节点保留（仅删除推理工作负载），如需缩容节点池：

```bash
# 缩容节点池到 0（保留池配置）
tccli tke ModifyClusterNodePool --region <Region> \
    --ClusterId CLUSTER_ID \
    --NodePoolId NP_ID \
    --DesiredNodesNum 0
# expected: exit 0
# 节点池保留，所有 CVM 实例被释放
```

验证清理：

```bash
tccli tke DescribeClusterNodePools --region <Region> \
    --ClusterId CLUSTER_ID
# expected: DesiredNodesNum: 0（如已缩容）
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

```bash
kubectl get pods -l ray.io/cluster=llm-ray-cluster 2>/dev/null
# expected: No resources found in default namespace.
```

```text
NAME  STATUS  AGE
...
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DescribeClusterNodePools` 返回空列表 | `tccli tke DescribeClusterNodePools --region <Region> --ClusterId CLUSTER_ID` 检查输出 | 集群无节点池或集群 ID 不正确 | 先通过 `DescribeClusters` 确认集群 ID，然后在控制台或 CLI 创建 GPU 节点池 |
| `DescribeClusterNodePools` 返回 GPU 节点数为 0 | 同上，检查 `NodeCountSummary` 字段 | 节点池中无就绪 GPU 节点（可能自动扩缩容缩为 0） | 执行缩容恢复：`tccli tke ModifyClusterNodePool ... --DesiredNodesNum 2` |
| `helm install aibrix` 失败 `ImagePullBackOff` | `kubectl describe pod -n aibrix-system` 查看事件 | AIBrix 镜像拉取失败，镜像仓库不可达 | 检查网络代理/VPN；使用国内镜像或配置 imagePullSecrets |

### 推理服务部署成功但无响应

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Ray Worker Pod 一直 `Pending` | `kubectl describe pod -l ray.io/cluster=llm-ray-cluster` 查看事件 | GPU 资源不足或节点标签不匹配 | 检查节点是否有足够 `nvidia.com/gpu` 资源。确认 nodeSelector 与节点标签匹配。保留 ClusterId、RequestId 排查 |
| AIBrix Serving Runtime 一直 `Loading` | `kubectl logs -l aibrix.io/runtime=llama-70b` 查看日志 | 模型下载/加载慢或 OOM | 检查模型存储路径是否可达；大模型加载需 10-30 分钟，等待超时后检查日志；`kubectl describe aibrixservingruntime llama-70b` 查看状态事件 |
| 推理返回 `503 Service Unavailable` | `kubectl get svc llama-70b` 确认 Service 存在，`kubectl get endpoints llama-70b` 确认有后端 Pod | Ray Serve 还未就绪或后端 Pod 重启 | 等待 Ray Serve 初始化完成（首次加载模型需几分钟）；检查 `kubectl logs` 确认模型加载进度 |
| 推理吞吐低（<10 tokens/s） | `kubectl exec` 进入 head Pod，执行 `ray status` 查看集群资源利用率 | pipelineParallelSize 设置过小或 Worker 数不足 | 增加 `pipelineParallelSize`（拆分更多 Stage）或增加 `minReplicas`
| 推理时 GPU OOM | `kubectl logs -l aibrix.io/runtime=llama-70b \| grep -i "out of memory"` 查看 OOM 日志 | 模型太大，单节点显存不足 | 增加 `pipelineParallelSize` 以拆分模型到更多节点，或降低 `--gpu-memory-utilization` 至 0.85 |

## 下一步

- [在 TKE 上使用 RDMA 加速 AI 推理和训练](https://cloud.tencent.com/document/product/457/116720) — RDMA 网络优化分布式推理通信
- [在 TKE 上部署满血版 DeepSeek-R1 (SGLang)](https://cloud.tencent.com/document/product/457/103984) — 超大模型部署方案
- [GPU 资源管理](https://cloud.tencent.com/document/product/457/32205) — GPU 共享与调度
- [vLLM 官方文档](https://docs.vllm.ai/) — vLLM 推理引擎参数调优
- [Ray Serve 文档](https://docs.ray.io/en/latest/serve/index.html) — 分布式推理服务化

## 控制台替代

[TKE 控制台 → 集群 → 工作负载](https://console.cloud.tencent.com/tke2/cluster)：通过 Helm 或 YAML 部署 AIBrix 工作负载，查看 Pod 状态和日志。
