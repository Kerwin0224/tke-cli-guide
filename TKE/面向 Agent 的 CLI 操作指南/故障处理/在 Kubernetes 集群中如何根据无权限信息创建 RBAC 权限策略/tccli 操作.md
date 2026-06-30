# 在 Kubernetes 集群中如何根据无权限信息创建 RBAC 权限策略（tccli）

> 对照官方：[在 Kubernetes 集群中如何根据无权限信息创建 RBAC 权限策略](https://cloud.tencent.com/document/product/457/108475) · page_id `108475`

## 概述

当用户或 ServiceAccount 在执行 kubectl 操作时收到 403 Forbidden 错误，需要根据缺少的权限信息创建最小 RBAC 权限策略（Role/ClusterRole + RoleBinding/ClusterRoleBinding）。

> **kubectl 数据面不可达**：CAM拒绝公网端点(strategyId:240463971)，内网需VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

## 前置条件

- [环境准备](../../环境准备.md)：`tccli` 已配置
- 已获取集群 kubeconfig（如需 kubectl 操作）
- 具备集群管理员权限（cluster-admin 或同等角色）
- 了解 Kubernetes RBAC 基本概念（Role、ClusterRole、RoleBinding、ClusterRoleBinding）
- 集群为 MANAGED_CLUSTER 类型，K8s 1.30.0，ap-guangzhou

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 查看集群状态 | `tccli tke DescribeClusterStatus --region <Region> --ClusterIds '["CLUSTER_ID"]'` | 是 |
| 查看 RBAC 资源 | `kubectl get roles,clusterroles,rolebindings,clusterrolebindings -A`（需 VPN/IOA） | 是 |
| 查看 ServiceAccount | `kubectl get sa -A`（需 VPN/IOA） | 是 |
| 创建 Role | `kubectl create role <name> --verb=<verbs> --resource=<resources> -n NAMESPACE`（需 VPN/IOA） | 否（同名报错） |
| 创建 ClusterRole | `kubectl create clusterrole <name> --verb=<verbs> --resource=<resources>`（需 VPN/IOA） | 否（同名报错） |
| 创建 RoleBinding | `kubectl create rolebinding <name> --role=<role> --serviceaccount=NAMESPACE:<sa> -n NAMESPACE`（需 VPN/IOA） | 否（同名报错） |
| 测试权限 | `kubectl auth can-i <verb> <resource> --as=system:serviceaccount:NAMESPACE:<sa> -n NAMESPACE`（需 VPN/IOA） | 是 |

## 操作步骤

### 1. 控制面：确认集群正常

```bash
tccli tke DescribeClusterStatus --region <Region> --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterState "Running"
```

```json
{
  "ClusterStatusSet": "<ClusterStatusSet>",
  "ClusterId": "<ClusterId>",
  "ClusterState": "<ClusterState>",
  "ClusterInstanceState": "<ClusterInstanceState>",
  "ClusterBMonitor": "<ClusterBMonitor>",
  "ClusterInitNodeNum": "<ClusterInitNodeNum>"
}
```

### 2. 数据面：定位缺少的权限（需 VPN/IOA）

**场景一：用户/SA 已报 403 错误**

从报错信息中提取缺失的权限：

```bash
# 典型 403 报错格式：
# Error from server (Forbidden): <resource> is forbidden:
# User "system:serviceaccount:NAMESPACE:<sa>" cannot <verb> resource "<resource>" in API group "<group>" in the namespace "NAMESPACE"
```

**场景二：审计日志分析（需审计日志开启）**

```bash
# 查看审计日志中 403 记录（需集群已开启审计）
kubectl get events -A --field-selector type!=Normal | grep -i forbidden
# expected: 返回 403 相关事件记录
```

```text
NAME  STATUS  AGE
...
```

**场景三：主动测试权限**

```bash
# 以目标 SA 身份测试特定操作
kubectl auth can-i list pods --as=system:serviceaccount:NAMESPACE:<sa> -n NAMESPACE
# expected: yes 或 no（no 表示无权限）

# 列出目标 SA 当前所有权限
kubectl auth can-i --list --as=system:serviceaccount:NAMESPACE:<sa> -n NAMESPACE
# expected: 列出该 SA 具有的所有 verbs 和 resources
```

### 3. 数据面：创建最小权限 RBAC 策略（需 VPN/IOA）

**步骤 3a：确定权限范围**

| 需求 | 使用资源类型 | 示例 |
|------|-------------|------|
| 单个命名空间内操作 | Role + RoleBinding | 仅操作 `default` 命名空间的 Pod |
| 全集群范围操作 | ClusterRole + ClusterRoleBinding | 查看所有命名空间的 Node、PV |
| 全集群范围但在特定命名空间绑定 | ClusterRole + RoleBinding | 复用 ClusterRole 到某命名空间 |

**步骤 3b：创建 Role/ClusterRole**

```bash
# 示例：允许在 NAMESPACE 命名空间中 get/list/watch Pod 和 Deployment
kubectl create role pod-reader \
    --verb=get --verb=list --verb=watch \
    --resource=pods --resource=deployments \
    -n NAMESPACE
# expected: role.rbac.authorization.k8s.io/pod-reader created

# 示例：允许全集群只读 Pod（ClusterRole）
kubectl create clusterrole cluster-pod-reader \
    --verb=get --verb=list --verb=watch \
    --resource=pods
# expected: clusterrole.rbac.authorization.k8s.io/cluster-pod-reader created
```

**步骤 3c：创建 RoleBinding/ClusterRoleBinding**

```bash
# 绑定 Role 到 ServiceAccount
kubectl create rolebinding pod-reader-binding \
    --role=pod-reader \
    --serviceaccount=NAMESPACE:<sa-name> \
    -n NAMESPACE
# expected: rolebinding.rbac.authorization.k8s.io/pod-reader-binding created

# 绑定 ClusterRole 到 ServiceAccount（全集群）
kubectl create clusterrolebinding cluster-pod-reader-binding \
    --clusterrole=cluster-pod-reader \
    --serviceaccount=NAMESPACE:<sa-name>
# expected: clusterrolebinding.rbac.authorization.k8s.io/cluster-pod-reader-binding created
```

**步骤 3d：YAML 声明式方式（推荐用于版本管理）**

```yaml
# role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: NAMESPACE
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-reader-binding
  namespace: NAMESPACE
subjects:
- kind: ServiceAccount
  name: <sa-name>
  namespace: NAMESPACE
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl apply -f role.yaml
# expected: role.rbac.authorization.k8s.io/pod-reader unchanged 或 created
```

## 验证

### 控制面（tccli）

```bash
tccli tke DescribeClusterStatus --region <Region> --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterState "Running"
```

```json
{
  "ClusterStatusSet": "<ClusterStatusSet>",
  "ClusterId": "<ClusterId>",
  "ClusterState": "<ClusterState>",
  "ClusterInstanceState": "<ClusterInstanceState>",
  "ClusterBMonitor": "<ClusterBMonitor>",
  "ClusterInitNodeNum": "<ClusterInitNodeNum>"
}
```

### 数据面（需 VPN/IOA）

```bash
# 验证目标 SA 已获得权限
kubectl auth can-i get pods --as=system:serviceaccount:NAMESPACE:<sa-name> -n NAMESPACE
# expected: yes

# 验证不具有不应有的权限（最小权限原则）
kubectl auth can-i delete pods --as=system:serviceaccount:NAMESPACE:<sa-name> -n NAMESPACE
# expected: no（如果确实不应有 delete 权限）

# 查看绑定关系
kubectl get rolebinding pod-reader-binding -n NAMESPACE -o yaml
# expected: 显示 subjects 和 roleRef 匹配预期
```

```text
NAME  STATUS  AGE
...
```

## 清理

排障页通常无需清理，但如有自建测试资源需删除：

```bash
kubectl delete rolebinding pod-reader-binding -n NAMESPACE
kubectl delete role pod-reader -n NAMESPACE
kubectl delete clusterrolebinding cluster-pod-reader-binding
kubectl delete clusterrole cluster-pod-reader
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl create rolebinding` 报错 "AlreadyExists" | `kubectl get rolebinding <name> -n NAMESPACE -o yaml` | 同名 RoleBinding 已存在 | 使用不同名称或 `kubectl edit rolebinding <name> -n NAMESPACE` 修改现有绑定 |
| `kubectl auth can-i` 返回 `no`，但 RoleBinding 已创建 | `kubectl describe rolebinding <name> -n NAMESPACE` 检查 subjects 和 roleRef | ServiceAccount 名称或命名空间拼写错误；Role 中 apiGroups 不匹配（如 pods 属于 `""` 核心组而非 `"apps"`） | 修正 RoleBinding subjects 中的 name 和 namespace；检查 Role rules 中 apiGroups 是否正确（核心资源: `[""]`，apps 资源: `["apps"]`) |
| `kubectl create clusterrole` 报错 "Permission denied" | `kubectl auth can-i create clusterroles` 检查当前用户权限 | 当前操作者无创建 ClusterRole 权限（非 cluster-admin） | 联系集群管理员授予创建 RBAC 资源的权限 |
| `kubectl apply -f role.yaml` 报错 "invalid apiVersion" | `kubectl api-resources \| grep roles` | 使用了错误的 apiVersion（如 `rbac.authorization.k8s.io/v1beta1` 已废弃） | 使用 `rbac.authorization.k8s.io/v1` |

### 创建成功但状态异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| SA 仍然 403 Forbidden | `kubectl auth can-i --list --as=system:serviceaccount:NAMESPACE:<sa> -n NAMESPACE` | RoleBinding 绑定到了不同命名空间的 SA；子资源权限遗漏（如 `pods/log` 不等于 `pods`） | `kubectl describe rolebinding <name> -n NAMESPACE` 确认 subjects 和 roleRef 指向正确；为子资源单独添加 rule（`pods/log`、`pods/exec` 需显式列出） |
| 权限分配过多（违反最小权限原则） | `kubectl describe role <name> -n NAMESPACE \| grep -A20 Rules` | Role 使用了通配符 `*` 或 `verbs: ["*"]` | 将 `verbs: ["*"]` 替换为具体需要的操作动词（get, list, watch, create, update, patch, delete） |
| ServiceAccount 未自动创建 | `kubectl get sa -n NAMESPACE` | 未使用 `serviceAccountName` 显式指定或命名空间下 SA 被误删 | `kubectl create sa <sa-name> -n NAMESPACE`；在 Deployment/Pod spec 中显式指定 `serviceAccountName: <sa-name>` |
| 审计日志无记录 | `kubectl get events -A --field-selector type!=Normal \| grep -i forbidden` | 集群未开启审计日志功能（TKE 托管集群默认不开启审计） | 通过 TKE 控制台开启集群审计日志：组件管理 → 开启集群审计；或依赖事件记录 `kubectl get events` |

## 下一步

- [授权腾讯云售后运维排障](../授权腾讯云售后运维排障/tccli%20操作.md) -- page_id `128647`
- [Pod 状态异常与处理措施](../Pod%20状态异常与处理措施/tccli%20操作.md) -- page_id `42945`

## 控制台替代

[容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) → 集群 → 授权管理 → RBAC → 创建 Role/ClusterRole。
