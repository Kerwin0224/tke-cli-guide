# 采集超级节点上的 Pod 日志

> 对照官方：[采集超级节点上的 Pod 日志](https://cloud.tencent.com/document/product/457/60701) · page_id `60701` · tccli ≥3.1.107 · API 2018-05-25

## 概述

超级节点上的 Pod 日志可通过 CLS（日志服务）或 Kafka 两种方式采集。本文档以 CLS 路径为主，演示从零搭建日志采集链路的完整 CLI 操作：创建 CLS 日志集/主题 -> 安装 tke-log-agent 组件 -> 安装日志采集 Agent -> 创建日志采集规则 -> 验证 -> 清理。

Kafka 路径通过 LogConfig CRD 的 `kafkaDetail` 字段指定消费端，创建方式与 CLS 类似，仅目标配置不同。

| 采集目标 | 适用场景 | tccli/K8s 支持 |
|---------|---------|:---:|
| CLS（日志服务） | 需要腾讯云原生日志查询、告警、仪表板 | tccli + CRD |
| Kafka（自建/CKafka） | 已有 Kafka 基础设施，需自定义消费逻辑 | CRD（仅 kubectl） |

## 前置条件

- [环境准备](../../../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0
```

**预期输出**：

```text
tccli version 1.0.x
```

```bash
# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置
```

**预期输出**：

```text
credential:
  secretId: *******************
  secretKey: *******************
  region: ap-guangzhou

[+1 more]
```

```bash
# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:CreateClusterVirtualNodePool, tke:CreateClusterVirtualNode
#    tke:DescribeClusterVirtualNodePools, tke:DescribeClusterVirtualNode
#    tke:InstallAddon, tke:DescribeAddon, tke:DeleteAddon
#    tke:CreateCLSLogConfig, tke:DescribeLogConfigs, tke:DeleteLogConfigs
#    tke:InstallLogAgent
#    vpc:DescribeVpcs, vpc:DescribeSubnets, vpc:DescribeSecurityGroups
#    cvm:DescribeInstances
#    cls:CreateLogset, cls:CreateTopic, cls:DescribeLogsets, cls:DescribeTopics
#    cls:DeleteLogset, cls:DeleteTopic
# 验证：执行 DescribeClusters 确认权限
tccli tke DescribeClusters --region <Region>
# expected: exit 0，返回集群列表
```

**预期输出**：

```json
{
    "TotalCount": 1,
    "Clusters": [{"ClusterId": "cls-example", "ClusterStatus": "Running", "ClusterVersion": "1.32.2"}]
}
```

### 资源检查

```bash
# 4. 确认集群存在且 Running
tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'
# expected: ClusterStatus == "Running"
```

**预期输出**：

```json
{
    "Clusters": [{"ClusterId": "cls-example", "ClusterStatus": "Running", "ClusterVersion": "1.32.2"}]
}
```

```bash
# 5. 确认超级节点池存在或创建
tccli tke DescribeClusterVirtualNodePools --region <Region> --ClusterId <ClusterId>
# expected: TotalCount >= 1，LifeState == "normal"
```

**预期输出**：

```json
{
    "TotalCount": 1,
    "NodePoolSet": [{"NodePoolId": "np-example", "LifeState": "normal"}]
}
```

```bash
# 6. 确认虚拟节点已就绪
tccli tke DescribeClusterVirtualNode --region <Region> --ClusterId <ClusterId> --NodePoolId <NodePoolId>
# expected: Phase == "Running"
```

**预期输出**：

```json
{
    "TotalCount": 1,
    "Nodes": [{"Name": "eklet-subnet-xxxxxxxx-xxxxxx", "Phase": "Running"}]
}
```

```bash
# 7. 检查 CLS 日志集和主题配额
tccli cls DescribeLogsets --region <Region>
# expected: 如需要新建，确认未达配额上限（单个地域最多 20 个日志集，每个日志集最多 50 个主题）
```

**预期输出**：

```json
{
    "TotalCount": 3,
    "Logsets": [{"LogsetId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx", "LogsetName": "<LogsetName>", "TopicCount": 0}]
}
```

### 服务角色授权（一次性，控制台操作）

日志采集依赖 TKE 的服务角色 `TKE_QCSLinkedRoleInEKSLog`。如果首次使用日志采集功能，需在 CAM 控制台完成授权：

1. 登录 [访问管理控制台](https://console.cloud.tencent.com/cam/role)
2. 单击**新建角色** -> 选择**腾讯云产品服务** -> **容器服务（tke）** -> **容器服务-EKS日志采集**
3. 确认角色策略 `QcloudAccessForTKELinkedRoleInEKSLog`
4. 完成创建

> 此步骤为一次性操作，CLI 不可自动化。授权后账号下所有集群的超级节点日志采集均可用。

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 创建日志集 | `cls CreateLogset` | 否 |
| 创建日志主题 | `cls CreateTopic` | 否 |
| 安装日志采集组件 | `tke InstallAddon --AddonName tke-log-agent` | 否 |
| 安装日志 Agent | `tke InstallLogAgent` | 否 |
| 创建日志采集规则 | `tke CreateCLSLogConfig` | 否 |
| 查看日志采集配置 | `tke DescribeLogConfigs` | 是 |
| 删除日志采集规则 | `tke DeleteLogConfigs` | 是 |
| 查看日志采集组件状态 | `tke DescribeAddon` | 是 |

## 关键字段说明

以下说明 `CreateCLSLogConfig`、`CreateClusterVirtualNodePool`、`InstallAddon` 的主要参数。

### CreateCLSLogConfig

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `ClusterId` | String | 是 | 集群 ID，如 `cls-xxxxxxxx` | 集群不存在 -> `ResourceNotFound` |
| `ClusterType` | String | 否 | 集群类型，标准集群填 `tke`，Serverless 集群填 `eks`。**超级节点日志采集：集群本身是 TKE 标准集群，填 `tke`，非 `eks`** | 填 `eks` -> `FailedOperation.KubernetesClientBuildError` |
| `LogsetId` | String | 否 | CLS 日志集 ID。不填则自动创建 | 日志集不存在 -> 取决于 cls 服务 |
| `LogConfig` | String | 是 | 日志采集规则 JSON，格式见下方模板。至少需指定 `inputDetail.type` 和 `clsDetail` 或 `kafkaDetail` | JSON 格式错误 -> `InvalidParameter.LogConfig` |

### CreateClusterVirtualNodePool

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `ClusterId` | String | 是 | 集群 ID | 不存在 -> `ResourceNotFound` |
| `Name` | String | 是 | 节点池名称，建议 ≤20 字符 | — |
| `SecurityGroupIds` | Array | 是 | 安全组 ID 列表，格式 `sg-xxxxxxxx` | 不存在或无权限 -> `InvalidParameter.Param` |
| `SubnetIds` | Array | 否 | 子网 ID 列表。与 VirtualNodes 互斥 | 子网不存在 -> `InvalidParameter.SubnetId` |

### InstallAddon

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `ClusterId` | String | 是 | 集群 ID | 不存在 -> `ResourceNotFound` |
| `AddonName` | String | 是 | 组件名称，如 `tke-log-agent` | 不存在 -> `UnknownParameter` |
| `AddonVersion` | String | 否 | 组件版本，不填默认安装最新版 | 版本不存在 -> `UnknownParameter`（"addon version not found"） |
| `RawValues` | String | 否 | 组件自定义配置 JSON | JSON 格式错误 -> 安装失败 |

## 操作步骤

### 步骤 1：创建 CLS 日志集和日志主题

**决策说明**：CLS 日志集（Logset）是日志主题（Topic）的容器，一个日志集下可创建多个主题。日志主题是日志的实际存储单元。超级节点日志采集需要一个配套的日志主题作为输出目标。

**最小示例** — 创建日志集：

```bash
tccli cls CreateLogset --region <Region> \
    --LogsetName <LogsetName>
# expected: exit 0, 返回 LogsetId
```

**预期输出**：

```json
{
    "LogsetId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

**最小示例** — 在日志集下创建日志主题：

```bash
tccli cls CreateTopic --region <Region> \
    --LogsetId <LogsetId> \
    --TopicName <TopicName>
# expected: exit 0, 返回 TopicId
```

**预期输出**：

```json
{
    "TopicId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

**验证**：

```bash
# 验证日志集
tccli cls DescribeLogsets --region <Region> \
    --Filters '[{"Key":"logsetId","Values":["<LogsetId>"]}]'
# expected: Logsets 数组含目标日志集
```

**预期输出**：

```json
{
    "Logsets": [{"LogsetId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx", "LogsetName": "<LogsetName>"}]
}
```

```bash
# 验证日志主题
tccli cls DescribeTopics --region <Region> \
    --Filters '[{"Key":"topicId","Values":["<TopicId>"]}]'
# expected: Topics 数组含目标主题，TopicName 匹配
```

**预期输出**：

```json
{
    "Topics": [{"TopicId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx", "TopicName": "<TopicName>", "LogsetId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"}]
}
```

### 步骤 2：安装 tke-log-agent 组件

**决策说明**：`tke-log-agent` 是 TKE 集群中管理日志采集规则的核心组件。它包含 LogConfig CRD 控制器（provisioner）和日志采集代理（log-agent）。安装此组件后，集群内才能创建 LogConfig CRD 来定义日志采集规则。`AddonVersion` 可选，不传时自动安装集群 K8s 版本对应的最新版。

> **注意**：`InstallAddon` **不是**幂等操作——重复安装已存在的组件会返回 `UnknownParameter` 错误。执行前先通过 `DescribeAddon` 检查组件是否已安装，已安装则跳过此步骤。

**检查是否已安装**：

```bash
tccli tke DescribeAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName tke-log-agent
# expected：未安装时返回空 Addons 数组，已安装时显示 Phase == "Succeeded"
```

**最小示例**（仅当未安装时执行）：

```bash
tccli tke InstallAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName tke-log-agent
# expected: exit 0, 返回 RequestId
```

**预期输出**：

```json
{
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

**验证组件安装状态**：

```bash
tccli tke DescribeAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName tke-log-agent
# expected: Addons[0].Phase == "Succeeded"
```

**预期输出**：

```json
{
    "Addons": [
        {
            "AddonName": "tke-log-agent",
            "AddonVersion": "1.1.25",
            "Phase": "Succeeded",
            "CreateTime": "2026-06-23T08:13:52Z"
        }
    ]
}
```

### 步骤 3：安装日志采集 Agent

`InstallLogAgent` 在集群节点上部署日志采集 Agent DaemonSet，负责实际采集容器标准输出和文件日志。该 DaemonSet 由 `tke-log-agent` 组件管理，卸载 `DeleteAddon tke-log-agent` 时会级联清理，无需单独操作。

```bash
tccli tke InstallLogAgent --region <Region> \
    --ClusterId <ClusterId>
# expected: exit 0, 返回 RequestId
```

**预期输出**：

```json
{
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

### 步骤 4：创建日志采集规则

**决策说明**：日志采集规则通过 `CreateCLSLogConfig` 创建（等价于在集群中创建 LogConfig CRD）。规则的 `inputDetail` 定义采集源（容器标准输出或文件），`clsDetail` 定义输出目标（CLS 日志集和主题 ID）。

`ClusterType` 参数是关键：标准 TKE 集群填 `tke`（即使使用超级节点），只有纯 EKS/Serverless 集群才填 `eks`。**填写 `eks` 会导致 `FailedOperation.KubernetesClientBuildError`。**

以下示例采集 `default` 命名空间中所有容器的标准输出（`container_stdout`），输出到指定的 CLS 日志主题。

**最小示例** — 采集 default 命名空间所有容器标准输出：

```bash
cat > log-config.json << 'EOF'
{
    "apiVersion": "cls.cloud.tencent.com/v1",
    "kind": "LogConfig",
    "metadata": {
        "name": "<ConfigName>"
    },
    "spec": {
        "clsDetail": {
            "logsetId": "<LogsetId>",
            "topicId": "<TopicId>"
        },
        "inputDetail": {
            "type": "container_stdout",
            "containerStdout": {
                "namespace": "default",
                "allContainers": true
            }
        }
    }
}
EOF

tccli tke CreateCLSLogConfig --region <Region> \
    --ClusterId <ClusterId> \
    --ClusterType tke \
    --LogsetId <LogsetId> \
    --LogConfig "$(python3 -c 'import sys,json; print(json.dumps(json.load(sys.stdin)))' < log-config.json)"
# expected: exit 0, 返回 RequestId
# 注意：--LogConfig 需要传入 JSON 字符串（单层编码），不要使用双重 json.dumps
```

**预期输出**：

```json
{
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

**增强示例** — 采集指定 workload（deployment）的日志：

```bash
cat > log-config-workload.json << 'EOF'
{
    "apiVersion": "cls.cloud.tencent.com/v1",
    "kind": "LogConfig",
    "metadata": {
        "name": "<ConfigName>-workload"
    },
    "spec": {
        "clsDetail": {
            "logsetId": "<LogsetId>",
            "topicId": "<TopicId>"
        },
        "inputDetail": {
            "type": "container_stdout",
            "containerStdout": {
                "namespace": "default",
                "allContainers": false,
                "workloads": [
                    {
                        "namespace": "default",
                        "name": "<DeploymentName>",
                        "kind": "deployment",
                        "container": "<ContainerName>"
                    }
                ]
            }
        }
    }
}
EOF
```

### 步骤 5：部署测试工作负载（kubectl）

> **注意**：以下 kubectl 命令需在集群端点可达的环境下执行。如果公网端点被 CAM 策略拒绝（常见于企业组织账号），需通过 IOA/VPN/专线或同 VPC CVM 连接内网端点。

在超级节点上部署一个测试 Pod（通过 `nodeSelector` 或 `tolerations` 调度到超级节点），验证日志是否能正常采集到 CLS。

**部署测试 Deployment**：

```yaml
# test-logger.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-logger
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: test-logger
  template:
    metadata:
      labels:
        app: test-logger
    spec:
      nodeSelector:
        node.kubernetes.io/instance-type: eklet
      containers:
      - name: logger
        image: busybox:latest
        command: ["/bin/sh", "-c"]
        args:
        - |
          while true; do
            echo "$(date) - Test log message from super node pod"
            sleep 10
          done
```

```bash
kubectl apply -f test-logger.yaml
# expected: deployment.apps/test-logger created
```

**预期输出**：

```text
deployment.apps/test-logger created
```

```bash
kubectl get pods -l app=test-logger -o wide
# expected: Pod Running, NODE 列为 eklet-xxx（超级节点）
```

**预期输出**：

```text
NAME                          READY   STATUS    RESTARTS   AGE   NODE
test-logger-xxxxxxxxx-xxxxx   1/1     Running   0          30s   eklet-subnet-xxxxxxxx-xxxxx
```

**验证日志输出**：

```bash
kubectl logs -l app=test-logger --tail=5
# expected: 最近 5 条日志，含 "Test log message from super node pod"
```

**在 CLS 控制台检索日志**：登录 [CLS 控制台](https://console.cloud.tencent.com/cls)，选择对应的日志集和日志主题，执行检索语句确认日志已上报。

## 验证

### 控制面（tccli）

**验证日志采集配置**：

```bash
tccli tke DescribeLogConfigs --region <Region> \
    --ClusterId <ClusterId> \
    --ClusterType tke
# expected: Total >= 1，LogConfigs 中包含目标配置
```

**预期输出**：

```json
{
    "Total": 1,
    "LogConfigs": "{\"Items\":[{\"metadata\":{\"name\":\"<ConfigName>\"},\"spec\":{\"clsDetail\":{\"logsetId\":\"...\",\"topicId\":\"...\"}}}]}"
}
```

**验证组件状态**：

```bash
tccli tke DescribeAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName tke-log-agent
# expected: Phase == "Succeeded"
```

**预期输出**：

```json
{
    "Addons": [
        {"AddonName": "tke-log-agent", "AddonVersion": "1.1.25", "Phase": "Succeeded"}
    ]
}
```

**验证虚拟节点池**：

```bash
tccli tke DescribeClusterVirtualNodePools --region <Region> \
    --ClusterId <ClusterId>
# expected: LifeState == "running"
```

**预期输出**：

```json
{
    "TotalCount": 1,
    "NodePoolSet": [{"NodePoolId": "np-example", "Name": "<PoolName>", "LifeState": "normal"}]
}
```

**验证虚拟节点**：

```bash
tccli tke DescribeClusterVirtualNode --region <Region> \
    --ClusterId <ClusterId> \
    --NodePoolId <NodePoolId>
# expected: Phase == "Running"
```

**预期输出**：

```json
{
    "TotalCount": 1,
    "Nodes": [{"Name": "eklet-subnet-xxxxxxxx-xxxxxx", "Phase": "Running"}]
}
```

### 数据面（kubectl）

```bash
# 5. 验证 log-agent 相关 Pod 运行中
kubectl get pods -n kube-system -l app=log-agent
# expected: at least 1 pod Running
```

**预期输出**：

```text
NAME              READY   STATUS    RESTARTS   AGE
log-agent-xxxxx   1/1     Running   0          5m
```

```bash
# 6. 验证 LogConfig CRD 存在
kubectl get logconfigs -A
# expected: 列表包含目标配置
```

**预期输出**：

```text
NAMESPACE   NAME                AGE
default     <ConfigName>        5m
```

```bash
# 7. 验证测试 Pod 日志正常输出
kubectl logs -l app=test-logger --tail=5
# expected: 最近 5 条日志
```

**预期输出**：

```text
Mon Jun 23 08:15:00 UTC 2026 - Test log message from super node pod
Mon Jun 23 08:15:10 UTC 2026 - Test log message from super node pod
Mon Jun 23 08:15:20 UTC 2026 - Test log message from super node pod
```

## 清理

> **警告**：清理顺序不可颠倒。错误的清理顺序可能导致资源残留或计费持续。先删除日志采集规则，再卸载组件，最后删除 CLS 资源。

### 数据面清理（先执行）

```bash
# 1. 删除测试 deployment
kubectl delete deployment test-logger -n default
# expected: deployment.apps "test-logger" deleted
```

### 控制面清理

```bash
# 2. 删除日志采集规则
tccli tke DeleteLogConfigs --region <Region> \
    --ClusterId <ClusterId> \
    --ClusterType tke \
    --LogConfigNames <ConfigName>
# expected: exit 0

# 3. 验证规则已删除
tccli tke DescribeLogConfigs --region <Region> \
    --ClusterId <ClusterId> \
    --ClusterType tke
# expected: ItemCount == 0

# 4. 卸载 tke-log-agent 组件
# ⚠️ 影响范围声明：DeleteAddon tke-log-agent 会使集群内**所有**日志采集规则
#   （包括其他 LogConfig，不限于本文档创建的）立即失效，且不可逆。
#   如有其他业务依赖此组件进行日志采集，请先确认后再卸载。
tccli tke DeleteAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName tke-log-agent
# expected: exit 0

# 5. 删除 CLS 日志主题
tccli cls DeleteTopic --region <Region> \
    --TopicId <TopicId>
# expected: exit 0

# 6. 删除 CLS 日志集
tccli cls DeleteLogset --region <Region> \
    --LogsetId <LogsetId>
# expected: exit 0
```

> **注意**：`DeleteLogConfigs` 只删除日志采集规则，不会自动删除 CLS 日志集和主题。如果不单独清理 `DeleteTopic` 和 `DeleteLogset`，CLS 将持续计费。虚拟节点池和虚拟节点的清理见「新建超级节点」文档。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `FailedOperation.KubernetesClientBuildError` | `tccli tke DescribeClusters --ClusterIds '["<ClusterId>"]'` 确认集群状态 | CreateCLSLogConfig 的 `ClusterType` 填了 `eks` 而非 `tke`。标准 TKE 集群即使使用超级节点，也需填 `tke` | 将 `ClusterType` 改为 `tke` 重试 |
| `InvalidParameter.Param`（公网端点） | 错误信息含 `condition tke:clusterExtranetEndpoint=true, strategyId:240463971` | 组织级 CAM 策略硬拒绝创建公网端点。`tke:clusterExtranetEndpoint=true` 条件触发 deny | 使用内网端点（`IsExtranet=false`）。内网端点需在同一 VPC 内或通过 IOA/VPN/专线连接 |
| `UnknownParameter`（"addon version not found"） | `tccli tke DescribeAddonValues --AddonName tke-log-agent` 查看可用配置 | `AddonVersion` 版本号不存在。如 `2.2.0` 等随意填写的版本号 | 不传 `AddonVersion`（默认安装最新版），或通过 DescribeAddonValues 确认可用版本 |
| `InvalidParameter.Param`（"only one of subnetId, subnetIds, virtualNodes, or count is allowed"） | CreateClusterVirtualNode 参数检查 | 同时传了 `SubnetIds` 和 `VirtualNodes`，二者互斥 | 四选一。用 `VirtualNodes` 时删除 `SubnetIds` 和 `SubnetId` |
| kubectl 返回 `InternalError "an error on the server"` | HTTP 可达但 API Server 返回错误 | 内网端点 HTTP 可达但 API 请求被拒绝。当前环境不在 VPC 内且无 IOA/VPN | 通过同 VPC CVM 或 IOA/VPN/专线连接内网端点。也可在 CLS 控制台直接检索日志确认采集生效 |

### 创建成功但状态异常

| 现象 | 诊断命令 | 根因 | 修复 |
|------|---------|------|------|
| 日志配置创建成功但 CLS 中无日志 | `kubectl get logconfigs -A` 确认 CRD 状态 + `kubectl get pods -n kube-system -l app=log-agent` 确认 Agent 运行 | 可能原因：(1) Pod 未调度到超级节点（检查 nodeSelector）；(2) LogConfig 的 namespace/allContainers 与 Pod 不匹配；(3) 服务角色 `TKE_QCSLinkedRoleInEKSLog` 未授权 | (1) 确认 Pod 的 `node.kubernetes.io/instance-type=eklet`；(2) 检查 LogConfig 的容器标准输出/文件配置；(3) 在 CAM 控制台完成服务角色授权 |
| tke-log-agent Phase 长时间为 `Installing` | `tccli tke DescribeAddon --AddonName tke-log-agent` | 组件安装中，通常 1-2 分钟完成 | 等待至 `Phase: Succeeded`。超过 5 分钟仍 Installing，检查集群资源是否充足 |
| 虚拟节点 Phase 长时间 Pending | `tccli tke DescribeClusterVirtualNode --NodePoolId <NodePoolId>` | 虚拟节点正在创建，正常需 1-3 分钟 | 等待至 `Phase: Running`。超时检查子网 IP 是否充足 |
| LogConfig 删除后 kafka 仍在推送 | `kubectl get logconfigs` 确认已删 | DeleteLogConfigs 只删除规则，不保证已采集的日志立即停止 | 删除后 kafka 消费者可能还有缓存，几分钟内自动停止 |

> 排查问题时建议保留 RequestId + ClusterId + 创建时的完整 JSON（如 `log-config.json`），便于复现和提单。

## 下一步

- [新建超级节点](../新建超级节点/tccli%20操作.md) — 创建超级节点池和虚拟节点
- [调度 Pod 至超级节点](../../超级节点调度说明/调度%20Pod%20至超级节点/tccli%20操作.md) — 将工作负载调度到超级节点
- [指定资源规格](../指定资源规格/tccli%20操作.md) — 为超级节点 Pod 指定资源
- [超级节点上支持运行 DaemonSet](../超级节点上支持运行%20DaemonSet/tccli%20操作.md) — DaemonSet 在超级节点上的运行方式

## 控制台替代

在 [TKE 控制台](https://console.cloud.tencent.com/tke2/cluster?rid=<Region>) 中，进入集群 -> **日志采集** -> **新建日志采集规则**，通过可视化界面配置日志源和输出目标。
