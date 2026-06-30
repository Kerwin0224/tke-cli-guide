# 使用 CRD 采集日志到 Kafka

> 对照官方：[使用 CRD 采集日志到 Kafka](https://cloud.tencent.com/document/product/457/61092) · page_id `61092`

## 概述

TKE Serverless（EKS）集群通过 **LogConfig CRD** 声明式定义日志采集规则，将容器 stdout 或文件日志投递至自建 Kafka 集群。与 CLS 方案不同，Kafka 方案适用于已有 Kafka 基础设施、需要自行消费和处理日志的场景（如对接自建 ELK、自定义分析平台）。

采集链路：**Pod stdout/文件日志 -> LogConfig CRD -> log-agent（平台管理）-> Kafka Topic**。

注意：TKE Serverless 集群无节点概念，log-agent 由平台自动部署和管理，无需手动安装。

> **kubectl 数据面不可达**：CAM 拒绝公网端点 (strategyId:240463971)，内网需 VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

## 前置条件

- [环境准备](../../../../../环境准备.md)
- 至少一个运行中的 TKE Serverless 集群
- 自建 Kafka 集群可达（内网或公网，需集群 VPC 与 Kafka 网络互通）
- 已在 Kafka 中创建目标 Topic

### 环境检查

```bash
tccli --version
# expected: tccli version 3.0.x

tccli configure list
# expected: secretId, secretKey, region 均已配置

tccli tke DescribeEKSClusters --region <Region> | jq '.Clusters[] | select(.ClusterStatus=="Running")'
# expected: 至少返回1个Running集群

# 验证 kubectl 可访问 APIServer（需 VPN/IOA）
kubectl cluster-info
# expected: Kubernetes control plane is running at ...

# 验证 Kafka 可达（使用 kubectl 运行临时 Pod 测试）
kubectl run kafka-test --rm -it --restart=Never --image=busybox:1.28 -n default -- \
    nc -zv KAFKA_BROKER_HOST KAFKA_BROKER_PORT
# expected: KAFKA_BROKER_HOST (KAFKA_BROKER_PORT) open
```

```json
{
  "TotalCount": "<TotalCount>",
  "Clusters": "<Clusters>",
  "ClusterId": "<ClusterId>",
  "ClusterName": "<ClusterName>",
  "VpcId": "<VpcId>",
  "SubnetIds": "<SubnetIds>"
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli / kubectl 命令 | 幂等 |
|-----------|---------------------|:--:|
| 创建日志采集规则 | `kubectl apply -f logconfig-kafka.yaml` | 是 |
| 查看采集规则 | `kubectl get logconfigs` | 是 |
| 查看规则详情 | `kubectl describe logconfig <name>` | 是 |
| 修改采集规则 | `kubectl apply -f logconfig-kafka.yaml` | 是 |
| 删除采集规则 | `kubectl delete logconfig <name>` | 否 |

## 操作步骤

### 占位符说明

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|----------|
| `NAMESPACE` | Kubernetes 命名空间 | 不超过 63 字符 | 自定义或使用 default |
| `KAFKA_BROKER_HOST` | Kafka Broker 地址 | IP 或域名 | 自建 Kafka 集群 |
| `KAFKA_BROKER_PORT` | Kafka Broker 端口 | 通常 9092 | 自建 Kafka 集群 |
| `KAFKA_TOPIC` | 目标 Kafka Topic 名称 | 不超过 255 字符 | Kafka 中已创建的 Topic |
| `CLUSTER_ID` | EKS 集群 ID | 格式 cls-xxxxxxxx | `DescribeEKSClusters` |

### 步骤1: 创建 LogConfig CRD 采集 stdout 日志到 Kafka

LogConfig 是 TKE 自定义资源，用于定义日志采集规则。以下 YAML 采集指定 namespace 下所有容器的 stdout/stderr 日志。

```bash
cat <<'EOF' > logconfig-kafka-stdout.yaml
apiVersion: cls.cloud.tencent.com/v1
kind: LogConfig
metadata:
  name: stdout-to-kafka
  namespace: NAMESPACE
spec:
  clsDetail:
    # Kafka 目标配置（type=kafka）
    logType: container_stdout
    extractRule: {}
  inputDetail:
    type: container_stdout
    containerStdout:
      allContainers: true
      namespace: NAMESPACE
  outputDetail:
    type: kafka
    kafkaDetail:
      brokers: "KAFKA_BROKER_HOST:KAFKA_BROKER_PORT"
      topic: "KAFKA_TOPIC"
      # SASL 认证（可选，如 Kafka 启用了 SASL）
      # securityProtocol: "SASL_PLAINTEXT"
      # mechanism: "PLAIN"
      # username: "KAFKA_USERNAME"
      # password: "KAFKA_PASSWORD"
EOF
kubectl apply -f logconfig-kafka-stdout.yaml
# expected: logconfig.cls.cloud.tencent.com/stdout-to-kafka created
```

### 步骤2: 创建 LogConfig CRD 采集指定文件日志到 Kafka

采集容器内指定路径的文件日志（如应用日志文件 `/var/log/app/*.log`）。

```bash
cat <<'EOF' > logconfig-kafka-file.yaml
apiVersion: cls.cloud.tencent.com/v1
kind: LogConfig
metadata:
  name: file-to-kafka
  namespace: NAMESPACE
spec:
  inputDetail:
    type: container_file
    containerFile:
      namespace: NAMESPACE
      workload:
        name: "DEPLOYMENT_NAME"
        kind: "Deployment"
      container: "CONTAINER_NAME"
      logPath: "/var/log/app/"
      filePattern: "*.log"
      customLabels:
        app: "my-app"
        env: "production"
  outputDetail:
    type: kafka
    kafkaDetail:
      brokers: "KAFKA_BROKER_HOST:KAFKA_BROKER_PORT"
      topic: "KAFKA_TOPIC_FILE"
      # 自定义 Kafka 消息 Key（基于日志字段）
      messageKey:
        valueFrom:
          fieldRef:
            fieldPath: "metadata.namespace"
      # 消息时间戳来源
      timestampKey: "@timestamp"
      timestampFormat: "rfc3339nano"
EOF
kubectl apply -f logconfig-kafka-file.yaml
# expected: logconfig.cls.cloud.tencent.com/file-to-kafka created
```

### 步骤3: 多源采集到同一 Kafka Topic

同一 namespace 下采集多种来源日志到同一 Kafka Topic。

```bash
cat <<'EOF' > logconfig-kafka-multi.yaml
apiVersion: cls.cloud.tencent.com/v1
kind: LogConfig
metadata:
  name: multi-source-to-kafka
  namespace: NAMESPACE
spec:
  inputDetail:
    type: container_multi
    containerMulti:
      sources:
        - type: container_stdout
          containerStdout:
            allContainers: true
            namespace: NAMESPACE
        - type: container_file
          containerFile:
            namespace: NAMESPACE
            workload:
              name: "app-deployment"
              kind: "Deployment"
            container: "app"
            logPath: "/var/log/nginx/"
            filePattern: "access.log"
  outputDetail:
    type: kafka
    kafkaDetail:
      brokers: "KAFKA_BROKER_HOST:KAFKA_BROKER_PORT"
      topic: "KAFKA_TOPIC"
EOF
kubectl apply -f logconfig-kafka-multi.yaml
# expected: logconfig.cls.cloud.tencent.com/multi-source-to-kafka created
```

## 验证

### 数据面（需 VPN/IOA）

```bash
# 1. 查看 LogConfig 列表及状态
kubectl get logconfigs -n NAMESPACE
# expected:
# NAME                    STATUS    AGE
# stdout-to-kafka         Running   5m
# file-to-kafka           Running   3m

# 2. 查看 LogConfig 详细状态
kubectl describe logconfig stdout-to-kafka -n NAMESPACE
# expected: Status.Conditions 中 Type 为 Running，Status 为 True

# 3. 验证 log-agent 组件正常运行（由平台管理）
kubectl get pods -n kube-system -l app=log-agent
# expected: 所有 Pod STATUS=Running, READY=1/1

# 4. 创建测试 Pod 产生日志并验证 Kafka 消费
kubectl run log-test --image=busybox:1.28 --restart=Never -n NAMESPACE -- \
    /bin/sh -c 'echo "KAFKA_TEST_MSG_$(date +%s)"; sleep 3600'
# expected: pod/log-test created

# 5. 等待 1-2 分钟后，通过 Kafka 消费者检查 Topic 中有对应消息
kubectl run kafka-consumer --rm -it --restart=Never --image=bitnami/kafka:3.6 -n default -- \
    kafka-console-consumer.sh --bootstrap-server KAFKA_BROKER_HOST:KAFKA_BROKER_PORT \
    --topic KAFKA_TOPIC --from-beginning --max-messages 5
# expected: 输出近期的日志消息（含 KAFKA_TEST_MSG_xxx 内容）
```

```text
NAME  STATUS  AGE
...
```

## 清理

> **计费警告**：Kafka 方案不通过腾讯云计费，但请确保清理不再使用的 LogConfig CRD 避免持续向 Kafka 写入无用数据。

```bash
# 1. 删除 LogConfig CRD 停止日志采集
kubectl delete logconfig stdout-to-kafka -n NAMESPACE
kubectl delete logconfig file-to-kafka -n NAMESPACE
kubectl delete logconfig multi-source-to-kafka -n NAMESPACE
# expected: logconfig.cls.cloud.tencent.com "stdout-to-kafka" deleted

# 2. 删除测试 Pod
kubectl delete pod log-test -n NAMESPACE --ignore-not-found
# expected: pod "log-test" deleted

# 3. 确认所有 LogConfig 已清除
kubectl get logconfigs -n NAMESPACE
# expected: No resources found in NAMESPACE namespace.
```

```text
NAME  STATUS  AGE
...
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl apply` 返回 `error: unable to recognize "logconfig.yaml": no matches for kind "LogConfig"` | `kubectl api-resources \| grep logconfig` | LogConfig CRD 未在集群中注册（EKS 集群自动注册，旧集群可能缺失） | 联系平台确认集群版本，TKE Serverless 集群应自动包含 LogConfig CRD |
| `kubectl describe logconfig` Status 显示 `Error` | 查看 Conditions 字段错误信息 | Kafka Broker 地址不可达或认证失败 | 检查 Kafka 网络连通性和 SASL 凭据，使用测试 Pod `nc -zv` 验证 |
| `kubectl get pods -n kube-system -l app=log-agent` 无 Pod | log-agent 组件未部署 | 集群日志采集组件未初始化（首次使用需平台触发） | 在 TKE 控制台中开启一次日志采集功能以触发组件部署，或联系工单处理 |
| Kafka 消费者无消息 | 检查 `kubectl logs -n kube-system log-agent-xxx` | LogConfig 配置的 namespace 或 container 选择器不匹配 | 确认 `namespace`、`workload.name`、`container` 与实际 Pod 一致 |

### 创建成功但状态异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| LogConfig Status 长时间 Pending | `kubectl describe logconfig LOG_CONFIG_NAME -n NAMESPACE` | log-agent 资源不足或 Kafka 连接超时 | 检查 Kafka 网络延迟，确认主题存在且可写入 |
| 部分容器日志未采集 | `kubectl get pods -n NAMESPACE` 确认容器状态 | LogConfig `allContainers: false` 且未列出目标容器 | 修改 LogConfig 添加 `allContainers: true` 或明确列出所有目标容器 |
| Kafka 消息乱序 | 检查 Kafka Topic 分区数和 LogConfig 消息 Key 配置 | Kafka 按 Key 分区，不同 Key 的消息可能跨分区导致乱序 | 配置 LogConfig `messageKey` 使用稳定的标识（如 pod-name）|
| 日志消息体过大被截断 | Kafka broker 日志中查看 `MessageTooLarge` 错误 | 单条日志超过 Kafka 消息大小限制（默认 1MB） | 在应用层限制单条日志大小，或调整 Kafka `message.max.bytes` 集群参数 |

## 下一步

- [开启日志采集](../使用%20CRD%20采集日志到%20CLS/开启日志采集/tccli%20操作.md)：采集日志到腾讯云 CLS 日志服务
- [通过 YAML 配置日志采集](../使用%20CRD%20采集日志到%20CLS/通过%20YAML%20配置日志采集/tccli%20操作.md)：LogConfig CRD 完整 YAML 参考
- [TKE LogConfig CRD 说明](https://cloud.tencent.com/document/product/457/78446)：LogConfig CRD 字段说明文档

## 控制台替代

控制台路径：**TKE 控制台 > Serverless 集群 > 集群 ID > 日志采集 > 新建日志采集规则**。在控制台中选择输出类型为 Kafka，填写 Broker 地址和 Topic 名称。控制台操作底层等效于创建 LogConfig CRD，但提供表单式配置界面，适合首次使用或不熟悉 CRD YAML 的用户。
