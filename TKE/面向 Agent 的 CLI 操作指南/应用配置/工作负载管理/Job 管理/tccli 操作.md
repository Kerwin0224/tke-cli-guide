# Job 管理（tccli）

> 对照官方：[Job 管理](https://cloud.tencent.com/document/product/457/31708) · page_id `31708`

## 概述

Job 是 Kubernetes 控制器，用于运行一次性任务至完成。与 Deployment 不同，Job 创建的 Pod 在成功执行后退出的容器不会被重启。支持并行执行、失败重试和完成确认。通过 kubectl 创建、查看和删除 Job。

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
| 创建 Job | `kubectl apply -f job.yaml` | 是 |
| 查看 Job 列表 | `kubectl get job` | 是 |
| 查看详情 | `kubectl describe job/<name>` | 是 |
| 查看 Pod 日志 | `kubectl logs job/<name>` | 是 |
| 删除 Job | `kubectl delete job/<name>` | 是 |

## 操作步骤

### 步骤 1：创建 Job

YAML 清单：

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: job-example
  namespace: default
spec:
  completions: 1
  parallelism: 1
  backoffLimit: 4
  activeDeadlineSeconds: 300
  template:
    spec:
      containers:
        - name: pi
          image: perl:latest
          command: ["perl", "-Mbignum=bpi", "-wle", "print bpi(2000)"]
      restartPolicy: Never
```

| 参数 | 说明 | 示例 |
|------|------|------|
| `completions` | 完成 Pod 总数 | `1` |
| `parallelism` | 并行 Pod 数 | `1` |
| `backoffLimit` | 失败重试次数 | `4` |
| `activeDeadlineSeconds` | 最长运行时间（秒） | `300` |
| `restartPolicy` | 重启策略 | `Never` 或 `OnFailure` |

```bash
kubectl apply -f job.yaml
# expected: job.batch/job-example created
```

**预期输出**：

```text
job.batch/job-example created
```

### 步骤 2：查看 Job 状态

```bash
kubectl get job job-example
# expected: COMPLETIONS 列显示 1/1
```

```text
NAME  STATUS  AGE
...
```

```bash
kubectl describe job/job-example
# expected: 显示 Job 详情、Pod 状态和事件
```

```text
NAME  STATUS  AGE
...
```

### 步骤 3：查看 Job 输出

```bash
kubectl logs job/job-example
# expected: 输出计算结果
```

**预期输出**（示例）：

```text
3.14159265358979323846...
```

### 步骤 4：并行 Job

设置 `parallelism` 和 `completions` 实现并行处理：

```yaml
spec:
  completions: 5
  parallelism: 2
  template:
    spec:
      containers:
        - name: worker
          image: busybox:latest
          command: ["echo", "task done"]
      restartPolicy: OnFailure
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
kubectl get job job-example
# expected: COMPLETIONS 1/1

kubectl get pods --selector=job-name=job-example
# expected: Pod 状态 Completed
```

```text
NAME  STATUS  AGE
...
```

## 清理

```bash
kubectl delete job job-example
# expected: job.batch "job-example" deleted
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `Unable to connect to the server` | `kubectl cluster-info` 检查连通性 | 公网端点被 CAM 策略 strategyId:240463971 以 `tke:clusterExtranetEndpoint=true` 条件拒绝 | 通过 IOA/VPN 连接内网，或使用同 VPC CVM 执行 kubectl 命令 |
| Pod `ImagePullBackOff` | `kubectl describe pod <pod>` 查看事件 | 镜像拉取失败 | 检查镜像地址和访问凭证 |
| Pod `Pending` | `kubectl describe pod <pod>` 检查 Events | 资源不足或调度约束 | 检查节点资源和调度规则 |

### 资源创建成功但状态异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Job 状态 `Failed` | `kubectl logs job/<name>` 查看日志 | 容器命令执行失败 | 检查 command/args 和镜像内容 |
| Job 超过重试次数 | `kubectl describe job/<name>` 查看 Events | `backoffLimit` 耗尽 | 检查应用错误，调整重试次数 |
| Job 长时间不完成 | `kubectl get pods` 检查 Pod 状态 | Pod 卡住不退出 | 检查应用是否有退出逻辑，或设置 `activeDeadlineSeconds` |

## 下一步

- [CronJob 管理](../CronJob%20管理/tccli%20操作.md) — page_id `31709`
- [Deployment 管理](../Deployment%20管理/tccli%20操作.md) — page_id `31705`

## 控制台替代

[容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) → 集群 → 工作负载 → Job → 新建。
