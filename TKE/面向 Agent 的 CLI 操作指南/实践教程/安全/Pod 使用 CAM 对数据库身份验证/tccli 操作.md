# Pod 使用 CAM 对数据库身份验证（tccli）

> 对照官方：[Pod 使用 CAM 对数据库身份验证](https://cloud.tencent.com/document/product/457/81989) · page_id `81989`

## 概述

通过 TKE 工作负载身份（Workload Identity）将 CAM 角色关联到 Kubernetes ServiceAccount，使 Pod 可以通过实例元数据服务获取临时安全凭证（STS），从而安全访问腾讯云数据库（TencentDB for MySQL、Redis、MongoDB 等），避免在 Pod 中硬编码 AK/SK。

核心流程：
1. 创建 CAM 角色并配置信任策略，允许指定集群/命名空间/ServiceAccount 扮演
2. 将 CAM 角色绑定到数据库访问策略
3. 在 ServiceAccount 上添加注解关联 CAM 角色
4. Pod 通过 ServiceAccount 自动注入环境变量，应用内通过元数据服务或 SDK 获取临时凭证

> **kubectl 数据面不可达**：CAM拒绝公网端点(strategyId:240463971)，内网需VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

## 前置条件

### 环境检查

```bash
tccli --version
# expected: tccli version >= 1.0.0

tccli configure list
# expected: secretId/secretKey 已配置，region: ap-guangzhou

tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]'
# expected: TotalCount >= 1, ClusterStatus "Running"

tccli cam DescribeRoleList --region <Region> \
    --Page 1 --Rp 20
# expected: exit 0

tccli sts GetCallerIdentity --region <Region>
# expected: 返回当前账号 UIN 和主账号 UIN
```

```json
{
  "TotalCount": "<TotalCount>",
  "Clusters": "<Clusters>",
  "ClusterId": "<ClusterId>",
  "ClusterName": "<ClusterName>",
  "ClusterDescription": "<ClusterDescription>",
  "ClusterVersion": "<ClusterVersion>"
}
```

### 资源检查

```bash
# 检查目标数据库实例是否存在
tccli cdb DescribeDBInstances --region <Region> \
    --InstanceIds '["DB_INSTANCE_ID"]'
# expected: InstanceInfo 返回实例详情，Status=1（运行中）

# 检查集群是否启用工作负载身份（TKE 1.24+ 默认支持）
tccli tke DescribeClusterAuthenticationOptions --region <Region> \
    --ClusterId CLUSTER_ID
# expected: ServiceAccounts 相关配置
```

```json
{
  "ServiceAccounts": "<ServiceAccounts>",
  "UseTKEDefault": "<UseTKEDefault>",
  "Issuer": "<Issuer>",
  "JWKSURI": "<JWKSURI>",
  "AutoCreateDiscoveryAnonymousAuth": "<AutoCreateDiscoveryAnonymousAuth>",
  "LatestOperationState": "<LatestOperationState>"
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli/kubectl 命令 | 幂等 |
|-----------|-------------------|:--:|
| 创建 CAM 角色 | `tccli cam CreateRole --RoleName CAM_ROLE_NAME` | 否 |
| 查看 CAM 角色 | `tccli cam GetRole --RoleName CAM_ROLE_NAME --region <Region>` | 是 |
| 绑定数据库策略 | `tccli cam AttachRolePolicy --AttachRoleName CAM_ROLE_NAME --PolicyId POLICY_ID` | 否 |
| 解绑策略 | `tccli cam DetachRolePolicy --DetachRoleName CAM_ROLE_NAME --PolicyId POLICY_ID` | 否 |
| 创建 ServiceAccount | `kubectl create sa SA_NAME -n NAMESPACE`（需 VPN/IOA） | 是 |
| 关联 CAM 角色 | `kubectl annotate sa SA_NAME -n NAMESPACE eks.tke.cloud.tencent.com/role-name=CAM_ROLE_NAME`（需 VPN/IOA） | 是 |
| 查看 SA 注解 | `kubectl describe sa SA_NAME -n NAMESPACE`（需 VPN/IOA） | 是 |
| 获取 STS 临时凭证 | `kubectl exec POD_NAME -n NAMESPACE -- curl -s http://metadata.tencentyun.com/latest/meta-data/cam/security-credentials/CAM_ROLE_NAME`（需 VPN/IOA） | 是 |
| Pod 绑定 SA | `kubectl set sa deploy/DEPLOY_NAME SA_NAME -n NAMESPACE`（需 VPN/IOA） | 是 |

## 操作步骤

### 步骤 1：创建 CAM 角色并配置信任策略

#### 选择依据

- **工作负载身份** vs **固定 AK/SK**：工作负载身份由 TKE 元数据服务自动轮换临时凭证（默认 2 小时过期），无需管理密钥生命周期
- **角色粒度**：按数据库实例或业务域创建独立角色，遵循最小权限原则
- **信任策略**：精确限定可扮演角色的 ServiceAccount（集群 ID + 命名空间 + SA 名称）

参数说明：

| 参数 | 说明 | 示例值 |
|------|------|--------|
| RoleName | CAM 角色名称，全局唯一 | `TKE-DB-Access-Role` |
| PolicyDocument | 信任策略 JSON，声明谁可以扮演此角色 | 见下方 |
| Description | 角色描述 | `Pod 访问 TencentDB 的角色` |

#### 最小创建

```bash
cat > create-db-role.json <<'EOF'
{
    "RoleName": "TKE-DB-Access-Role",
    "PolicyDocument": "{\"version\":\"2.0\",\"statement\":[{\"effect\":\"allow\",\"principal\":{\"service\":\"tke.cloud.tencent.com\"},\"action\":\"sts:AssumeRole\"}]}",
    "Description": "Pod通过工作负载身份访问TencentDB的角色",
    "ConsoleLogin": 0
}
EOF
tccli cam CreateRole --region <Region> \
    --cli-input-json file://create-db-role.json
# expected: 返回 RoleId, RoleName
```

### 步骤 2：绑定数据库访问策略

#### 选择依据

- **预设策略** vs **自定义策略**：预设策略（如 `QcloudCDBReadOnlyAccess`）覆盖常见权限；自定义策略精确控制到实例、操作级别
- **数据库产品**：CDB（MySQL）、Redis、MongoDB 各有独立策略

参数说明：

| 参数 | 说明 | 示例值 |
|------|------|--------|
| AttachRoleName | 目标角色名 | `TKE-DB-Access-Role` |
| PolicyId | 策略 ID（预设或自定义） | `POLICY_ID` |

```bash
# 查看 CDB 相关预设策略
tccli cam ListPolicies --region <Region> \
    --Keyword "CDB" \
    --Scope "QCS"
# expected: 返回含 QcloudCDBReadOnlyAccess 等策略

# 绑定只读策略
tccli cam AttachRolePolicy --region <Region> \
    --AttachRoleName TKE-DB-Access-Role \
    --PolicyId CDB_READ_POLICY_ID
# expected: exit 0

# 验证策略绑定
tccli cam ListAttachedRolePolicies --region <Region> \
    --RoleName TKE-DB-Access-Role
# expected: List 中返回已绑定的 PolicyId
```

### 步骤 3：创建 ServiceAccount 并关联 CAM 角色

数据面（需 VPN/IOA）：

```bash
# 创建命名空间（如不存在）
kubectl create ns NAMESPACE --dry-run=client -o yaml | kubectl apply -f -
# expected: namespace/NAMESPACE created (or unchanged)

# 创建 ServiceAccount
kubectl create sa db-access-sa -n NAMESPACE
# expected: serviceaccount/db-access-sa created

# 关联 CAM 角色注解
kubectl annotate sa db-access-sa -n NAMESPACE \
    eks.tke.cloud.tencent.com/role-name=TKE-DB-Access-Role \
    --overwrite
# expected: serviceaccount/db-access-sa annotated

# 验证注解
kubectl describe sa db-access-sa -n NAMESPACE | grep -A1 Annotations
# expected: Annotations: eks.tke.cloud.tencent.com/role-name: TKE-DB-Access-Role
```

### 步骤 4：创建 Pod 使用工作负载身份

#### 选择依据

- **Deployment** vs **Pod**：生产环境使用 Deployment 管理，测试可用裸 Pod
- **环境变量注入**：元数据服务自动注入 `TENCENTCLOUD_ROLE_ARN` 等环境变量
- **SDK 支持**：推荐使用腾讯云 SDK 的 `DefaultCredentialProvider` 自动获取 STS 凭证

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-db-access
  namespace: NAMESPACE
spec:
  replicas: 1
  selector:
    matchLabels:
      app: db-access-demo
  template:
    metadata:
      labels:
        app: db-access-demo
    spec:
      serviceAccountName: db-access-sa
      containers:
      - name: app
        image: APP_IMAGE
        env:
        - name: DB_HOST
          value: "DB_INSTANCE_ID.mysql.ap-guangzhou.cdb.myqcloud.com"
        - name: DB_PORT
          value: "3306"
        - name: DB_USER
          value: "admin"
        - name: DB_NAME
          value: "app_db"
        - name: TENCENTCLOUD_ROLE_ARN
          value: "qcs::cam::uin/ACCOUNT_UIN:roleName/TKE-DB-Access-Role"
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
```

```bash
kubectl apply -f deployment-db-access.yaml
# expected: deployment.apps/app-db-access created

kubectl get pods -n NAMESPACE -l app=db-access-demo
# expected: 1 Pod Running
```

```text
NAME  STATUS  AGE
...
```

## 验证

### 控制面（tccli）

```bash
# 验证 CAM 角色创建成功
tccli cam GetRole --region <Region> \
    --RoleName TKE-DB-Access-Role
# expected: 返回 RoleId, RoleName, Description

# 验证 STS 可正常获取（调用方认证）
tccli sts AssumeRole --region <Region> \
    --RoleArn qcs::cam::uin/ACCOUNT_UIN:roleName/TKE-DB-Access-Role \
    --RoleSessionName test-session
# expected: 返回 TmpSecretId, TmpSecretKey, Token, ExpiredTime
```

### 数据面（需 VPN/IOA）

```bash
# 验证 Pod 环境变量含角色 ARN
kubectl exec -n NAMESPACE deployment/app-db-access -- env \
    | grep TENCENTCLOUD
# expected: TENCENTCLOUD_ROLE_ARN=qcs::cam::uin/...:roleName/TKE-DB-Access-Role

# 验证元数据服务返回临时凭证
POD_NAME=$(kubectl get pods -n NAMESPACE -l app=db-access-demo \
    -o jsonpath='{.items[0].metadata.name}')
kubectl exec $POD_NAME -n NAMESPACE -- \
    curl -s http://metadata.tencentyun.com/latest/meta-data/cam/security-credentials/TKE-DB-Access-Role | jq .
# expected: 返回 TmpSecretId, TmpSecretKey, Token, Expiration

# 验证数据库连接（需安装 mysql-client）
kubectl exec $POD_NAME -n NAMESPACE -- \
    mysql -h DB_INSTANCE_ID.mysql.ap-guangzhou.cdb.myqcloud.com \
    -u admin -pPASSWORD -e "SELECT 1"
# expected: +---+ \n | 1 | \n +---+
```

```text
NAME  STATUS  AGE
...
```

## 清理

### 数据面（需 VPN/IOA）

```bash
# 删除 Deployment
kubectl delete deployment app-db-access -n NAMESPACE
# expected: deployment "app-db-access" deleted

# 删除 ServiceAccount
kubectl delete sa db-access-sa -n NAMESPACE
# expected: serviceaccount "db-access-sa" deleted

# ⚠️ 警告：删除 SA 后，绑定该 SA 的 Pod 将失去 CAM 角色凭证
kubectl delete ns NAMESPACE --dry-run=client
# 如仅为此测试创建 namespace，可删除（确认无其他资源）
```

### 控制面（tccli）

```bash
# 解绑策略
tccli cam DetachRolePolicy --region <Region> \
    --DetachRoleName TKE-DB-Access-Role \
    --PolicyId CDB_READ_POLICY_ID
# expected: exit 0

# ⚠️ 警告：解绑策略后，该角色将失去数据库访问权限

# 删除角色
tccli cam DeleteRole --region <Region> \
    --RoleName TKE-DB-Access-Role
# expected: exit 0
# ⚠️ 警告：角色删除不可逆，关联此角色的所有 SA 将立即失效
```

```bash
# 验证删除
tccli cam GetRole --region <Region> --RoleName TKE-DB-Access-Role 2>&1
# expected: ResourceNotFound: role not exist
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Pod 启动后无法获取 STS 凭证 | `kubectl exec POD_NAME -n NAMESPACE -- curl -v http://metadata.tencentyun.com/latest/meta-data/cam/ 2>&1` | ServiceAccount 未注解 CAM 角色名 | `kubectl annotate sa SA_NAME -n NAMESPACE eks.tke.cloud.tencent.com/role-name=CAM_ROLE_NAME --overwrite` |
| 元数据服务返回 404 | `kubectl exec POD_NAME -n NAMESPACE -- curl -s http://metadata.tencentyun.com/latest/meta-data/cam/security-credentials/` | CAM 角色不存在或名称拼写错误 | `tccli cam GetRole --RoleName CAM_ROLE_NAME` 确认角色存在 |
| 数据库连接被拒绝 (AccessDenied) | 应用日志中搜索 `AuthFailure` 或 `AccessDenied` | CAM 角色未绑定数据库访问策略 | `tccli cam AttachRolePolicy --AttachRoleName CAM_ROLE_NAME --PolicyId POLICY_ID` |
| 元数据服务返回 InvalidParameter | `kubectl describe sa SA_NAME -n NAMESPACE` | 注解 key 拼写错误（如缺少 `eks.` 前缀） | 确认注解为 `eks.tke.cloud.tencent.com/role-name` |

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 临时凭证过期后 API 调用失败 | 应用日志 `ExpiredToken` 错误 | SDK 未自动刷新凭证（部分老旧 SDK） | 升级到最新 SDK，或实现 `CachedCredentialProvider` 在过期前自动调用 `AssumeRole` 刷新 |
| CAM CreateRole 返回 RoleNameInUse | `tccli cam GetRole --RoleName CAM_ROLE_NAME` | 角色名已存在 | 使用不同 RoleName 或先删除旧角色（确认无在用） |
| Pod 环境变量无 TENCENTCLOUD_ROLE_ARN | `kubectl exec POD_NAME -n NAMESPACE -- env \| grep TENCENT` | Pod 创建在 SA 注解之前，未触发注入 | 重建 Pod：`kubectl delete pod POD_NAME -n NAMESPACE`，Deployment 自动重建 |
| sts:AssumeRole 返回 AccessDenied | `tccli cam GetRole --RoleName CAM_ROLE_NAME` 检查信任策略 | 信任策略 `principal.service` 不是 `tke.cloud.tencent.com` | 更新 CAM 角色信任策略 `PolicyDocument` |

## 下一步

- [使用 ExternalSecretOperator 导入腾讯云 SSM 凭据](../使用%20ExternalSecretOperator%20导入腾讯云%20SSM%20凭据/tccli%20操作.md) -- page_id `102120`
- [在 TKE 中自定义 RBAC 授权](../../运维/在%20TKE%20中自定义%20RBAC%20授权/tccli%20操作.md) -- page_id `51683`
- [容器镜像签名及验证](../容器镜像签名及验证/tccli%20操作.md) -- page_id `80909`

## 控制台替代

[CAM 控制台](https://console.cloud.tencent.com/cam/role) -> 角色 -> 新建角色 -> 腾讯云产品服务 -> 容器服务（TKE）。
控制台可通过可视化界面创建角色、配置信任策略和绑定权限策略，但 ServiceAccount 注解仍需 kubectl（或通过 TKE 控制台 YAML 编辑器）。
