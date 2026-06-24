# 使用集群审计排查问题（tccli）

> 对照官方：[使用集群审计排查问题](https://cloud.tencent.com/document/product/457/48980) · page_id `48980`
## 概述

通过 TKE 集群审计（Audit）功能排查操作问题和安全事件。集群审计记录对 API Server 的所有请求，支持查询和告警。

> **kubectl 数据面不可达**：CAM拒绝公网端点(strategyId:240463971)，内网需VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

## 前置条件

- [环境准备](../../../环境准备.md)：`tccli` 已配置，`kubectl` >= v1.28
- 已获取集群 kubeconfig：

```bash
tccli tke DescribeClusterKubeconfig --region <Region> --ClusterId <ClusterId> \
    --filter "Kubeconfig" > kubeconfig.yaml
export KUBECONFIG=$(pwd)/kubeconfig.yaml
# expected: 生成 kubeconfig.yaml
```

```json
{
  "Kubeconfig": "<Kubeconfig>",
  "RequestId": "<RequestId>"
}
```

- 需要 CAM 权限：`tke:DescribeClusters`、`tke:DescribeClusterKubeconfig`

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 获取集群凭证 | `tccli tke DescribeClusterKubeconfig` | 是 |
| 查看集群状态 | `tccli tke DescribeClusterStatus --region <Region> --ClusterIds '["<ClusterId>"]'` | 是 |
| 相关 kubectl 操作 | 见 Procedure | -- |

## 操作步骤

### 1. 控制面：开启集群审计

```bash
# 查看审计配置
tccli tke DescribeLogSwitches --region <Region> --ClusterId <ClusterId>
# expected: 返回审计配置状态
```

```json
{
  "SwitchSet": "<SwitchSet>",
  "Audit": "<Audit>",
  "Enable": "<Enable>",
  "ErrorMsg": "<ErrorMsg>",
  "LogsetId": "<LogsetId>",
  "Status": "<Status>"
}
```

### 2. 查看审计日志

集群审计日志存储在 CLS（日志服务），可通过 CLS 控制台或 API 查询：

```bash
# 通过 tccli CLS 查询审计日志
tccli cls SearchLog --region <Region> \
    --TopicId <TopicId> \
    --From <Timestamp> \
    --To <Timestamp> \
    --Query "NOT user.username:system:*"
# expected: 返回审计日志条目
```

### 3. 常见审计查询场景

| 场景 | CLS 查询语句 | 目的 |
|------|-------------|------|
| 删除操作 | `verb:delete` | 排查误删除 |
| 特定用户 | `user.username:<user>` | 审计用户行为 |
| 失败请求 | `responseStatus.code:>=400` | 排查权限问题 |
| 资源变更 | `verb:create OR verb:update OR verb:patch` | 变更审计 |
| 敏感操作 | `objectRef.resource:secrets` | 安全审计 |

### 4. 审计事件（需 VPN/IOA）

```bash
# 通过 kubectl 查看最近事件（非审计日志）
kubectl get events -A --sort-by=.lastTimestamp
# expected: 集群事件列表
```

```text
NAME  STATUS  AGE
...
```

## 验证

### 控制面

```bash
tccli tke DescribeLogSwitches --region <Region> --ClusterId <ClusterId>
# expected: 审计配置信息

tccli tke DescribeClusterStatus --region <Region> --ClusterIds '["<ClusterId>"]'
# expected: ClusterState "Running"
```

```json
{
  "SwitchSet": "<SwitchSet>",
  "Audit": "<Audit>",
  "Enable": "<Enable>",
  "ErrorMsg": "<ErrorMsg>",
  "LogsetId": "<LogsetId>",
  "Status": "<Status>"
}
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `tccli cls SearchLog` 返回空 | `tccli tke DescribeLogSwitches --region ap-guangzhou --ClusterId cls-xxxxxxxx` | 审计未开启或 CLS Topic 未绑定 | 确认 `DescribeLogSwitches` 返回 `Enable:true`；重新 `EnableClusterAudit` 并指定有效 TopicId |
| CLS 查询无结果 | `tccli tke DescribeClusters --region ap-guangzhou --ClusterId cls-xxxxxxxx --filter "Clusters[].ClusterStatus"` | 时间范围错误或查询语句语法错误 | 扩大时间范围；简化 CLS 查询条件（如先 `*` 全量检索） |
| `kubectl get events` 报 `Unable to connect to the server` | `tccli tke DescribeClusters --region ap-guangzhou --ClusterId cls-xxxxxxxx --filter "Clusters[].ClusterStatus"` | CAM 拒绝公网端点（strategyId 240463971），数据面不可达 | 接入 VPN/IOA 后在内网执行 kubectl |

## 清理

<!-- 无需特殊清理 -->

## 下一步

- [在 TKE 中自定义 RBAC 授权](../../安全/在%20TKE%20中自定义%20RBAC%20授权/tccli%20操作.md) -- page_id `51683`
- [集群 API Server 网络无法访问排障处理](../../../故障处理/集群%20API%20Server%20网络无法访问排障处理/tccli%20操作.md) -- page_id `80912`

## 控制台替代

[容器服务控制台](https://console.cloud.tencent.com/tke2/cluster)
