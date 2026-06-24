# 通过 YAML 配置日志采集

> 对照官方：[通过 YAML 配置日志采集](https://cloud.tencent.com/document/product/457/56750) · page_id `56750`

## 概述

本文档提供 TKE Serverless（EKS）集群中 **LogConfig CRD** 的完整 YAML 配置参考和 CLI 操作指南。LogConfig CRD 支持声明式定义日志采集规则，包括 stdout 采集、容器文件日志采集、多源聚合、正则解析、多行合并等高级特性。

通过 `kubectl apply -f` 管理 LogConfig YAML，配合 `tccli cls` 查询 CLS 侧资源状态，实现日志采集配置的版本化管理和 GitOps 工作流。

TKE Serverless 集群由平台自动管理 log-agent，LogConfig YAML 提交后 agent 自动感知变更并在数十秒内生效。

> **kubectl 数据面不可达**：CAM 拒绝公网端点 (strategyId:240463971)，内网需 VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

## 前置条件

- [环境准备](../../../../../../环境准备.md)
- 至少一个运行中的 TKE Serverless 集群
- 已完成 CLS 日志集和日志主题创建
- kubectl 已配置并可访问集群 APIServer（需 VPN/IOA）

### 环境检查

```bash
tccli --version
# expected: tccli version 3.0.x

tccli configure list
# expected: secretId, secretKey, region 均已配置

# 验证 CLS 日志集和主题
tccli cls DescribeLogsets --region <Region> --Limit 10 | jq '.Logsets[] | {LogsetId, LogsetName}'
# expected: 至少返回1个日志集

# 验证 kubectl 可达
kubectl cluster-info
# expected: Kubernetes control plane is running at ...

# 验证 LogConfig CRD 已注册
kubectl api-resources | grep logconfigs
# expected: logconfigs  cls.cloud.tencent.com/v1  true  LogConfig

# 查看现有 LogConfig（可选）
kubectl get logconfigs -A
# expected: 列出已有 LogConfig，或 No resources found
```

## 控制台与 CLI 参数映射

| 控制台操作 | kubectl / tccli 命令 | 幂等 |
|-----------|---------------------|:--:|
| 提交 YAML 配置 | `kubectl apply -f logconfig.yaml` | 是 |
| 查看配置 YAML | `kubectl get logconfig <name> -n <ns> -o yaml` | 是 |
| 编辑配置 | `kubectl edit logconfig <name> -n <ns>` | 是（声明式） |
| 删除配置 | `kubectl delete -f logconfig.yaml` | 否 |
| 批量部署 | `kubectl apply -f logconfigs/` | 是 |
| 查看 CLS 主题 | `tccli cls DescribeTopics --TopicId TOPIC_ID` | 是 |
| 检索日志验证 | `tccli cls SearchLog --TopicId TOPIC_ID` | 是 |

## 操作步骤

### 占位符说明

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|----------|
| `NAMESPACE` | Kubernetes 命名空间 | 不超过 63 字符 | 自定义或使用 default |
| `LOGSET_ID` | CLS 日志集 ID | UUID 格式 xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx | `tccli cls DescribeLogsets` |
| `TOPIC_ID` | CLS 日志主题 ID | UUID 格式 | `tccli cls DescribeTopics` |
| `REGION` | 地域 | 如 ap-guangzhou | 固定值 |
| `CONTAINER_NAME` | 容器名称 | spec.containers[].name | `kubectl get pod <name> -o jsonpath='{.spec.containers[*].name}'` |

### 步骤1: 基础 stdout 采集 YAML

最简配置：采集指定 namespace 下所有容器的 stdout/stderr 日志到 CLS。

```yaml
# logconfig-basic-stdout.yaml
apiVersion: cls.cloud.tencent.com/v1
kind: LogConfig
metadata:
  name: basic-stdout
  namespace: NAMESPACE
spec:
  clsDetail:
    logsetId: "LOGSET_ID"
    topicId: "TOPIC_ID"
    logType: container_stdout
  inputDetail:
    type: container_stdout
    containerStdout:
      allContainers: true
      namespace: NAMESPACE
```

```bash
kubectl apply -f logconfig-basic-stdout.yaml
# expected: logconfig.cls.cloud.tencent.com/basic-stdout created
```

### 步骤2: 指定 Workload 的 stdout 采集

精准控制：只采集特定 Deployment 的特定容器 stdout。

```yaml
# logconfig-workload-stdout.yaml
apiVersion: cls.cloud.tencent.com/v1
kind: LogConfig
metadata:
  name: workload-stdout
  namespace: NAMESPACE
spec:
  clsDetail:
    logsetId: "LOGSET_ID"
    topicId: "TOPIC_ID"
    logType: container_stdout
  inputDetail:
    type: container_stdout
    containerStdout:
      allContainers: false
      namespace: NAMESPACE
      workloads:
        - name: "my-deployment"
          kind: "Deployment"
          container: "app"
        - name: "my-statefulset"
          kind: "StatefulSet"
          container: "worker"
```

```bash
kubectl apply -f logconfig-workload-stdout.yaml
# expected: logconfig.cls.cloud.tencent.com/workload-stdout created
```

### 步骤3: 容器文件日志采集（单文件）

采集容器内指定路径的文件日志。

```yaml
# logconfig-single-file.yaml
apiVersion: cls.cloud.tencent.com/v1
kind: LogConfig
metadata:
  name: single-file-log
  namespace: NAMESPACE
spec:
  clsDetail:
    logsetId: "LOGSET_ID"
    topicId: "TOPIC_ID"
    logType: container_file
  inputDetail:
    type: container_file
    containerFile:
      namespace: NAMESPACE
      workload:
        name: "nginx-deployment"
        kind: "Deployment"
      container: "nginx"
      logPath: "/var/log/nginx/"
      filePattern: "access.log"
      customLabels:
        service: "nginx"
        tier: "frontend"
```

```bash
kubectl apply -f logconfig-single-file.yaml
# expected: logconfig.cls.cloud.tencent.com/single-file-log created
```

### 步骤4: 多文件路径采集

同一 LogConfig 采集多个文件路径（逗号分隔）。

```yaml
# logconfig-multi-file.yaml
apiVersion: cls.cloud.tencent.com/v1
kind: LogConfig
metadata:
  name: multi-file-log
  namespace: NAMESPACE
spec:
  clsDetail:
    logsetId: "LOGSET_ID"
    topicId: "TOPIC_ID"
    logType: container_file
  inputDetail:
    type: container_file
    containerFile:
      namespace: NAMESPACE
      workload:
        name: "app-deployment"
        kind: "Deployment"
      container: "app"
      # 多个日志路径，逗号分隔
      logPath: "/var/log/app/,/var/log/nginx/"
      filePattern: "*.log"
      excludeLabels: {}
```

```bash
kubectl apply -f logconfig-multi-file.yaml
# expected: logconfig.cls.cloud.tencent.com/multi-file-log created
```

### 步骤5: 带正则解析的文件日志

使用正则表达式解析日志字段，生成结构化日志。

```yaml
# logconfig-regex-parse.yaml
apiVersion: cls.cloud.tencent.com/v1
kind: LogConfig
metadata:
  name: regex-parse-log
  namespace: NAMESPACE
spec:
  clsDetail:
    logsetId: "LOGSET_ID"
    topicId: "TOPIC_ID"
    logType: container_file
    extractRule:
      # 多行合并：以时间戳开头的行作为新日志起点
      beginningRegex: '^\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2}\.\d{3}'
      # 提取字段名
      keys:
        - "timestamp"
        - "level"
        - "logger"
        - "thread"
        - "message"
      # 正则表达式匹配（如 Java 日志格式）
      logRegex: '^(\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2}\.\d{3})\s+(\S+)\s+\d+\s+---\s+\[([^\]]+)\]\s+(\S+)\s+:\s+(.*)'
      timeKey: "timestamp"
      timeFormat: "%Y-%m-%d %H:%M:%S.%f"
      # 过滤特定日志级别
      filterKeys:
        - "level"
      filterRegex:
        level: "^(ERROR|WARN)$"
  inputDetail:
    type: container_file
    containerFile:
      namespace: NAMESPACE
      workload:
        name: "spring-app"
        kind: "Deployment"
      container: "app"
      logPath: "/var/log/app/"
      filePattern: "application.log"
```

```bash
kubectl apply -f logconfig-regex-parse.yaml
# expected: logconfig.cls.cloud.tencent.com/regex-parse-log created
```

### 步骤6: 多行日志合并采集

处理 Java 异常堆栈、Python traceback 等多行日志。

```yaml
# logconfig-multiline.yaml
apiVersion: cls.cloud.tencent.com/v1
kind: LogConfig
metadata:
  name: multiline-log
  namespace: NAMESPACE
spec:
  clsDetail:
    logsetId: "LOGSET_ID"
    topicId: "TOPIC_ID"
    logType: container_file
  inputDetail:
    type: container_file
    containerFile:
      namespace: NAMESPACE
      workload:
        name: "java-app"
        kind: "Deployment"
      container: "app"
      logPath: "/var/log/app/"
      filePattern: "app.log"
      # 多行配置：以日期开头的行作为新日志起点
      multiline:
        pattern: '^\d{4}-\d{2}-\d{2}'
        negate: true
        match: after
```

```bash
kubectl apply -f logconfig-multiline.yaml
# expected: logconfig.cls.cloud.tencent.com/multiline-log created
```

### 步骤7: 多源混合采集

同一 LogConfig 同时采集 stdout 和文件日志。

```yaml
# logconfig-hybrid.yaml
apiVersion: cls.cloud.tencent.com/v1
kind: LogConfig
metadata:
  name: hybrid-collection
  namespace: NAMESPACE
spec:
  clsDetail:
    logsetId: "LOGSET_ID"
    topicId: "TOPIC_ID"
    logType: minimalist_log
    # 混合日志使用自定义标签区分来源
    extractRule:
      keys:
        - "source_type"
        - "content"
      logRegex: '(.*)'
  inputDetail:
    type: container_multi
    containerMulti:
      sources:
        - type: container_stdout
          containerStdout:
            allContainers: true
            namespace: NAMESPACE
            customLabels:
              source_type: "stdout"
        - type: container_file
          containerFile:
            namespace: NAMESPACE
            workload:
              name: "app-deployment"
              kind: "Deployment"
            container: "app"
            logPath: "/var/log/app/"
            filePattern: "*.log"
            customLabels:
              source_type: "file"
```

```bash
kubectl apply -f logconfig-hybrid.yaml
# expected: logconfig.cls.cloud.tencent.com/hybrid-collection created
```

## 验证

### 数据面（需 VPN/IOA）

```bash
# 1. 查看所有 LogConfig 状态
kubectl get logconfigs -n NAMESPACE
# expected:
# NAME                 STATUS    AGE
# basic-stdout         Running   5m
# single-file-log      Running   3m

# 2. 检查单个 LogConfig 详情
kubectl describe logconfig basic-stdout -n NAMESPACE
# expected:
# Status:
#   Conditions:
#     Last Transition Time:  2024-01-01T00:00:00Z
#     Status:                True
#     Type:                  Running

# 3. 验证采集组件健康
kubectl get pods -n kube-system -l app=log-agent
# expected: Running 1/1

# 4. 产生测试日志
kubectl run yaml-log-test --image=busybox:1.28 --restart=Never -n NAMESPACE -- \
    sh -c 'echo "YAML_LOG_TEST_$(date +%s)"; sleep 3600'
# expected: pod/yaml-log-test created
```

```text
NAME  STATUS  AGE
...
```

### 控制面（tccli）

```bash
# 5. CLS 检索验证日志投递（等待 2-3 分钟后执行）
tccli cls SearchLog \
    --region <Region> \
    --TopicId TOPIC_ID \
    --From $(python3 -c "import time; print(int((time.time()-600)*1000))") \
    --To $(python3 -c "import time; print(int(time.time()*1000))") \
    --Query "YAML_LOG_TEST"
# expected: Results 数组含 YAML_LOG_TEST_xxx 条目

# 6. 查看 CLS 主题当前日志量
tccli cls DescribeTopics \
    --region <Region> \
    --TopicId TOPIC_ID | jq '.Topics[0] | {TopicName, Status}'
# expected: Status="1"（正常）
```

## 清理

```bash
# 1. 按名称删除 LogConfig（停止采集）
kubectl delete logconfig basic-stdout single-file-log multi-file-log \
    regex-parse-log multiline-log hybrid-collection -n NAMESPACE
# expected: logconfig.cls.cloud.tencent.com "basic-stdout" deleted ...

# 2. 或按 YAML 文件批量删除
kubectl delete -f logconfig-basic-stdout.yaml -f logconfig-single-file.yaml
# expected: logconfig.cls.cloud.tencent.com "basic-stdout" deleted ...

# 3. 清理测试 Pod
kubectl delete pod yaml-log-test -n NAMESPACE --ignore-not-found
# expected: pod "yaml-log-test" deleted

# 4. 确认全部清除
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
| `kubectl apply` 返回 `validation error` | 检查 YAML 缩进和字段拼写 | LogConfig spec 格式错误或字段名不正确 | 对照 TKE LogConfig CRD 文档核对字段，确认 `apiVersion: cls.cloud.tencent.com/v1` |
| LogConfig Status 为 `Error` | `kubectl describe logconfig NAME -n NAMESPACE` 查看 Conditions | 日志集或主题 ID 不存在或格式错误 | 确认 LogsetId/TopicId 为有效 UUID，日志集和主题状态正常 |
| `containerMulti` 类型报 schema 错误 | `kubectl explain logconfig.spec.inputDetail` | 部分旧版 CRD 不支持 `containerMulti` | 升级集群或拆分为多个独立 LogConfig |
| log-agent Pod CrashLoopBackOff | `kubectl logs -n kube-system -l app=log-agent --tail=50` | log-agent 配置加载失败或 OOM | 检查 LogConfig 是否过于复杂，简化 regex 或减少采集源数量 |

### 创建成功但状态异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Status 长时间 Pending | `kubectl describe logconfig NAME -n NAMESPACE` | log-agent 尚未处理或 CLS API 限流 | 等待 1 分钟重试，检查集群是否有多个 LogConfig 竞争 |
| stdout 日志丢失部分时间段的日志 | `tccli cls SearchLog --TopicId TOPIC_ID --Query "*" \| jq '.Results \| length'` | Pod 短时间内创建销毁，log-agent 未捕获全生命周期 | 确保 LogConfig 在 Pod 创建前已部署，增加 `terminationGracePeriodSeconds` |
| 文件日志尾部丢失 | `kubectl exec CONTAINER -- wc -l LOG_PATH/FILE` 对比采集量 | 容器退出时日志文件未 flush，log-agent 停止读取 | 在应用中配置日志 flush 后等待数秒再退出，或使用 stdout 替代文件日志 |
| 正则解析不生效 | 在 CLS 控制台检索查看原始日志内容 | `logRegex` 与实际日志格式不匹配 | 使用 https://regex101.com/ 验证正则表达式，注意转义 |
| 多行日志合并失败 | CLS 检索中异常堆栈被拆分为多条 | `beginningRegex` 未正确匹配新日志行的起始模式 | 调整 `beginningRegex` 精确匹配日志时间戳或前缀模式 |

## 下一步

- [通过控制台配置日志采集](../通过控制台配置日志采集/tccli%20操作.md)：kubectl 配合 tccli 的操作指南
- [开启日志采集](../开启日志采集/tccli%20操作.md)：首次创建 CLS 日志集和主题
- [LogConfig CRD 说明](https://cloud.tencent.com/document/product/457/78446)：TKE LogConfig CRD 官方字段文档
- [CLS 检索语法](https://cloud.tencent.com/document/product/614/47044)：CLS 检索分析语法参考

## 控制台替代

控制台路径：**TKE 控制台 > Serverless 集群 > 集群 ID > 日志采集 > 新建日志采集规则**，通过表单配置日志源、CLS 目标、解析规则。控制台底层自动生成 LogConfig CRD YAML 并 apply。kubectl 直接管理 YAML 更适合 GitOps 场景：YAML 可版本化存入 Git 仓库，通过 CI/CD 自动 apply，支持 review 和回滚。
