# 使用 kubecm 管理多集群 kubeconfig（tccli）

> 对照官方：[使用 kubecm 管理多集群 kubeconfig](https://cloud.tencent.com/document/product/457/50659) · page_id `50659`

## 概述

kubecm 是 kubeconfig 管理工具，支持多集群 kubeconfig 合并、切换、别名管理。配合 TKE 多集群场景，方便在多个集群间切换操作。

## 前置条件

- 已安装 kubecm（`kubecm version`）

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 添加集群 | `kubecm add -f <kubeconfig-file>` | 是 |
| 合并配置 | `kubecm merge -f <kubeconfig-file>` | 是 |
| 列出集群 | `kubecm list` | 是 |
| 切换集群 | `kubecm switch <name>` | 是 |
| 设置别名 | `kubecm alias <old> <new>` | 是 |

## 操作步骤

### 1. 获取多个集群 kubeconfig

```bash
tccli tke DescribeClusterKubeconfig --ClusterId cls-example-1 --region ap-guangzhou > kc1
tccli tke DescribeClusterKubeconfig --ClusterId cls-example-2 --region ap-guangzhou > kc2
```

```json
{
  "Kubeconfig": "<Kubeconfig>",
  "RequestId": "<RequestId>"
}
```

### 2. 合并到 kubecm

```bash
kubecm add -f kc1 --name cluster-prod
kubecm add -f kc2 --name cluster-staging
```

### 3. 查看和切换

```bash
kubecm list
```

```output
+----------------+---------------------+----------------------+
|   CURRENT      |        NAME         |       CONTEXT        |
+----------------+---------------------+----------------------+
|                | cluster-prod        | cluster-prod-context |
|        *       | cluster-staging     | cluster-staging-ctx  |
+----------------+---------------------+----------------------+
```

```bash
kubecm switch cluster-prod
kubectl get nodes
```

```text
NAME  STATUS  AGE
...
```

## 验证

```bash
kubecm list
kubectl config current-context
```

## 清理

> 本文操作为只读/非持久化操作，无需清理。

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubecm list` 无集群 | `tccli tke DescribeClusters --region ap-guangzhou --filter "Clusters[].{Id:ClusterId,Status:ClusterStatus}"` | `DescribeClusterKubeconfig` 未执行或集群未 Running | 确认集群 Running；重新拉取 kubeconfig 后 `kubecm add` |
| `kubecm merge` 报 `duplicate context` | `kubecm list` | 多个 kubeconfig context 名称冲突 | 用 `kubecm add -f <file> --name <unique>` 指定唯一名称 |
| `kubectl get nodes` 报 `Unable to connect to the server` | `tccli tke DescribeClusters --region ap-guangzhou --ClusterId cls-xxxxxxxx --filter "Clusters[].ClusterStatus"` | CAM 拒绝公网端点（strategyId 240463971），数据面不可达 | 接入 VPN/IOA 后在内网执行 kubectl |

## 下一步

- [使用 TKE 审计和事件服务快速排查问题](../使用%20TKE%20审计和事件服务快速排查问题/tccli%20操作.md)
- [在 TKE 中自定义 RBAC 授权](../在%20TKE%20中自定义%20RBAC%20授权/tccli%20操作.md)

## 控制台替代

无控制台界面，CLI 工具。
