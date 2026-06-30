# IDC 集群添加超级节点用于弹性扩容（tccli）

> 对照官方：[IDC 集群添加超级节点用于弹性扩容](https://cloud.tencent.com/document/product/457/62028) · page_id `62028`

## 概述

在混合云场景中，通过 `CreateExternalNodePool` 将超级节点添加到 IDC 注册集群，利用腾讯云弹性容器实例（EKS CI）作为弹性资源池，实现 IDC 集群的秒级弹性扩容。

## 前置条件

- 已有注册集群（通过 `CreateCluster` 注册外部节点）

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 创建超级节点池 | `tccli tke CreateClusterVirtualNodePool --ClusterId <ClusterId>` | 否 |
| 调度到超级节点 | Pod `nodeSelector` / `tolerations` | 是 |
| 查看超级节点 | `tccli tke DescribeClusterVirtualNode` | 是 |

## 操作步骤

### 1. 创建超级节点池

```bash
tccli tke CreateClusterVirtualNodePool --region ap-guangzhou --cli-input-json file://vnpool.json
```

```json
{
    "ClusterId": "<ClusterId>",
    "Name": "burst-pool",
    "SubnetIds": ["subnet-example"],
    "SecurityGroupIds": ["sg-example"]
}
```

### 2. 调度到超级节点

```yaml
nodeSelector:
  node.kubernetes.io/instance-type: eklet
tolerations:
- key: "eks.tke.cloud.tencent.com/eklet"
  operator: "Exists"
```

## 验证

```bash
kubectl get nodes -l node.kubernetes.io/instance-type=eklet
```

```text
NAME  STATUS  AGE
...
```

## 清理

> 本文操作为只读/非持久化操作，无需清理。

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl` 连接超时/拒绝 | `tccli tke DescribeClusters` 返回集群 Running 但 kubectl 不可达 | CAM 策略 240463971 拒绝公网访问 APIServer 端点 | 通过 VPN/IOA 接入内网后执行 kubectl 命令 |
| 超级节点不显示 | `tccli tke DescribeClusterVirtualNode --ClusterId <ClusterId>` 检查节点池状态 | 节点池创建失败或 SubnetIds/SecurityGroupIds 无效 | 确认子网和安全组在同一 VPC 且有可用资源 |
| Pod 无法调度到超级节点 | `kubectl describe pod <pod>` 查看 Events | Pod 未配置正确的 nodeSelector 或 tolerations | 添加 `nodeSelector: node.kubernetes.io/instance-type: eklet` 和对应 tolerations |

## 下一步

- [超级节点使用指南](../../../集群配置/节点管理/超级节点/tccli%20操作.md)
- [集群弹性伸缩实践](../../弹性伸缩/集群弹性伸缩实践/tccli%20操作.md)

## 控制台替代

控制台：集群 → 节点管理 → 超级节点 → 新建节点池。
