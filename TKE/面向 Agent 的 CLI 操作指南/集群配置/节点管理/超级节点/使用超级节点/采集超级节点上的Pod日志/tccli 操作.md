# 采集超级节点上的 Pod 日志

> 对照官方：[采集超级节点上的 Pod 日志](https://cloud.tencent.com/document/product/457/60701) · page_id `60701`

## 概述

将超级节点上 Pod 的容器日志采集到**日志服务（CLS）**，支持标准输出（stdout/stderr）和文件日志。通过在 Pod Annotation 中声明 CLS 日志集、日志主题等配置，超级节点后端（eklet）自动将日志投递到 CLS。

| 采集方式 | 配置位置 | 适用场景 |
|---------|---------|---------|
| Pod Annotation（标准输出） | `.spec.template.metadata.annotations` | 采集 stdout/stderr 日志到 CLS |
| LogConfig CRD | 集群级 CRD 资源 | 精细控制：多路径采集、提取规则、多行日志 |
| TKE 日志采集组件（log-agent） | Addon 安装 + 采集规则 | 全局日志采集策略 |

**控制面验证**：通过 `DescribeClusterVirtualNodePools` 确认超级节点池状态；通过 `InstallAddon` 安装日志组件。

## 前置条件

- [环境准备](../../../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查 kubectl 已安装且可连接集群
kubectl version --client
# expected: 显示 Client Version

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:DescribeClusterVirtualNodePools, tke:DescribeClusterKubeconfig
#    cls:CreateTopic, cls:CreateLogset, cls:DescribeTopics
# 验证：
tccli tke DescribeClusterVirtualNodePools --region <Region> --ClusterId CLUSTER_ID
# expected: exit 0

# 验证 CLS 权限
tccli cls DescribeTopics --region <Region>
# expected: exit 0，返回日志主题列表（可为空）
```

```json
{
  "TotalCount": "<TotalCount>",
  "NodePoolSet": "<NodePoolSet>",
  "NodePoolId": "<NodePoolId>",
  "SubnetIds": "<SubnetIds>",
  "Name": "<Name>",
  "LifeState": "<LifeState>"
}
```

### 资源检查

```bash
# 4. 确认超级节点池已创建且状态正常
tccli tke DescribeClusterVirtualNodePools --region <Region> \
    --ClusterId CLUSTER_ID
# expected: NodePoolSet 非空，LifeState 为 normal

# 5. 获取 kubeconfig
tccli tke DescribeClusterKubeconfig --region <Region> --ClusterId CLUSTER_ID \
    | jq -r '.Kubeconfig' > ~/.kube/config-super
kubectl --kubeconfig ~/.kube/config-super cluster-info
# expected: Kubernetes control plane 可达

# 6. 开通 CLS 并创建日志集（如未开通）
tccli cls DescribeLogsets --region <Region>
# expected: 返回日志集列表（可为空，需已开通 CLS 服务）
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli / kubectl | 幂等 |
|-----------|-----------|:--:|
| 查看超级节点池 | `DescribeClusterVirtualNodePools` | 是 |
| 获取 kubeconfig | `DescribeClusterKubeconfig` | 是 |
| 安装日志组件 | `InstallAddon` | 是 |
| 创建 CLS 日志集 | `cls CreateLogset` | 否 |
| 创建 CLS 日志主题 | `cls CreateTopic` | 否 |
| 声明 Pod 日志 Annotation | `kubectl apply -f` | 是 |
| 查询 CLS 日志 | `cls SearchLog` | 是 |

## 关键字段说明

### Pod 日志采集 Annotation

| Annotation Key | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `eks.tke.cloud.tencent.com/cls-logset-id` | String | 是 | CLS 日志集 ID，`cls DescribeLogsets` 获取 | 日志集不存在 → 日志无法投递 |
| `eks.tke.cloud.tencent.com/cls-topic-id` | String | 是 | CLS 日志主题 ID，`cls DescribeTopics` 获取 | 日志主题不存在 → 日志无法投递 |
| `eks.tke.cloud.tencent.com/log-dir` | String | 否 | 文件日志路径，如 `"/var/log/app"` | 路径不存在 → 对应目录日志不采集 |
| `eks.tke.cloud.tencent.com/enable-log` | String | 否 | `"stdout"` 表示采集标准输出 | 值错误 → 日志采集不启动 |

## 操作步骤

### 步骤 1：创建 CLS 日志集和日志主题

#### 选择依据

- **日志集**：按业务/环境维度创建，如一个集群一个日志集。
- **日志主题**：按应用/命名空间粒度创建，如每个 Deployment 一个日志主题。
- **保存周期**：按合规和成本需求设置，默认 30 天。

#### 创建日志集

`cls-logset.json`：

```json
{
  "LogsetName": "LOG_SET_NAME",
  "Period": 30
}
```

```bash
tccli cls CreateLogset --region <Region> \
    --cli-input-json file://cls-logset.json
# expected: exit 0，返回 LogsetId
```

**预期输出**：

```json
{
    "LogsetId": "logset-example",
    "RequestId": "..."
}
```

#### 创建日志主题

`cls-topic.json`：

```json
{
  "LogsetId": "LOG_SET_ID",
  "TopicName": "TOPIC_NAME"
}
```

```bash
tccli cls CreateTopic --region <Region> \
    --cli-input-json file://cls-topic.json
# expected: exit 0，返回 TopicId
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `LOG_SET_NAME` | CLS 日志集名称 | 全局唯一，长度 1-255 | 自定义 |
| `LOG_SET_ID` | CLS 日志集 ID | `CreateLogset` 返回 | `cls DescribeLogsets --region <Region>` |
| `TOPIC_NAME` | CLS 日志主题名称 | 日志集内唯一 | 自定义 |
| `REGION` | 地域 | 须与集群所在地域一致 | `tccli configure list` |

### 步骤 2：部署带日志 Annotation 的 Pod

`pod-log-annotation.yaml`：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: POD_NAME
  annotations:
    eks.tke.cloud.tencent.com/cpu: "1"
    eks.tke.cloud.tencent.com/mem: "2Gi"
    eks.tke.cloud.tencent.com/cls-logset-id: "LOG_SET_ID"
    eks.tke.cloud.tencent.com/cls-topic-id: "TOPIC_ID"
    eks.tke.cloud.tencent.com/enable-log: "stdout"
spec:
  containers:
  - name: CONTAINER_NAME
    image: nginx:1.25
    resources:
      requests:
        cpu: "1"
        memory: "2Gi"
```

```bash
kubectl --kubeconfig ~/.kube/config-super apply -f pod-log-annotation.yaml
# expected: pod/POD_NAME created
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `POD_NAME` | Pod 名称 | 须遵循 K8s 命名规范 | 自定义 |
| `CONTAINER_NAME` | 容器名称 | 须遵循 K8s 命名规范 | 自定义 |

## 验证

### 控制面（tccli）

```bash
# 验证超级节点池状态
tccli tke DescribeClusterVirtualNodePools --region <Region> \
    --ClusterId CLUSTER_ID
# expected: NodePoolSet 非空，LifeState 为 normal

# 验证 CLS 日志主题存在
tccli cls DescribeTopics --region <Region> \
    --Filters '[{"Key":"topicId","Values":["TOPIC_ID"]}]'
# expected: TotalCount >= 1
```

```json
{
  "TotalCount": "<TotalCount>",
  "NodePoolSet": "<NodePoolSet>",
  "NodePoolId": "<NodePoolId>",
  "SubnetIds": "<SubnetIds>",
  "Name": "<Name>",
  "LifeState": "<LifeState>"
}
```

### 数据面（kubectl）

```bash
# 验证 Pod 状态和 Annotation
kubectl --kubeconfig ~/.kube/config-super describe pod POD_NAME
# expected: Annotations 包含 cls-logset-id、cls-topic-id

# 验证 Pod 产生日志
kubectl --kubeconfig ~/.kube/config-super logs POD_NAME
# expected: 返回容器标准输出日志

# 验证 CLS 中有日志数据（等待 5-10 秒后）
tccli cls SearchLog --region <Region> \
    --TopicId TOPIC_ID \
    --From FROM_TIMESTAMP \
    --To TO_TIMESTAMP \
    --Query ""
# expected: 返回日志记录
```

| 验证维度 | 命令 | 预期 |
|---------|------|------|
| 节点池就绪 | `DescribeClusterVirtualNodePools --ClusterId CLUSTER_ID` | `LifeState: "normal"` |
| Pod Running | `kubectl get pod POD_NAME` | `STATUS: "Running"` |
| Annotation 生效 | `kubectl describe pod POD_NAME` | Annotations 含 CLS 相关键 |
| 容器日志 | `kubectl logs POD_NAME` | 有标准输出日志 |
| CLS 日志投递 | `cls SearchLog --TopicId TOPIC_ID` | 返回日志记录 |

## 清理

> **警告**：删除 CLS 日志主题将**永久删除**该主题下的所有日志数据。删除前请确认数据已备份或不再需要。
> 删除 CLS 日志集将级联删除其下所有日志主题。CLS 按日志写入量和存储量计费，未清理的日志主题将持续产生费用。

### 1. 清理测试 Pod

```bash
kubectl --kubeconfig ~/.kube/config-super delete pod POD_NAME
# expected: pod "POD_NAME" deleted
```

### 2. 删除 CLS 日志主题

```bash
tccli cls DeleteTopic --region <Region> --TopicId TOPIC_ID
# expected: exit 0
```

### 3. 删除 CLS 日志集

```bash
tccli cls DeleteLogset --region <Region> --LogsetId LOG_SET_ID
# expected: exit 0
```

### 4. 验证已删除

```bash
tccli cls DescribeTopics --region <Region> \
    --Filters '[{"Key":"topicId","Values":["TOPIC_ID"]}]'
# expected: TotalCount: 0

tccli cls DescribeLogsets --region <Region> \
    --Filters '[{"Key":"logsetId","Values":["LOG_SET_ID"]}]'
# expected: TotalCount: 0
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `cls CreateTopic` 返回 `InvalidParameter.LogsetNotExist` | `cls DescribeLogsets --region <Region>` 确认日志集存在 | LogsetId 不存在或已被删除 | 用步骤 1 重新创建日志集 |
| `cls SearchLog` 返回 `UnauthorizedOperation` | `tccli configure list` 检查当前凭据 | 子账号缺少 `cls:SearchLog` 权限（此为环境限制，非命令错误） | 联系主账号授予 CLS 只读权限（`QcloudCLSReadOnlyAccess`） |
| Pod 创建失败，Events 显示 `Failed to create pod sandbox` | `kubectl describe pod POD_NAME \| tail -20` | CLS 日志集或主题 ID 不存在 | 确认 `cls-logset-id` 和 `cls-topic-id` 指向已存在的 CLS 资源 |

### 创建成功但日志未投递

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Pod 正常运行但 CLS 无日志 | `kubectl logs POD_NAME` 确认容器有输出 | Annotation 键名拼写错误（`cls` 大小写、连字符） | 核对 Annotation Key：`cls-logset-id` 和 `cls-topic-id` 全小写、中划线分隔 |
| CLS 中只有部分日志 | `kubectl logs POD_NAME --tail 50` 查看近期日志 | 日志投递有延迟（通常 3-10 秒） | 等待 30 秒后重新 `cls SearchLog` |
| 文件日志未采集 | `kubectl exec POD_NAME -- ls 日志目录路径` 确认文件存在 | 未声明 `log-dir` Annotation，或路径不正确 | 检查 Annotation `log-dir` 值，确保为容器内绝对路径 |
| 日志集 ID 正确但日志主题 ID 错误 | `cls DescribeTopics --region <Region>` 列出所有主题 | TopicId 与 LogsetId 不匹配（Topic 不在该 Logset 下） | 用 `cls DescribeTopics --region <Region> --Filters '[{"Key":"logsetId","Values":["LOG_SET_ID"]}]'` 查找日志集下的正确 TopicId |

## 下一步

- [超级节点 Annotation 说明](https://cloud.tencent.com/document/product/457/44173) — 完整 Annotation 列表
- [超级节点可调度 Pod 说明](https://cloud.tencent.com/document/product/457/74015) — 可调度规格范围
- [超级节点常见问题](https://cloud.tencent.com/document/product/457/60411) — 日志采集相关 FAQ
- [CLS 产品文档](https://cloud.tencent.com/document/product/614) — 日志服务完整文档

## 控制台替代

[TKE 控制台 → 运维中心 → 日志采集](https://console.cloud.tencent.com/tke2/cluster)：在日志采集页面创建采集规则，绑定 CLS 日志集和日志主题。
