# 手动搭建 Hello World 服务

> 对照官方：[手动搭建 Hello World 服务](https://cloud.tencent.com/document/product/457/7204) · page_id `7204`
## 概述

通过 kubectl YAML 文件在 TKE 集群中部署一个简单的 Hello World Web 应用，使用 LoadBalancer Service 对外暴露。

> **注意**：需集群端点可达（公网端点被 CAM 策略 strategyId:240463971 硬拒绝，内网端点创建失败）。在具备 IOA/VPN 或同 VPC CVM 环境下执行。
## 前置条件

### 环境检查

```bash
tccli --version
# expected: tccli version >= 1.0

kubectl version --client --short
# expected: Client Version >= v1.28
```

### 资源检查

```bash
tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]' \
    --filter "Clusters[0].ClusterStatus"
# expected: "Running"
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

- 所需 CAM 权限：`tke:DescribeClusterKubeconfig`

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 获取集群凭证 | `tccli tke DescribeClusterKubeconfig --region ap-guangzhou --ClusterId cls-xxxxxxxx` | 是 |
| 创建 Hello World Deployment | `kubectl apply -f hello-world.yaml` | 是 |
| 创建 Service | `kubectl apply -f hello-world-svc.yaml` | 是 |
| 查看 Deployment | `kubectl get deployment hello-world` | 是 |
| 获取访问地址 | `kubectl get service hello-world` | 是 |
| 删除应用 | `kubectl delete -f hello-world.yaml && kubectl delete -f hello-world-svc.yaml` | 是 |

## 操作步骤

### 1. 控制面：获取集群凭证

```bash
tccli tke DescribeClusterKubeconfig --region ap-guangzhou --ClusterId cls-xxxxxxxx \
    --filter "Kubeconfig" > kubeconfig.yaml
export KUBECONFIG=$(pwd)/kubeconfig.yaml
# expected: kubeconfig 写入本地文件
```

### 2. 数据面：准备并部署 Hello World 应用（需 VPN/IOA）

创建 `hello-world.yaml`：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-world
  namespace: default
  labels:
    app: hello-world
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hello-world
  template:
    metadata:
      labels:
        app: hello-world
    spec:
      containers:
        - name: hello-world
          image: nginx:alpine
          ports:
            - containerPort: 80
          volumeMounts:
            - name: html
              mountPath: /usr/share/nginx/html
      volumes:
        - name: html
          emptyDir: {{}}
```

创建 `hello-world-svc.yaml`：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: hello-world
  namespace: default
spec:
  type: LoadBalancer
  selector:
    app: hello-world
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

```bash
kubectl apply -f hello-world.yaml
# expected: deployment.apps/hello-world created

kubectl apply -f hello-world-svc.yaml
# expected: service/hello-world created
```

### 3. 数据面：验证部署状态

```bash
kubectl get deployment hello-world
# expected: READY 2/2

kubectl get pods -l app=hello-world
# expected: 2 个 Running Pod
```

预期输出：

```text
NAME                           READY   STATUS    RESTARTS   AGE
hello-world-5d8c4f9b6c-abc12   1/1     Running   0          2m
hello-world-5d8c4f9b6c-def34   1/1     Running   0          2m
```

```bash
kubectl get service hello-world
# expected: EXTERNAL-IP 非 <pending>
```

预期输出：

```text
NAME          TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
hello-world   LoadBalancer   10.96.0.101    1.2.3.5       80:32345/TCP   3m
```

### 4. 数据面：访问 Hello World 服务

```bash
curl http://<EXTERNAL-IP>
# expected: 返回 Nginx 默认页面
```

## 验证

### 控制面（tccli）

```bash
tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]' \
    --filter "Clusters[0].ClusterStatus"
# expected: "Running"
```

预期输出：

```json
{
    "Clusters": [
        {
            "ClusterId": "cls-xxxxxxxx",
            "ClusterStatus": "Running"
        }
    ]
}
```

### 数据面

```bash
kubectl get deployment hello-world
# expected: READY 2/2, AVAILABLE 2
```

```text
NAME  STATUS  AGE
...
```

```bash
kubectl get service hello-world
# expected: EXTERNAL-IP 非 <pending>
```

```text
NAME  STATUS  AGE
...
```

```bash
kubectl get pods -l app=hello-world
# expected: 2 个 Pod STATUS 为 Running
```

预期输出：

```text
NAME                           READY   STATUS    RESTARTS   AGE
hello-world-5d8c4f9b6c-abc12   1/1     Running   0          3m
hello-world-5d8c4f9b6c-def34   1/1     Running   0          3m
```

## 清理

> **注意**：LoadBalancer 类型 Service 会创建 CLB 实例并持续产生费用，验证完成后请及时删除以免产生额外费用。

### 数据面

```bash
kubectl delete -f hello-world-svc.yaml
# expected: service "hello-world" deleted

kubectl delete -f hello-world.yaml
# expected: deployment.apps "hello-world" deleted
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `Unable to connect to the server: dial tcp ... i/o timeout` | `kubectl cluster-info` | 集群公网端点被 CAM 策略 strategyId:240463971 硬拒绝，内网端点创建失败 | 通过 IOA/VPN 或同 VPC CVM 连接内网端点后重试 |
| `error: the path "hello-world.yaml" does not exist` | `ls hello-world.yaml hello-world-svc.yaml` | YAML 文件未创建或路径错误 | 确认已按步骤创建 YAML 文件，检查当前工作目录 |

### 操作成功但异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Pod `Pending` | `kubectl describe pod <pod-name>` 查看 Events | 资源不足或节点不可调度 | 扩容节点或释放资源；检查节点状态 |
| Pod `ImagePullBackOff` | `kubectl describe pod <pod-name>` | 镜像拉取失败 | 检查节点公网连通性；换用 TCR 镜像 `ccr.ccs.tencentyun.com/library/nginx:alpine` |
| Service EXTERNAL-IP 长时间 `<pending>` | `kubectl describe service hello-world` | CLB 创建中 | 等待 1–2 分钟 |
| `Connection refused` 访问 LB IP | `kubectl get endpoints hello-world` | Service 无后端 Pod 或安全组未放通 | 确认 `ENDPOINTS` 有后端 IP；检查 CLB 安全组入站规则 `TCP:80` |

## 下一步

- [创建简单的 Nginx 服务](../创建简单的 Nginx 服务/tccli%20操作.md) -- page_id `7851`
- [单实例版 WordPress](../单实例版 WordPress/tccli%20操作.md) -- page_id `7205`

## 控制台替代

[容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) -> 集群 -> 工作负载 -> Deployment -> 新建。
