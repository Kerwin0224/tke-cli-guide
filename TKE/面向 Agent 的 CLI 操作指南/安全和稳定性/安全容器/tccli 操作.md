# 安全容器（Kata）（tccli）

> 对照官方：[安全容器（Kata）](https://cloud.tencent.com/document/product/457/129421) · page_id `129421`

## 概述

TKE 安全容器基于 Kata Containers 技术，为每个 Pod 提供独立的轻量级虚拟机（microVM），实现 VM 级别的强隔离。与标准容器共享宿主机内核不同，安全容器中的 Pod 拥有独立的内核，适用于多租户环境、不可信工作负载等安全敏感场景。TKE 提供两种运行时类型：`kata-clh`（基于 Cloud Hypervisor，性能优先）和 `kata-qemu`（基于 QEMU，兼容性优先）。

安全容器节点池需通过 TKE 控制台或 MachineSet YAML 创建，无直接 tccli API。节点池创建后，TKE 自动注册 RuntimeClass 并为节点添加 `katacontainers.io/kata-runtime=true` 标签。本页演示如何在已具备 Kata 节点的集群中通过 kubectl 部署和验证安全容器工作负载。

## 前置条件

- 演示集群 `cls-xxxxxxxx`（ap-guangzhou, v1.30.0, Running），kubectl 因 CAM 策略限制不可达，本页 kubectl 命令需在端点可达的环境中执行
- 完成 [环境准备](../../环境准备.md)
- **严格前置条件：**
  - Kubernetes 版本 >= 1.30
  - 仅支持原生节点（Native Node）
  - 仅支持 BMS5 裸金属服务器实例类型
  - 节点操作系统为 Tencent OS 4
- 安全容器节点池需通过 [TKE 控制台](https://console.cloud.tencent.com/tke2/cluster) 或 MachineSet YAML 创建（无直接 tccli API）

### 环境检查

```bash
# 1. 确认集群状态
tccli tke DescribeClusters --ClusterIds '["cls-xxxxxxxx"]' --region ap-guangzhou --output json
```

```json
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "cls-xxxxxxxx",
            "ClusterName": "demo-cluster",
            "ClusterType": "MANAGED_CLUSTER",
            "ClusterStatus": "Running",
            "ClusterVersion": "1.30.0",
            "ClusterNodeNum": 3,
            "ClusterOs": "tlinux3.1",
            "VpcId": "vpc-xxxxxxxx",
            "CreatedTime": "2024-06-15T10:30:00Z"
        }
    ],
    "RequestId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
}
```

```bash
# 2. 确认 Kata 节点存在（需端点可达）
kubectl get nodes -l katacontainers.io/kata-runtime=true
```

```text
NAME  STATUS  AGE
...
```

```output
NAME            STATUS   ROLES    AGE   VERSION
kata-node-001   Ready    <none>   10d   v1.30.x-tke.x
kata-node-002   Ready    <none>   10d   v1.30.x-tke.x
```

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 查看 Kata 节点 | `kubectl get nodes -l katacontainers.io/kata-runtime=true` | 是 |
| 查看 RuntimeClass | `kubectl get runtimeclass` | 是 |
| 部署安全容器工作负载 | `kubectl apply -f <deployment-yaml>` | 是（apply 幂等） |
| 查看 Pod 运行时 | `kubectl describe pod <pod-name>` | 是 |
| 创建 Kata 节点池 | 控制台 / MachineSet YAML（无 tccli API） | — |

## 操作步骤

### 1. 查看可用的 RuntimeClass（需端点可达）

```bash
kubectl get runtimeclass
```

```text
NAME  STATUS  AGE
...
```

```output
NAME        HANDLER     AGE
kata-clh    kata-clh    10d
kata-qemu   kata-qemu   10d
```

| RuntimeClass | 底层 Hypervisor | 适用场景 |
|-------------|----------------|----------|
| `kata-clh` | Cloud Hypervisor | 性能优先，启动速度快 |
| `kata-qemu` | QEMU | 兼容性优先，支持更多设备模拟 |

### 2. 部署安全容器工作负载（需端点可达）

以下 Deployment YAML 使用 `kata-qemu` 运行时部署 Nginx：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-kata
  labels:
    app: nginx-kata
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-kata
  template:
    metadata:
      labels:
        app: nginx-kata
    spec:
      runtimeClassName: kata-qemu
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: "500m"
            memory: "512Mi"
          limits:
            cpu: "1"
            memory: "1Gi"
```

```bash
kubectl apply -f nginx-kata-deployment.yaml
```

```text
# command executed successfully
```

```output
deployment.apps/nginx-kata created
```

### 3. 验证 Pod 运行状态（需端点可达）

```bash
kubectl get pods -l app=nginx-kata -o wide
```

```text
NAME  STATUS  AGE
...
```

```output
NAME                          READY   STATUS    RESTARTS   AGE   IP           NODE
nginx-kata-xxxxxxxxxx-xxxxx   1/1     Running   0          30s   10.0.x.x     kata-node-001
nginx-kata-xxxxxxxxxx-yyyyy   1/1     Running   0          30s   10.0.x.y     kata-node-002
```

查看 Pod 详情，确认使用了 Kata 运行时：

```bash
kubectl describe pod <nginx-kata-pod-name>
```

```text
Name:         ...
Status:       Running
...
```

在 describe 输出中可看到 `RuntimeClass: kata-qemu` 以及 Kata 相关的 Annotation。

### 4. 创建 Service 暴露服务（需端点可达）

```bash
kubectl expose deployment nginx-kata --port=80 --type=ClusterIP
```

```output
service/nginx-kata exposed
```

验证服务可达：

```bash
kubectl run curl-test --image=curlimages/curl:latest --rm -it --restart=Never -- curl -s -o /dev/null -w "%{http_code}" http://nginx-kata
```

```output
200
```

### 5. 不兼容字段说明

安全容器不支持以下 Pod Spec 字段（包含这些字段将导致创建失败）：

| 不兼容字段 | 说明 |
|-----------|------|
| `hostNetwork: true` | Kata Pod 无法使用宿主机网络 |
| `hostIPC: true` | Kata Pod 无法共享宿主机 IPC |
| `hostPID: true` | Kata Pod 无法共享宿主机 PID 命名空间 |
| `shareProcessNamespace: true` | Kata 已有独立的内核，不支持进程命名空间共享 |
| `securityContext.privileged: true` | 特权模式与 VM 级隔离冲突 |

### 6. 资源规划建议

安全容器的每个 Pod 需要额外的 VM 开销（约 128Mi 内存），建议 CPU 和内存的 requests 设置比标准容器高 10%-20%。Kata Pod 启动时间较标准容器长（约 1-3 秒的 VM 启动开销），建议适当增大 `initialDelaySeconds`。

## 验证

### 控制面（tccli）

```bash
# 确认集群状态正常
tccli tke DescribeClusters --ClusterIds '["cls-xxxxxxxx"]' --region ap-guangzhou --output json
```

```json
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "cls-xxxxxxxx",
            "ClusterName": "demo-cluster",
            "ClusterType": "MANAGED_CLUSTER",
            "ClusterStatus": "Running",
            "ClusterVersion": "1.30.0",
            "ClusterNodeNum": 3,
            "ClusterOs": "tlinux3.1",
            "VpcId": "vpc-xxxxxxxx",
            "CreatedTime": "2024-06-15T10:30:00Z"
        }
    ],
    "RequestId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
}
```

### 数据面（kubectl，需端点可达）

```bash
# 确认 Deployment 就绪
kubectl get deployment nginx-kata

# 确认 Pod 运行在 Kata 节点上
kubectl get pods -l app=nginx-kata -o wide

# 确认 RuntimeClass 符合预期
kubectl get pod <pod-name> -o jsonpath='{.spec.runtimeClassName}'

# 确认 Service 可访问
kubectl get svc nginx-kata
```

```text
NAME  STATUS  AGE
...
```

```output
kata-qemu
```

## 清理

```bash
kubectl delete deployment nginx-kata --ignore-not-found
kubectl delete service nginx-kata --ignore-not-found
kubectl delete pod curl-test --ignore-not-found
rm -f nginx-kata-deployment.yaml
```

> 安全容器节点池的删除需通过 [TKE 控制台](https://console.cloud.tencent.com/tke2/cluster) 完成。节点上的 Kata 运行时组件由 TKE 管理，请勿手动卸载。

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Pod 长时间 Pending | `kubectl get nodes -l katacontainers.io/kata-runtime=true` 确认节点存在且 Ready | 无符合标签的 Kata 节点，或节点资源不足 | 创建 Kata 节点池或减少副本数 |
| `runtimeClassName: kata-qemu` 报 NotFound | `kubectl get runtimeclass` 确认 RuntimeClass 存在 | 节点池未正确创建，RuntimeClass 未注册 | 通过控制台重新创建 Kata 节点池 |
| Pod 创建报不兼容字段错误 | 检查 Pod Spec 是否包含 `hostNetwork`、`hostPID`、`privileged` 等字段 | Kata 不支持共享宿主机命名空间或特权模式 | 移除不兼容字段后重新部署 |
| 启动延迟较长 | Kata VM 启动约需 1-3 秒，属正常现象 | VM 启动开销 | 调大 `initialDelaySeconds` 避免健康检查误报 |
| 网络不通 | 确认未使用 `hostNetwork` | 安全容器不支持 `hostNetwork` | 使用 CNI 网络模式（VPC-CNI 或 Global Router） |
| 内存不足 | Kata Pod 有额外 VM 开销（约 128Mi/Pod） | 节点规格不足 | 加大节点规格或减少副本数 |

## 下一步

- [集群安全增强能力](../集群安全增强能力/tccli%20操作.md) — 一键开启集群级安全加固
- [策略管理（OPA/Gatekeeper）](../应用安全/策略管理/tccli%20操作.md) — 准入控制与删除保护
- [Kata Containers 官方文档](https://katacontainers.io/docs/)

## 控制台替代

[TKE 控制台 → 集群详情](https://console.cloud.tencent.com/tke2/cluster) → 节点管理 → 节点池 → 新建节点池 → 选择「原生节点」类型 → 选择 BMS5 裸金属机型 → 操作系统选择 Tencent OS 4 → 开启「安全容器（Kata）」选项 → 选择运行时类型（kata-clh / kata-qemu）→ 完成创建。创建后 TKE 自动注册 RuntimeClass 并为节点添加 `katacontainers.io/kata-runtime=true` 标签。
