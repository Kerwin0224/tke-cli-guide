# 单实例版 WordPress

> 对照官方：[单实例版 WordPress](https://cloud.tencent.com/document/product/457/7205) · page_id `7205`
## 概述

在 TKE 集群中部署单实例版 WordPress，使用 Deployment 部署 WordPress 容器，通过 LoadBalancer Service 对外暴露，数据存储在容器本地。

> **注意**：需集群端点可达（公网端点被 CAM 策略 strategyId:240463971 硬拒绝，内网端点创建失败）。在具备 IOA/VPN 或同 VPC CVM 环境下执行。
## 前置条件

### 环境检查

```bash
tccli --version
# expected: tccli version 1.x.x

tccli configure list
# expected: secretId, secretKey, region 均已配置
```

所需 CAM 权限：`tke:DescribeClusterKubeconfig`

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
| 部署 WordPress | `kubectl apply -f wordpress.yaml` | 是 |
| 创建 Service | `kubectl apply -f wordpress-svc.yaml` | 是 |
| 查看部署状态 | `kubectl get deployment wordpress` | 是 |
| 获取访问 IP | `kubectl get service wordpress` | 是 |
| 删除应用 | `kubectl delete -f wordpress.yaml && kubectl delete -f wordpress-svc.yaml` | 是 |

## 操作步骤

### 1. 控制面：获取集群凭证

```bash
tccli tke DescribeClusterKubeconfig --region ap-guangzhou --ClusterId cls-xxxxxxxx \
    --filter "Kubeconfig" > kubeconfig.yaml
export KUBECONFIG=$(pwd)/kubeconfig.yaml
# expected: kubeconfig 写入本地文件
```

### 2. 数据面：部署 WordPress（需 VPN/IOA）

创建 `wordpress.yaml`：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
  namespace: default
  labels:
    app: wordpress
spec:
  replicas: 1
  selector:
    matchLabels:
      app: wordpress
  template:
    metadata:
      labels:
        app: wordpress
    spec:
      containers:
        - name: wordpress
          image: wordpress:latest
          ports:
            - containerPort: 80
          env:
            - name: WORDPRESS_DB_HOST
              value: "127.0.0.1"
            - name: WORDPRESS_DB_USER
              value: "root"
            - name: WORDPRESS_DB_PASSWORD
              value: "password"
```

创建 `wordpress-svc.yaml`：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: wordpress
  namespace: default
spec:
  type: LoadBalancer
  selector:
    app: wordpress
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

```bash
kubectl apply -f wordpress.yaml
# expected: deployment.apps/wordpress created

kubectl apply -f wordpress-svc.yaml
# expected: service/wordpress created
```

### 3. 数据面：验证部署

```bash
kubectl get deployment wordpress
# expected: READY 1/1

kubectl get pods -l app=wordpress
# expected: STATUS Running
```

预期输出：

```text
NAME                         READY   STATUS    RESTARTS   AGE
wordpress-6d8c4f9b6c-xyz89   1/1     Running   0          3m
```

```bash
kubectl get service wordpress
# expected: EXTERNAL-IP 非 <pending>
```

```text
NAME  STATUS  AGE
...
```

### 4. 访问 WordPress

浏览器访问 `http://<EXTERNAL-IP>` 进入 WordPress 初始化向导。

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
kubectl get deployment wordpress
# expected: READY 1/1

kubectl get service wordpress
# expected: EXTERNAL-IP 非 <pending>

curl -sI http://<EXTERNAL-IP>
# expected: HTTP/1.1 200 OK
```

```text
NAME  STATUS  AGE
...
```

## 清理

> **注意**：删除 LoadBalancer Service 前确保关联的 CLB 已不再需要，否则 CLB 将持续产生费用。CLB 需在 [CLB 控制台](https://console.cloud.tencent.com/clb) 确认已随 Service 自动释放。

### 数据面

```bash
kubectl delete -f wordpress-svc.yaml
# expected: service "wordpress" deleted

kubectl delete -f wordpress.yaml
# expected: deployment.apps "wordpress" deleted
```

### 控制面（tccli）

无需控制面操作，集群资源保留。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Pod `Pending` | `kubectl describe pod -l app=wordpress` 查看 Events | 节点资源不足（CPU/内存）或 PVC 未绑定 | 扩容节点或降低资源 requests；检查节点状态 `kubectl get nodes` |
| `ImagePullBackOff` | `kubectl describe pod -l app=wordpress` 查看 Events | 镜像拉取失败（公网不可达或镜像仓库鉴权） | 换用 `ccr.ccs.tencentyun.com/library/wordpress:latest`；或配置 imagePullSecrets |

### 操作成功但异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| WordPress 无法连接数据库 | `kubectl logs deployment/wordpress` 查看启动日志 | 单实例版数据库在容器内（`127.0.0.1`），仅作演示用途 | 如需独立数据库，请参考「使用 TencentDB 的 WordPress」 |
| 访问 LB IP 超时 | `kubectl get svc wordpress -o yaml` 确认 EXTERNAL-IP；`kubectl describe svc wordpress` 查看 Events | CLB 安全组未放通 TCP:80 入站规则 | 在 [CLB 控制台](https://console.cloud.tencent.com/clb) 的安全组中添加 `TCP:80` 入站规则 |

## 下一步

- [使用 TencentDB 的 WordPress](../使用 TencentDB 的 WordPress/tccli%20操作.md) -- page_id `7447`
- [创建简单的 Nginx 服务](../创建简单的 Nginx 服务/tccli%20操作.md) -- page_id `7851`

## 控制台替代

[容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) -> 集群 -> 工作负载 -> Deployment -> 新建，选择 WordPress 镜像。
