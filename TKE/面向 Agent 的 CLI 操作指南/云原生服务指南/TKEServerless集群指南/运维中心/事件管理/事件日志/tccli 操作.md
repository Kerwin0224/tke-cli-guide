# 事件日志

> 对照官方：[事件日志](https://cloud.tencent.com/document/product/457/50988) · page_id `50988`

## 概述

TKE Serverless（EKS）集群的 **事件日志（Event）** 记录了集群内资源的状态变更和操作通知，是排障和监控的首要入口。Kubernetes Event 包括 Normal 和 Warning 两种类型，涵盖 Pod 调度、容器启动、健康检查、资源扩缩、镜像拉取等关键生命周期节点。

TKE Serverless 集群的事件与传统 TKE 集群的主要区别：
- **无节点级事件**：Serverless 集群无 Node 资源，不存在节点状态、节点磁盘压力、kubelet 等事件
- **Pod 沙箱事件**：EKS 特有的事件类型，如安全组分配、ENI 创建、沙箱启动/销毁
- **平台管理事件**：log-agent、审计组件等由平台管理的组件事件

事件持久化方式：默认通过 APIServer 内存存储（默认保留 1 小时），可通过 CLS 日志服务持久化存储，支持长期检索和分析。

> **kubectl 数据面不可达**：CAM 拒绝公网端点 (strategyId:240463971)，内网需 VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

## 前置条件

- [环境准备](../../../../../环境准备.md)
- 至少一个运行中的 TKE Serverless 集群
- kubectl 已配置并可访问集群 APIServer（需 VPN/IOA）

### 环境检查

```bash
tccli --version
# expected: tccli version 3.0.x

tccli configure list
# expected: secretId, secretKey, region 均已配置

# 验证集群可达（需 VPN/IOA）
kubectl cluster-info
# expected: Kubernetes control plane is running at ...

# 验证集群 Running
tccli tke DescribeEKSClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterStatus 为 Running

# 测试事件 API 可达
kubectl get events -n default --limit 1
# expected: (返回最近1条事件，或 No resources found)
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

| 控制台操作 | kubectl / tccli 命令 | 幂等 |
|-----------|---------------------|:--:|
| 查看集群事件 | `kubectl get events -A` | 是 |
| 按 namespace 查看 | `kubectl get events -n <ns>` | 是 |
| 按资源过滤事件 | `kubectl get events --field-selector involvedObject.name=POD_NAME` | 是 |
| 查看 Warning 事件 | `kubectl get events -A --field-selector type=Warning` | 是 |
| 按时间排序事件 | `kubectl get events -A --sort-by='.lastTimestamp'` | 是 |
| 事件详情（describe） | `kubectl describe pod <name> -n <ns>`（底部 Events 段） | 是 |
| 事件持久化到 CLS | `tccli tke EnableClusterAudit`（含事件审计） | 是 |
| 检索持久化事件 | `tccli cls SearchLog` | 是 |

## 操作步骤

### 占位符说明

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|----------|
| `CLUSTER_ID` | EKS 集群 ID | 格式 cls-xxxxxxxx | `DescribeEKSClusters` |
| `NAMESPACE` | Kubernetes 命名空间 | 不超过 63 字符 | `kubectl get ns` |
| `POD_NAME` | Pod 名称 | Kubernetes 资源名规范 | `kubectl get pods -n NAMESPACE` |
| `REGION` | 地域 | 如 ap-guangzhou | 固定值 |
| `AUDIT_TOPIC_ID` | 审计/事件日志主题 ID | UUID 格式（如已开启持久化） | `DescribeLogSwitches` |

### 步骤1: 查看全局事件

查看集群所有 namespace 的事件，按最后时间戳排序，关注 Warning 类型。

```bash
# 全局事件概览（最近事件优先）
kubectl get events -A --sort-by='.lastTimestamp'
# expected:
# NAMESPACE   LAST SEEN   TYPE      REASON              OBJECT                         MESSAGE
# default     1m          Normal    Scheduled           pod/nginx-app                  Successfully assigned default/nginx-app
# default     1m          Normal    Pulling             pod/nginx-app                  Pulling image "nginx:latest"
# default     30s         Normal    Pulled              pod/nginx-app                  Successfully pulled image "nginx:latest"
# default     30s         Normal    Created             pod/nginx-app                  Created container nginx
# default     30s         Normal    Started             pod/nginx-app                  Started container nginx

# 仅查看 Warning 事件（排障优先）
kubectl get events -A --field-selector type=Warning --sort-by='.lastTimestamp'
# expected: (列出所有 Warning 事件，如 BackOff、Failed、Unhealthy 等)
```

```text
NAME  STATUS  AGE
...
```

### 步骤2: 按 Namespace 和资源过滤事件

```bash
# 查看特定 namespace 事件
kubectl get events -n NAMESPACE --sort-by='.lastTimestamp'
# expected: (仅 NAMESPACE 下的事件，按时间排序)

# 查看特定 Pod 的事件
kubectl get events -n NAMESPACE --field-selector involvedObject.name=POD_NAME
# expected:
# LAST SEEN   TYPE      REASON      OBJECT          MESSAGE
# 1m          Normal    Scheduled   pod/POD_NAME    Successfully assigned ...
# 1m          Normal    Pulled      pod/POD_NAME    Container image "..." already present
# 30s         Normal    Created     pod/POD_NAME    Created container app

# 查看最近 10 条事件（任意 namespace）
kubectl get events -A --sort-by='.lastTimestamp' | tail -10
# expected: 最近 10 条事件
```

```text
NAME  STATUS  AGE
...
```

### 步骤3: 通过 Describe 查看 Pod 的聚合事件

`kubectl describe pod` 在底部展示该 Pod 完整的生命周期事件链，是排障常用入口。

```bash
kubectl describe pod POD_NAME -n NAMESPACE
# expected: (底部 Events 段)
# Events:
#   Type    Reason     Age   From               Message
#   ----    ------     ----  ----               -------
#   Normal  Scheduled  5m    default-scheduler  Successfully assigned NAMESPACE/POD_NAME
#   Normal  Pulling    5m    kubelet            Pulling image "nginx:latest"
#   Normal  Pulled     4m    kubelet            Successfully pulled image "nginx:latest"
#   Normal  Created    4m    kubelet            Created container nginx
#   Normal  Started    4m    kubelet            Started container nginx
```

```text
NAME  STATUS  AGE
...
```

### 步骤4: 检索 Serverless 集群特有事件

TKE Serverless 集群包含沙箱创建、ENI 分配、安全组绑定等特有事件。

```bash
# 查看 Pod 沙箱相关事件（EKS 特有）
kubectl get events -A --field-selector reason=SandboxCreated
# expected: (列出 EKS 沙箱创建事件)

kubectl get events -A --field-selector reason=FailedCreatePodSandBox
# expected: (列出沙箱创建失败事件，常见原因: 子网 IP 不足、安全组配置错误)

# 查看 ENI/IP 分配事件
kubectl get events -A --field-selector reason=FailedCreatePodSandBox | grep -E "eni|ip|subnet"
# expected: (过滤网络相关失败事件)

# 查看平台组件事件
kubectl get events -n kube-system --sort-by='.lastTimestamp'
# expected: (log-agent、coredns 等系统组件的事件)
```

```text
NAME  STATUS  AGE
...
```

### 步骤5: 通过 tccli 查看事件持久化（需开启审计）

若集群已[开启审计日志](../../审计管理/审计日志/tccli%20操作.md)，事件日志也可通过 CLS 检索。

```bash
# 检索与 Pod 相关的操作事件（Events 实为 APIServer 审计记录的一种）
tccli cls SearchLog \
    --region <Region> \
    --TopicId AUDIT_TOPIC_ID \
    --From $(python3 -c "import time; print(int((time.time()-3600)*1000))") \
    --To $(python3 -c "import time; print(int(time.time()*1000))") \
    --Query "objectRef.resource:events"
# expected: Results 含 Event 资源的 create/patch 操作

# 统计各 namespace 的 Warning 事件数量
tccli cls SearchLog \
    --region <Region> \
    --TopicId AUDIT_TOPIC_ID \
    --From $(python3 -c "import time; print(int((time.time()-3600)*1000))") \
    --To $(python3 -c "import time; print(int(time.time()*1000))") \
    --Query "objectRef.resource:events AND responseStatus.code:200" \
    --Limit 100 | jq '[.Results[] | .Content | fromjson | .objectRef.namespace] | group_by(.) | map({namespace: .[0], count: length})'
# expected: [{"namespace": "default", "count": 15}, ...]
```

### 步骤6: 监测事件流（实时）

```bash
# 实时 watch 所有 namespace 的事件
kubectl get events -A --watch
# expected: (持续输出新事件，Ctrl+C 停止)

# 实时 watch 特定 namespace 的 Warning 事件
kubectl get events -n NAMESPACE --field-selector type=Warning --watch
# expected: (持续输出 NAMESPACE 下的 Warning 事件)
```

```text
NAME  STATUS  AGE
...
```

## 验证

### 数据面（需 VPN/IOA）

```bash
# 1. 验证事件 API 正常工作
kubectl get events -A --limit 1
# expected: 返回至少 1 条事件，或无错误（新集群可能无事件）

# 2. 触发事件生成（创建 Pod 后检查事件）
kubectl run event-test --image=busybox:1.28 --restart=Never -n default -- sleep 10
# expected: pod/event-test created

# 3. 验证事件已生成
kubectl get events -n default --field-selector involvedObject.name=event-test
# expected: 包含 Scheduled、Pulling、Pulled、Created、Started 事件

# 4. 等待 Pod 完成，验证 Completed 事件
sleep 15
kubectl get events -n default --field-selector involvedObject.name=event-test | grep -E "Completed|Succeeded"
# expected: Pod 状态变更相关事件
```

```text
NAME  STATUS  AGE
...
```

### 控制面（tccli）

```bash
# 5. 验证集群状态（控制面健康是事件正常记录的前提）
tccli tke DescribeEKSClusters --region <Region> --ClusterIds '["CLUSTER_ID"]' | jq '.Clusters[0].ClusterStatus'
# expected: "Running"
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

## 清理

> **注意**：Kubernetes Event 由 APIServer 自动管理，默认保留 1 小时，无需手动清理。如果开启了 CLS 持久化存储，参见 [审计日志](../../审计管理/审计日志/tccli%20操作.md) 清理章节。

```bash
# 1. 删除测试 Pod
kubectl delete pod event-test -n default --ignore-not-found
# expected: pod "event-test" deleted

# 2. 停止事件 watch（如有运行中的 watch 进程）
# Ctrl+C 终止 kubectl get events --watch
```

```text
NAME  STATUS  AGE
...
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl get events` 返回 `Unable to connect to the server` | `kubectl cluster-info` | APIServer 不可达或 kubeconfig 未配置 | 确认 VPN/IOA 连接正常，kubeconfig 正确配置 |
| `kubectl get events` 返回 `forbidden` | `kubectl auth can-i list events -n NAMESPACE` | RBAC 权限不足，无 events 资源 list/watch 权限 | 绑定 ClusterRole `view` 或自定义 Role 授予 events 读权限 |
| 事件大量 `FailedCreatePodSandBox` | `kubectl get events -A --field-selector reason=FailedCreatePodSandBox` | 子网 IP 耗尽或安全组配置错误 | 检查子网可用 IP 数 `tccli vpc DescribeSubnets`，确认安全组规则正确 |
| `kubectl describe pod` 无事件输出 | `kubectl get pod POD_NAME -n NAMESPACE` 确认 Pod 存在 | Pod 刚创建或事件已过期（> 1 小时） | 等待数秒后重试 describe，或通过 CLS 检索历史事件 |

### 创建成功但状态异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Pod 一直 Pending，事件显示 `FailedCreatePodSandBox` | `kubectl describe pod POD_NAME` 查看详细错误 | EKS 沙箱创建失败（子网 IP 不足、安全组限制） | 扩充子网 CIDR 或更换子网，检查安全组配额和规则 |
| 镜像拉取失败事件反复出现 `Failed`/`ErrImagePull` | `kubectl get events -n NAMESPACE --field-selector type=Warning` | 镜像地址无效、私有仓库认证失败、或 TCR 网络不通 | 检查 image 拼写，验证 imagePullSecrets，确认 TCR 可达性 |
| 无 Warning 事件但 Pod 异常 | `kubectl describe pod POD_NAME` 检查 Conditions | 非 Kubernetes 层面的问题（应用自身 CrashLoopBackOff 无显式 Warning 事件） | `kubectl logs POD_NAME` 查看容器日志定位应用层错误 |
| Serverless 集群事件不同于普通 TKE 集群 | `kubectl get events -A --field-selector reason=NodeHasNoDiskPressure` | Serverless 集群无 Node 资源，缺少节点级事件 | 这是预期行为，非异常。Serverless 集群仅关注 Pod 和平台事件 |

## 下一步

- [审计日志](../../审计管理/审计日志/tccli%20操作.md)：APIServer 操作审计日志的开启和检索
- [容器实例](../../容器实例/tccli%20操作.md)：查看 Pod 实例列表和详情
- [监控和告警](../../监控和告警/tccli%20操作.md)：基于指标的数据面监控
- [TKE Serverless 常见问题](https://cloud.tencent.com/document/product/457/60223)：集群故障排查指南

## 控制台替代

控制台路径：**TKE 控制台 > Serverless 集群 > 集群 ID > 运维中心 > 事件**。控制台提供可视化事件列表，支持按类型（Normal/Warning）、时间范围、资源类型筛选，并提供事件趋势图表和聚合统计。控制台事件页面自动持久化到 CLS，保留时效更长（默认 30 天）。kubectl 直接查询适合快速排障和脚本化监控，但与控制台相比受限于 APIServer 内存事件保留时间（约 1 小时）。
