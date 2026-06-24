# docker run 参数适配（tccli）

> 对照官方：[docker run 参数适配](https://cloud.tencent.com/document/product/457/9883) · page_id `9883`
## 概述

将 `docker run` 命令参数转换为 Kubernetes Pod/Deployment YAML 配置，涵盖常用 docker run 参数的 K8s 等价配置。

> **kubectl 数据面不可达**：CAM拒绝公网端点(strategyId:240463971)，内网需VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

## 前置条件

- [环境准备](../../../环境准备.md)：`tccli` 已配置，`kubectl` >= v1.28
- 已获取集群 kubeconfig：

```bash
tccli tke DescribeClusterKubeconfig --region <Region> --ClusterId <ClusterId> \
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

- 需要 CAM 权限：`tke:DescribeClusters`、`tke:DescribeClusterKubeconfig`

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 获取集群凭证 | `tccli tke DescribeClusterKubeconfig` | 是 |
| 查看集群状态 | `tccli tke DescribeClusterStatus --region <Region> --ClusterIds '["<ClusterId>"]'` | 是 |
| 相关 kubectl 操作 | 见 Procedure | -- |

## 操作步骤

### docker run 到 Kubernetes 参数映射

| docker run 参数 | Kubernetes 等价配置 | 说明 |
|----------------|---------------------|------|
| `--name` | `metadata.name` | 资源名称 |
| `-d` | Deployment/ReplicaSet 保证 | K8s 默认后台运行 |
| `-p 8080:80` | `ports.containerPort: 80` + Service | 端口映射 |
| `-v /host:/container` | `volumes.hostPath` 或 PVC | 卷挂载 |
| `-e KEY=VALUE` | `env[].name` / `env[].value` | 环境变量 |
| `--restart=always` | `restartPolicy: Always`（Pod）/ Deployment 自动 | 重启策略 |
| `--cpus=2` | `resources.limits.cpu: "2"` | CPU 限制 |
| `-m 512m` | `resources.limits.memory: "512Mi"` | 内存限制 |
| `--network=host` | `hostNetwork: true` | 主机网络 |
| `--privileged` | `securityContext.privileged: true` | 特权模式 |
| `--entrypoint` | `command` | 启动命令 |
| `CMD` 参数 | `args` | 启动参数 |

### 示例：docker run 转 K8s Deployment

原始 docker run 命令：

```bash
docker run -d --name webapp \
  -p 8080:80 \
  -v /data/html:/usr/share/nginx/html \
  -e APP_ENV=production \
  --cpus=1 \
  -m 256m \
  nginx:alpine
```

等价 Kubernetes Deployment：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  replicas: 1
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
          image: nginx:alpine
          ports:
            - containerPort: 80
          env:
            - name: APP_ENV
              value: "production"
          resources:
            limits:
              cpu: "1"
              memory: "256Mi"
          volumeMounts:
            - name: html
              mountPath: /usr/share/nginx/html
      volumes:
        - name: html
          hostPath:
            path: /data/html
```

```bash
kubectl apply -f webapp.yaml
# expected: deployment.apps/webapp created
```

## 验证

### 控制面

```bash
tccli tke DescribeClusterStatus --region <Region> --ClusterIds '["<ClusterId>"]'
# expected: ClusterState "Running"
```

```json
{
  "ClusterStatusSet": "<ClusterStatusSet>",
  "ClusterId": "<ClusterId>",
  "ClusterState": "<ClusterState>",
  "ClusterInstanceState": "<ClusterInstanceState>",
  "ClusterBMonitor": "<ClusterBMonitor>",
  "ClusterInitNodeNum": "<ClusterInitNodeNum>"
}
```

### 数据面（需 VPN/IOA）

```bash
kubectl get deployment webapp
# expected: READY 1/1
```

```text
NAME  STATUS  AGE
...
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl` 连接超时/拒绝 | `tccli tke DescribeClusters` 返回集群 Running 但 kubectl 不可达 | CAM 策略 240463971 拒绝公网访问 APIServer 端点 | 通过 VPN/IOA 接入内网后执行 kubectl 命令 |
| hostPath 卷不可用 | `kubectl describe pod <pod>` 查看 Events | 节点路径不存在 | 确认路径存在或改用 PVC |
| 特权容器被拒绝 | `kubectl get pod <pod>` 查看 STATUS | PodSecurityPolicy/PSA 限制 | 移除非必要的 `securityContext.privileged: true` |
| hostNetwork 端口冲突 | `kubectl describe pod <pod>` 查看 Events | Node 端口已被占用 | 换用 ClusterIP Service |

## 清理

```bash
kubectl delete deployment webapp
# expected: deployment.apps "webapp" deleted
```

## 下一步

- [构建简单 Web 应用](../../../快速入门/入门示例/构建简单 Web 应用/tccli%20操作.md) -- page_id `6996`
- [设置工作负载的运行命令和参数](../../../应用配置/工作负载管理/设置工作负载的运行命令和参数/tccli%20操作.md) -- page_id `32816`

## 控制台替代

[容器服务控制台](https://console.cloud.tencent.com/tke2/cluster)
