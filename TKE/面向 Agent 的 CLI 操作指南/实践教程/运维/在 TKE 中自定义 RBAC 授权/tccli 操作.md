# 在 TKE 中自定义 RBAC 授权（tccli）

> 对照官方：[在 TKE 中自定义 RBAC 授权](https://cloud.tencent.com/document/product/457/51683) · page_id `51683`

## 概述

通过 Kubernetes 原生 RBAC（ClusterRole、Role、ClusterRoleBinding、RoleBinding）实现精细化权限控制。配合 `GrantUserPermissions` 将 CAM 子账号绑定到自定义 RBAC 角色。

## 前置条件

- [环境准备](../../../环境准备.md)

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 创建 ClusterRole | `kubectl apply -f clusterrole.yaml` | 是 |
| 创建 RoleBinding | `kubectl apply -f rolebinding.yaml` | 是 |
| 验证权限 | `kubectl auth can-i --as=<user> <verb> <resource>` | 是 |

## 操作步骤

### 1. 创建命名空间级角色

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]
```

### 2. 绑定到用户

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  namespace: default
  name: read-pods
subjects:
- kind: User
  name: "<SubUin>"
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### 3. 验证

```bash
kubectl auth can-i get pods --as=<SubUin> -n default
kubectl auth can-i delete pods --as=<SubUin> -n default
```

```output
yes
no
```

## 验证

```bash
kubectl get role pod-reader -n default
kubectl describe rolebinding read-pods -n default
```

```text
NAME  STATUS  AGE
...
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl auth can-i` 报 `Unable to connect to the server` | `tccli tke DescribeClusters --region ap-guangzhou --ClusterId cls-xxxxxxxx --filter "Clusters[].ClusterStatus"` | CAM 拒绝公网端点（strategyId 240463971），数据面不可达 | 接入 VPN/IOA 后在内网执行 kubectl |
| `kubectl auth can-i` 返回 `no` | `kubectl describe rolebinding read-pods -n default`（VPN/IOA） | RoleBinding 中 `subjects[].name` 与目标子账号 UIN 不一致 | 核对 `<SubUin>` 填入正确的子账号 UIN，重新 `kubectl apply` |
| 子账号仍无权限 | `tccli tke DescribeUserPermissions --ClusterId cls-xxxxxxxx --region ap-guangzhou` | CAM 侧未授予 `tke:DescribeClusters` 等基础权限 | 在 CAM 策略中为子账号添加 QcloudTKEFullAccess 或自定义策略 |

## 清理

```bash
kubectl delete role pod-reader -n default
kubectl delete rolebinding read-pods -n default
```

## 下一步

- [使用预设身份授权](../../../安全和稳定性/身份验证和授权/使用预设身份授权/tccli%20操作.md)
- [自定义策略授权](../../../安全和稳定性/身份验证和授权/自定义策略授权/tccli%20操作.md)

## 控制台替代

控制台：集群 → 授权管理 → RBAC 策略生成器。
