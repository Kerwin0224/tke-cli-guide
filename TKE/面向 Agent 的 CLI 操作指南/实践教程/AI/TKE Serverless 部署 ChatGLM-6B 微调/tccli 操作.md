# TKE Serverless 运行 ChatGLM-6B 微调（tccli）

> 对照官方：[TKE Serverless 运行 ChatGLM-6B 微调](https://cloud.tencent.com/document/product/457/97823) · page_id `97823`

## 概述

基于 TKE Serverless 集群，利用 P-Tuning v2 方法对 ChatGLM-6B 进行微调。涉及：TCR 镜像制作与推送、CFS 文件存储挂载（模型/数据集/checkpoint）、Serverless Job 创建。使用单张 V100 GPU 完成微调。

## 前置条件

- 已创建 [TKE Serverless 集群](https://console.cloud.tencent.com/tke2/ecluster/startUp)。
- 已创建 [TCR 镜像仓库](https://console.cloud.tencent.com/tcr)（个人版/企业版均可）。
- 已开通 [CFS 文件存储](https://console.cloud.tencent.com/cfs)。
- 已 [连接集群](../../../集群配置/集群管理/连接集群/tccli%20操作.md)。

```bash
tccli tke DescribeClusters --region ap-guangzhou --output json \
  --filter "Clusters[?ClusterId=='<ClusterId>'] | [0].{ClusterId:ClusterId,ClusterStatus:ClusterStatus}"
```

```
{
  "ClusterId": "cls-example",
  "ClusterStatus": "Running"
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 克隆微调代码 | `git clone https://github.com/coderwangke/tke-run-chatglm.git` | 否 |
| 制作 Docker 镜像 | `docker build -f Dockerfile.V100 -t <tag> .` | 否 |
| 推送镜像到 TCR | `docker push <tag>` | 是（覆盖） |
| 下载模型到 CFS | 挂载 CFS 后 `git clone` 模型仓库 | 否 |
| 安装 CFS 组件 | `tccli tke InstallAddon --AddonName CFS` | 是 |
| 创建 PV/PVC (CFS) | `kubectl apply -f cfs-pv.yaml` | 否 |
| 创建 Job | `kubectl apply -f finetune-job.yaml` | 否 |

## 操作步骤

### 1. 镜像准备

```bash
git clone https://github.com/coderwangke/tke-run-chatglm.git
cd tke-run-chatglm/finetune
docker build -f Dockerfile.V100 -t ccr.ccs.tencentyun.com/chatglm/chatglm-6b-ptv2:v1.0 .
docker login ccr.ccs.tencentyun.com
docker push ccr.ccs.tencentyun.com/chatglm/chatglm-6b-ptv2:v1.0
```

### 2. 存储准备

在 CFS 中创建目录结构并下载模型：

```bash
sudo mount -t nfs -o vers=3,nolock,proto=tcp,noresvport <CFS_IP>:/ /mnt/cfs
mkdir -p /mnt/cfs/models /mnt/cfs/datasets /mnt/cfs/output
cd /mnt/cfs/models
git lfs install
git clone https://huggingface.co/THUDM/chatglm-6b
```

下载 ADGEN 数据集到 `/mnt/cfs/datasets`。

### 3. 安装 CFS 组件

```bash
tccli tke InstallAddon --ClusterId <ClusterId> --AddonName CFS --region ap-guangzhou
```

### 4. 创建 PV/PVC

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: cfs-pv
spec:
  capacity:
    storage: 200Gi
  accessModes:
  - ReadWriteMany
  csi:
    driver: com.tencent.cloud.csi.cfs
    volumeHandle: <CFS_ID>:
    volumeAttributes:
      path: /
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cfs-pvc
  namespace: default
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 200Gi
  volumeName: cfs-pv
```

```bash
kubectl apply -f cfs-pv-pvc.yaml
kubectl get pvc cfs-pvc
```

```
NAME      STATUS   VOLUME   CAPACITY   ACCESS MODES   AGE
cfs-pvc   Bound    cfs-pv   200Gi      RWX            10s
```

### 5. 创建微调 Job

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: chatglm-finetune
spec:
  template:
    spec:
      containers:
      - name: finetune
        image: ccr.ccs.tencentyun.com/chatglm/chatglm-6b-ptv2:v1.0
        command:
        - bash
        - run_train.sh
        resources:
          limits:
            nvidia.com/gpu: "1"
        volumeMounts:
        - name: cfs-data
          mountPath: /data
      volumes:
      - name: cfs-data
        persistentVolumeClaim:
          claimName: cfs-pvc
      restartPolicy: Never
```

```bash
kubectl apply -f finetune-job.yaml
kubectl get pods -w
kubectl logs job/chatglm-finetune
```

```
NAME                     READY   STATUS    RESTARTS   AGE
chatglm-finetune-xxx     1/1     Running   0          10s
...
Training completed. Checkpoint saved to /data/output/
```

## 验证

```bash
kubectl get job chatglm-finetune
```

```
NAME               COMPLETIONS   DURATION   AGE
chatglm-finetune   1/1           30m        30m
```

```bash
ls /mnt/cfs/output/
```

```
checkpoint-1000  checkpoint-2000  checkpoint-3000  ...
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| kubectl 无法连接集群（超时或 `Unauthorized`） | `tccli tke DescribeClusters --ClusterId <ClusterId> --region ap-guangzhou` 确认 `ClusterStatus` 为 `Running`；本地 `kubectl get nodes` 失败 | CAM 240463971：Serverless 集群 API Server 为内网地址，未通过 VPN/IOA 接入 | 连接腾讯云 VPN/IOA 客户端后重试 kubectl；未接入时改用 `tccli tke` 命令完成诊断 |
| Job 长时间 Pending，Events 提示 GPU 不足 | `kubectl describe pod -l job-name=chatglm-finetune` 查看 Events；`tccli tke DescribeClusters --ClusterId <ClusterId> --region ap-guangzhou` 确认集群 Running | Serverless 集群 GPU 配额不足（V100 需单独申请） | 在配额管理页申请 V100 GPU 配额；或更换可用 GPU 规格 |
| 模型下载失败（Hugging Face 不可达） | `kubectl logs job/chatglm-finetune` 查看日志中的网络错误 | 公网无法直接访问 Hugging Face | 使用镜像站点或提前下载模型后上传到 CFS |
| 系统盘空间不足 | `kubectl describe pod <pod>` 查看 Events 中的磁盘错误 | 默认系统盘空间不足以存放模型+数据集 | 调大系统盘至 50 GiB |
| CFS 挂载失败 | `kubectl describe pod <pod>` 查看 Volume 挂载错误 | CFS 组件未安装或 VPC/子网不匹配 | 确认 CFS 组件已安装；确认 VPC/子网与集群一致 |
| `git lfs` 未安装 | `kubectl logs job/chatglm-finetune \| grep "git-lfs"` | 镜像未预装 git-lfs | `yum install git-lfs -y` 或使用预装 git-lfs 的基础镜像 |

## 清理

```bash
kubectl delete job chatglm-finetune
kubectl delete pvc cfs-pvc
kubectl delete pv cfs-pv
```

## 下一步

- [使用 TKE 快速部署 ChatGLM](../使用 TKE 快速部署 ChatGLM/tccli 操作.md)（page_id `92795`）
- [在 TKE 上部署 AI 大模型](https://cloud.tencent.com/document/product/457/116253)

## 控制台替代

控制台 **集群 → 工作负载 → Job → 新建**，选择实例类型 GPU V100，配置系统盘 50 GiB，挂载 CFS PVC，填写镜像和运行命令。
