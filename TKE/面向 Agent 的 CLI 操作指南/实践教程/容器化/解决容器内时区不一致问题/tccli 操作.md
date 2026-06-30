# 解决容器内时区不一致问题（tccli）

> 对照官方：[解决容器内时区不一致问题](https://cloud.tencent.com/document/product/457/41877) · page_id `41877`
## 概述

解决容器内时区与宿主机不一致的问题。通过挂载 `/etc/localtime` 或设置 `TZ` 环境变量统一时区。

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

- 需要 CAM 权限：`tke:DescribeClusters`、`tke:DescribeClusterKubeconfig`

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 获取集群凭证 | `tccli tke DescribeClusterKubeconfig` | 是 |
| 查看集群状态 | `tccli tke DescribeClusterStatus --region <Region> --ClusterIds '["<ClusterId>"]'` | 是 |
| 相关 kubectl 操作 | 见 Procedure | -- |

## 操作步骤

### 1. 方法一：挂载宿主机时区文件（需 VPN/IOA）

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: tz-demo
spec:
  containers:
    - name: app
      image: nginx:alpine
      volumeMounts:
        - name: timezone
          mountPath: /etc/localtime
          readOnly: true
  volumes:
    - name: timezone
      hostPath:
        path: /etc/localtime
```

### 2. 方法二：设置 TZ 环境变量

```yaml
spec:
  containers:
    - name: app
      image: nginx:alpine
      env:
        - name: TZ
          value: "Asia/Shanghai"
```

### 3. 方法三：Deployment 级别配置

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: timezone-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: timezone-app
  template:
    metadata:
      labels:
        app: timezone-app
    spec:
      containers:
        - name: app
          image: nginx:alpine
          env:
            - name: TZ
              value: "Asia/Shanghai"
          volumeMounts:
            - name: timezone
              mountPath: /etc/localtime
              readOnly: true
      volumes:
        - name: timezone
          hostPath:
            path: /etc/localtime
```

```bash
kubectl apply -f timezone-deployment.yaml
# expected: deployment.apps/timezone-app created
```

### 4. 验证时区

```bash
kubectl exec <pod> -- date
# expected: 显示 Asia/Shanghai 时间
```

| 方法 | 优点 | 缺点 |
|------|------|------|
| 挂载 /etc/localtime | 与宿主机完全一致 | 依赖宿主机时区配置 |
| TZ 环境变量 | 不依赖宿主机，灵活 | 部分基础镜像可能不支持 |

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
kubectl exec <pod> -- date
# expected: CST 时间（UTC+8）
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl` 连接超时/拒绝 | `tccli tke DescribeClusters` 返回集群 Running 但 kubectl 不可达 | CAM 策略 240463971 拒绝公网访问 APIServer 端点 | 通过 VPN/IOA 接入内网后执行 kubectl 命令 |
| TZ 环境变量无效 | `kubectl exec <pod> -- date` 仍显示 UTC | 镜像不读取 TZ 变量 | 改用挂载 /etc/localtime 方式 |
| /etc/localtime 挂载失败 | `kubectl describe pod <pod>` 查看 Events | 节点时区文件路径不同 | 检查宿主机 `/etc/localtime` 路径是否存在 |

## 清理

```bash
kubectl delete deployment timezone-app
# expected: deployment.apps "timezone-app" deleted
```

## 下一步

- [docker run 参数适配](../docker%20run%20参数适配/tccli%20操作.md) -- page_id `9883`
- [设置工作负载的运行命令和参数](../../../应用配置/工作负载管理/设置工作负载的运行命令和参数/tccli%20操作.md) -- page_id `32816`

## 控制台替代

[容器服务控制台](https://console.cloud.tencent.com/tke2/cluster)
