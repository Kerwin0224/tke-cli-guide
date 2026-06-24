# 使用 TKE 快速部署 ChatGLM（tccli）

> 对照官方：[使用 TKE 快速部署 ChatGLM](https://cloud.tencent.com/document/product/457/92795) · page_id `92795`

## 概述

在 TKE GPU 集群中部署 ChatGLM-6B 对话模型，使用 huggingface 模型下载 initContainer、Gradio Web 界面和 GPU 资源。

## 前置条件

- GPU 节点（NVIDIA T4 或以上，≥12GB 显存）

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 部署 ChatGLM | `kubectl apply -f chatglm.yaml` | 是 |
| 查看日志 | `kubectl logs deploy/chatglm` | 是 |

## 操作步骤

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: chatglm
spec:
  replicas: 1
  template:
    spec:
      nodeSelector:
        accelerator: nvidia-tesla-t4
      initContainers:
      - name: download-model
        image: python:3.10
        command: ["pip", "install", "huggingface_hub", "&&", "huggingface-cli", "download", "THUDM/chatglm-6b"]
      containers:
      - name: chatglm
        image: pytorch/pytorch:2.0.0-cuda11.7-cudnn8-runtime
        ports:
        - containerPort: 7860
        resources:
          limits:
            nvidia.com/gpu: 1
            memory: 16Gi
```

```bash
kubectl apply -f chatglm.yaml
kubectl expose deploy chatglm --type=LoadBalancer --port=7860
```

```text
# command executed successfully
```

## 验证

```bash
kubectl logs deploy/chatglm
kubectl get svc chatglm
```

```text
NAME  STATUS  AGE
...
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| kubectl 无法连接集群（超时或 `Unauthorized`） | `tccli tke DescribeClusters --ClusterId <ClusterId> --region ap-guangzhou` 确认 `ClusterStatus` 为 `Running`；本地 `kubectl get nodes` 失败 | CAM 240463971：集群 API Server 为内网地址，未通过 VPN/IOA 接入 | 连接腾讯云 VPN/IOA 客户端后重试 kubectl；未接入时改用 `tccli tke` 命令完成诊断 |
| Pod 长时间 Pending，Events 提示 `Insufficient nvidia.com/gpu` | `tccli tke DescribeClusterInstances --ClusterId <ClusterId> --InstanceRole WORKER --region ap-guangzhou` 确认 GPU 节点数；`kubectl describe pod <pod>` 查看 Events | GPU 配额不足或 GPU 节点未就绪 | 在配额管理页申请 GPU 配额；确认节点为 GPU 机型（T4/V100/A10）且 GPU 驱动与 device-plugin 已安装 |

## 清理

```bash
kubectl delete svc,deploy chatglm
```

## 下一步

- [Stable Diffusion 部署](../使用%20TKE%20完整部署生产级%20Stable%20Diffusion/tccli%20操作.md)
- [GPU 共享使用指南](../../../集群配置/GPU%20资源管理/GPU%20共享/tccli%20操作.md)

## 控制台替代

控制台：工作负载 → 新建 → GPU 镜像。
