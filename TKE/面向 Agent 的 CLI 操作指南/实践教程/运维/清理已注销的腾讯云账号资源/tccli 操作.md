# 清理已注销的腾讯云账号资源（tccli）

> 对照官方：[清理已注销的腾讯云账号资源](https://cloud.tencent.com/document/product/457/77336) · page_id `77336`

## 概述

当子账号注销后，需清理其在集群中残留的 RBAC 绑定、ServiceAccount 等资源。通过 `DescribeUserPermissions` 列出授权，`DeleteUserPermissions` 移除。

## 前置条件

- [环境准备](../../../环境准备.md)
- 已知注销账号的 UIN

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 查看用户权限 | `tccli tke DescribeUserPermissions --ClusterId <ClusterId>` | 是 |
| 移除权限 | `tccli tke DeleteUserPermissions --UserName <Uin>` | 否 |
| 清理 RBAC 绑定 | `kubectl delete clusterrolebinding,rolebinding -l user=<Uin>` | 否 |

## 操作步骤

### 1. 查看已授权用户

```bash
tccli tke DescribeUserPermissions --region ap-guangzhou --ClusterId <ClusterId>
```

```json
{
  "Permissions": [],
  "ClusterId": "<ClusterId>",
  "RoleName": "<RoleName>",
  "RoleType": "<RoleType>",
  "IsCustom": true,
  "Namespace": "<Namespace>",
  "TargetUin": "<TargetUin>",
  "RequestId": "<RequestId>"
}
```

### 2. 删除用户权限

```bash
tccli tke DeleteUserPermissions --region ap-guangzhou --cli-input-json "{\"ClusterId\":\"<ClusterId>\",\"UserName\":\"<Uin>\"}"
```

### 3. 清理残留 RBAC

```bash
kubectl get clusterrolebindings -o json | jq '.items[] | select(.subjects[]?.name=="<Uin>") | .metadata.name' | xargs kubectl delete clusterrolebinding
kubectl get rolebindings -A -o json | jq '.items[] | select(.subjects[]?.name=="<Uin>") | "\(.metadata.namespace) \(.metadata.name)"' | xargs -n2 sh -c 'kubectl delete rolebinding $1 -n $0'
```



```json
{
  "Permissions": [],
  "ClusterId": "<ClusterId>",
  "RoleName": "<RoleName>",
  "RoleType": "<RoleType>",
  "IsCustom": true,
  "Namespace": "<Namespace>",
  "TargetUin": "<TargetUin>",
  "RequestId": "<RequestId>"
}
```## 验证

```bash
tccli tke DescribeUserPermissions --region ap-guangzhou --ClusterId <ClusterId> --filter "length(Permissions)"
kubectl get clusterrolebindings -o json | jq '[.items[] | select(.subjects[]?.name=="<Uin>")] | length'
```

```text
NAME  STATUS  AGE
...
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DeleteUserPermissions` 返回 `UserNotFound` | `tccli tke DescribeUserPermissions --region ap-guangzhou --ClusterId cls-xxxxxxxx` | UIN 错误或权限已被他人删除 | 核对 `<Uin>`；用 `DescribeUserPermissions` 确认现存授权 |
| `kubectl delete clusterrolebinding` 报 `Unable to connect to the server` | `tccli tke DescribeClusters --region ap-guangzhou --ClusterId cls-xxxxxxxx --filter "Clusters[].ClusterStatus"` | CAM 拒绝公网端点（strategyId 240463971），数据面不可达 | 接入 VPN/IOA 后在内网执行 kubectl 清理 RBAC |
| 清理后仍残留 RBAC 绑定 | `kubectl get clusterrolebindings,rolebindings -A -o json \| jq '.items[] \| select(.subjects[]?.name=="<Uin>")'`（VPN/IOA） | RoleBinding 未按 UIN 过滤或跨命名空间遗漏 | 逐命名空间扫描并删除所有引用该 UIN 的 binding |

## 清理

清理完成后建议注销 CAM 子账号。

## 下一步

- [在 TKE 中自定义 RBAC 授权](../在%20TKE%20中自定义%20RBAC%20授权/tccli%20操作.md)
- [使用预设身份授权](../../../安全和稳定性/身份验证和授权/使用预设身份授权/tccli%20操作.md)

## 控制台替代

控制台：集群 → 授权管理 → 删除用户。
