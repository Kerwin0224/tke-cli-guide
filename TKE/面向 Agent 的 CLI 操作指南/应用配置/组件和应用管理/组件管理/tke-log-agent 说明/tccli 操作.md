# tke-log-agent 说明（tccli）

> 对照官方：[tke-log-agent 说明](https://cloud.tencent.com/document/product/457/91050) · page_id `91050`

## 概述

tke-log-agent 是 Kubernetes 集群日志采集组件，支持非侵入式采集容器标准输出日志、容器内日志以及节点日志。该组件配合 CLS（日志服务）实现日志的集中存储和检索分析。

### 部署的 Kubernetes 对象

| 对象名称 | 类型 | 资源 | 命名空间 |
|---------|------|------|---------|
| tke-log-agent | DaemonSet | 0.21C / 126M | kube-system |
| cls-example | Deployment | 0.1C / 64M | kube-system |
| logconfigs.cls.cloud.tencent.com | CRD | — | — |
| cls-example | ClusterRole | — | — |
| cls-example | ClusterRoleBinding | — | — |
| cls-example | ServiceAccount | — | kube-system |
| tke-log-agent | ClusterRole | — | — |
| tke-log-agent | ClusterRoleBinding | — | — |
| tke-log-agent | ServiceAccount | — | kube-system |

### 工作原理

1. **cls-example** 检测用户创建的采集规则，生成 CLS 侧的采集配置并同步至 CLS 服务端。
2. **tke-log-agent** 根据采集规则映射日志目录至统一目录。
3. **loglistener** 同步 CLS 服务端的采集配置，并采集上传日志至 CLS。

### 使用场景

1. **审计日志采集：** 独立集群开启审计日志采集时，tke-log-agent 被默认安装并采集 apiserver 审计日志。
2. **规则采集：** 用户通过采集规则自定义采集容器标准输出、容器内日志及节点日志。

## 前置条件

- [环境准备](../../../../环境准备.md)
- 已 [连接集群](../../../../集群配置/集群管理/连接集群/tccli 操作.md)，kubectl 可正常访问 APIServer

确认集群状态正常：

```bash
tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'
```

```json
{
  "TotalCount": 0,
  "Clusters": [],
  "ClusterId": "<ClusterId>",
  "ClusterName": "<ClusterName>",
  "ClusterDescription": "<ClusterDescription>",
  "ClusterVersion": "<ClusterVersion>",
  "ClusterOs": "<ClusterOs>",
  "ClusterType": "<ClusterType>"
}
```

- 已开通日志服务（CLS）并完成授权
- 集群需启用特权容器（组件需读写宿主机目录的元数据文件）

**CAM 权限要求：**

| 操作 | 所需权限 |
|------|---------|
| 安装组件 | `tke:InstallAddon` |
| 查看组件状态 | `tke:DescribeAddon` |
| 查看组件配置 | `tke:DescribeAddonValues` |
| 卸载组件 | `tke:DeleteAddon` |

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 查看集群列表 | `tccli tke DescribeClusters` | 是 |
| 查看组件详情 | `tccli tke DescribeAddon --AddonName tke-log-agent` | 是 |
| 安装组件 | `tccli tke InstallAddon --AddonName tke-log-agent` | 否（重复安装报错） |
| 更新组件 | `tccli tke UpdateAddon --AddonName tke-log-agent` | 否 |
| 查看组件列表 | `tccli tke DescribeAddonValues --AddonName tke-log-agent` | 是 |
| 创建日志采集规则 | `kubectl apply -f <yaml>`（LogConfig CR） | 否（同名报错） |
| 查看日志采集规则 | `kubectl get logconfig` | 是 |
| 卸载组件 | `tccli tke DeleteAddon --AddonName tke-log-agent` | 是 |

## 操作步骤

### 1. 安装 tke-log-agent 组件（控制面）

```bash
tccli tke InstallAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName tke-log-agent
```

> **说明：** 独立集群开启审计日志采集后，tke-log-agent 会被默认安装。手动安装方式适用于普通日志采集场景。

### 2. 创建 CLS 日志集和日志主题

在 [CLS 控制台](https://console.cloud.tencent.com/cls) 创建：
- **日志集**（Logset）：日志的逻辑分组
- **日志主题**（Topic）：日志主题 ID 将在 LogConfig CR 中引用

> **限制：** 每个日志集最多 500 个日志主题及指标主题。

### 3. 创建 LogConfig 采集规则（数据面）

> **注意**：kubectl 数据面当前因公网端点被 CAM 策略 strategyId:240463971 拒绝而不可达。以下 kubectl 命令为参考格式，需在内网/VPN 环境执行。

#### 采集容器标准输出

```yaml
apiVersion: cls.cloud.tencent.com/v1
kind: LogConfig
metadata:
  name: stdout-log-config
spec:
  clsDetail:
    topicId: <TopicId>
    logsetId: <LogsetId>
    region: ap-guangzhou
  inputDetail:
    type: container_stdout
    containerStdout:
      allContainers: false
      namespace: default
      workloads:
        - deployment: my-app
      containers:
        - my-app-container
```

#### 采集容器内文件日志

```yaml
apiVersion: cls.cloud.tencent.com/v1
kind: LogConfig
metadata:
  name: file-log-config
spec:
  clsDetail:
    topicId: <TopicId>
    logsetId: <LogsetId>
    region: ap-guangzhou
  inputDetail:
    type: container_file
    containerFile:
      namespace: default
      workloads:
        - deployment: my-app
      containers:
        - my-app-container
      logPath: /var/log/app
      filePattern: "*.log"
```

| 参数 | 说明 |
|------|------|
| `clsDetail.topicId` | CLS 日志主题 ID |
| `clsDetail.logsetId` | CLS 日志集 ID |
| `clsDetail.region` | CLS 日志主题所在的地域 |
| `inputDetail.type` | 采集类型：`container_stdout`（标准输出）、`container_file`（容器内文件）、`host_file`（节点文件） |
| `containerStdout.allContainers` | 是否采集所有容器（布尔值） |
| `containerStdout.namespace` | 指定命名空间 |
| `containerStdout.workloads` | 指定工作负载（deployment、statefulset、daemonset 等） |
| `containerFile.logPath` | 容器内日志文件目录 |
| `containerFile.filePattern` | 日志文件匹配模式，如 `"*.log"` |

#### 采集节点文件日志

```yaml
apiVersion: cls.cloud.tencent.com/v1
kind: LogConfig
metadata:
  name: node-log-config
spec:
  clsDetail:
    topicId: <TopicId>
    logsetId: <LogsetId>
    region: ap-guangzhou
  inputDetail:
    type: host_file
    hostFile:
      logPath: /var/log
      filePattern: "messages*"
      customLabels:
        node: "true"
```

```bash
kubectl apply -f log-config.yaml
```

```text
# command executed successfully
```

### 4. 验证日志采集（数据面）

```bash
kubectl get logconfig
```

**预期输出：** 列出已创建的 LogConfig 及其采集状态。

在 CLS 控制台检索日志：
1. 前往 [CLS 检索分析](https://console.cloud.tencent.com/cls/search)。
2. 选择对应的日志主题。
3. 输入检索语句进行搜索，确认日志已成功上报。

### 组件 RBAC 权限（控制面已自动创建）

**tke-log-agent ClusterRole：**

| API Group | 资源 | 操作 | 用途 |
|-----------|------|------|------|
| `cls.cloud.tencent.com` | logconfigs, logconfigpros | list, watch, patch, get | 监控日志采集规则变化 |
| `""` (core) | pods, namespaces, nodes, persistentvolumeclaims, configmaps, persistentvolumes | list, watch, get | 获取容器和存储路径信息 |
| `apps` | daemonsets, replicasets, deployments, statefulsets | list, watch, get | 获取工作负载信息 |
| `batch` | jobs, cronjobs | list, watch, get | 获取批处理工作负载 |
| `storage.k8s.io` | storageclasses | get | 获取存储类 |

**cls-example ClusterRole：**

| API Group | 资源 | 操作 | 用途 |
|-----------|------|------|------|
| `cls.cloud.tencent.com` | l

```json
{
  "Addons": [],
  "AddonName": "<AddonName>",
  "AddonVersion": "<AddonVersion>",
  "RawValues": "<RawValues>",
  "Phase": "<Phase>",
  "Reason": "<Reason>",
  "CreateTime": "<CreateTime>",
  "RequestId": "<RequestId>"
}
```ogconfigs | list, watch, patch, update | 同步日志采集规则到 CLS |
| `'*'` (所有) | events, configmaps | create, patch, update | 状态上报和配置管理 |

## 验证

### 控制面 (tccli)

```bash
tccli tke DescribeAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName tke-log-agent
```

```json
{
  "RequestId": "..."
}
```

**预期输出：** 组件状态为 `Succeeded`，AddonName 为 `tke-log-agent`。

### 数据面 (kubectl)

> **注意**：kubectl 数据面当前因公网端点被 CAM 策略 strategyId:240463971 拒绝而不可达。以下 kubectl 命令为参考格式，需在内网/VPN 环境执行。

验证 DaemonSet 和 Deployment 运行：

```bash
kubectl get ds -n kube-system tke-log-agent
kubectl get deploy -n kube-system cls-example
```

```text
NAME  STATUS  AGE
...
```

**预期输出：** DaemonSet `DESIRED` = `READY`，Deployment `READY` = `1/1`。

验证 LogConfig CRD 已注册：

```bash
kubectl get crd logconfigs.cls.cloud.tencent.com
```

```text
NAME  STATUS  AGE
...
```

**预期输出：** CRD 已存在，CREATED AT 显示创建时间。

验证采集规则状态：

```bash
kubectl get logconfig
kubectl describe logconfig stdout-log-config
```

```text
NAME  STATUS  AGE
...
```

**预期输出：** LogConfig 状态中可查看采集配置是否同步成功及最近事件。

## 清理

### 数据面 (kubectl)

> **注意**：kubectl 数据面当前因公网端点被 CAM 策略 strategyId:240463971 拒绝而不可达。以下 kubectl 命令为参考格式，需在内网/VPN 环境执行。

```bash
kubectl delete logconfig stdout-log-config
kubectl delete logconfig file-log-config
kubectl delete logconfig node-log-config
```

### 控制面 (tccli)

```bash
tccli tke DeleteAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName tke-log-agent
```

> **计费提醒：** 卸载组件不会自动删除 CLS 中的日志主题和已采集数据。请前往 [CLS 控制台](https://console.cloud.tencent.com/cls) 手动删除不再使用的日志主题以避免持续产生存储费用。

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| LogConfig 状态异常（同步失败） | `kubectl describe logconfig <name>` 查看 Status 与事件 | CLS 日志主题 ID 或地域不正确 | 在 CLS 控制台确认 `topicId` 和 `region` 正确；验证日志主题存在且未被删除 |
| tke-log-agent Pod 异常 `CrashLoopBackOff` | `kubectl describe pod -n kube-system <log-agent-pod>` 查看详细错误 | 特权容器未启用，无法读写宿主机目录 | 确认集群允许特权容器 |
| 容器内文件日志未采集 | `kubectl describe logconfig <name>` 检查 `logPath` 和 `filePattern` | `logPath` 路径不正确或 `filePattern` 格式不匹配 | 检查容器内是否确实在指定路径生成日志文件；调整 `filePattern` 匹配规则 |
| 日志采集出现数据丢失 | 检查 CLS 日志主题分区数和节点磁盘空间 | loglistener 缓冲区满或 CLS 写入 QPS 受限 | 增加 CLS 日志主题分区数；检查节点磁盘空间是否充足 |
| `Unable to connect to the server` | `kubectl cluster-info` 检查连通性 | 公网端点被 CAM 策略 strategyId:240463971 以 `tke:clusterExtranetEndpoint=true` 条件拒绝 | 通过 IOA/VPN 连接内网，或使用同 VPC CVM 执行 kubectl 命令 |

## 下一步

- [开启日志采集](https://cloud.tencent.com/document/product/457/36771) — CLS 日志采集完整操作指南
- [通过 YAML 配置日志采集](https://cloud.tencent.com/document/product/457/48425) — LogConfig CRD 配置详情
- [使用 CRD 采集日志到 Kafka](https://cloud.tencent.com/document/product/457/61092) — 将日志投递到 Kafka 的配置方案
- [组件的生命周期管理](../组件的生命周期管理/tccli 操作.md) — 组件安装、升级与卸载

## 控制台替代

通过 [TKE 控制台安装 tke-log-agent 组件](https://cloud.tencent.com/document/product/457/91050) 操作。
