# 高并发场景优化（tccli）

> 对照官方：[高并发场景优化](https://cloud.tencent.com/document/product/457/104859) · page_id `104859`

## 概述

针对高并发场景（≥ 10000 QPS）优化 Nginx Ingress Controller 的 Worker 进程数、连接池、keepalive、backlog 及节点内核参数（sysctl），确保在高流量下延迟稳定、不丢连接。

**优化维度**：

| 维度 | 默认值 | 高并发建议值 | 影响 |
|------|--------|-------------|------|
| Worker 进程数 | `auto`（CPU 核数） | 保持 `auto`，或 `16-32` | 处理并发连接的能力 |
| max-worker-connections | 16384 | 65536 | 单 Worker 最大并发连接 |
| keepalive | 75s | 300s | 减少 TLS 握手开销 |
| proxy-body-size | 1MB | 100MB | 支持大请求体 |
| backlog（net.core.somaxconn） | 128 | 65535 | 半连接/全连接队列大小 |

> **kubectl 数据面不可达**：CAM 拒绝公网端点（strategyId:240463971），内网需 VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

## 前置条件

- [环境准备](../../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId、secretKey、region 均已配置

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:DescribeClusters, tke:DescribeClusterKubeconfig
# 验证
tccli tke DescribeClusters --region <Region>
# expected: exit 0

# 4. 检查 kubectl 和 Helm（须 VPN/IOA）
kubectl version --client
# expected: Client Version v1.30+
helm version --short
# expected: v3.x
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

### 资源检查

```bash
# 5. 确认目标集群
tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterStatus "Running"

# 6. 确认 Nginx Ingress 已安装
helm list -n ingress-nginx
# expected: nginx-ingress 状态 deployed

# 7. 查看当前节点规格
tccli tke DescribeClusterInstances --region <Region> \
    --ClusterId CLUSTER_ID
# expected: 节点实例配置信息
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `CLUSTER_ID` | TKE 集群 ID | 格式 `cls-xxxxxxxx` | `tccli tke DescribeClusters` |

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 查看集群节点 | `tccli tke DescribeClusterInstances` | 是 |
| 查看集群配置 | `tccli tke DescribeClusters` | 是 |
| 修改 Nginx Ingress 配置 | `helm upgrade`（修改 values） | 否 |
| 修改节点内核参数 | `kubectl apply -f sysctl-daemonset.yaml` | 否 |

## 关键字段说明

以下说明 Nginx Ingress Controller 高并发相关的 Helm Values 配置项。

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `controller.maxWorkerConnections` | Integer | 否 | 单 Worker 最大并发连接数，默认 16384。高并发建议 65536 | 设置过低 → 高并发时连接被拒绝 |
| `controller.keepalive` | Integer | 否 | 客户端 keepalive 超时（秒），默认 75。建议 300 | 过短 → 频繁 TLS 握手影响性能 |
| `controller.config.proxy-body-size` | String | 否 | 最大请求体大小，默认 `"1m"`。支持 `"0"` 禁用限制 | 过小 → 大请求被 413 拒绝 |
| `controller.config.upstream-keepalive-connections` | Integer | 否 | 与上游（后端）的 keepalive 连接池大小，默认 320 | 过小 → 高并发时连接池耗尽 |
| `controller.config.use-gzip` | Boolean | 否 | 启用 Gzip 压缩，默认 `false`。建议 `true` | 启用后 CPU 占用略增 |
| `controller.replicaCount` | Integer | 是 | Controller 副本数，高并发建议 ≥ 3 | 副本不足 → 单点过载 |
| `controller.resources.limits.cpu` | String | 否 | CPU 限制，建议 ≥ `2`（核心） | CPU 不足 → 限流导致延迟增高 |

## 操作步骤

### 步骤 1：配置 Nginx Ingress Controller 高并发参数

#### 选择依据

- **maxWorkerConnections**：默认 16384 对应约 16000 并发连接。10000+ QPS 且长连接场景下，建议提升至 65536。计算公式：`maxWorkerConnections × worker_processes × 2（keepalive）≈ 总并发连接数`。
- **keepalive**：默认 75s 偏短，高频 TLS 握手增加延迟。增加到 300s 可复用 TLS 会话，适合 API 网关、CDN 回源场景。
- **proxy-body-size**：默认 1m 太小，文件上传、API 请求体经常超限。适当增大或设为 `"0"` 关闭限制。
- **upstream-keepalive-connections**：控制与后端 Pod 的 keepalive 连接池。高并发时多个 Pod 共享连接池，建议 >= 512。

#### 最小优化（仅调整关键参数）

`nginx-ingress-perf-minimal.yaml`：

```yaml
controller:
  replicaCount: 3
  maxWorkerConnections: 65536
  keepalive: 300
  config:
    proxy-body-size: "100m"
    upstream-keepalive-connections: "512"
```

```bash
helm upgrade nginx-ingress ingress-nginx/ingress-nginx \
    --namespace ingress-nginx \
    -f nginx-ingress-perf-minimal.yaml
# expected: STATUS: deployed
```

> **kubectl 数据面不可达**：CAM 拒绝公网端点（strategyId:240463971），内网需 VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

#### 增强优化（含 Gzip、超时调优）

`nginx-ingress-perf-enhanced.yaml`：

```yaml
controller:
  replicaCount: 3
  maxWorkerConnections: 65536
  keepalive: 300
  config:
    proxy-body-size: "100m"
    upstream-keepalive-connections: "512"
    use-gzip: "true"
    proxy-connect-timeout: "5"
    proxy-read-timeout: "60"
    proxy-send-timeout: "60"
    client-header-timeout: "60"
    client-body-timeout: "60"
    server-tokens: "false"
  resources:
    requests:
      cpu: 500m
      memory: 512Mi
    limits:
      cpu: 2
      memory: 2Gi
```

```bash
helm upgrade nginx-ingress ingress-nginx/ingress-nginx \
    --namespace ingress-nginx \
    -f nginx-ingress-perf-enhanced.yaml
# expected: STATUS: deployed
```

### 步骤 2：调优节点内核参数（sysctl）

#### 选择依据

- **somaxconn**：TCP 全连接队列最大值。默认 128 太小，高并发下连接建立时可能丢失 SYN。建议 65535。
- **tcp_max_syn_backlog**：半连接队列大小。默认 1024，高并发新建连接场景需要提升。
- **netdev_max_backlog**：网卡收包队列。流量突增时如有丢包，需调大。
- **sysctl 调优放在 DaemonSet initContainer 或节点初始化脚本中**，避免 Pod 权限不足无法修改主机内核参数。

`sysctl-daemonset.yaml`（参考格式，需在内网/VPN 环境下执行）：

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: sysctl-tuner
  namespace: kube-system
spec:
  selector:
    matchLabels:
      name: sysctl-tuner
  template:
    metadata:
      labels:
        name: sysctl-tuner
    spec:
      hostNetwork: true
      hostPID: true
      initContainers:
      - name: sysctl-apply
        image: busybox:1.36
        securityContext:
          privileged: true
        command:
        - sh
        - -c
        - |
          sysctl -w net.core.somaxconn=65535
          sysctl -w net.ipv4.tcp_max_syn_backlog=65535
          sysctl -w net.core.netdev_max_backlog=65535
          sysctl -w net.ipv4.tcp_tw_reuse=1
          sysctl -w net.ipv4.tcp_fin_timeout=30
          sysctl -w net.ipv4.ip_local_port_range="1024 65535"
      containers:
      - name: pause
        image: busybox:1.36
        command: ["sleep", "infinity"]
```

```bash
kubectl apply -f sysctl-daemonset.yaml
# expected: daemonset.apps/sysctl-tuner created
```

| 内核参数 | 默认值 | 建议值 | 说明 |
|---------|--------|--------|------|
| `net.core.somaxconn` | 128 | 65535 | 全连接队列最大值 |
| `net.ipv4.tcp_max_syn_backlog` | 1024 | 65535 | 半连接队列 |
| `net.core.netdev_max_backlog` | 1000 | 65535 | 网卡收包队列 |
| `net.ipv4.tcp_tw_reuse` | 0 | 1 | TIME_WAIT 复用 |
| `net.ipv4.tcp_fin_timeout` | 60 | 30 | FIN_WAIT2 超时 |

### 步骤 3：验证配置生效

```bash
# 查看 Nginx Ingress 实际配置
kubectl -n ingress-nginx exec -it deployment/nginx-ingress-ingress-nginx-controller -- nginx -T | grep -E "worker_connections|keepalive_timeout|client_max_body_size|upstream_keepalive"
# expected: worker_connections 65536, keepalive_timeout 300s, client_max_body_size 100m

# 验证内核参数（登录节点后执行，须 VPN/IOA + SSH）
sysctl net.core.somaxconn net.ipv4.tcp_max_syn_backlog
# expected: net.core.somaxconn = 65535
```

## 验证

### 控制面（tccli）

```bash
# 验证 Pod 副本数
tccli tke DescribeClusterInstances --region <Region> \
    --ClusterId CLUSTER_ID
# expected: 返回 Running 的节点数，确认节点数量 ≥ 3

# 验证 CLB 状态
tccli clb DescribeLoadBalancers --region <Region> \
    --LoadBalancerIds '["CLB_ID"]'
# expected: Status 为 1
```

```json
{
  "TotalCount": "<TotalCount>",
  "InstanceSet": "<InstanceSet>",
  "InstanceId": "<InstanceId>",
  "InstanceRole": "<InstanceRole>",
  "FailedReason": "<FailedReason>",
  "InstanceState": "<InstanceState>"
}
```

### 数据面（需 VPN/IOA）

```bash
# 验证 Helm 配置
helm get values nginx-ingress -n ingress-nginx
# expected: 显示 maxWorkerConnections、keepalive、config 等自定义配置

# 验证 Pod 数量
kubectl -n ingress-nginx get pods
# expected: ≥ 3 个 Controller Pod Running

# 验证 Nginx 运行配置
kubectl -n ingress-nginx exec -it deployment/nginx-ingress-ingress-nginx-controller -- nginx -T 2>/dev/null | grep -E "worker_connections|keepalive_timeout"
# expected: worker_connections 65536, keepalive_timeout 300s

# 验证内核参数（须节点 SSH 访问）
kubectl -n kube-system get pods -l name=sysctl-tuner
# expected: sysctl-tuner DaemonSet Pod 在每节点 Running

# 压测验证（使用 ab/wrk）
# kubectl run -it --rm load-generator --image=busybox:1.36 -- sh
# 在 Pod 内用 wget/curl 测试并发
```

## 清理

### 数据面（需 VPN/IOA）

```bash
# 删除 sysctl DaemonSet
kubectl delete daemonset sysctl-tuner -n kube-system
# expected: daemonset.apps "sysctl-tuner" deleted

# 恢复 Nginx Ingress 到默认配置
helm rollback nginx-ingress -n ingress-nginx
# expected: Rollback was a success
```

### 控制面（tccli）

```bash
# 验证 CLB 和集群状态无变化
tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterStatus "Running"
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

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `helm upgrade` 返回 `UPGRADE FAILED` | `helm history nginx-ingress -n ingress-nginx` | 上次部署失败，Release 处于 failed 状态 | `helm rollback nginx-ingress -n ingress-nginx` 回滚后重试 |
| `sysctl -w` 返回 `Read-only file system` | 容器挂载 `/proc/sys` 为只读 | 非 privileged 容器无法修改内核参数 | 在 Pod SecurityContext 中设置 `privileged: true` |
| `nginx -T` 命令返回 `command not found` | Pod 镜像中 Nginx 路径不同 | 某些发行版 Nginx 二进制路径为 `/usr/sbin/nginx` | 使用完整路径或 `which nginx` 确定位置 |

### 配置生效后功能异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 高并发下出现 502/504 | `kubectl -n ingress-nginx logs -l app.kubernetes.io/name=ingress-nginx --tail=100` | 后端连接池耗尽（upstream-keepalive-connections 不足）或 Pod 资源不足 | 增加 `upstream-keepalive-connections` 值；增加 Pod 副本数和资源限制 |
| 大量 413 错误 | `kubectl -n ingress-nginx logs -l app.kubernetes.io/name=ingress-nginx | grep 413` | `proxy-body-size` 设置过小 | 增大 `proxy-body-size` 或设为 `"0"` |
| Controller CPU 高但 QPS 不增 | `kubectl top pod -n ingress-nginx` 查看资源使用 | Gzip 压缩或 SSL/TLS 卸载消耗大量 CPU | 增加 CPU limit，或关闭 Gzip（`use-gzip: "false"`） |
| 连接建立时有 SYN 丢包 | 登录节点执行 `netstat -s | grep -i listen` 检查 `ListenOverflows` | `somaxconn` / `tcp_max_syn_backlog` 不足 | 增加 sysctl 参数值 |

## 下一步

- [高可用配置优化](https://cloud.tencent.com/document/product/457/104860) — PDB、拓扑约束、健康检查
- [可观测性集成](https://cloud.tencent.com/document/product/457/104861) — Prometheus 监控、CLS 日志
- [启用 CLB 直连](https://cloud.tencent.com/document/product/457/104866) — CLB 直连 Pod 减少一层转发

## 控制台替代

通过 [TKE 控制台 - 工作负载](https://console.cloud.tencent.com/tke2/cluster/sub/list/deployment/deployment?rid=1) 查看 Pod 状态和资源使用。内核参数调优需通过 [CVM 控制台](https://console.cloud.tencent.com/cvm) 登录节点或使用自定义镜像。
