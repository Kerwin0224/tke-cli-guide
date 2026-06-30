# 实现独立集群的 Master 容灾（tccli）

> 对照官方：[实现独立集群的 Master 容灾](https://cloud.tencent.com/document/product/457/48956) · page_id `48956`

## 概述

独立集群（INDEPENDENT_CLUSTER）模式下，用户可管理 Master 节点。通过 `ScaleOutClusterMaster` 和 `ScaleInClusterMaster` API 调整 Master 节点数量实现高可用。

## 前置条件

- 独立集群已创建

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 扩容 Master | `tccli tke ScaleOutClusterMaster --ClusterId <ClusterId> --InstanceIds ["ins-xxx"]` | 否 |
| 缩容 Master | `tccli tke ScaleInClusterMaster` | 否 |
| 查看 Master 信息 | `tccli tke DescribeMasterComponent --ClusterId <ClusterId>` | 是 |

## 操作步骤

### 扩容 Master 至 3 节点

```bash
tccli tke ScaleOutClusterMaster --region ap-guangzhou --cli-input-json file://scale-master.json
```

```json
{
    "ClusterId": "<ClusterId>",
    "InstanceIds": ["ins-master-2", "ins-master-3"]
}
```

### 查看 Master 状态

```bash
tccli tke DescribeMasterComponent --region ap-guangzhou --ClusterId <ClusterId>
```

```json
{
  "Component": "<Component>",
  "Status": "<Status>",
  "RequestId": "<RequestId>"
}
```

## 验证

```bash
kubectl get nodes -l node-role.kubernetes.io/master
```

```text
NAME  STATUS  AGE
...
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl get nodes -l node-role.kubernetes.io/master` 报 `Unable to connect to the server` | `tccli tke DescribeClusters --region ap-guangzhou --ClusterId cls-xxxxxxxx --filter "Clusters[].ClusterStatus"` | CAM 拒绝公网端点（strategyId 240463971），数据面不可达 | 接入 VPN/IOA 后在内网执行 kubectl |
| `ScaleOutClusterMaster` 返回 `ClusterTypeNotSupport` | `tccli tke DescribeClusters --region ap-guangzhou --ClusterId cls-xxxxxxxx --filter "Clusters[].ClusterType"` | 集群非 INDEPENDENT_CLUSTER，不支持 Master 扩容 | 仅独立集群可操作；托管集群 Master 由腾讯云管理 |
| 扩容后 Master 节点 `NotReady` | `tccli tke DescribeMasterComponent --region ap-guangzhou --ClusterId cls-xxxxxxxx` | etcd/control-plane 初始化未完成或 CVM 异常 | 等待 2-3 分钟；`kubectl describe node`（VPN/IOA）查看事件 |

## 清理

缩容恢复单 Master：

```bash
tccli tke ScaleInClusterMaster --region ap-guangzhou --cli-input-json file://scale-in.json
```

## 下一步

- [组建集群选型推荐](../组建集群选型推荐/tccli%20操作.md)
- [集群生命周期](../../../集群配置/集群管理/集群生命周期/tccli%20操作.md)

## 控制台替代

控制台：集群 → Master 管理 → 添加 Master 节点。
