---
doc_type: Troubleshooting
subtype: 7B
---
# TKE 故障排查

> 官方文档：[常见高危操作](https://cloud.tencent.com/document/product/457/39539)
>
> 配额：地域集群数默认 20、安全组规则数等限制可能影响排查路径，超限信号见 [配额说明](https://cloud.tencent.com/document/product/457/9087)。

## 从这里开始

```bash
tccli tke DescribeClusterStatus --region ap-guangzhou --ClusterIds '["<CLUSTER_ID>"]'
# expected: ClusterStatusSet[0].ClusterState = "Running", ClusterDeletionProtection = true
```

正常输出： `ClusterState: "Running"` + `ClusterInstanceState: "AllNormal"` 表示集群正常。任何其他状态见 [状态机](reference/states.md) 或下方。

## 诊断工具箱

| 命令 | 检查什么 | 何时使用 |
|---------|---------------|------------|
| `tccli tke DescribeClusterStatus --ClusterIds '["<ID>"]'` | 集群状态 + 删除保护状态 + 节点计数 | 任何集群异常时第一步 |
| `tccli tke DescribeClusters --ClusterIds '["<ID>"]'` | 集群全量配置 | 怀疑配置被修改时 |
| `tccli tke DescribeClusterInstances --ClusterId "<ID>"` | 节点列表 + 各节点状态 | 节点不能加入/运行异常时 |
| `tccli tke DescribeClusterKubeconfig --ClusterId "<ID>"` | kubeconfig 是否可获取 | kubectl 连接失败时 |
| `tccli tke DescribeClusterEndpoints --ClusterId "<ID>"` | 访问端点状态 | 无法连接 API Server 时 |
| `tccli tke DescribeTasks --Filter '[{"Name":"ClusterId","Values":["<ID>"]},{"Name":"TaskType","Values":["node_upgrade"]}]' --Latest true` | 异步任务进度 | 操作卡住时 |

> `DescribeTasks` 入参是 `Filter`+`Latest`，无 `TaskIds` 参数。**`Filter` 内 `TaskType` 必传**（不传报 `InvalidParameter.Param: PARAM_ERROR(TaskType is empty)`），取值如 `node_upgrade`/`add_cluster_cidr`/`node_upgrade_ctl`；`ClusterId` 在 `Filter` 内（注意大小写 `ClusterId` 非 `cluster-id`）。`Latest=true` 只取最新一条。无匹配任务返回空 `Tasks[]` 或 `FailedOperation.TaskNotFound`。

```bash
# 查询集群的异步任务进度（Filter 内 ClusterId + TaskType 必传，Latest=true 取最新）
tccli tke DescribeTasks --region ap-guangzhou \
  --Filter '[{"Name":"ClusterId","Values":["<CLUSTER_ID>"]},{"Name":"TaskType","Values":["node_upgrade"]}]' \
  --Latest true
# expected: exit 0, 返回 {"Tasks": [...], "RequestId": "..."}；无任务时 Tasks 为空或报 FailedOperation.TaskNotFound
```

## 常见问题

### 集群卡在 Creating 状态超过 30 分钟

**可能原因**： 可用区资源不足或 VPC 配置冲突。

**诊断**：

```bash
tccli tke DescribeClusterStatus --region ap-guangzhou --ClusterIds '["<CLUSTER_ID>"]'
# expected: 查看 ClusterState 值和状态变更时间
```

如果 `ClusterState: "Creating"` 持续超过 30 分钟，多为底层资源问题。

**修复**：

```bash
# 1. 删除卡住的集群
tccli tke DisableClusterDeletionProtection --region ap-guangzhou --ClusterId "<CLUSTER_ID>"
tccli tke DeleteCluster --region ap-guangzhou --ClusterId "<CLUSTER_ID>" --InstanceDeleteMode terminate
# 2. 换可用区重建
tccli tke CreateCluster --region ap-guangzhou --ClusterType MANAGED_CLUSTER \
  --ClusterBasicSettings '{"ClusterName":"<NEW_NAME>","ClusterVersion":"1.34.1",...}'
```

**验证**：

```bash
tccli tke DescribeClusterStatus --ClusterIds '["<NEW_CLUSTER_ID>"]'
# expected: 5-10 分钟内 ClusterState 变为 "Running"
```

### 节点一直 initializing 或 NotReady

**可能原因**： 安全组规则阻止 kubelet 与 API Server 通信，或节点脚本执行失败。

**诊断**：

```bash
# 1. 查看节点状态
tccli tke DescribeClusterInstances --region ap-guangzhou --ClusterId "<CLUSTER_ID>"
# expected: 找到 InstanceState 为 "initializing" 或 "failed" 的节点

# 2. 查看节点池详情
tccli tke DescribeClusterNodePoolDetail --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" --NodePoolId "<NODEPOOL_ID>"
# expected: 检查 LifeState 和伸缩配置
```

**修复**： 常见原因及修复:

1. **安全组问题**: 确保节点安全组出方向允许 443 端口到集群 API Server 地址
2. **镜像拉取失败**: 检查 `DescribeClusterInstances` 中的 `InstanceAdvancedSettings.UserScript` 是否有错误
3. **磁盘空间不足**: 检查 DataDisk 配置是否足够 (建议 ≥50GB)

```bash
# 如果节点完全无法恢复，移出并销毁
tccli tke RemoveNodeFromNodePool --ClusterId "<CLUSTER_ID>" --NodePoolId "<POOL>" \
  --InstanceIds '["<INSTANCE_ID>"]'
tccli tke DeleteClusterInstances --ClusterId "<CLUSTER_ID>" \
  --InstanceIds '["<INSTANCE_ID>"]'
```

**验证**：

```bash
tccli tke DescribeClusterInstances --region ap-guangzhou --ClusterId "<CLUSTER_ID>"
# expected: 所有节点 InstanceState 为 "running"，DrainStatus 为空
```

### kubectl 连接集群失败

**可能原因**： kubeconfig 过期、访问端点未开启、或凭证配置错误。

**诊断**：

```bash
# 1. 检查端点状态
tccli tke DescribeClusterEndpoints --region ap-guangzhou --ClusterId "<CLUSTER_ID>"
# expected: 至少有一个端点 Status 为 Created

# 2. 重新获取 kubeconfig
tccli tke DescribeClusterKubeconfig --region ap-guangzhou --ClusterId "<CLUSTER_ID>"
# expected: 返回有效 kubeconfig (base64)
```

**修复**：

```bash
# 如果端点未开启
tccli tke CreateClusterEndpoint --region ap-guangzhou --ClusterId "<CLUSTER_ID>"

# 如果 kubeconfig 过期
tccli tke UpdateClusterKubeconfig --region ap-guangzhou --ClusterId "<CLUSTER_ID>"
```

**验证**：

> kubectl（K8s 原生命令，非 tccli；TCCLI 管 TKE 抽象层不提供 K8s 资源操作能力）
<!-- tccli不提供K8s集群连通验证与节点诊断(kubectl cluster-info/nodes/describe)，排查辅助非tccli边界 -->
```bash
# kubectl 验证 kubeconfig 连通 (K8s 原生命令, TCCLI 不提供集群连通验证)
kubectl --kubeconfig <KUBECONFIG_FILE> cluster-info
# expected: Kubernetes control plane is running at https://...
```

### 删除保护阻止删除集群

**可能原因**： 集群创建时或手动开启了删除保护。

**诊断**：

```bash
tccli tke DescribeClusterStatus --region ap-guangzhou --filter "ClusterStatusSet[?ClusterId=='<CLUSTER_ID>'] | [0].ClusterDeletionProtection"
# expected: true（开启删除保护）
```

**修复**：

```bash
tccli tke DisableClusterDeletionProtection --region ap-guangzhou --ClusterId "<CLUSTER_ID>"
# expected: exit 0

# 验证
tccli tke DescribeClusterStatus --region ap-guangzhou --filter "ClusterStatusSet[?ClusterId=='<CLUSTER_ID>'] | [0].ClusterDeletionProtection"
# expected: false（已关闭）
```

**验证**：

```bash
tccli tke DeleteCluster --region ap-guangzhou --ClusterId "<CLUSTER_ID>"
# expected: exit 0 (不再返回删除保护错误)
```

## 高危操作后果速查

> 下列操作易导致业务故障；部分**不可恢复**。排障前先对照：是否刚改过安全组、内核参数、LB 控制台、CBS 挂载。完整高危操作清单见官方 [常见高危操作](https://cloud.tencent.com/document/product/457/39539)。

### 集群 / 节点

| 对象 | 高危操作 | 后果 | 误操作处理 |
|:-----|:---------|:-----|:-----------|
| Master/Etcd | 改节点安全组未按推荐放通 | Master 可能不可用 | 按 [安全组](https://cloud.tencent.com/document/product/457/9084) 放通 |
| Master/Etcd | 节点到期/销毁、重装 OS、删 `/etc/kubernetes`、自行换证书 | Master 不可用 | **不可恢复**（须重建） |
| Master/Etcd | 自行升级 master/etcd 组件版本 | 集群可能不可用 | 回退到原始版本 |
| Master/Etcd | 更改节点 IP | Master 不可用 | 改回原 IP |
| Worker | 改安全组 / 改规格强制关机 / 重装 OS / 改 IP | 节点不可用 | 移出再加入；到期销毁则**不可恢复** |
| Worker | 自行改核心组件参数 / OS 配置 | 节点可能不可用 | 还原配置或删节点重购 |
| 账号 | CAM 权限变更 | CLB 等资源可能创建失败 | 恢复权限 |

### 网络 / LB / 日志 / CBS

| 高危操作 | 后果 | 误操作处理 |
|:---------|:-----|:-----------|
| `net.ipv4.ip_forward=0` | 网络不通 | 改回 `=1` |
| `net.ipv4.tcp_tw_recycle=1` | NAT 异常 | 改回 `=0` |
| 安全组未放通容器 CIDR 的 53/udp | 集群 DNS 失败 | 按推荐放通安全组 |
| 改/删 TKE 管理的 LB 标签 | 可能触发新购 LB | 恢复标签 |
| 在 LB 控制台改 TKE 管理的监听器/后端 RS/证书/监听器名 | 被 TKE 重置或禁止 | 用 Service/Ingress YAML 管理 |
| 删宿主机 `/tmp/ccs-log-collector/pos` | 日志重复采集 | 无（pos 记采集位点） |
| 删 `/tmp/ccs-log-collector/buffer` | 日志丢失 | 无 |
| 控制台手动解挂 CBS / 节点 umount / 直接操作块设备 | Pod IO 异常或写本地盘 | 清 mount 后重调度；或重新 mount |

## 升级

如果以上步骤无法解决问题，收集以下信息提交工单:

```bash
# 1. 集群基本信息
tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["<CLUSTER_ID>"]' > cluster-info.json

# 2. 集群状态详情
tccli tke DescribeClusterStatus --region ap-guangzhou --ClusterIds '["<CLUSTER_ID>"]' > cluster-status.json

# 3. 最近操作日志（从控制台获取 CloudAudit 日志）
# 或运行: tccli cloudaudit LookUpEvents --LookupAttributes '[{"AttributeKey":"ResourceName","AttributeValue":"<CLUSTER_ID>"}]'

# 4. 错误 RequestId
# 从之前失败的 API 响应中获取 RequestId
```

提交到: [腾讯云工单系统](https://console.cloud.tencent.com/workorder)，附带以上 JSON 文件和 RequestId。

## 下一步

- [状态机](reference/states.md) — 集群/节点池/节点状态枚举与含义
- [错误码](reference/error-codes.md) — `UnsupportedOperation`/`LimitExceeded` 等诊断
- [配额和限制](reference/quotas.md) — `LimitExceeded` 的配额阈值
- [管理访问端点](networking/endpoints.md) — kubectl 连接失败的端点配置
- [认证配置](security/auth.md) — kubeconfig 获取与轮转
- [删除集群](clusters/delete.md) — 删除保护与残留清理

