# CronJob 管理（tccli）

> 对照官方：[CronJob 管理](https://cloud.tencent.com/document/product/457/31709) · page_id `31709`

## 概述

CronJob 是 Kubernetes 控制器，按 Cron 表达式定时创建 Job，执行周期性任务（如数据备份、报表生成、日志清理）。通过 kubectl 创建、查看和删除 CronJob。

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
| 创建 CronJob | `kubectl apply -f cronjob.yaml` | 是 |
| 查看 CronJob 列表 | `kubectl get cronjob` | 是 |
| 查看详情 | `kubectl describe cronjob/<name>` | 是 |
| 查看触发的 Job | `kubectl get jobs` | 是 |
| 暂停 CronJob | `kubectl patch cronjob/<name> -p '{"spec":{"suspend":true}}'` | 是 |
| 删除 CronJob | `kubectl delete cronjob/<name>` | 是 |

## 操作步骤

### 步骤 1：创建 CronJob

YAML 清单：

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cronjob-example
  namespace: default
spec:
  schedule: "*/5 * * * *"
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: hello
              image: busybox:latest
              command: ["/bin/sh", "-c", "date; echo Hello from CronJob"]
          restartPolicy: OnFailure
```

| 参数 | 说明 | 示例 |
|------|------|------|
| `schedule` | Cron 表达式 | `"*/5 * * * *"` 每 5 分钟 |
| `concurrencyPolicy` | 并发策略 | `Forbid` 禁止并发、`Allow` 允许、`Replace` 替换旧 Job |
| `successfulJobsHistoryLimit` | 保留成功 Job 数 | `3` |
| `failedJobsHistoryLimit` | 保留失败 Job 数 | `1` |
| `suspend` | 暂停调度 | `true` / `false` |

```bash
kubectl apply -f cronjob.yaml
# expected: cronjob.batch/cronjob-example created
```

**预期输出**：

```text
cronjob.batch/cronjob-example created
```

### 步骤 2：查看 CronJob 状态

```bash
kubectl get cronjob cronjob-example
# expected: SCHEDULE 列显示 Cron 表达式，LAST SCHEDULE 显示上次触发时间
```

```text
NAME  STATUS  AGE
...
```

```bash
kubectl describe cronjob/cronjob-example
# expected: 显示 CronJob 详情、最近 Job 和事件
```

```text
NAME  STATUS  AGE
...
```

### 步骤 3：查看触发的 Job

```bash
kubectl get jobs -l job-name
# expected: CronJob 定时创建的 Job 列表
```

```text
NAME  STATUS  AGE
...
```

```bash
kubectl get pods --selector=job-name
# expected: Job 关联的 Pod
```

```text
NAME  STATUS  AGE
...
```

### 步骤 4：暂停和恢复

```bash
# 暂停 CronJob
kubectl patch cronjob cronjob-example -p '{"spec":{"suspend":true}}'
# expected: cronjob.batch/cronjob-example patched

# 恢复 CronJob
kubectl patch cronjob cronjob-example -p '{"spec":{"suspend":false}}'
# expected: cronjob.batch/cronjob-example patched
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
kubectl get cronjob cronjob-example
# expected: ACTIVE 列显示正在运行的 Job 数

kubectl get jobs
# expected: 定时触发的 Job 记录

kubectl logs job/<job-name>
# expected: Hello from CronJob
```

```text
NAME  STATUS  AGE
...
```

## 清理

```bash
kubectl delete cronjob cronjob-example
# expected: cronjob.batch "cronjob-example" deleted
```

> **注意**：删除 CronJob 不会删除已创建的 Job，需手动清理：

```bash
kubectl delete jobs -l job-name
# expected: 关联 Job 已删除
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `Unable to connect to the server` | `kubectl cluster-info` 检查连通性 | 公网端点被 CAM 策略 strategyId:240463971 以 `tke:clusterExtranetEndpoint=true` 条件拒绝 | 通过 IOA/VPN 连接内网，或使用同 VPC CVM 执行 kubectl 命令 |
| Pod `ImagePullBackOff` | `kubectl describe pod <pod>` 查看事件 | 镜像拉取失败 | 检查镜像地址和访问凭证 |

### 资源创建成功但状态异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| CronJob 未触发新 Job | `kubectl describe cronjob/<name>` 查看 Events | `suspend` 为 `true` 或 Cron 表达式错误 | 检查 `schedule` 字段格式和 `suspend` 状态 |
| Job 持续失败 | `kubectl logs job/<job-name>` 查看日志 | 容器命令执行失败 | 检查 command/args 和镜像内容 |
| 多个 Job 并发运行 | `kubectl get jobs` 查看 | `concurrencyPolicy` 设置为 `Allow` | 改为 `Forbid` 或 `Replace` |

## 下一步

- [Job 管理](../Job%20管理/tccli%20操作.md) — page_id `31708`
- [Deployment 管理](../Deployment%20管理/tccli%20操作.md) — page_id `31705`

## 控制台替代

[容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) → 集群 → 工作负载 → CronJob → 新建。
