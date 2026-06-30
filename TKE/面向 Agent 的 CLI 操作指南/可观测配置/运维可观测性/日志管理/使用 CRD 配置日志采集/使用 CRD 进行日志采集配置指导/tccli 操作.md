# 使用 CRD 进行日志采集配置指导（tccli）

> 对照官方：[使用 CRD 进行日志采集配置指导](https://cloud.tencent.com/document/product/457/70991) · page_id `70991`

## 概述

除控制台配置外，TKE 还支持通过 CRD（CustomResourceDefinitions，`LogConfig` 资源）方式配置日志采集。CRD 方式支持采集容器标准输出、容器文件和主机文件，支持投递到 CLS 和 Kafka 等不同消费端。`LogConfig` CRD 的 apiVersion 为 `cls.cloud.tencent.com/v1`。

## 前置条件

- `tccli` 已配置凭证
- kubectl 不可达：演示集群 cls-xxxxxxxx 为托管集群，公网端点可能受 CAM 策略限制（strategyId:240463971）；内网无 VPN 时不可达。以下 kubectl 命令作为文档参考。
- 已在 [运维功能管理](https://console.cloud.tencent.com/tke2/ops/list) 中开启日志采集（`tccli tke InstallLogAgent`）
- 已在 CLS 中创建日志集和日志主题（或自动创建）

```bash
tccli tke DescribeClusters --ClusterIds '["cls-xxxxxxxx"]' --region ap-guangzhou
```

```json
{
  "RequestId": "1e8b2c3d-4a5f-6789-abcd-ef0123456789",
  "TotalCount": 1,
  "Clusters": [
    {
      "ClusterId": "cls-xxxxxxxx",
      "ClusterName": "my-test-cluster",
      "ClusterType": "MANAGED_CLUSTER",
      "ClusterStatus": "Running",
      "ClusterVersion": "1.30.0",
      "ClusterNodeNum": 2
    }
  ]
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 创建日志采集配置 | `kubectl apply -f logconfig.yaml` | 是 |
| 查看采集配置 | `kubectl get logconfig -A` | 是 |
| 查看配置详情 | `kubectl describe logconfig <Name>` | 是 |
| 更新采集配置 | `kubectl apply -f logconfig.yaml` | 是 |
| 删除采集配置 | `kubectl delete logconfig <Name>` | 否 |
| 查看 CRD 同步状态 | `kubectl get logconfig <Name> -o jsonpath='{.status.status}'` | 是 |
| 检查 Loglistener DaemonSet | `kubectl get ds -n kube-system loglistener-tke` | 是 |

## 操作步骤

### 示例一：容器标准输出（container_stdout）

采集 default 命名空间所有容器的标准输出，投递到 CLS：

```yaml
apiVersion: cls.cloud.tencent.com/v1
kind: LogConfig
metadata:
  name: test-stdout
spec:
  clsDetail:
    logsetName: example-logset
    topicName: example-topic
    logType: minimalist_log
    period: 30
  inputDetail:
    type: container_stdout
    containerStdout:
      allContainers: true
      namespace: default
```

```bash
kubectl apply -f logconfig-stdout.yaml
```

```text
logconfig.cls.cloud.tencent.com/test-stdout created
```

### 示例二：容器文件路径（container_file）

采集 default 命名空间 nginx 容器的 /var/log/nginx 目录日志：

```yaml
apiVersion: cls.cloud.tencent.com/v1
kind: LogConfig
metadata:
  name: test-container-file
spec:
  clsDetail:
    topicId: xxxxxx-xx-xx-xx-xxxxxxxx
    logType: minimalist_log
  inputDetail:
    type: container_file
    containerFile:
      namespace: default
      container: nginx
      logPath: /var/log/nginx
      filePattern: "*.log"
```

```bash
kubectl apply -f logconfig-container-file.yaml
```

```text
logconfig.cls.cloud.tencent.com/test-container-file created
```

### 示例三：节点文件路径（host_file）

采集节点 /var/log 目录下所有 .log 文件：

```yaml
apiVersion: cls.cloud.tencent.com/v1
kind: LogConfig
metadata:
  name: test-host-file
spec:
  clsDetail:
    topicId: xxxxxx-xx-xx-xx-xxxxxxxx
    logType: minimalist_log
  inputDetail:
    type: host_file
    hostFile:
      logPath: /var/log
      filePattern: "*.log"
      customLabels:
        env: production
```

```bash
kubectl apply -f logconfig-host-file.yaml
```

```text
logconfig.cls.cloud.tencent.com/test-host-file created
```

### 示例四：Kafka 消费端

采集容器标准输出并投递到 CKafka：

```yaml
apiVersion: cls.cloud.tencent.com/v1
kind: LogConfig
metadata:
  name: test-kafka
spec:
  inputDetail:
    type: container_stdout
    containerStdout:
      allContainers: true
      namespace: default
  kafkaDetail:
    brokers: x.x.x.x:9092
    topic: test-topic
    kafkaType: CKafka
    instanceId: ckafka-xxxxxx
    logType: minimalist_log
    outputType: json
```

```bash
kubectl apply -f logconfig-kafka.yaml
```

```text
logconfig.cls.cloud.tencent.com/test-kafka created
```

### 查看所有 LogConfig

```bash
kubectl get logconfig -A
```

```text
NAME                 AGE
test-stdout          5m
test-container-file  3m
test-host-file       1m
test-kafka           30s
```

### 查看 CRD 同步状态

`status.status` 为 `Synced` 表示采集配置处理成功，为 `Stale` 表示失败：

```bash
kubectl get logconfig test-stdout -o jsonpath='{.status.status}'
```

```text
Synced
```

### 删除 LogConfig

```bash
kubectl delete logconfig test-stdout
```

```text
logconfig.cls.cloud.tencent.com "test-stdout" deleted
```

## 验证

### Data plane (kubectl) — 参考

```bash
kubectl get logconfig -A && kubectl get logconfig <Name> -o jsonpath='{.status}'
```

```text
NAME            AGE
test-stdout     5m
{"status":"Synced"}
```

## 清理

### Data plane (kubectl) — 参考

```bash
kubectl delete logconfig --all
```

## 排障

| 现象 | 处理 |
|------|------|
| status 为 Stale | 检查 inputDetail 配置是否正确；确保日志路径在容器中存在 |
| Kafka 采集限制 | 单规则最多监听约 256 个文件；超出需提交工单 |
| 日志单行长度限制 | 标准输出最大 16KB；容器/节点文件最大 2MB |
| 容器文件路径采集失败 | 不能为符号链接或硬链接；确保路径在采集器容器内存在 |
| metadataLabels 不生效 | LogConfig status 为空表示初始状态，正确配置后变为 Synced |
| kubectl 不可达 | 托管集群公网端点可能受 CAM 策略限制（strategyId:240463971）；内网无 VPN 时不可达。CRD 配置方式的 kubectl 命令作为文档参考；采集规则也可通过控制台或 `tccli tke CreateCLSLogConfig` 创建 |

## 下一步

- [LogConfig JSON 格式说明](../LogConfig json 格式说明/tccli 操作.md) — LogConfig CRD 字段完整参考
- [采集容器日志到 CLS](../../采集容器日志到 CLS/tccli 操作.md) — 控制台方式配置日志采集
- [日志采集概述](../../日志采集概述/tccli 操作.md) — 日志采集基本概念

## 控制台替代

登录 [容器服务控制台](https://console.cloud.tencent.com/tke2) > 选择集群 > 日志 > 业务日志 > 新增采集配置。
