# 创建简单的 Nginx 服务

> 对照官方：[创建简单的 Nginx 服务](https://cloud.tencent.com/document/product/457/7851) · page_id `7851`
## 概述

在已有 TKE 集群的 `default` 命名空间中部署 Nginx Deployment，创建 LoadBalancer 类型 Service 对外暴露 80 端口。

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

至少有 1 个 Worker 节点。

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
| 查看集群列表 | `tccli tke DescribeClusters --region ap-guangzhou` | 是 |
| 创建 Deployment（名称 nginx） | `kubectl create deployment nginx --image=nginx:latest --replicas=1` | 否 |
| 创建公网 LB Service | `kubectl expose deployment nginx --type=LoadBalancer --port=80` | 否 |
| 查看 Deployment | `kubectl get deployment nginx` | 是 |
| 查看 Service / 获取 LB IP | `kubectl get service nginx` | 是 |
| 删除工作负载 | `kubectl delete deployment nginx && kubectl delete service nginx` | 是 |

## 操作步骤

### 1. 控制面：获取集群凭证

```bash
tccli tke DescribeClusterKubeconfig --region ap-guangzhou --ClusterId cls-xxxxxxxx \
    --filter "Kubeconfig" > kubeconfig.yaml
export KUBECONFIG=$(pwd)/kubeconfig.yaml
# expected: kubeconfig 写入本地文件
```

```json
{
  "Kubeconfig": "<Kubeconfig>",
  "RequestId": "<RequestId>"
}
```

### 2. 数据面：创建 Deployment（需 VPN/IOA）

```bash
kubectl create deployment nginx --image=nginx:latest --replicas=1
# expected: deployment.apps/nginx created
```

预期输出：

```text
deployment.apps/nginx created
```

### 3. 数据面：创建 LoadBalancer Service

```bash
kubectl expose deployment nginx --type=LoadBalancer --port=80 --target-port=80
# expected: service/nginx exposed
```

预期输出：

```text
service/nginx exposed
```

### 4. 数据面：获取外部访问 IP

```bash
kubectl get service nginx --watch
# expected: EXTERNAL-IP 从 <pending> 变为公网 IP
```

预期输出：

```text
NAME    TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)        AGE
nginx   LoadBalancer   10.96.0.100    <pending>        80:31234/TCP   5s
nginx   LoadBalancer   10.96.0.100    1.2.3.4          80:31234/TCP   60s
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
kubectl get deployment nginx
# expected: READY 1/1, AVAILABLE 1
```

预期输出：

```text
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
nginx   1/1     1            1           2m
```

```bash
kubectl get pods -l app=nginx
# expected: STATUS Running
```

预期输出：

```text
NAME                     READY   STATUS    RESTARTS   AGE
nginx-7848d4b86f-x7k2j   1/1     Running   0          2m
```

```bash
kubectl get service nginx
# expected: EXTERNAL-IP 非 <pending>
```

```text
NAME  STATUS  AGE
...
```

```bash
curl -sI http://<EXTERNAL-IP>
# expected: HTTP/1.1 200 OK
```

预期输出：

```text
HTTP/1.1 200 OK
Server: nginx/1.27.0
Content-Type: text/html
```

## 清理

> **注意**：LoadBalancer 类型 Service 会创建 CLB 实例并持续产生费用，验证完成后请及时删除以免产生额外费用。

### 数据面

按依赖倒序删除：

```bash
kubectl delete service nginx
# expected: service "nginx" deleted

kubectl delete deployment nginx
# expected: deployment.apps "nginx" deleted

kubectl get deployment,service nginx
# expected: NotFound
```

```text
NAME  STATUS  AGE
...
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `Unable to connect to the server: dial tcp ... i/o timeout` | `kubectl cluster-info` | 集群公网端点被 CAM 策略 strategyId:240463971 硬拒绝，内网端点创建失败 | 通过 IOA/VPN 或同 VPC CVM 连接内网端点后重试 |
| `kubectl create deployment` 失败 | `kubectl cluster-info` 验证连通性 | kubeconfig 未配置或 APIServer 不可达 | 确认已连 VPN/IOA，重新执行 `tccli tke DescribeClusterKubeconfig` 并导出 `KUBECONFIG` |

### 操作成功但异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Service EXTERNAL-IP 长时间 `<pending>` | `kubectl describe service nginx` 查看 Events | CLB 创建中或安全组未放通 | 等待 1–2 分钟；确认安全组放通 NodePort 范围 `30000–32767` |
| 访问 LB IP 无响应 | `curl -v http://<EXTERNAL-IP>` | CLB 安全组未放通入站 80 端口 | 在 CLB 控制台添加入站规则 `TCP:80` |
| Pod `ImagePullBackOff` | `kubectl describe pod <pod-name>` | 节点无法拉取镜像 | 检查节点公网连通性；换用 `ccr.ccs.tencentyun.com/library/nginx:latest` |
| Pod `CrashLoopBackOff` | `kubectl logs deployment/nginx` | 容器启动失败 | 根据日志定位启动错误并修正镜像或配置 |

## 下一步

- [手动搭建 Hello World 服务](../手动搭建 Hello World 服务/tccli%20操作.md) -- page_id `7204`
- [单实例版 WordPress](../单实例版 WordPress/tccli%20操作.md) -- page_id `7205`
- [Deployment 管理](../../../应用配置/工作负载管理/Deployment%20管理/tccli%20操作.md) -- page_id `31705`

## 控制台替代

[容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) -> 集群 -> 工作负载 -> Deployment -> 新建。
