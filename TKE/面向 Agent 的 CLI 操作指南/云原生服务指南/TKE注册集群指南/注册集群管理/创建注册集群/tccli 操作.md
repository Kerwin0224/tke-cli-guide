# 创建注册集群

> 对照官方：[创建注册集群](https://cloud.tencent.com/document/product/457/63218) · page_id `63218`

## 概述

注册集群（External Cluster）将 IDC 数据中心或其他公有云上已运行的 Kubernetes 集群注册到 TKE 管理面，由 TKE 提供统一管理能力。注册后，TKE 仅作为管理平面，不承载数据面 Master/Etcd，外部集群自行维护控制面。

本文演示通过 `tccli tke CreateExternalCluster` 在 TKE 侧创建一个注册集群条目，获取 agent 部署命令，在外部集群执行 kubectl 完成 agent 安装，并验证注册成功。涉及两类操作：**tccli**（TKE 管理面）和 **kubectl**（外部集群数据面），需在各自环境中分别执行。

## 前置条件

- [环境准备](../../../../环境准备.md)

### 环境检查

```bash
tccli --version
# expected: tccli version >= 1.0.0

tccli configure list
# expected: secretId, secretKey, region 均已配置

tccli tke DescribeExternalClusters --region <Region>
# expected: exit 0，返回注册集群列表（可为空）
```

```json
{"RequestId": "..."}
```

### 资源检查

```bash
# 确认外部集群 kubectl 可用（在外部集群执行）
kubectl version --client --output yaml
# expected: 返回 clientVersion

kubectl cluster-info
# expected: 外部集群控制面可达

kubectl get nodes
# expected: 外部集群节点列表

# 确认外部集群 Kubernetes 版本（需 >= 1.16）
kubectl version --output json | jq -r '.serverVersion.gitVersion'
# expected: v1.xx.x
```

```text
NAME  STATUS  AGE
...
```

### 版本与网络模式选择

- **Kubernetes 版本**：需 >= 1.16，与外部集群实际版本一致
- **网络模式**：`HostNetwork`（简单，适合小规模）或 `CiliumBGP`（适合跨数据中心，需外部集群已部署 Cilium）
- **Agent**：TKE 会在外部集群部署 agent（`tke-` 命名空间），负责与控制面通信

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 创建注册集群 | `tccli tke CreateExternalCluster --region <Region> --ClusterName "<name>" --K8sVersion "<ver>"` | 否（同名不可重复） |
| 查看注册集群列表 | `tccli tke DescribeExternalClusters --region <Region>` | 是（只读） |
| 查看指定集群 | `tccli tke DescribeExternalClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'` | 是（只读） |
| 获取 agent 部署 YAML | `tccli tke EnableExternalNodeSupport --region <Region> --ClusterId CLUSTER_ID` | 是（幂等） |
| 在外部集群部署 agent | `kubectl apply -f agent.yaml` | 是（幂等） |
| 查看注册集群状态 | `tccli tke DescribeExternalClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'` | 是（只读） |
| 集群名称 | `--ClusterName` | 不可重复 |
| Kubernetes 版本 | `--K8sVersion` | 需与外部集群一致 |
| 网络模式 | `--NetworkType`（`HostNetwork` / `CiliumBGP`） | 创建后不可修改 |
| 标签 | `--ClusterDesc` 或 `--TagSpecification` | 覆盖 |

## 操作步骤

### 步骤1：在 TKE 侧创建注册集群条目

```bash
tccli tke CreateExternalCluster --region <Region> \
  --ClusterName "CLUSTER_NAME" \
  --K8sVersion "K8S_VERSION" \
  --NetworkType "NETWORK_TYPE" \
  --output json
# expected: exit 0，返回 ClusterId
```

预期输出：

```json
{
    "ClusterId": "cls-xxxxxxxx",
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `REGION` | 地域 | 如 `ap-guangzhou` | `tccli configure list` |
| `CLUSTER_NAME` | 集群名称 | 长度 1-60，不可重复 | 自定义 |
| `K8S_VERSION` | Kubernetes 版本 | 如 `1.28.0`，需与外部集群一致 | `kubectl version --output json`（外部集群执行） |
| `NETWORK_TYPE` | 网络模式 | `HostNetwork` 或 `CiliumBGP` | 根据外部集群网络方案选择 |

> **注意**：`CreateExternalCluster` 仅在 TKE 侧创建管理记录，不会在外部集群部署任何资源。外部集群此时不受影响。

### 步骤2：轮询确认集群条目创建成功

```bash
tccli tke DescribeExternalClusters --region <Region> \
  --ClusterIds '["CLUSTER_ID"]' --output json
# expected: ClusterStatus 为 Running 或 Normal
```

```json
{"RequestId": "..."}
```

### 步骤3：获取 Agent 部署命令

```bash
tccli tke EnableExternalNodeSupport --region <Region> \
  --ClusterId CLUSTER_ID --output json
# expected: exit 0，返回 agent 部署 YAML（Base64 编码）或 agent install command
```

将返回的 agent 部署内容保存为文件：

```bash
tccli tke EnableExternalNodeSupport --region <Region> \
  --ClusterId CLUSTER_ID --output json \
  | jq -r '.Result' > /tmp/tke-agent.yaml
# expected: 生成 agent YAML 文件
```

### 步骤4：在外部集群执行 Agent 部署

> 以下命令在**外部集群**的 kubectl 管理节点执行。

```bash
# 在外部集群部署 TKE agent
kubectl apply -f /tmp/tke-agent.yaml
# expected: namespace/tke-xxx created, deployment/tke-agent created, ...

# 等待 agent Pod 就绪
kubectl get pods -n tke-xxx --watch
# expected: 所有 agent Pod 状态变为 Running

# 检查 agent 日志
kubectl logs -n tke-xxx deployment/tke-agent --tail=20
# expected: 日志无连接错误
```

```text
NAME  STATUS  AGE
...
```

| 占位符 | 说明 | 获取方式 |
|--------|------|---------|
| `tke-xxx` | agent 所在的命名空间 | 查看 agent.yaml 文件中的 namespace 字段 |

### 步骤5：在 TKE 侧验证注册状态

```bash
tccli tke DescribeExternalClusters --region <Region> \
  --ClusterIds '["CLUSTER_ID"]' --output json
# expected: ClusterId 正确，状态为正常（如 Running/Normal）
```

```json
{"RequestId": "..."}
```

## 验证

### TKE 管理面验证（tccli）

- `DescribeExternalClusters` 返回 `ClusterId: CLUSTER_ID` 且状态正常。
- `DescribeExternalClusterCredential` 能返回有效的 kubeconfig（见 [连接注册集群](../连接注册集群/tccli%20操作.md)）。

### 外部集群数据面验证（kubectl）

```bash
# agent Pod 运行正常
kubectl get pods -n tke-xxx
# expected: 所有 agent Pod STATUS 为 Running

# agent 日志无异常
kubectl logs -n tke-xxx deployment/tke-agent --tail=50
# expected: 无 ERROR 级别日志，持续打印心跳/连接成功信息
```

```text
NAME  STATUS  AGE
...
```

### 跨面验证

```bash
# TKE 侧查询集群节点（若 agent 已同步）
tccli tke DescribeExternalClusters --region <Region> \
  --ClusterIds '["CLUSTER_ID"]' --output json \
  | jq '.ClusterSet[0] | {ClusterId, ClusterStatus, NodeCount}'
# expected: NodeCount >= 外部集群节点数
```

```json
{"RequestId": "..."}
```

## 清理

> **警告**：删除注册集群会解除 TKE 管理面关联，但**不会删除外部集群上的 agent 资源**。需手动清理。
> **计费提醒**：注册集群本身不收取管理费，但若启用了 TKE 日志/审计等附加功能，相关 CLS 资源会持续计费。

### 1. 解除 TKE 侧注册

```bash
tccli tke DeleteExternalCluster --region <Region> \
  --ClusterId CLUSTER_ID --output json
# expected: exit 0
```

### 2. 确认已解除

```bash
tccli tke DescribeExternalClusters --region <Region> \
  --ClusterIds '["CLUSTER_ID"]'
# expected: ResourceNotFound 或返回空
```

```json
{"RequestId": "..."}
```

### 3. 清理外部集群 agent 资源（在外部集群执行）

```bash
kubectl delete -f /tmp/tke-agent.yaml
# expected: namespace "tke-xxx" deleted, 相关资源全部清理
```

> **警告**：清理外部集群 agent 资源后，TKE 管理面将彻底失去与该集群的连接，该操作不可逆。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `CreateExternalCluster` 返回 `InvalidParameter.ClusterName` | 检查集群名称格式 | 名称重复或包含非法字符 | 修改 `ClusterName`，确保地域内唯一 |
| `CreateExternalCluster` 返回 `InvalidParameter.K8sVersion` | `kubectl version --output json`（外部集群）确认版本 | 版本号格式错误或过低（< 1.16） | 修正版本号为 `Major.Minor.Patch` 格式 |
| `CreateExternalCluster` 返回 `InvalidParameter.NetworkType` | 检查参数值 | 网络模式仅支持 `HostNetwork` 或 `CiliumBGP` | 确认使用合法枚举值 |
| `EnableExternalNodeSupport` 返回 `ResourceNotFound` | `DescribeExternalClusters` 检查集群存在性 | 集群 ID 不存在或地域不匹配 | 确认 `ClusterId` 和 `--region` 正确 |
| `EnableExternalNodeSupport` 返回 `FailedOperation.ClusterState` | `DescribeExternalClusters` 查看集群状态 | 集群状态异常 | 等待集群状态变为 Normal 后重试 |
| `kubectl apply` 返回 `AlreadyExists` | `kubectl get ns tke-xxx` 确认 | agent 已部署，重复 apply | 该错误无害；若需重装，先 `kubectl delete -f /tmp/tke-agent.yaml` |

### 操作成功但异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| agent Pod 持续 `CrashLoopBackOff` | `kubectl describe pod -n tke-xxx <pod>` | 镜像拉取失败或资源不足 | 确认外部集群能拉取 TKE agent 镜像，增加资源配额 |
| agent Pod Running 但 TKE 控制面看不到节点 | `kubectl logs -n tke-xxx deployment/tke-agent` 查看日志 | 网络不通（agent 到 TKE 控制面） | 确认外部集群节点能出公网访问 TKE API，检查防火墙/NAT 配置 |
| agent 部署后 `DescribeExternalClusters` 状态仍为 `Pending` | 等待 3-5 分钟再查 | agent 注册心跳有延迟 | 继续等待；超过 10 分钟检查 agent 日志中的连接地址是否正确 |
| agent YAML 为空或格式异常 | 检查 `EnableExternalNodeSupport` 返回的 `Result` 字段 | API 返回异常或 Base64 解码失败 | 确认 `jq` 正确提取字段，若为 Base64 需 `base64 -d` |

## 下一步

- [连接注册集群](../连接注册集群/tccli%20操作.md) — 获取 kubeconfig 并连接外部集群（page_id `63216`）
- [日志采集](../../运维指南/日志采集/tccli%20操作.md) — 配置外部集群日志采集到 CLS（page_id `72148`）
- [集群审计](../../运维指南/集群审计/tccli%20操作.md) — 启用集群审计日志（page_id `72149`）
- [事件存储](../../运维指南/事件存储/tccli%20操作.md) — 持久化 Kubernetes Events 到 CLS（page_id `72150`）

## 控制台替代

[容器服务控制台 → 集群列表 → 注册集群](https://console.cloud.tencent.com/tke2/external) → 点击 **注册**，填写集群名称、版本、网络模式，获取 agent 部署命令后在外部集群执行。
