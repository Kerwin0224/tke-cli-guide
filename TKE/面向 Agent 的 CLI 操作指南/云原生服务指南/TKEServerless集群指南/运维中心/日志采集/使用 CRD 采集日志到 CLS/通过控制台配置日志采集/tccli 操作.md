# 通过控制台配置日志采集

> 对照官方：[通过控制台配置日志采集](https://cloud.tencent.com/document/product/457/56320) · page_id `56320`

## 概述

本文档覆盖 TKE Serverless（EKS）集群中通过 **LogConfig CRD** 配置日志采集规则的 CLI 操作方法。虽然官方页面标题为"通过控制台配置"，但控制台底层实际是创建 LogConfig CRD 资源。本文使用 `kubectl apply` 声明式管理 LogConfig，配合 `tccli cls` 查询 CLS 侧资源状态，实现端到端的日志采集规则配置。

适用场景：已通过 [开启日志采集](../开启日志采集/tccli%20操作.md) 完成 CLS 日志集/主题创建，需要细化配置采集规则的场景（如按 namespace、workload、容器维度筛选日志）。

> **kubectl 数据面不可达**：CAM 拒绝公网端点 (strategyId:240463971)，内网需 VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

## 前置条件

- [环境准备](../../../../../../环境准备.md)
- 至少一个运行中的 TKE Serverless 集群
- 已完成 CLS 日志集和日志主题创建（参见 [开启日志采集](../开启日志采集/tccli%20操作.md)）
- kubectl 已配置并可访问集群 APIServer（需 VPN/IOA）

### 环境检查

```bash
tccli --version
# expected: tccli version 3.0.x

tccli configure list
# expected: secretId, secretKey, region 均已配置

# 验证 CLS 日志集存在
tccli cls DescribeLogsets --region <Region> | jq '.Logsets[] | {LogsetId, LogsetName}'
# expected: 至少返回1个日志集

# 验证 CLS 日志主题存在
tccli cls DescribeTopics --region <Region> --Filters '[{"Key":"logsetId","Values":["LOGSET_ID"]}]'
# expected: TotalCount >= 1

# 验证 kubectl 集群可达（需 VPN/IOA）
kubectl cluster-info
# expected: Kubernetes control plane is running at ...

# 验证 LogConfig CRD 可用
kubectl api-resources | grep logconfigs
# expected: logconfigs  cls.cloud.tencent.com/v1  true  LogConfig
```

## 控制台与 CLI 参数映射

| 控制台操作 | kubectl / tccli 命令 | 幂等 |
|-----------|---------------------|:--:|
| 创建采集配置 | `kubectl apply -f logconfig.yaml` | 是 |
| 查看采集配置列表 | `kubectl get logconfigs -A` | 是 |
| 查看采集配置详情 | `kubectl describe logconfig <name> -n <ns>` | 是 |
| 修改采集配置 | `kubectl apply -f logconfig.yaml` | 是 |
| 删除采集配置 | `kubectl delete logconfig <name> -n <ns>` | 否 |
| 查询 CLS 主题状态 | `tccli cls DescribeTopics --TopicId TOPIC_ID` | 是 |
| 查询 CLS 索引状态 | `tccli cls DescribeIndex --TopicId TOPIC_ID` | 是 |
| 检索日志验证 | `tccli cls SearchLog --TopicId TOPIC_ID --Query "..."` | 是 |

## 操作步骤

### 占位符说明

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|----------|
| `NAMESPACE` | Kubernetes 命名空间 | 不超过 63 字符 | 自定义或使用 default |
| `LOGSET_ID` | CLS 日志集 ID | UUID 格式 | `CreateLogset` 返回值或 `DescribeLogsets` |
| `TOPIC_ID` | CLS 日志主题 ID | UUID 格式 | `CreateTopic` 返回值或 `DescribeTopics` |
| `REGION` | 地域 | 如 ap-guangzhou | 固定值 |
| `DEPLOYMENT_NAME` | 目标 Deployment 名称 | Kubernetes 资源名规范 | `kubectl get deployments -n NAMESPACE` |

### 步骤1: 配置 stdout 日志采集规则

采集指定 namespace 下所有容器的 stdout/stderr 到 CLS 日志主题。

```bash
cat <<'EOF' > logconfig-stdout.yaml
apiVersion: cls.cloud.tencent.com/v1
kind: LogConfig
metadata:
  name: stdout-collection
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
EOF

# 替换占位符
sed -i '' "s/NAMESPACE/NAMESPACE/g" logconfig-stdout.yaml
sed -i '' "s/LOGSET_ID/LOGSET_ID/g" logconfig-stdout.yaml
sed -i '' "s/TOPIC_ID/TOPIC_ID/g" logconfig-stdout.yaml

kubectl apply -f logconfig-stdout.yaml
# expected: logconfig.cls.cloud.tencent.com/stdout-collection created
```

### 步骤2: 配置容器文件日志采集规则

采集指定 Deployment 下特定容器的文件日志。

```bash
cat <<'EOF' > logconfig-file.yaml
apiVersion: cls.cloud.tencent.com/v1
kind: LogConfig
metadata:
  name: file-collection
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
        name: "DEPLOYMENT_NAME"
        kind: "Deployment"
      container: "CONTAINER_NAME"
      logPath: "/var/log/app/"
      filePattern: "*.log"
      customLabels:
        source: "file"
        app: "my-app"
EOF
kubectl apply -f logconfig-file.yaml
# expected: logconfig.cls.cloud.tencent.com/file-collection created
```

### 步骤3: 配置多路径文件日志采集

同一 LogConfig 中配置多个文件日志路径源。

```bash
cat <<'EOF' > logconfig-multi-path.yaml
apiVersion: cls.cloud.tencent.com/v1
kind: LogConfig
metadata:
  name: multi-path-collection
  namespace: NAMESPACE
spec:
  clsDetail:
    logsetId: "LOGSET_ID"
    topicId: "TOPIC_ID"
    logType: container_file
    extractRule:
      # 使用正则解析日志字段
      beginningRegex: '^\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2}'
      keys:
        - "timestamp"
        - "level"
        - "message"
      logRegex: '^(\S+ \S+) (\S+) (.*)'
      timeKey: "timestamp"
      timeFormat: "%Y-%m-%d %H:%M:%S"
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
      # 排除特定文件
      excludeFilePattern: "*.gz"
      # 多行合并（如 Java 堆栈）
      multiline:
        pattern: '^\d{4}-\d{2}-\d{2}'
        negate: true
        match: after
EOF
kubectl apply -f logconfig-multi-path.yaml
# expected: logconfig.cls.cloud.tencent.com/multi-path-collection created
```

### 步骤4: 查询和管理 LogConfig

```bash
# 查看所有 namespace 的 LogConfig
kubectl get logconfigs -A
# expected:
# NAMESPACE    NAME                    STATUS    AGE
# default      stdout-collection       Running   5m
# default      file-collection         Running   3m

# 查看特定 LogConfig 详情
kubectl describe logconfig stdout-collection -n NAMESPACE
# expected: Status.Conditions.Type=Running, Status=True

# 查看 LogConfig YAML（用于备份或修改）
kubectl get logconfig stdout-collection -n NAMESPACE -o yaml
# expected: 输出完整 CRD 配置
```

```text
NAME  STATUS  AGE
...
```

## 验证

### 数据面（需 VPN/IOA）

```bash
# 1. 确认 LogConfig Status 为 Running
kubectl get logconfigs -n NAMESPACE --no-headers | awk '{print $1, $3}'
# expected: stdout-collection Running

# 2. 确认 log-agent 正常运行
kubectl get pods -n kube-system -l app=log-agent
# expected: Running 1/1

# 3. 创建测试 Pod 产生日志
kubectl run verify-log --image=busybox:1.28 --restart=Never -n NAMESPACE -- \
    /bin/sh -c 'for i in $(seq 1 5); do echo "LOGCFG_TEST_LINE_$i"; sleep 1; done; sleep 3600'
# expected: pod/verify-log created
```

```text
NAME  STATUS  AGE
...
```

### 控制面（tccli）

```bash
# 4. 等待 2-3 分钟后，通过 CLS 检索验证日志投递
tccli cls SearchLog \
    --region <Region> \
    --TopicId TOPIC_ID \
    --From $(python3 -c "import time; print(int((time.time()-600)*1000))") \
    --To $(python3 -c "import time; print(int(time.time()*1000))") \
    --Query "LOGCFG_TEST"
# expected: Results 数组含 LOGCFG_TEST_LINE_1 ~ LOGCFG_TEST_LINE_5

# 5. 验证 LogConfig 未产生错误事件
kubectl get events -n NAMESPACE --field-selector involvedObject.name=stdout-collection
# expected: No events (或仅 Normal 类型事件)
```

```text
NAME  STATUS  AGE
...
```

## 清理

```bash
# 1. 删除 LogConfig CRD（停止日志采集规则）
kubectl delete logconfig stdout-collection -n NAMESPACE
kubectl delete logconfig file-collection -n NAMESPACE
kubectl delete logconfig multi-path-collection -n NAMESPACE
# expected: logconfig.cls.cloud.tencent.com "stdout-collection" deleted

# 2. 删除测试 Pod
kubectl delete pod verify-log -n NAMESPACE --ignore-not-found
# expected: pod "verify-log" deleted

# 3. 确认 LogConfig 已清除
kubectl get logconfigs -n NAMESPACE
# expected: No resources found in NAMESPACE namespace.
```

```text
NAME  STATUS  AGE
...
```

> **注意**：删除 LogConfig 仅停止采集规则，不会删除 CLS 日志主题中的已有日志。如需删除 CLS 资源（日志主题/日志集），参见 [开启日志采集](../开启日志采集/tccli%20操作.md) 清理章节。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl apply` 返回 `no matches for kind "LogConfig"` | `kubectl api-resources \| grep logconfig` | LogConfig CRD 未在集群中注册 | 确认集群为 TKE Serverless 类型，标准 TKE 集群需先安装 CRD |
| LogConfig Status 为 `Error` | `kubectl describe logconfig NAME -n NAMESPACE` | LogsetId 或 TopicId 无效或 CLS 权限不足 | 检查 LogsetId/TopicId 格式正确，确认集群服务角色有 CLS 写入权限 |
| LogConfig 创建后长时间 `Pending` | 查看 log-agent 日志 `kubectl logs -n kube-system -l app=log-agent` | log-agent 未运行或无法连接 CLS API | 检查集群网络连通性，确认 CLS 地域与集群一致 |
| `tccli cls SearchLog` 返回空结果 | 确认时间范围和 Query 语法 | 日志投递延迟（通常 1-3 分钟）或 Query 不匹配 | 扩大时间范围至 `--From $(python3 -c "import time; print(int((time.time()-600))*1000)")`，或使用 `--Query "*"` |

### 创建成功但状态异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 部分容器日志未采集 | `kubectl get pods -n NAMESPACE -l app=deployment-name` 核对标签选择器 | LogConfig `containerStdout.allContainers` 为 false 且未匹配 | 设置 `allContainers: true` 或通过 `workload` 字段精确匹配 |
| 文件日志采集不到 | `kubectl exec CONTAINER -- ls LOG_PATH/` 确认文件存在 | logPath 路径错误或容器未写入日志 | 确认 `logPath` 为容器内绝对路径，`filePattern` 匹配实际文件名 |
| 多行日志被拆分 | 检索 CLS 查看日志条目是否正确合并 | LogConfig 未配置 `multiline` 合并规则 | 添加 `multiline.pattern` 匹配日志行起始正则，如 Java 异常堆栈 |
| LogConfig 修改后未生效 | `kubectl get logconfig stdout-collection -n NAMESPACE -o yaml \| grep resourceVersion` | LogConfig 更新后 log-agent 自动感知变更，但延迟通常 < 30s | 等待 1 分钟后重试验证 |

## 下一步

- [通过 YAML 配置日志采集](../通过%20YAML%20配置日志采集/tccli%20操作.md)：LogConfig CRD 完整 YAML 参考和高级配置
- [开启日志采集](../开启日志采集/tccli%20操作.md)：首次创建 CLS 日志集和主题的初始化操作
- [CLS 检索语法](https://cloud.tencent.com/document/product/614/47044)：CLS 检索分析语法参考
- [LogConfig CRD 说明](https://cloud.tencent.com/document/product/457/78446)：TKE LogConfig CRD 官方字段文档

## 控制台替代

控制台路径：**TKE 控制台 > Serverless 集群 > 集群 ID > 日志采集 > 新建日志采集规则**。在控制台中通过表单选择日志源（stdout/文件）、目标 CLS 主题、日志解析规则。控制台底层自动创建对应的 LogConfig CRD，可通过 `kubectl get logconfigs` 查看。tccli 方式通过 kubectl 声明式管理 CRD，支持版本控制和 GitOps 工作流。
