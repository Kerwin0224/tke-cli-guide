# DaemonSet 管理（tccli）

> 对照官方：[DaemonSet 管理](https://cloud.tencent.com/document/product/457/31706) · page_id `31706`

## 概述

DaemonSet 是 Kubernetes 控制器，确保在每个（或指定）节点上运行一个 Pod 副本。常用于部署日志采集、监控 Agent、网络插件等节点级守护进程。通过 kubectl 创建、更新和回滚 DaemonSet。

## 前置条件

- [环境准备](../../../环境准备.md)：`tccli` 已配置，`kubectl` >= v1.28
- 需要 CAM 权限：`tke:DescribeClusters`、`tke:DescribeClusterKubeconfig`

```bash
# 检查集群状态
tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'
# expected: ClusterStatus "Running"

# 获取 kubeconfig
tccli tke DescribeClusterKubeconfig --region <Region> --ClusterId <ClusterId> \
    | jq -r '.Kubeconfig' > kubeconfig.yaml
export KUBECONFIG=$(pwd)/kubeconfig.yaml
```

> **注意**：kubectl 数据面当前因公网端点被组织级 CAM 策略拒绝（strategyId:240463971）而不可达。外网端点被 `tke:clusterExtranetEndpoint=true` 条件拦截，内网端点需通过 IOA/VPN 或同 VPC 内 CVM 访问。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 获取集群凭证 | `tccli tke DescribeClusterKubeconfig --region <Region> --ClusterId <ClusterId>` | 是 |
| 创建 DaemonSet | `kubectl apply -f daemonset.yaml` | 是 |
| 查看 DaemonSet 列表 | `kubectl get daemonset` | 是 |
| 查看详情 | `kubectl describe daemonset/<name>` | 是 |
| 更新镜像 | `kubectl set image daemonset/<name> <container>=<image>` | 否 |
| 滚动更新状态 | `kubectl rollout status daemonset/<name>` | 是 |
| 版本回滚 | `kubectl rollout undo daemonset/<name>` | 否 |
| 删除 DaemonSet | `kubectl delete daemonset/<name>` | 是 |

## 操作步骤

### 步骤 1：创建 DaemonSet

YAML 清单：

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: daemonset-example
  namespace: kube-system
  labels:
    app: daemonset-example
spec:
  selector:
    matchLabels:
      app: daemonset-example
  template:
    metadata:
      labels:
        app: daemonset-example
    spec:
      containers:
        - name: log-collector
          image: fluentd:latest
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 200m
              memory: 256Mi
          volumeMounts:
            - name: varlog
              mountPath: /var/log
            - name: dockercontainers
              mountPath: /var/lib/docker/containers
              readOnly: true
      volumes:
        - name: varlog
          hostPath:
            path: /var/log
        - name: dockercontainers
          hostPath:
            path: /var/lib/docker/containers
```

```bash
kubectl apply -f daemonset.yaml
# expected: daemonset.apps/daemonset-example created
```

**预期输出**：

```text
daemonset.apps/daemonset-example created
```

### 步骤 2：查看 DaemonSet 状态

```bash
kubectl get daemonset daemonset-example -n kube-system
# expected: DESIRED 和 CURRENT 列等于节点数
```

```text
NAME  STATUS  AGE
...
```

```bash
kubectl get pods -l app=daemonset-example -n kube-system -o wide
# expected: 每个节点一个 Pod
```

```text
NAME  STATUS  AGE
...
```

```bash
kubectl describe daemonset/daemonset-example -n kube-system
# expected: 显示 DaemonSet 详情和事件
```

```text
NAME  STATUS  AGE
...
```

### 步骤 3：滚动更新与回滚

```bash
kubectl set image daemonset/daemonset-example log-collector=fluentd:v1.16 -n kube-system
# expected: daemonset.apps/daemonset-example image updated

kubectl rollout status daemonset/daemonset-example -n kube-system
# expected: daemonset "daemonset-example" successfully rolled out

kubectl rollout undo daemonset/daemonset-example -n kube-system
# expected: daemonset.apps/daemonset-example rolled back
```

### 步骤 4：限定运行节点

通过 `nodeSelector` 或 `tolerations` 控制 DaemonSet 运行的节点范围：

```yaml
spec:
  template:
    spec:
      nodeSelector:
        disktype: ssd
      tolerations:
        - key: "node-role.kubernetes.io/master"
          operator: "Exists"
          effect: "NoSchedule"
```

## 验证

### 控制面（tccli）

```bash
tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'
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

### 数据面（kubectl）

```bash
kubectl get daemonset daemonset-example -n kube-system
# expected: DESIRED=CURRENT=NUMBER AVAILABLE

kubectl get pods -l app=daemonset-example -n kube-system -o wide
# expected: 每个匹配节点一个 Running Pod
```

```text
NAME  STATUS  AGE
...
```

## 清理

```bash
kubectl delete daemonset daemonset-example -n kube-system
# expected: daemonset.apps "daemonset-example" deleted
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `Unable to connect to the server` | `kubectl cluster-info` 检查连通性 | 公网端点被 CAM 策略 strategyId:240463971 以 `tke:clusterExtranetEndpoint=true` 条件拒绝 | 通过 IOA/VPN 连接内网，或使用同 VPC CVM 执行 kubectl 命令 |
| Pod `ImagePullBackOff` | `kubectl describe pod <pod>` 查看事件 | 镜像拉取失败 | 检查镜像地址和访问凭证 |
| Pod `Pending` | `kubectl describe pod <pod>` 检查 Events | 节点污点未容忍或资源不足 | 添加 toleration 或检查节点资源 |

### 资源创建成功但状态异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| DESIRED 与 CURRENT 不匹配 | `kubectl describe daemonset/<name>` 查看 Events | 部分 Pod 创建失败 | 检查节点资源和镜像拉取状态 |
| Pod `CrashLoopBackOff` | `kubectl logs <pod>` 查看容器日志 | 容器启动崩溃 | 检查应用配置和 hostPath 挂载权限 |

## 下一步

- [Deployment 管理](../Deployment%20管理/tccli%20操作.md) — page_id `31705`
- [设置工作负载的调度规则](../设置工作负载的调度规则/tccli%20操作.md) — page_id `32814`

## 控制台替代

[容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) → 集群 → 工作负载 → DaemonSet → 新建。
