# 自定义策略授权（tccli）

> 对照官方：[自定义策略授权（Kubernetes 对象级）](https://cloud.tencent.com/document/product/457/46106) · page_id `46106`

## 概述

在 TKE 预设身份（`tke:admin` 等）之外，可通过 Kubernetes 原生 RBAC 机制实现更细粒度的权限控制。自定义 Role/ClusterRole 配合 RoleBinding/ClusterRoleBinding，可精确控制子账号对特定 Namespace、资源类型和操作动词的权限。

本页所有 kubectl 操作均需端点可达。子账号的用户标识为其 x509 证书 CN（即 UIN），在 RoleBinding/ClusterRoleBinding 的 `subjects[].name` 中使用。

本页 YAML 创建操作需 kubectl 访问集群 APIServer。演示集群 `cls-xxxxxxxx`（Running，v1.30.0，ap-guangzhou）端点不可达，kubectl 创建步骤无法在本集群执行；本页仅通过 tccli 完成控制面凭证获取。演示子账号 UIN `100012345679`。

## 前置条件

- 已完成 [环境准备](../../../../../../环境准备.md)，tccli 已安装并配置凭据。
- 演示集群 `cls-xxxxxxxx`（Running，v1.30.0，ap-guangzhou），kubectl 不可达——本页 YAML 创建步骤需在端点可达的集群执行；tccli 获取 kubeconfig 步骤可在本集群完成。
- 主账号 UIN `100012345678`，目标子账号 UIN `100012345679`（已通过 `GrantFullClusterAccess` 授权）。
- CAM 调用账号具备以下 Action 权限：`tke:DescribeClusterKubeconfig`。

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置
```

### 资源检查

```bash
# 3. 确认 kubeconfig 包含 x509 证书（新模式确认）
tccli tke DescribeClusterKubeconfig --ClusterId cls-xxxxxxxx --region ap-guangzhou \
    --output json | jq -r '.Kubeconfig' | base64 -d \
    | grep -c "client-certificate-data"
# expected: 1
```

```json
{
  "Kubeconfig": "<Kubeconfig>",
  "RequestId": "<RequestId>"
}
```

> 以下 kubectl 命令需端点可达，演示集群 `cls-xxxxxxxx` 不可达，请在端点可达的集群执行：

```bash
# 4. 确认目标 Namespace 存在（需端点可达）
kubectl get ns default
# expected: NAME=default，STATUS=Active
```

```text
NAME  STATUS  AGE
...
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli/kubectl 命令 | 幂等 |
|-----------|-------------------|:--:|
| 获取子账号 kubeconfig | `tccli tke DescribeClusterKubeconfig --ClusterId cls-xxxxxxxx --region ap-guangzhou` | 是 |
| YAML 创建 Role | `kubectl create -f role.yaml`（需端点可达） | 否 |
| YAML 创建 ClusterRole | `kubectl create -f clusterrole.yaml`（需端点可达） | 否 |
| RBAC 策略生成器绑定 | `kubectl create rolebinding` / `kubectl create clusterrolebinding`（需端点可达） | 否 |
| 查看 Role | `kubectl get role -n NAMESPACE`（需端点可达） | 是 |
| 查看 ClusterRole | `kubectl get clusterrole`（需端点可达） | 是 |
| 确认子账号权限 | `kubectl auth can-i --as 100012345679`（需端点可达） | 是 |

## 操作步骤

### 步骤 1：获取子账号 kubeconfig 和证书 CN

TKE 为每个子账号签发独立 x509 证书，证书 CN（Common Name）即子账号 UIN，作为集群中该用户的标识。

```bash
tccli tke DescribeClusterKubeconfig --ClusterId cls-xxxxxxxx --region ap-guangzhou
# expected: exit 0，返回 base64 编码的 Kubeconfig
```

```json
{
    "Kubeconfig": "YXBpVmVyc2lvbjogdjEKY2x1c3RlcnM6...",
    "RequestId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
}
```

解码后 kubeconfig 中 `users[].user.client-certificate-data` 和 `client-key-data` 即为该子账号的专属凭证。证书 CN 格式为纯数字 UIN，如 `100012345679`。

### 步骤 2：创建命名空间级别的 Role

> 以下 kubectl 操作需端点可达，演示集群 `cls-xxxxxxxx` 不可达，请在端点可达的集群执行。

以下 Role 授予对 `default` 命名空间内 Pod 资源的只读权限。

#### 选择依据

- **Role vs ClusterRole**：Role 限定在单个 Namespace，ClusterRole 覆盖所有 Namespace。按最小权限原则选择 Role（除非需要跨 Namespace 授权）。
- **Verbs 选择**：`get`、`list`、`watch` 为只读三件套。不包括 `create`、`delete` 防止误操作。需要写入权限时按需添加具体 verb。
- **apiGroups**：核心 API 组用空字符串 `""`；apps 组用 `"apps"`；networking.k8s.io 用 `"networking.k8s.io"`。

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

| 字段 | 含义 |
|------|------|
| `apiGroups: [""]` | 核心 API 组（空字符串 = core/v1） |
| `resources: ["pods"]` | 授权操作的资源类型 |
| `verbs: ["get", "list", "watch"]` | 授权操作动词 |

### 步骤 3：创建 RoleBinding 绑定给用户

将 `pod-reader` Role 绑定到子账号用户（证书 CN = `100012345679`）：

`rolebinding-read-pods.yaml`：

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: User
  name: "100012345679"
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

以下 ClusterRole 授予所有 Namespace 内 Pod 资源的完整操作权限。

#### Verb 说明表

| 动词 | 含义 | 适用场景 |
|------|------|---------|
| `create` | 创建资源 | 部署新 Pod/Deployment |
| `delete` | 删除单个资源 | 清理指定 Pod |
| `deletecollection` | 删除资源集合 | 批量清理（如 `kubectl delete pods --all`） |
| `get` | 获取单个资源 | 查看指定 Pod 详情 |
| `list` | 列出资源 | 查看 Pod 列表 |
| `patch` | 局部更新资源 | 修改标签、注解等 |
| `update` | 完整替换资源 | 更新资源完整规格 |
| `watch` | 监听资源变更 | 监控 Pod 状态变化 |

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
  name: "100012345679"
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

### 控制面（tccli）

```bash
# 确认 kubeconfig 可用
tccli tke DescribeClusterKubeconfig --ClusterId cls-xxxxxxxx --region ap-guangzhou
# expected: exit 0，返回 Kubeconfig
```

```json
{
    "Kubeconfig": "YXBpVmVyc2lvbjogdjEKY2x1c3RlcnM6...",
    "RequestId": "a7b8c9d0-e1f2-3456-abcd-4567890abcdef"
}
```

### 数据面（kubectl）

> 需端点可达，演示集群 `cls-xxxxxxxx` 不可达，请在端点可达的集群执行：

```bash
# 验证 Role 已创建（需端点可达）
kubectl get role pod-reader -n default
# expected: NAME=pod-reader

# 验证 RoleBinding 已创建
kubectl get rolebinding read-pods -n default
# expected: NAME=read-pods，ROLE=Role/pod-reader

# 验证 ClusterRole 已创建
kubectl get clusterrole pod-full-access
# expected: NAME=pod-full-access

# 验证 ClusterRoleBinding 已创建
kubectl get clusterrolebinding pod-full-access-binding
# expected: NAME=pod-full-access-binding

# 验证子账号用户的实际权限（模拟子账号身份检查）
kubectl auth can-i list pods --as 100012345679
# expected: pod-reader 绑定后 → yes

kubectl auth can-i delete pods --as 100012345679
# expected: pod-reader 绑定后 → no；pod-full-access 绑定后 → yes
```

```text
NAME  STATUS  AGE
...
```

## 清理

> **警告**：删除 ClusterRole 可能导致依赖该角色的所有绑定失效。确认无其他用户依赖后再执行清理。以下操作需端点可达。

### 数据面（kubectl）

```bash
# 按创建逆序清理（需端点可达）
kubectl delete rolebinding read-pods -n default
# expected: rolebinding.rbac.authorization.k8s.io "read-pods" deleted

kubectl delete role pod-reader -n default
# expected: role.rbac.authorization.k8s.io "pod-reader" deleted

kubectl delete clusterrolebinding pod-full-access-binding
# expected: clusterrolebinding.rbac.authorization.k8s.io "pod-full-access-binding" deleted

kubectl delete clusterrole pod-full-access
# expected: clusterrole.rbac.authorization.k8s.io "pod-full-access" deleted
```

### 验证已删

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

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl auth can-i` 返回 `no` | 检查 RBAC 绑定中的 subjects：`kubectl get rolebinding read-pods -n default -o yaml \| grep -A3 subjects` | RoleBinding/ClusterRoleBinding 中 `subjects[].name` 与 kubeconfig 证书 CN 不一致 | 确保 `name` 值与子账号证书 CN（即 `100012345679`）完全一致 |
| `Error from server (Forbidden)` | 检查 Role 的 verbs 列表：`kubectl describe role pod-reader -n default` | 当前子账号缺少目标操作的 verb 权限 | 在 Role YAML 的 `verbs` 中补充需要的动词 |
| 创建 Role 时报 `AlreadyExists` | 检查是否存在同名 Role：`kubectl get role pod-reader -n default` | 同名 Role 已存在 | 使用 `kubectl apply -f` 替代 `kubectl create -f`，或先删除再创建 |
| kubectl 无法连接 APIServer | `tccli tke DescribeClusterSecurity --ClusterId cls-xxxxxxxx --region ap-guangzhou` 检查 PgwEndpoint | 集群 APIServer 端点不可达（如演示集群 cls-xxxxxxxx） | 在端点可达的集群执行 kubectl；或通过控制台内嵌 Cloud Shell |

### 配置不生效

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `apiGroups: [""]` 不生效 | 检查 YAML 中的引号是否为英文双引号 | 使用了中文引号或智能引号 | 确保 YAML 中所有引号为英文半角双引号（`""`），空字符串代表核心 API 组 core/v1 |
| 绑定后子账号仍无权限 | 检查当前 kubectl 身份：`kubectl auth whoami` 或 `kubectl config current-context` | 当前 kubeconfig 可能属于其他子账号 | 确认使用的 kubeconfig 是目标子账号的专属文件（`~/.kube/config-cls-xxxxxxxx`） |

## 下一步

- [使用预设身份授权](../使用预设身份授权/tccli%20操作.md) — GrantFullClusterAccess 快速授权
- [更新子账号的 TKE 集群访问凭证](../更新子账号的%20TKE%20集群访问凭证/tccli%20操作.md) — 凭证轮换操作
- [Kubernetes 官方 RBAC 文档](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) — 完整 RBAC 参考

## 控制台替代

[TKE 控制台 → 集群详情 → 授权管理](https://console.cloud.tencent.com/tke2/cluster) → **RBAC 策略生成器** → 选择子账号 → 选择自定义 Role/ClusterRole → 完成绑定。Role 和 ClusterRole 的 YAML 可通过控制台 **YAML 创建** 功能导入。
