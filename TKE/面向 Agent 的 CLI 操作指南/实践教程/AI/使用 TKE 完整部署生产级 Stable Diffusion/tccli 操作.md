# 使用 TKE 完整部署生产级 Stable Diffusion（tccli）

> 对照官方：[使用 TKE 完整部署生产级 Stable Diffusion](https://cloud.tencent.com/document/product/457/92794) · page_id `92794`

## 概述

在 TKE 集群中部署 Stable Diffusion WebUI，使用 GPU 节点、PVC 持久化模型和输出、Service 暴露访问。

## 前置条件

- GPU 节点可用

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 创建 PVC | `kubectl apply -f pvc.yaml` | 是 |
| 部署 SD WebUI | `kubectl apply -f sd-deploy.yaml` | 是 |
| 暴露服务 | `kubectl expose deploy sd-webui --type=LoadBalancer --port=7860` | 是 |

## 操作步骤

### 创建 PVC + Deployment

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: sd-models
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 50Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sd-webui
spec:
  replicas: 1
  template:
    spec:
      nodeSelector:
        accelerator: nvidia-tesla-t4
      containers:
      - name: webui
        image: ghcr.io/automatic1111/stable-diffusion-webui:latest
        ports:
        - containerPort: 7860
        resources:
          limits:
            nvidia.com/gpu: 1
        volumeMounts:
        - name: models
          mountPath: /stable-diffusion-webui/models
      volumes:
      - name: models
        persistentVolumeClaim:
          claimName: sd-models
```

```bash
kubectl apply -f sd.yaml
kubectl expose deploy sd-webui --type=LoadBalancer --port=7860
```

```text
# command executed successfully
```

```output
deployment.apps/sd-webui created
service/sd-webui exposed
```

## 验证

```bash
kubectl get svc sd-webui
kubectl logs deploy/sd-webui --tail=10
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
kubectl delete svc sd-webui
kubectl delete deploy sd-webui
kubectl delete pvc sd-models
```

## 下一步

- [使用 TKE 快速部署 ChatGLM](../使用%20TKE%20快速部署%20ChatGLM/tccli%20操作.md)
- [GPU 共享使用指南](../../../集群配置/GPU%20资源管理/GPU%20共享/tccli%20操作.md)

## 控制台替代

控制台：工作负载 → Deployment → 新建 → 选择 GPU 镜像。
