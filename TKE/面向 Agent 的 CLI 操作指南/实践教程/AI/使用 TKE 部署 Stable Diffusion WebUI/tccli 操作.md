# 使用 TKE 完整部署生产级 Stable Diffusion（tccli）

> 对照官方：[使用 TKE 完整部署生产级 Stable Diffusion](https://cloud.tencent.com/document/product/457/92794) · page_id `92794`

## 概述

在 TKE 集群上部署 Stable Diffusion Web UI 推理服务。涉及：TCR 镜像管理、TKE GPU 集群创建、CFS 文件存储挂载模型文件、Deployment + LoadBalancer Service 对外暴露、qGPU 共享切分、云原生 API 网关限流及会话保持、TACO 模型推理优化。

## 前置条件

- 已创建 TKE 集群（需 GPU 节点 PNV4 / A10；Kubernetes ≥ 1.32；网络模式 GlobalRouter）。
- 已创建 [TCR 企业版实例](https://cloud.tencent.com/document/product/1141/40716)。
- 已开通 [CFS 文件存储](https://console.cloud.tencent.com/cfs)。
- 已 [连接集群](../../../集群配置/集群管理/连接集群/tccli%20操作.md)。

```bash
tccli tke DescribeClusters --region ap-guangzhou --output json \
  --filter "Clusters[?ClusterId=='<ClusterId>'] | [0].{ClusterId:ClusterId,ClusterStatus:ClusterStatus}"

tccli tke DescribeClusterNodePools --ClusterId <ClusterId> --region ap-guangzhou --output json \
  --filter "NodePoolSet[0].{Name:Name,NodeCountSummary:NodeCountSummary}"
```

```
{
  "ClusterId": "cls-example",
  "ClusterStatus": "Running"
}
{
  "Name": "gpu-pool",
  "NodeCountSummary": { "AutoscalingAdded": { "Total": 1 } }
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 拉取 SD 镜像 | `docker pull gpulab.tencentcloudcr.com/ai/stable-diffusion:1.0.7` | 是 |
| 推送镜像到 TCR | `docker tag` + `docker push` | 是（覆盖） |
| 查询集群 GPU 节点 | `tccli tke DescribeClusterInstances --ClusterId <ClusterId> --InstanceRole WORKER` | 是 |
| 创建 CFS StorageClass | `kubectl apply -f cfs-sc.yaml` | 否 |
| 创建 PV/PVC (CFS) | `kubectl apply -f cfs-pv.yaml` | 否 |
| 部署 SD Deployment | `kubectl apply -f sd-deployment.yaml` | 是 |
| 创建 LoadBalancer Service | `kubectl apply -f sd-svc.yaml` | 否 |
| qGPU 配置 | 修改 Deployment 资源 Request（annotation） | 否 |

## 操作步骤

### 1. 准备 Stable Diffusion 容器镜像

```bash
docker pull gpulab.tencentcloudcr.com/ai/stable-diffusion:1.0.7
docker tag gpulab.tencentcloudcr.com/ai/stable-diffusion:1.0.7 <tcr-registry>/ai/stable-diffusion:1.0.7
docker push <tcr-registry>/ai/stable-diffusion:1.0.7
```

### 2. 准备 GPU 集群

确认 GPU 节点就绪：

```bash
kubectl get nodes -l node.kubernetes.io/instance-type=GPU
```

```
NAME            STATUS   ROLES    AGE   VERSION
10.0.4.10       Ready    <none>   1h    v1.32.2-tke.1
```

确认 GPU 驱动可用：

```bash
kubectl describe node 10.0.4.10 | grep nvidia.com/gpu
```

```
nvidia.com/gpu:  1
nvidia.com/gpu:  1
```

### 3. 创建 CFS 存储并挂载模型

#### 3.1 准备 CFS

在 CFS 控制台创建文件系统，选择与集群相同的 VPC 和子网，创建目录 `/models/Stable-diffusion`，下载 `v1-5-pruned-emaonly.safetensors` 模型文件。

#### 3.2 创建 StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: cfs-standard
provisioner: com.tencent.cloud.csi.cfs
parameters:
  vers: "3"
  vpcid: <VpcId>
  subnetid: <SubnetId>
```

```bash
kubectl apply -f cfs-sc.yaml
```

#### 3.3 创建 PV/PVC

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: sd-models-pv
spec:
  capacity:
    storage: 100Gi
  accessModes:
  - ReadWriteMany
  csi:
    driver: com.tencent.cloud.csi.cfs
    volumeHandle: <CFS_ID>:/models/Stable-diffusion
    volumeAttributes:
      path: /models/Stable-diffusion
      vers: "3"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: sd-models-pvc
  namespace: default
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 100Gi
  volumeName: sd-models-pv
```

```bash
kubectl apply -f cfs-pv-pvc.yaml
kubectl get pvc sd-models-pvc
```

```
NAME            STATUS   VOLUME         CAPACITY   ACCESS MODES   STORAGECLASS   AGE
sd-models-pvc   Bound    sd-models-pv   100Gi      RWX                           30s
```

### 4. 部署 Stable Diffusion Web UI

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: stable-diffusion-webui
spec:
  replicas: 1
  selector:
    matchLabels:
      app: stable-diffusion
  template:
    metadata:
      labels:
        app: stable-diffusion
    spec:
      containers:
      - name: webui
        image: <tcr-registry>/ai/stable-diffusion:1.0.7
        args: ["--listen"]
        ports:
        - containerPort: 7860
        resources:
          limits:
            nvidia.com/gpu: "1"
        volumeMounts:
        - name: models
          mountPath: /dockerx/stable-diffusion-webui/models/Stable-diffusion
      volumes:
      - name: models
        persistentVolumeClaim:
          claimName: sd-models-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: sd-webui-svc
spec:
  type: LoadBalancer
  ports:
  - port: 7860
    targetPort: 7860
  selector:
    app: stable-diffusion
```

```bash
kubectl apply -f sd-deployment.yaml
kubectl get pods -w
kubectl get svc sd-webui-svc
```

```
NAME                                      READY   STATUS    AGE
stable-diffusion-webui-xxx-yyy            1/1     Running   5m
NAME           TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)          AGE
sd-webui-svc   LoadBalancer   10.0.200.100    119.xx.xx.xx     7860:30001/TCP   1m
```

浏览器访问 `http://<EXTERNAL-IP>:7860` 可打开 Stable Diffusion Web UI。

### （进阶）qGPU 切分配置

如需单卡运行多个 Pod 实例，修改资源请求：

```yaml
resources:
  limits:
    tke.cloud.tencent.com/qgpu-core: "50"
    tke.cloud.tencent.com/qgpu-memory: "10"
```

```bash
kubectl patch deployment stable-diffusion-webui -p '{"spec":{"replicas":2}}'  # 扩展到 2 个 Pod
```

### （进阶）推理性能优化（TACO）

替换镜像为 `sd_v1.5_demo`（含 TACO 优化的 UNet），创建额外的 CFS 目录 `/data` 用于存放优化模型，推理速度可提升约 30%。

## 验证

```bash
kubectl get pods -l app=stable-diffusion
kubectl logs stable-diffusion-webui-xxx-yyy | grep "Running on"
```

```
Running on local URL: http://0.0.0.0:7860
```

浏览器访问返回 Web UI 页面即为成功。

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| kubectl 无法连接集群（超时或 `Unauthorized`） | `tccli tke DescribeClusters --ClusterId <ClusterId> --region ap-guangzhou` 确认 `ClusterStatus` 为 `Running`；本地 `kubectl get nodes` 失败 | CAM 240463971：集群 API Server 为内网地址，未通过 VPN/IOA 接入 | 连接腾讯云 VPN/IOA 客户端后重试 kubectl；未接入时改用 `tccli tke` 命令完成诊断 |
| Pod 长时间 Pending，Events 提示 `Insufficient nvidia.com/gpu` | `tccli tke DescribeClusterInstances --ClusterId <ClusterId> --InstanceRole WORKER --region ap-guangzhou` 确认 GPU 节点数；`kubectl describe pod <pod>` 查看 Events | GPU 配额不足或 GPU 节点未就绪 | 在配额管理页申请 GPU 配额；确认节点为 GPU 机型（T4/V100/A10）且 GPU 驱动与 device-plugin 已安装 |
| 镜像拉取超时（大型镜像） | `kubectl describe pod <pod> \| grep -A5 Events` 查看拉取失败原因 | Stable Diffusion 镜像较大（通常 > 10GB），公网拉取可能超时 | 启用 TCR 按需加载；确认节点可访问 TCR VPC 内网地址 |
| GPU 不可用 (`nvidia.com/gpu` = 0) | `kubectl describe node <gpu-node> \| grep nvidia.com/gpu` | 节点不是 GPU 机型或 GPU 驱动未安装 | 确认节点为 GPU 机型；安装 GPU 驱动和 device-plugin |
| CFS PVC 未绑定 | `kubectl describe pvc sd-models-pvc` | CFS ID 错误或不在同一 VPC | 确认 CFS ID 正确；Cluster 与 CFS 同一 VPC |
| qGPU 显存分配不生效 | `kubectl describe pod <pod> \| grep qgpu` | `qgpu-core < 100` 时未同时指定 `qgpu-memory` | 在 Pod annotation 中同时设置 `tke.cloud.tencent.com/qgpu-core` 和 `tke.cloud.tencent.com/qgpu-memory` |

## 清理

```bash
kubectl delete service sd-webui-svc
kubectl delete deployment stable-diffusion-webui
kubectl delete pvc sd-models-pvc
kubectl delete pv sd-models-pv
```

## 下一步

- [使用 TKE 快速部署 ChatGLM](../使用 TKE 快速部署 ChatGLM/tccli 操作.md)（page_id `92795`）
- [容器服务使用 qGPU](https://cloud.tencent.com/document/product/457/65734)

## 控制台替代

控制台 **集群 → 工作负载 → Deployment → 新建**，选择 TCR 镜像，挂载 CFS PVC，配置 GPU 资源和 Service。
