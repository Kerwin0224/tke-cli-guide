# 集群审计（tccli）

> 对照官方：[集群审计](https://cloud.tencent.com/document/product/457/72149) · page_id `72149`

## 概述

为注册集群配置 **Kubernetes API Server 审计日志**，记录集群中所有对 API Server 的请求操作。审计策略定义在外部集群的 Master 节点上（`audit-policy.yaml` + `kube-apiserver.yaml` 参数），审计日志通过 TKE 控制台启用后**自动投递至腾讯云日志服务（CLS）**，供安全审计、合规检查与排障使用。

这是一个**混合操作场景**：Master 节点配置（kubectl/SSH）+ CLS 目标配置（tccli/控制台）。

## 前置条件

- 注册集群已创建，Master 节点可通过 SSH 登录并具备 root/sudo 权限。
- CLS 日志服务已开通，已创建日志集与日志主题（参见 [日志采集](../日志采集/tccli%20操作.md) 步骤 1）。
- 了解 Kubernetes 审计策略语法：[audit.k8s.io/v1beta1 Policy](https://kubernetes.io/docs/tasks/debug/debug-cluster/audit/#audit-policy)。
- 审计日志量较大（生产集群日均 GB 级），请确保 CLS 存储容量与成本预算。

## 控制台与 CLI 参数映射

| 控制台 | kubectl / tccli | 幂等 |
|--------|-----------------|------|
| 配置审计策略 | SSH 到 Master 节点编辑 `/etc/kubernetes/audit-policy.yaml` | 是 |
| 配置 API Server 审计参数 | 编辑 `/etc/kubernetes/manifests/kube-apiserver.yaml` | 否（触发 API Server 重启） |
| 开启集群审计（控制台） | 控制台操作：选择投递方式 + CLS 日志主题 | 否 |
| 查看审计日志 | `tccli cls SearchLog` + 审计 Topic | 是 |
| 关闭审计 | 控制台关闭 + Master 节点清理参数（可选） | 否 |

## 操作步骤

### 1. 在 Master 节点配置审计策略

SSH 登录到集群任意 Master 节点，创建审计策略文件：

```bash
# SSH 到 Master 节点
ssh root@<master-node-ip>

# 创建审计策略文件
sudo tee /etc/kubernetes/audit-policy.yaml << 'EOF'
apiVersion: audit.k8s.io/v1beta1
kind: Policy
omitStages:
  - "RequestReceived"
rules:
  # 系统组件请求：不记录（避免日志爆炸）
  - level: None
    users:
      - "system:kube-proxy"
      - "system:kubelet"
      - "system:kube-controller-manager"
      - "system:apiserver"
      - "kubelet"
  # Secret / ConfigMap 的 get/list/watch：只记录元数据
  - level: Metadata
    resources:
      - group: ""
        resources: ["secrets", "configmaps"]
    verbs: ["get", "list", "watch"]
  # 读操作（非 Secret/ConfigMap）：记录请求信息
  - level: Request
    verbs: ["get", "list", "watch"]
    omitStages:
      - "RequestReceived"
  # 写操作：记录完整请求与响应
  - level: RequestResponse
    verbs: ["create", "update", "patch", "delete"]
    omitStages:
      - "RequestReceived"
  # 默认：记录元数据（兜底）
  - level: Metadata
EOF
```

| 审计级别 | 记录内容 | 适用场景 |
|---------|---------|---------|
| `None` | 不记录 | 系统组件 |
| `Metadata` | 请求元数据（用户、时间、资源、动词），不含 body | Secret/ConfigMap 读取 |
| `Request` | 元数据 + 请求体 | 一般读操作 |
| `RequestResponse` | 元数据 + 请求体 + 响应体 | 写操作（创建/更新/删除） |

### 2. 修改 API Server 配置（启用审计日志输出）

编辑 kube-apiserver 静态 Pod 的 manifest，添加审计相关参数：

```bash
# 编辑 API Server 配置
sudo vi /etc/kubernetes/manifests/kube-apiserver.yaml
```

在 `spec.containers[0].command` 中添加以下参数：

```yaml
spec:
  containers:
  - command:
    - kube-apiserver
    # ... 现有参数 ...
    - --audit-log-maxbackup=10       # 最多保留 10 个归档文件
    - --audit-log-maxsize=100        # 单个日志文件最大 100 MB
    - --audit-log-path=/var/log/kubernetes/kubernetes.audit  # 审计日志路径
    - --audit-log-maxage=30          # 日志文件保留 30 天
    - --audit-policy-file=/etc/kubernetes/audit-policy.yaml   # 审计策略文件路径
```

保存后 **API Server Pod 会自动重启**（kubelet 检测 manifest 变更后重建静态 Pod）。

```bash
# 验证 API Server Pod 已重启
sudo crictl ps | grep kube-apiserver
```

| 参数 | 值 | 说明 |
|------|---|------|
| `--audit-log-maxbackup` | `10` | 最大备份文件数 |
| `--audit-log-maxsize` | `100` | 单个文件限制（MB） |
| `--audit-log-path` | `/var/log/kubernetes/kubernetes.audit` | 审计日志输出路径 |
| `--audit-log-maxage` | `30` | 日志保留天数 |
| `--audit-policy-file` | `/etc/kubernetes/audit-policy.yaml` | 审计策略文件 |

### 3. 在控制台开启集群审计（投递至 CLS）

此步骤需在控制台操作（当前无直接 tccli tke API）：

1. 进入 TKE 控制台 → **注册集群** → 选择集群 → **运维功能管理**
2. 点击 **集群审计** → **设置**
3. 选择**投递方式**：
   - **公网**：集群节点通过互联网访问 CLS（适用于节点有公网 IP）
   - **内网**：集群节点通过 CLS 内网域名访问（适用于 VPC 内网互通，速度更快无公网流量费）
4. 选择**日志集**和**日志主题**（使用步骤 1 创建的 CLS 资源）
5. 点击**开启**

开启后 TKE 控制台自动配置审计日志采集 agent，无需额外 Master 节点操作。

### 4. 通过 CLI 查询审计日志

审计日志开启后，通过 `tccli cls` 检索：

```bash
tccli cls SearchLog \
  --TopicId <audit-topic-id> \
  --From 1715900000000 \
  --To 1715903600000 \
  --Query "verb:delete" \
  --Limit 10 \
  --region ap-guangzhou \
  --output json
```

```json
{
  "Context": "",
  "ListOver": true,
  "Results": [
    {
      "Time": 1715901234567,
      "LogJson": "{\"kind\":\"Event\",\"apiVersion\":\"audit.k8s.io/v1beta1\",\"level\":\"RequestResponse\",\"auditID\":\"abc123-def456-ghi789\",\"stage\":\"ResponseComplete\",\"requestURI\":\"/api/v1/namespaces/default/pods/nginx-app\",\"verb\":\"delete\",\"user\":{\"username\":\"admin\",\"groups\":[\"system:masters\",\"system:authenticated\"]},\"sourceIPs\":[\"10.0.1.100\"],\"objectRef\":{\"resource\":\"pods\",\"namespace\":\"default\",\"name\":\"nginx-app-7d9f8c5b4d-abc12\"},\"responseStatus\":{\"metadata\":{},\"code\":200},\"requestObject\":{...},\"responseObject\":{...}}"
    }
  ],
  "RequestId": "e8f9a0b1-2345-6789-0abc-def123456789"
}
```

常用 CLS 检索条件：

```bash
# 查询特定用户的删除操作
--Query "user.username:admin AND verb:delete"

# 查询 Secret 访问记录
--Query 'objectRef.resource:secrets AND verb:(get OR list)'

# 查询失败的请求
--Query "responseStatus.code:>=400"

# 查询特定命名空间的操作
--Query "objectRef.namespace:default"

# 时间范围内的 Pod 创建记录
--Query "objectRef.resource:pods AND verb:create"
```

## 验证

### 验证审计日志文件已在 Master 节点生成

```bash
# SSH 到 Master 节点
ssh root@<master-node-ip>

# 检查审计日志文件
sudo ls -lh /var/log/kubernetes/kubernetes.audit
# -rw------- 1 root root 24M Jan 15 10:00 /var/log/kubernetes/kubernetes.audit

# 查看最近的审计事件
sudo tail -5 /var/log/kubernetes/kubernetes.audit | jq '.'
```

```json
{
  "kind": "Event",
  "apiVersion": "audit.k8s.io/v1beta1",
  "level": "RequestResponse",
  "auditID": "de31f8a3-53d7-4dd5-a3a1-82c1e4b8c7f2",
  "stage": "ResponseComplete",
  "requestURI": "/api/v1/namespaces/default/services",
  "verb": "create",
  "user": {
    "username": "admin",
    "groups": ["system:masters", "system:authenticated"]
  },
  "sourceIPs": ["10.0.1.100"],
  "objectRef": {
    "resource": "services",
    "namespace": "default",
    "name": "new-service"
  },
  "responseStatus": {
    "metadata": {},
    "code": 201
  }
}
```

### 验证 API Server 审计参数已生效

```bash
# 检查 API Server Pod 的启动参数
kubectl -n kube-system get pod -l component=kube-apiserver -o yaml \
  | grep -A1 "audit"
```

```
- --audit-log-maxbackup=10
- --audit-log-maxsize=100
- --audit-log-path=/var/log/kubernetes/kubernetes.audit
- --audit-log-maxage=30
- --audit-policy-file=/etc/kubernetes/audit-policy.yaml
```

### 验证 CLS 中已有审计日志

```bash
tccli cls DescribeTopics \
  --region ap-guangzhou \
  --output json \
  --filter "Topics[?contains(TopicName, 'audit')].{TopicId:TopicId,TopicName:TopicName}"
```

## 清理

### 关闭 CLS 审计投递

在控制台进入 **运维功能管理 → 集群审计 → 关闭**，停止审计日志向 CLS 的投递。

审计数据在 CLS 中可保留，关闭仅停止新的写入。

### 移除 Master 节点审计配置（可选）

如需完全回退：

```bash
# SSH 到 Master 节点
ssh root@<master-node-ip>

# 移除 kube-apiserver.yaml 中的审计参数
sudo sed -i '/--audit-log-/d' /etc/kubernetes/manifests/kube-apiserver.yaml
sudo sed -i '/--audit-policy-file/d' /etc/kubernetes/manifests/kube-apiserver.yaml

# 删除审计策略文件（可选）
sudo rm /etc/kubernetes/audit-policy.yaml

# API Server 会自动重启（静态 Pod 检测到 manifest 变更）
```

### 清理 CLS 审计 Topic（如完全不再需要）

```bash
tccli cls DeleteTopic --TopicId <audit-topic-id> --region ap-guangzhou
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| API Server 重启失败（审计参数错误） | `kubectl logs -n kube-system kube-apiserver-<NodeName>` 查看 API Server 日志 | `/var/log/kubernetes/` 目录不存在或无写权限，或 `audit-policy.yaml` 路径/YAML 非法 | 确认目录存在且可写；`kubectl apply --dry-run=client -f audit-policy.yaml` 验证 YAML 语法 |
| 审计日志文件不增长 | `ls -la /var/log/kubernetes/audit*.log` 检查文件大小 | 审计策略 `None` 规则过于宽泛未匹配任何请求，或 `omitStages` 排除了所有阶段 | 调整策略匹配目标请求；确认 `omitStages` 未排除 `RequestReceived`/`ResponseComplete` |
| Master 节点磁盘空间耗尽 | `df -h /var/log/kubernetes/` 检查磁盘使用 | 审计日志文件过大未及时轮转 | 调整 `--audit-log-maxsize`/`--audit-log-maxbackup` 参数；设置 logrotate；将 CLS 投递作主存储 |
| CLS 中无审计日志 | `tccli cls SearchLog --TopicId <TopicId> --From <StartTime> --To <EndTime>` 直接查询 CLS | 集群审计未启用或投递网络不通 | 确认控制台中「集群审计」状态为「已开启」；检查投递方式网络连通性 |
| 审计日志过多导致性能下降 | `wc -l /var/log/kubernetes/audit*.log` 检查日志量 | 审计级别过高（`RequestResponse`）或资源匹配范围过广 | 降低级别：`RequestResponse`→`Request`→`Metadata`；按 namespace 过滤缩小范围 |
| `omitStages` 配置不生效 | `kubectl api-versions \| grep audit` 检查支持的 API 版本 | K8s 1.18+ 需 `audit.k8s.io/v1`，旧版语法不兼容 | 更新 API version 为 `audit.k8s.io/v1` |

## 下一步

- [日志采集](../日志采集/tccli%20操作.md) — 配置容器与节点日志采集
- [事件存储](../../运维指南/事件存储/tccli%20操作.md) — 将集群 Event 持久化到 CLS
- [Kubernetes 官方审计文档](https://kubernetes.io/docs/tasks/debug/debug-cluster/audit/) — 审计策略高级配置

## 控制台替代

[控制台 → 注册集群 → 运维功能管理 → 集群审计](https://console.cloud.tencent.com/tke2/cluster?rid=1) — 控制台提供一键开启/关闭审计日志投递，选择 CLS 目标及投递方式。Master 节点审计策略配置仍需手动操作（本指南步骤 1-2）。
