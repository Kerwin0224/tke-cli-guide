# 在 TKE 中自定义 RBAC 授权（tccli）

> 对照官方：[在 TKE 中自定义 RBAC 授权](https://cloud.tencent.com/document/product/457/51683) · page_id `51683`
## 概述

在 TKE 集群中通过 Role、ClusterRole、RoleBinding、ClusterRoleBinding 自定义 Kubernetes RBAC 授权策略。

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

### 1. 控制面：获取集群凭证

```bash
tccli tke DescribeClusterKubeconfig --region <Region> --ClusterId <ClusterId> \
    --filter "Kubeconfig" > kubeconfig.yaml
export KUBECONFIG=$(pwd)/kubeconfig.yaml
# expected: kubeconfig 写入本地文件
```

### 2. 数据面：创建命名空间级 Role（需 VPN/IOA）

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: default
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]
```

### 3. 数据面：创建 RoleBinding

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-reader-binding
  namespace: default
subjects:
  - kind: User
    name: "developer@example.com"
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### 4. 数据面：创建集群级 ClusterRole

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: secret-reader
rules:
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get", "list", "watch"]
```

### 5. 数据面：创建 ClusterRoleBinding

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: secret-reader-binding
subjects:
  - kind: Group
    name: "developers"
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl apply -f rbac.yaml
# expected: 各 RBAC 资源创建完成
```

### 6. 数据面：验证权限

```bash
kubectl auth can-i get pods --as developer@example.com
# expected: yes

kubectl auth can-i delete pods --as developer@example.com
# expected: no

kubectl auth can-i get secrets --as developer@example.com
# expected: yes（由 ClusterRoleBinding 赋予）
```

### RBAC 层级总结

| 资源 | 作用域 | 适用于 |
|------|--------|--------|
| Role | 命名空间 | 命名空间内资源授权 |
| RoleBinding | 命名空间 | 绑定 Role 到用户/组/SA |
| ClusterRole | 集群 | 集群级资源或跨命名空间 |
| ClusterRoleBinding | 集群 | 绑定 ClusterRole 到用户/组/SA |

## 验证

### 控制面

```bash
tccli tke DescribeClusterStatus --region <Region> --ClusterIds '["<ClusterId>"]'
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
kubectl get roles,rolebindings -n default
kubectl get clusterroles,clusterrolebindings | grep -E "secret-reader"
# expected: RBAC 资源已创建
```

```text
NAME  STATUS  AGE
...
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl` 连接超时/拒绝 | `tccli tke DescribeClusters` 返回集群 Running 但 kubectl 不可达 | CAM 策略 240463971 拒绝公网访问 APIServer 端点 | 通过 VPN/IOA 接入内网后执行 kubectl 命令 |
| `kubectl auth can-i` 返回 `no` | `kubectl auth can-i get pods --as <user>` 检查权限 | RoleBinding/ClusterRoleBinding 的 subjects 与 CAM 用户不匹配 | 确认 subjects 中的 name 与 CAM 用户一致 |
| `cannot get resource` 权限不足 | `kubectl describe rolebinding <name> -n <ns>` 查看 roleRef | Role 未包含所需资源或 verbs | 检查 Role rules 是否覆盖目标 resources 和 verbs |
| ClusterRole 与 CAM 冲突 | `tccli cam ListPoliciesForUser` 查看 CAM 策略 | TKE 同时支持 K8s RBAC + CAM，CAM 优先级更高 | 调整 CAM 策略或使用 CAM 侧授权 |

## 清理

```bash
kubectl delete role pod-reader -n default
kubectl delete rolebinding pod-reader-binding -n default
kubectl delete clusterrole secret-reader
kubectl delete clusterrolebinding secret-reader-binding
# expected: 资源删除成功
```

## 下一步

- [使用 kubecm 管理多集群 kubeconfig](../../集群/使用%20kubecm%20管理多集群%20kubeconfig/tccli%20操作.md) -- page_id `50659`
- [TKE Kubernetes 对象级权限控制](../../../安全和稳定性/身份验证和授权/TKE%20Kubernetes%20对象级权限控制/概述/tccli%20操作.md)

## 控制台替代

[容器服务控制台](https://console.cloud.tencent.com/tke2/cluster)
