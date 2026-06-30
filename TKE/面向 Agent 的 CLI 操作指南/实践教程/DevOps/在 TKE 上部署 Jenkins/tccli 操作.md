# 使用 TKE 部署 Jenkins（tccli）

> 对照官方：[使用 TKE 部署 Jenkins](https://cloud.tencent.com/document/product/457/52330) · page_id `52330`

## 概述

在 TKE 中部署 Jenkins CI/CD 服务，使用 PVC 持久化 Jenkins 数据，LoadBalancer Service 对外暴露 Web UI。

## 前置条件

- [环境准备](../../../环境准备.md)

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 部署 Jenkins | `kubectl apply -f jenkins.yaml` | 是 |
| 获取密码 | `kubectl exec <pod> -- cat /var/jenkins_home/secrets/initialAdminPassword` | 是 |
| 暴露服务 | `kubectl expose deploy jenkins --type=LoadBalancer --port=8080` | 是 |

## 操作步骤

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jenkins
spec:
  replicas: 1
  template:
    spec:
      containers:
      - name: jenkins
        image: jenkins/jenkins:lts
        ports:
        - containerPort: 8080
        - containerPort: 50000
        volumeMounts:
        - name: jenkins-home
          mountPath: /var/jenkins_home
      volumes:
      - name: jenkins-home
        persistentVolumeClaim:
          claimName: jenkins-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: jenkins
spec:
  type: LoadBalancer
  ports:
  - port: 8080
  selector:
    app: jenkins
```

```bash
kubectl apply -f jenkins.yaml
```

```text
# command executed successfully
```

```output
deployment.apps/jenkins created
service/jenkins created
```

## 验证

```bash
kubectl get svc jenkins
kubectl logs deploy/jenkins --tail=5
```

```text
NAME  STATUS  AGE
...
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl` 命令报错/无响应（本环境实测） | `tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]' --filter "Clusters[0].ClusterStatus"` | CAM 拒绝公网端点（strategyId:240463971），数据面不可达 | 接入 VPN/IOA 内网后执行 kubectl；控制面 tccli 不受影响 |
| Jenkins Pod `Pending` | `kubectl describe pod -l app=jenkins` | PVC 未绑定或节点资源不足 | 确认 StorageClass 可用；检查节点剩余资源 |
| LoadBalancer 无外网 IP | `kubectl get svc jenkins` | CLB 创建中或子账号无 CLB 权限 | 等待 1-2 分钟；确认 CAM 含 `clb:*` |
| 初始密码无法获取 | `kubectl logs deploy/jenkins --tail=20` | Jenkins 未完成初始化 | 等待 Pod Ready 后重试 `kubectl exec` |

## 清理

```bash
kubectl delete svc,deploy jenkins
kubectl delete pvc jenkins-pvc
```

## 下一步

- [在 containerd 集群中使用 Docker 构建镜像](../在%20containerd%20集群中使用%20Docker%20做镜像构建服务/tccli%20操作.md)
- [应用市场](../../../应用配置/组件和应用管理/应用管理/应用市场/tccli%20操作.md)

## 控制台替代

应用市场 → 搜索 Jenkins → 一键部署。
