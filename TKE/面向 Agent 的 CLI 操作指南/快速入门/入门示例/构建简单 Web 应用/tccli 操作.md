# 构建简单 Web 应用

> 对照官方：[构建简单 Web 应用](https://cloud.tencent.com/document/product/457/6996) · page_id `6996`
## 概述

在 TKE 集群中构建一个完整的简单 Web 应用，涵盖镜像构建、推送 TCR、部署、配置更新、扩缩容和版本回滚的全流程操作。

> **注意**：需集群端点可达（公网端点被 CAM 策略 strategyId:240463971 硬拒绝，内网端点创建失败）。在具备 IOA/VPN 或同 VPC CVM 环境下执行。
## 前置条件

### 环境检查

```bash
tccli --version
# expected: tccli version 1.x.x

tccli configure list
# expected: secretId, secretKey, region 均已配置
```

所需 CAM 权限：`tke:DescribeClusterKubeconfig`、`tcr:CreateRepository`、`tcr:DescribeRepositories`

### 资源检查

```bash
tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]'
# expected: ClusterStatus "Running", ClusterType "MANAGED_CLUSTER", ClusterVersion "1.30.0"
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

确认 kubectl 工具可用：

```bash
kubectl version --client --short 2>/dev/null || kubectl version --client
# expected: Client Version >= v1.28
```

确认 Docker 已安装并登录 TCR：

```bash
docker --version
# expected: Docker version 2x.x.x

docker login ccr.ccs.tencentyun.com --username=<TCR_USERNAME>
# expected: Login Succeeded
```

获取 kubeconfig：

```bash
tccli tke DescribeClusterKubeconfig --region ap-guangzhou --ClusterId cls-xxxxxxxx \
    --filter "Kubeconfig" > kubeconfig.yaml
export KUBECONFIG=$(pwd)/kubeconfig.yaml
# expected: 生成 kubeconfig.yaml
```

```json
{
  "Kubeconfig": "<Kubeconfig>",
  "RequestId": "<RequestId>"
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 获取集群凭证 | `tccli tke DescribeClusterKubeconfig` | 是 |
| 构建 Docker 镜像 | `docker build -t <image> .` | 否 |
| 登录 TCR | `docker login ccr.ccs.tencentyun.com` | 否 |
| 推送镜像到 TCR | `docker push <image>` | 否 |
| 部署应用 | `kubectl apply -f webapp.yaml` | 是 |
| 更新镜像 | `kubectl set image deployment/webapp` | 否 |
| 扩缩容 | `kubectl scale deployment/webapp --replicas=<N>` | 是 |
| 回滚版本 | `kubectl rollout undo deployment/webapp` | 否 |
| 删除应用 | `kubectl delete -f webapp.yaml` | 是 |

## 操作步骤

### 1. 构建 Docker 镜像

创建 `Dockerfile`：

```dockerfile
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/index.html
```

创建 `index.html`：

```html
<!DOCTYPE html>
<html>
<body>
<h1>Hello from TKE!</h1>
<p>Version 1.0</p>
</body>
</html>
```

```bash
docker build -t ccr.ccs.tencentyun.com/<Namespace>/webapp:v1 .
# expected: Successfully built <image-id>, Successfully tagged ccr.ccs.tencentyun.com/<Namespace>/webapp:v1
```

### 2. 推送镜像到 TCR

```bash
docker push ccr.ccs.tencentyun.com/<Namespace>/webapp:v1
# expected: The push refers to repository [ccr.ccs.tencentyun.com/<Namespace>/webapp]
# v1: digest: sha256:... size: ...
```

### 3. 控制面：获取集群凭证

```bash
tccli tke DescribeClusterKubeconfig --region ap-guangzhou --ClusterId cls-xxxxxxxx \
    --filter "Kubeconfig" > kubeconfig.yaml
export KUBECONFIG=$(pwd)/kubeconfig.yaml
# expected: kubeconfig 写入本地文件
```

### 4. 数据面：部署应用（需 VPN/IOA）

创建 `webapp.yaml`：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
        - name: webapp
          image: ccr.ccs.tencentyun.com/<Namespace>/webapp:v1
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 256Mi
---
apiVersion: v1
kind: Service
metadata:
  name: webapp
spec:
  type: LoadBalancer
  selector:
    app: webapp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

```bash
kubectl apply -f webapp.yaml
# expected: deployment.apps/webapp created, service/webapp created
```

### 5. 数据面：更新应用（滚动升级）

构建并推送 v2 版本：

```bash
# 修改 index.html 中的版本号为 Version 2.0
docker build -t ccr.ccs.tencentyun.com/<Namespace>/webapp:v2 .
# expected: Successfully tagged ccr.ccs.tencentyun.com/<Namespace>/webapp:v2

docker push ccr.ccs.tencentyun.com/<Namespace>/webapp:v2
# expected: v2: digest: sha256:... size: ...
```

更新 Deployment 镜像：

```bash
kubectl set image deployment/webapp webapp=ccr.ccs.tencentyun.com/<Namespace>/webapp:v2
# expected: deployment.apps/webapp image updated
```

查看滚动更新状态：

```bash
kubectl rollout status deployment/webapp
# expected: deployment "webapp" successfully rolled out
```

### 6. 数据面：版本回滚

```bash
kubectl rollout undo deployment/webapp
# expected: deployment.apps/webapp rolled back

kubectl rollout status deployment/webapp
# expected: deployment "webapp" successfully rolled out
```

查看回滚历史：

```bash
kubectl rollout history deployment/webapp
# expected: 显示 REVISION 列表，包含 CHANGE-CAUSE
```

### 7. 数据面：扩缩容

```bash
kubectl scale deployment/webapp --replicas=5
# expected: deployment.apps/webapp scaled

kubectl get pods -l app=webapp
# expected: 5 Pods Running
```

```text
NAME  STATUS  AGE
...
```

## 验证

### 控制面（tccli）

```bash
tccli tke DescribeClusterStatus --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]'
# expected: ClusterState "Running"
```

**预期输出**：

```json
{
    "TotalCount": 1,
    "ClusterStatusSet": [
        {
            "ClusterId": "cls-xxxxxxxx",
            "ClusterState": "Running",
            "ClusterInstanceState": "Available"
        }
    ],
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

### 数据面

```bash
kubectl get deployment webapp
# expected: READY 5/5

kubectl get pods -l app=webapp
# expected: 5 Pods Running

kubectl get service webapp
# expected: EXTERNAL-IP 非 <pending>

curl -s http://<EXTERNAL-IP>
# expected: "Hello from TKE!"
```

```text
NAME  STATUS  AGE
...
```

## 清理

> **注意**：删除 LoadBalancer Service 前确保关联的 CLB 已不再需要，否则 CLB 将持续产生费用。CLB 需在 [CLB 控制台](https://console.cloud.tencent.com/clb) 确认已随 Service 自动释放。

### 数据面

```bash
kubectl delete -f webapp.yaml
# expected: deployment.apps "webapp" deleted, service "webapp" deleted
```

### 控制面（tccli）

TCR 镜像需手动清理（可选，避免继续占用存储）：

```bash
tccli tcr DeleteImage --region ap-guangzhou --RegistryId <RegistryId> \
    --NamespaceName <Namespace> --RepositoryName webapp --ImageVersion v1
# expected: RequestId 返回成功
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `docker login` 失败 | `docker login --username=<TCR_USERNAME> ccr.ccs.tencentyun.com` 查看错误信息 | TCR 用户名/密码错误，或长期未登录 token 已过期 | 在 [TCR 控制台](https://console.cloud.tencent.com/tcr) 重置访问凭证，重新 `docker login` |
| `docker push` 失败 (denied) | `docker images ccr.ccs.tencentyun.com/<Namespace>/webapp` 确认镜像存在；`tccli tcr DescribeRepositories --RegistryId <RegistryId>` 确认仓库存在 | 未登录 TCR，或命名空间/仓库不存在，或无推送权限 | 执行 `docker login ccr.ccs.tencentyun.com`；若仓库不存在，在 TCR 控制台创建命名空间和仓库 |
| `ImagePullBackOff` | `kubectl describe pod -l app=webapp` 查看 Events；`kubectl get events --field-selector type=Warning` 查看集群事件 | 集群节点无 TCR 镜像拉取凭证 | 创建 `docker-registry` 类型 Secret 并关联 ServiceAccount：`kubectl create secret docker-registry tcr-secret --docker-server=ccr.ccs.tencentyun.com --docker-username=<TCR_USERNAME> --docker-password=<TCR_PASSWORD>` |
| `CrashLoopBackOff` | `kubectl logs deployment/webapp` 查看容器日志 | 容器启动失败（镜像入口脚本错误、端口冲突等） | 检查 Dockerfile 的 CMD/ENTRYPOINT 是否正确；确认应用监听 `0.0.0.0:80` |

### 操作成功但异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 滚动更新卡住 | `kubectl rollout status deployment/webapp`；`kubectl describe deployment webapp` 查看 Conditions | 新 Pod 无法通过 readiness probe，或资源不足无法调度新 Pod | `kubectl get pods -l app=webapp -w` 观察 Pod 状态；若资源不足则扩容节点或降低 requests |
| 回滚未生效 | `kubectl rollout history deployment/webapp` 查看 revision 记录；`kubectl describe deployment webapp` 确认当前镜像版本 | `--to-revision` 指定的 revision 不存在或已被清理 | 确认 revision 列表：`kubectl rollout history deployment/webapp`；使用有效的 revision 编号回滚：`kubectl rollout undo deployment/webapp --to-revision=<N>` |
| 旧版本 Pod 仍运行 | `kubectl get pods -l app=webapp -o wide` 查看各 Pod 镜像版本 | 滚动更新策略 `maxUnavailable` 设置不当，或 PodDisruptionBudget 阻止驱逐 | 检查 `kubectl get deployment webapp -o yaml | grep -A10 strategy`；必要时手动删除旧 Pod 触发重建 |
| 服务间歇不可用 | `kubectl describe svc webapp` 确认 Endpoints；`kubectl get endpoints webapp` | 滚动更新期间 Pod 数量短暂低于 `minReadySeconds` 阈值 | 正常现象，短暂恢复；若持续不可用，检查 `maxSurge` 和 `maxUnavailable` 配置 |

## 下一步

- [Deployment 管理](../../../应用配置/工作负载管理/Deployment%20管理/tccli%20操作.md) -- page_id `31705`
- [使用 TCR 企业版实例内容器镜像创建工作负载](../../../应用配置/工作负载管理/使用%20TCR%20企业版实例内容器镜像创建工作负载/tccli%20操作.md) -- page_id `45624`

## 控制台替代

[容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) -> 集群 -> 工作负载 -> Deployment -> 新建。
