# UserGroupAccessControl 说明（tccli）

> 对照官方：[UserGroupAccessControl 说明](https://cloud.tencent.com/document/product/457/86977) · page_id `86977`

## 概述

UserGroupAccessControl 组件支持将 Kubernetes RBAC 权限管理与腾讯云 CAM 用户组打通，方便对子账号进行细粒度的访问控制。通过将具备相同功能的用户（子账号）放入 CAM 用户组，可实现快速为同一角色的子账号设置相同的 Kubernetes 对象访问权限。

**部署在集群内的 Kubernetes 对象：**

| Kubernetes 对象名称 | 类型 | 资源占用 | 所属 Namespaces |
|---|---|---|---|
| user-group-access-control | ServiceAccount | - | kube-system |
| user-group-access-control | ClusterRole | - | - |
| user-group-access-control | ClusterRoleBinding | - | - |
| user-group-access-control | Service | - | kube-system |
| user-group-access-control | ConfigMap | - | kube-system |
| user-group-access-control | Deployment | 0.5C1G（新建） | kube-system |

**使用限制：**

- 仅支持 TKE 托管集群
- Kubernetes 集群版本 ≥ 1.16
- 需通过工单申请开通

## 前置条件

- [环境准备](../../../../环境准备.md)
- 已 [连接集群](../../../../集群配置/集群管理/连接集群/tccli 操作.md)，kubectl 可正常访问 APIServer
- 已创建 CAM 用户组（如无，需先在 [访问管理控制台](https://console.cloud.tencent.com/cam) 创建）
- 已通过工单申请开通 UserGroupAccessControl 功能

确认集群状态正常：

```bash
tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'
```

```json
{
  "TotalCount": 0,
  "Clusters": [],
  "ClusterId": "<ClusterId>",
  "ClusterName": "<ClusterName>",
  "ClusterDescription": "<ClusterDescription>",
  "ClusterVersion": "<ClusterVersion>",
  "ClusterOs": "<ClusterOs>",
  "ClusterType": "<ClusterType>"
}
```

**CAM 权限要求：**

| 操作 | 所需权限 |
|------|---------|
| 安装组件 | `tke:InstallAddon` |
| 查看组件状态 | `tke:DescribeAddon` |
| 查看组件配置 | `tke:DescribeAddonValues` |
| 更新组件 | `tke:UpdateAddon` |
| 卸载组件 | `tke:DeleteAddon` |

## 控制台与 CLI 参数映射

| Console 操作 | tccli 命令 | 幂等 |
|---|---|---|
| 查看集群列表 | `tccli tke DescribeClusters` | 是 |
| 查看组件详情 | `tccli tke DescribeAddon --AddonName UserGroupAccessControl` | 是 |
| 组件管理 → 新建 → 勾选 UserGroupAccessControl | `tccli tke InstallAddon --AddonName UserGroupAccessControl --AddonVersion <AddonVersion>` | 是（已安装时报错 ResourceAlreadyExists） |
| 更新组件 | `tccli tke UpdateAddon --AddonName UserGroupAccessControl` | 否 |
| 组件管理 → 删除组件 | `tccli tke DeleteAddon --AddonName UserGroupAccessControl` | 是（已删除时返回 NotFound） |

## 操作步骤

### 步骤 1：通过工单申请开通

使用 UserGroupAccessControl 组件需通过控制台提交工单申请开通。

### 步骤 2：创建 CAM 用户组（如无）

在 CAM 控制台创建用户组，并将需要相同权限的子账号加入该组。如已有用户组可跳过此步。记录用户组 ID（`<UserGroupId>`）。

参考文档：新建用户组

### 步骤 3：安装组件（tccli）

安装前需完成**服务授权**：TKE 需获取读取您账户下用户组信息的权限，必须完成指引授权，将角色 `TKE_QCSRole` 与预设策略 `QcloudAccessForTKERoleInGroupsForUser` 关联。

```bash
tccli tke InstallAddon \
    --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName UserGroupAccessControl \
    --AddonVersion <AddonVersion>
```

输出示例：

```json
{
    "Response": {
        "RequestId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
    }
}
```

> **注意：** 首次安装需在控制台完成服务授权（角色 TKE_QCSRole 关联策略 QcloudAccessForTKERoleInGroupsForUser）。如已授权可跳过。

### 步骤 4：为用户组创建 ClusterRole 并绑定策略（数据面）

> **注意**：kubectl 数据面当前因公网端点被 CAM 策略 strategyId:240463971 拒绝而不可达。以下 kubectl 命令为参考格式，需在内网/VPN 环境执行。

创建绑定到 CAM 用户组的 ClusterRole：

```bash
cat > ugac-clusterrole.yaml << 'EOF'
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: ugac-readonly
  annotations:
    rbac.tencentcloud.com/usergroup-id: "<UserGroupId>"
rules:
- apiGroups: [""]
  resources: ["pods", "services", "configmaps"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch"]
EOF

kubectl apply -f ugac-clusterrole.yaml
```

```text
# command executed successfully
```

创建对应的 ClusterRoleBinding：

```bash
cat > ugac-clusterrolebinding.yaml << 'EOF'
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: ugac-readonly-binding
  annotations:
    rbac.tencentcloud.com/usergroup-id: "<UserGroupId>"
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: ugac-readonly
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: User
  name: "*"
EOF

kubectl apply -f ugac-clusterrolebinding.yaml
```

```text
# command executed successfully
```

> **注意：** ClusterRoleBinding 名称前缀为 CAM 用户组 ID。在控制台的 **授权管理 > ClusterRoleBinding** 中可查看以用户组 ID 为前缀的权限策略。

### 步骤 5：查看角色绑定策略

```bash
kubectl get clusterrolebinding | grep <UserGroupId>
```

输出示例：

```text
<UserGroupId>-ugac-readonly-binding   ClusterRole/ugac-r

```json
{
  "Addons": [],
  "AddonName": "<AddonName>",
  "AddonVersion": "<AddonVersion>",
  "RawValues": "<RawValues>",
  "Phase": "<Phase>",
  "Reason": "<Reason>",
  "CreateTime": "<CreateTime>",
  "RequestId": "<RequestId>"
}
```eadonly   5m
```

> **权限变更说明：** 如需要调整腾讯云资源操作权限（例如迁移组内子账号、增删云产品操作权限），只需在 CAM 侧修改用户组即可，更改立即在已创建的角色绑定策略中生效。

## 验证

### Control plane（tccli）

```bash
tccli tke DescribeAddon \
    --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName UserGroupAccessControl
```

**预期输出：** 组件状态为 `Running`，AddonName 为 `UserGroupAccessControl`。

### Data plane（kubectl）

> **注意**：kubectl 数据面当前因公网端点被 CAM 策略 strategyId:240463971 拒绝而不可达。以下 kubectl 命令为参考格式，需在内网/VPN 环境执行。

检查组件 Deployment 状态：

```bash
kubectl get deployment user-group-access-control \
    -n kube-system
```

**预期输出：** `READY` = `1/1`。

检查角色绑定策略：

```bash
kubectl get clusterrolebinding | grep <UserGroupId>
```

## 清理

> **计费提醒：** UserGroupAccessControl 组件本身不产生额外费用。卸载后 CAM 用户组中的子账号将失去基于用户组的 Kubernetes RBAC 权限，请确认不影响业务后再执行。

### Data plane（kubectl）

> **注意**：kubectl 数据面当前因公网端点被 CAM 策略 strategyId:240463971 拒绝而不可达。以下 kubectl 命令为参考格式，需在内网/VPN 环境执行。

删除 ClusterRoleBinding 和 ClusterRole：

```bash
kubectl delete clusterrolebinding <UserGroupId>-ugac-readonly-binding
kubectl delete clusterrole ugac-readonly
```

### Control plane（tccli）

卸载组件：

```bash
tccli tke DeleteAddon \
    --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName UserGroupAccessControl
```

验证已卸载：

```bash
tccli tke DescribeAddon \
    --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName UserGroupAccessControl
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 安装时提示服务授权未完成 | 检查 CAM 控制台角色 TKE_QCSRole 是否关联策略 | 首次安装需关联 TKE_QCSRole 角色 | 在控制台新建组件页面完成服务授权，或通过 CAM 控制台为角色 TKE_QCSRole 添加策略 QcloudAccessForTKERoleInGroupsForUser |
| 子账号无法访问集群资源 | 检查 CAM 用户组成员（CAM 控制台 > 用户组 > 成员管理）；`kubectl describe clusterrolebinding <UserGroupId>-*` 检查绑定 | CAM 用户组未包含该子账号，或 ClusterRoleBinding 未正确绑定 | 将子账号加入 CAM 用户组；确认 ClusterRoleBinding 正确绑定 |
| 未申请工单时组件不可用 | 检查是否已通过工单申请开通 | 功能需通过工单申请开通 | 通过控制台提交工单申请开通 UserGroupAccessControl |
| 独立集群中组件不生效 | `kubectl get nodes` 检查节点是否包含 `node-role.kubernetes.io/master` 标签 | 仅支持托管集群，独立集群不支持 | 确认集群类型为托管集群 |
| `Unable to connect to the server` | `kubectl cluster-info` 检查连通性 | 公网端点被 CAM 策略 strategyId:240463971 以 `tke:clusterExtranetEndpoint=true` 条件拒绝 | 通过 IOA/VPN 连接内网，或使用同 VPC CVM 执行 kubectl 命令 |

## 下一步

- [为用户组添加/移除用户](https://cloud.tencent.com/document/product/598/xxxxx) — 管理 CAM 用户组成员
- [Kubernetes RBAC 配置](https://cloud.tencent.com/document/product/457/xxxxx) — TKE RBAC 权限管理
- [新建用户组](https://cloud.tencent.com/document/product/598/xxxxx) — CAM 用户组创建指南

## 控制台替代

[组件管理 - 新建组件](https://console.cloud.tencent.com/tke2/cluster)
