# 自定义策略授权（tccli）

> 对照官方：[自定义策略授权](https://cloud.tencent.com/document/product/457/46106) · page_id `46106`

## 概述

在 TKE 中，除了集群级别的 CAM 权限控制之外，还可通过 Kubernetes 原生的 RBAC 机制实现命名空间级别、甚至资源级别的细粒度授权。本文介绍如何创建自定义 Role 和 ClusterRole，并通过 kubectl 将其绑定到子账号，实现对 Kubernetes 对象（Pod、Deployment 等）的精细化权限控制。

## 前置条件

- 演示集群 `cls-xxxxxxxx`（ap-guangzhou, v1.30.0, Running），kubectl 因 CAM 策略限制不可达，以下 kubectl 命令需在数据面（VPN/IOA 内网环境）执行
- 已安装 tccli 并完成 `tccli configure`
- 已安装 kubectl（Client Version >= 1.30.0）
- 当前账号具备 `tke:DescribeClusterKubeconfig` 权限
- 目标子账号 UIN `100012345678` 已通过 `GrantFullClusterAccess` 获得集群访问权限

### 环境检查

```bash
# 1. 确认集群存在且处于 Running 状态
tccli tke DescribeClusters --ClusterIds '["cls-xxxxxxxx"]' --region ap-guangzhou --output json
```

```json
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "cls-xxxxxxxx",
            "ClusterName": "demo-cluster",
            "ClusterType": "MANAGED_CLUSTER",
            "ClusterStatus": "Running",
            "ClusterVersion": "1.30.0",
            "ClusterNodeNum": 3,
            "VpcId": "vpc-xxxxxxxx",
            "CreatedTime": "2024-06-15T10:30:00Z"
        }
    ],
    "RequestId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
}
```

```bash
# 2. 获取子账号专属 kubeconfig（需确认证书 CN）
tccli tke DescribeClusterKubeconfig --ClusterId cls-xxxxxxxx --region ap-guangzhou \
    --output json | jq -r '.Kubeconfig' | base64 -d | grep "client-certificate-data"
# expected: 包含 client-certificate-data 字段
```

```json
{
  "Kubeconfig": "<Kubeconfig>",
  "RequestId": "<RequestId>"
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| YAML 创建 Role | `kubectl create -f role.yaml` | 否 |
| YAML 创建 ClusterRole | `kubectl create -f clusterrole.yaml` | 否 |
| RBAC 策略生成器绑定 | `kubectl create rolebinding` / `kubectl create clusterrolebinding` | 否 |
| 查看 Role | `kubectl get role -n default` | 是 |
| 查看 ClusterRole | `kubectl get clusterrole` | 是 |

## 操作步骤

### 步骤 1：获取子账号的 kubeconfig

每个子账号拥有独立的 x509 客户端证书，证书 CN（Common Name）即为该子账号在集群中的用户标识。

```bash
tccli tke DescribeClusterKubeconfig --ClusterId cls-xxxxxxxx --region ap-guangzhou --output json
```

```json
{
    "Kubeconfig": "YXBpVmVyc2lvbjogdjEKY2x1c3RlcnM6...",
    "RequestId": "b2c3d4e5-f6a7-8901-bcde-f12345678901"
}
```

### 步骤 2：创建命名空间级别的 Role

以下 Role 授予对 `default` 命名空间内 Pod 资源的只读权限。

> **选择依据**：Role 限定在单个 Namespace，ClusterRole 覆盖所有 Namespace。选择 Role 实现最小权限。Verbs 选择 `get`/`list`/`watch` 只读三件套，不加 `create`/`delete` 防止误操作。

`role-pod-reader.yaml`：

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
```

```bash
kubectl create -f role-pod-reader.yaml
# expected: role.rbac.authorization.k8s.io/pod-reader created
```

| 字段 | 说明 |
|------|------|
| `apiGroups: [""]` | 核心 API 组（空字符串 = core/v1） |
| `resources: ["pods"]` | 授权操作的资源类型 |
| `verbs: ["get", "list", "watch"]` | 授权操作动词：get（获取单个）、list（列出）、watch（监听变更） |

### 步骤 3：创建 RoleBinding 绑定给用户

`rolebinding-read-pods.yaml`：

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: User
  name: "100012345678"
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl create -f rolebinding-read-pods.yaml
# expected: rolebinding.rbac.authorization.k8s.io/read-pods created
```

### 步骤 4：创建集群级别的 ClusterRole

以下 ClusterRole 授予所有 Namespace 内 Pod 资源的完整操作权限（8 个动词）。

`clusterrole-pod-full-access.yaml`：

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: pod-full-access
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs:
    - create
    - delete
    - deletecollection
    - get
    - list
    - patch
    - update
    - watch
```

```bash
kubectl create -f clusterrole-pod-full-access.yaml
# expected: clusterrole.rbac.authorization.k8s.io/pod-full-access created
```

### 步骤 5：创建 ClusterRoleBinding 绑定给用户

`clusterrolebinding-pod-full-access.yaml`：

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: pod-full-access-binding
subjects:
- kind: User
  name: "100012345678"
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: pod-full-access
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl create -f clusterrolebinding-pod-full-access.yaml
# expected: clusterrolebinding.rbac.authorization.k8s.io/pod-full-access-binding created
```

## 验证

```bash
# 验证 Role 已创建
kubectl get role pod-reader -n default
# expected: NAME=pod-reader, CREATED AT 显示时间

# 验证 RoleBinding 已创建
kubectl get rolebinding read-pods -n default
# expected: NAME=read-pods, ROLE=Role/pod-reader

# 验证 ClusterRole 已创建
kubectl get clusterrole pod-full-access
# expected: NAME=pod-full-access, CREATED AT 显示时间

# 验证 ClusterRoleBinding 已创建
kubectl get clusterrolebinding pod-full-access-binding
# expected: NAME=pod-full-access-binding

# 验证子账号用户的实际权限
kubectl auth can-i list pods --as=100012345678
# expected: pod-reader 绑定后 → yes

kubectl auth can-i delete pods --as=100012345678
# expected: pod-reader 绑定后 → no。pod-full-access 绑定后 → yes
```

```text
NAME  STATUS  AGE
...
```

## 清理

> **警告**：删除 ClusterRole 可能导致依赖该角色的所有绑定失效。确认无其他用户依赖后再执行清理。

```bash
# 按创建逆序清理
kubectl delete rolebinding read-pods -n default
kubectl delete role pod-reader -n default
kubectl delete clusterrolebinding pod-full-access-binding
kubectl delete clusterrole pod-full-access
# expected: 每条返回 deleted
```

验证已删：

```bash
kubectl get rolebinding read-pods -n default
# expected: Error from server (NotFound)

kubectl get clusterrolebinding pod-full-access-binding
# expected: Error from server (NotFound)
```

```text
NAME  STATUS  AGE
...
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl auth can-i` 返回 `no` | `kubectl get rolebinding read-pods -n default -o yaml` 检查 subjects | RoleBinding/ClusterRoleBinding 中 `subjects[].name` 与 kubeconfig 证书 CN 不一致 | 确保 `name` 值与子账号证书 CN（即 UIN `100012345678`）完全一致 |
| 创建 Role 时报 `AlreadyExists` | `kubectl get role pod-reader -n default` 检查是否存在 | 同名 Role 已存在 | 使用 `kubectl apply -f` 替代 `kubectl create -f`，或先删除再创建 |
| `Error from server (Forbidden)` | `kubectl describe role pod-reader -n default` 检查 verbs 列表 | 子账号无对应操作的 verb 权限（如缺少 `create` 却尝试创建） | 在 Role YAML 中补充需要的 verb |
| `apiGroups: [""]` 不生效 | 检查 YAML 中的引号是否为英文双引号 | 使用了中文引号或智能引号 | 确保 YAML 中所有引号为英文双引号（`""`），空字符串代表核心 API 组 core/v1 |

## 下一步

- [使用预设身份授权](../使用预设身份授权/tccli%20操作.md) — page_id `46105`
- [更新子账号的 TKE 集群访问凭证](../更新子账号的%20TKE%20集群访问凭证/tccli%20操作.md) — page_id `46108`
- [Kubernetes 官方 RBAC 文档](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)

## 控制台替代

登录 [容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) → 选择集群 `cls-xxxxxxxx` → **授权管理** → **RBAC 策略生成器** → 选择子账号 → 选择自定义 Role/ClusterRole → 完成绑定。Role 和 ClusterRole 的 YAML 可通过控制台 "YAML 创建" 功能导入。
