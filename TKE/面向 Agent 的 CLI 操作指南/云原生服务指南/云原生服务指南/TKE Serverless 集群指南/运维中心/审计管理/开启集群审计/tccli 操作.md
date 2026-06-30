# 开启集群审计（tccli）

> 对照官方：[审计日志](https://cloud.tencent.com/document/product/457/58242) · page_id `58242`

## 概述

为 TKE Serverless 集群开启 Kubernetes 审计日志（Audit Log）功能。审计日志基于 `audit.k8s.io/v1` 审计策略，记录对 kube-apiserver 的所有请求，用于安全合规审计、操作追溯和故障排查。

开启集群审计后，审计事件将以 JSON 格式投递到 CLS（日志服务）指定的日志主题。每条审计事件包含请求用户、来源 IP、操作资源、HTTP 动词、响应状态码和审计阶段等信息。

> **注意**：开启或修改审计配置会触发 kube-apiserver 滚动重启，期间可能短暂影响集群管理操作。避免在业务高峰期频繁开启/关闭审计功能。

## 前置条件

- TKE Serverless 集群已创建且状态为 `Running`，参见 [创建集群](../../../TKE%20Serverless%20集群管理/创建集群/tccli%20操作.md)。
- [环境准备](../../../../环境准备.md)：`tccli` 已安装并配置 `region`（本文以 `ap-guangzhou` 为例）。
- 已开通 [CLS（日志服务）](https://console.cloud.tencent.com/cls)。
- CLS 日志集和日志主题已创建（参见 [采集容器内日志](../../日志采集/采集容器内日志/tccli%20操作.md) 中的 CLS 准备步骤）。

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|------|
| 查询集群信息 | `tke DescribeEKSClusters` | 是 |
| 查看 CLS 日志集 | `cls DescribeLogsets` | 是 |
| 创建 CLS 日志集 | `cls CreateLogset` | 否（同名可能冲突） |
| 查看 CLS 日志主题 | `cls DescribeTopics` | 是 |
| 创建 CLS 日志主题 | `cls CreateTopic` | 否（同名可能冲突） |
| 开启审计（控制台操作） | 控制台 **运维中心 → 审计管理 → 开启集群审计** | 否（重复开启幂等） |
| 修改审计配置 | 控制台操作 | 否 |
| 关闭审计 | 控制台操作 | 否 |

## 操作步骤

### 1. 理解 Kubernetes 审计策略

Kubernetes 审计 (`audit.k8s.io/v1`) 定义了审计事件的详细程度和记录阶段。

#### 1.1 审计级别（Audit Level）

| 级别 | 记录内容 | 适用场景 |
|------|---------|---------|
| `None` | 不记录任何请求 | 禁止审计匹配到的资源类型 |
| `Metadata` | 记录请求元数据（用户、时间戳、资源、动词等），**不记录** request/response body | 日常合规审计，降低日志量 |
| `Request` | 记录元数据 + request body，不记录 response body | 需要分析请求内容的场景 |
| `RequestResponse` | 记录元数据 + request body + response body | 完整安全审计、故障排查 |

#### 1.2 审计阶段（Audit Stage）

| 阶段 | 触发时机 | 说明 |
|------|---------|------|
| `RequestReceived` | API Server 接收到请求 | 最早阶段，所有请求都会触发 |
| `ResponseStarted` | 响应头已发送，body 未发送完毕 | 长连接 watch 等流式响应在此阶段记录 |
| `ResponseComplete` | 响应完全发送完毕 | 记录完整的请求-响应周期 |
| `Panic` | API Server panic 时 | 紧急事件，用于排查 API Server 异常 |

#### 1.3 审计事件 JSON 结构示例

```json
{
    "kind": "Event",
    "apiVersion": "audit.k8s.io/v1",
    "level": "RequestResponse",
    "auditID": "abc12345-def6-7890-abcd-ef1234567890",
    "stage": "ResponseComplete",
    "requestURI": "/api/v1/namespaces/default/pods",
    "verb": "create",
    "user": {
        "username": "kubernetes-admin",
        "uid": "abc-def-ghi",
        "groups": ["system:masters", "system:authenticated"]
    },
    "sourceIPs": ["10.0.1.100"],
    "userAgent": "kubectl/v1.28.0 (darwin/amd64) kubernetes/abc123",
    "objectRef": {
        "resource": "pods",
        "namespace": "default",
        "name": "nginx-deployment-7b9f8c6d5-x2k9m",
        "apiVersion": "v1"
    },
    "responseStatus": {
        "metadata": {},
        "code": 201,
        "status": "Created"
    },
    "requestObject": {
        "kind": "Pod",
        "apiVersion": "v1",
        "metadata": {
            "name": "nginx-deployment-7b9f8c6d5-x2k9m",
            "namespace": "default"
        }
    },
    "responseObject": {
        "kind": "Pod",
        "apiVersion": "v1",
        "metadata": {
            "name": "nginx-deployment-7b9f8c6d5-x2k9m",
            "namespace": "default",
            "uid": "uid-example-001"
        }
    },
    "requestReceivedTimestamp": "2024-01-15T10:30:00.000000Z",
    "stageTimestamp": "2024-01-15T10:30:00.050000Z",
    "annotations": {
        "authorization.k8s.io/decision": "allow",
        "authorization.k8s.io/reason": "RBAC: allowed by ClusterRoleBinding \"kubernetes-admin\" of ClusterRole \"cluster-admin\" to User \"kubernetes-admin\""
    }
}
```

### 2. 准备 CLS 审计日志主题

建议为审计日志创建独立的日志集和日志主题，推荐使用 JSON 解析模式（审计事件为 JSON 格式）。

#### 2.1 创建审计专用日志集

```bash
tccli cls CreateLogset \
    --LogsetName "tke-audit-logs" \
    --region ap-guangzhou \
    --output json
```

示例输出：

```json
{
    "LogsetId": "logset-example-audit",
    "RequestId": "abc123-def456-ghi789"
}
```

#### 2.2 创建审计日志主题

```bash
tccli cls CreateTopic \
    --LogsetId "logset-example-audit" \
    --TopicName "cluster-audit-events" \
    --Period 180 \
    --region ap-guangzhou \
    --output json
```

示例输出：

```json
{
    "TopicId": "topic-example-audit",
    "RequestId": "abc123-def456-ghi789"
}
```

> **建议**：审计日志保存时间（`Period`）推荐设置为 180 天以满足安全合规要求（默认 30 天可修改）。

### 3. 确认集群状态

确认目标集群状态为 `Running`，以便开启审计功能：

```bash
tccli tke DescribeEKSClusters \
    --ClusterIds '["cls-example"]' \
    --region ap-guangzhou \
    --output json
```

示例输出：

```json
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "cls-example",
            "ClusterName": "my-serverless-cluster",
            "Status": "Running",
            "VpcId": "vpc-example",
            "K8SVersion": "1.28.3",
            "CreatedTime": "2024-01-15T10:30:00Z"
        }
    ]
}
```

### 4. 开启集群审计

集群审计的开启操作在控制台中完成。TKE Serverless 集群的审计功能由托管 kube-apiserver 实现，不暴露独立的 `EnableClusterAudit` API。

#### 4.1 控制台操作步骤

1. 进入 [TKE 控制台](https://console.cloud.tencent.com/tke2/cluster) 集群列表。
2. 点击目标 Serverless 集群 ID（`cls-example`）。
3. 左侧导航栏选择 **运维中心 → 审计管理 → 开启集群审计**。
4. 选择审计日志的 CLS 地域（需与集群地域一致）、日志集和日志主题。
5. 选择审计级别：**Metadata**（推荐，降低日志量且满足合规）或 **RequestResponse**（完整审计）。
6. 点击 **开启**。

#### 4.2 操作说明

- 开启后 kube-apiserver 会自动滚动重启，审计配置生效（通常 1-3 分钟）。
- 审计级别选择 **Metadata** 可显著降低日志量（仅记录元数据，不记录 request/response body），适合大多数合规审计场景。
- 若需要排查 API 请求的具体内容（如排查非法请求），可临时切换至 **RequestResponse** 级别，排查完成后切回 **Metadata**。
- **频繁切换审计级别会反复触发 kube-apiserver 重启**，建议每次修改间隔至少 10 分钟。

> **CLI 局限性说明：** 当前 `tccli tke` 未暴露 `ModifyClusterAudit` 或等效接口，集群审计的开启/关闭/级别切换仅能通过控制台操作。审计事件查询可通过 `tccli cls SearchLog` 进行，见 [查看审计](../查看审计/tccli%20操作.md)（page_id `58243`）。

### 5. 审计策略说明

TKE Serverless 默认审计策略记录以下 API 组和资源：

| 资源类别 | 资源 | 审计级别 |
|---------|------|---------|
| 核心资源 | `pods`, `services`, `configmaps`, `secrets`, `namespaces` | Metadata+ |
| 工作负载 | `deployments`, `statefulsets`, `daemonsets`, `jobs`, `cronjobs` | Metadata+ |
| 网络 | `ingresses`, `networkpolicies` | Metadata+ |
| RBAC | `roles`, `rolebindings`, `clusterroles`, `clusterrolebindings` | Metadata+ |
| 存储 | `persistentvolumes`, `persistentvolumeclaims`, `storageclasses` | Metadata+ |
| 节点 | `nodes` | Metadata+ |
| 系统事件 | `/healthz`, `/metrics`, `/version` | None（不记录） |

## 验证

### 验证审计日志到达 CLS

```bash
# 检索审计事件（最近 10 分钟）
tccli cls SearchLog \
    --TopicId "topic-example-audit" \
    --From $(date -v-10M +%s)000 \
    --To $(date +%s)000 \
    --Query 'verb:create' \
    --Limit 3 \
    --region ap-guangzhou \
    --output json
```

示例输出（审计事件已上报时）：

```json
{
    "Results": [
        {
            "Time": 1705311000000,
            "TopicId": "topic-example-audit",
            "LogJson": "{\"auditID\":\"abc12345-def6-7890-abcd-ef1234567890\",\"verb\":\"create\",\"user\":{\"username\":\"kubernetes-admin\"},\"objectRef\":{\"resource\":\"pods\",\"namespace\":\"default\",\"name\":\"nginx-deployment-7b9f8c6d5-x2k9m\"},\"responseStatus\":{\"code\":201},\"stage\":\"ResponseComplete\",\"level\":\"Metadata\"}"
        }
    ],
    "TotalCount": 1
}
```

### 验证审计配置生效

在控制台 **运维中心 → 审计管理** 页面确认：
- 审计开关状态为 **已开启**
- 审计级别与所选一致
- 目标日志地域、日志集和日志主题与配置一致

## 清理

关闭集群审计（控制台操作）：

1. 进入 [TKE 控制台](https://console.cloud.tencent.com/tke2/cluster) 集群列表。
2. 点击目标 Serverless 集群 ID。
3. 左侧导航栏选择 **运维中心 → 审计管理**。
4. 点击 **关闭审计**。

> **注意**：关闭审计会再次触发 kube-apiserver 滚动重启。已投递到 CLS 的审计日志不受影响，继续保留至日志主题设定的保存期。

如需清理 CLS 审计日志主题：

```bash
tccli cls DeleteTopic \
    --TopicId "topic-example-audit" \
    --region ap-guangzhou \
    --output json

tccli cls DeleteLogset \
    --LogsetId "logset-example-audit" \
    --region ap-guangzhou \
    --output json
```

## 排障

| 现象 | 原因 | 处理 |
|------|------|------|
| 控制台审计开关置灰不可点击 | 集群状态非 `Running`，或 CLS 未开通 | 确认集群 `Running`，CLS 已开通 |
| 审计开启后 CLS 无审计事件 | kube-apiserver 重启中或审计事件延迟（通常 2-5 分钟） | 等待 5 分钟后通过 `cls SearchLog` 查询 |
| 审计日志中含有大量 `/healthz` 等系统请求 | 默认审计策略可能包含系统请求 | 系统事件（`/healthz`, `/metrics`）默认不记录；如出现，联系售后调整审计策略 |
| CLS 检索 `*` 查询无结果 | 审计事件未上报或日志主题不正确 | 确认 **审计管理** 页面绑定的日志主题与查询的 `TopicId` 一致 |
| 频繁开启关闭审计 | kube-apiserver 反复重启，集群不稳定 | 每次修改审计配置至少间隔 10 分钟 |
| 审计日志量过大导致 CLS 成本高 | 选择了 `RequestResponse` 级别 | 切换为 `Metadata` 级别可降低日志量约 80% |

## 下一步

- [查看审计](../查看审计/tccli%20操作.md)（page_id `58243`） — 使用审计仪表盘和 CLS 检索查询审计日志
- [集群事件](../../事件管理/集群事件/tccli%20操作.md) — 查看集群事件（与审计日志互补）
- [监控和告警](../../监控和告警/查看监控/tccli%20操作.md) — 为审计事件配置告警

## 控制台替代

[容器服务控制台 → 集群 → 运维中心 → 审计管理 → 开启集群审计](https://console.cloud.tencent.com/tke2/cluster) — 选择审计日志地域、日志集和日志主题，选择审计级别后开启。
