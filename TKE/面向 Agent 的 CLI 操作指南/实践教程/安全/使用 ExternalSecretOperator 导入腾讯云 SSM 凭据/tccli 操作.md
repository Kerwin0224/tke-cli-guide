# 使用 ExternalSecretOperator 导入腾讯云 SSM 凭据（tccli）

> 对照官方：[使用 ExternalSecretOperator 导入腾讯云 SSM 凭据](https://cloud.tencent.com/document/product/457/102120) · page_id `102120`

## 概述

通过 External Secrets Operator（ESO）将腾讯云凭据管理系统（SSM）中存储的凭据（数据库密码、API 密钥、TLS 证书等）自动同步到 Kubernetes Secret，实现凭据的统一管理、自动轮换和审计。

核心能力：
- **一次配置，自动同步**：SSM 凭据更新后，ESO 按 refreshInterval 自动同步到 K8s Secret
- **支持版本管理**：可指定凭据版本号，实现灰度切换和回滚
- **多命名空间隔离**：通过 SecretStore（命名空间级）和 ClusterSecretStore（集群级）控制凭据可见范围
- **CAM 认证集成**：支持通过 AK/SK 或 CAM 角色（IRSA/工作负载身份）认证 SSM API

> **kubectl 数据面不可达**：CAM拒绝公网端点(strategyId:240463971)，内网需VPN/IOA。以下 kubectl/helm 命令为参考格式，需在内网/VPN 环境下执行。

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

tccli ssm ListSecrets --region <Region> \
    --Offset 0 --Limit 20
# expected: SecretList 返回凭据列表

helm version --short
# expected: Helm >= v3.8.0
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
# 验证 ESO 未安装（或检查已有版本）
kubectl get ns external-secrets-system 2>/dev/null || echo "not installed"
# expected: "not installed"（首次安装时）

# 验证 CAM 权限
tccli cam DescribeRoleList --region <Region> --Page 1 --Rp 20
# expected: exit 0

# 验证 SSM 可用
tccli ssm DescribeRegions --region <Region> | jq '.Regions[] | select(.Region=="REGION")'
# expected: Region, RegionName 信息
```

```text
NAME  STATUS  AGE
...
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli/kubectl/helm 命令 | 幂等 |
|-----------|------------------------|:--:|
| 创建 SSM 凭据 | `tccli ssm CreateSecret --region <Region>` | 否 |
| 查看凭据列表 | `tccli ssm ListSecrets --region <Region>` | 是 |
| 获取凭据值 | `tccli ssm GetSecretValue --region <Region> --SecretName SECRET_NAME --VersionId VERSION_ID` | 是 |
| 更新凭据值 | `tccli ssm PutSecretValue --region <Region> --SecretName SECRET_NAME --SecretString NEW_VALUE` | 否 |
| 删除凭据 | `tccli ssm DeleteSecret --region <Region> --SecretName SECRET_NAME` | 否 |
| 安装 ESO | `helm install external-secrets external-secrets/external-secrets --namespace external-secrets-system --create-namespace`（需 VPN/IOA） | 否 |
| 创建 SecretStore | `kubectl apply -f secretstore.yaml`（需 VPN/IOA） | 是 |
| 创建 ClusterSecretStore | `kubectl apply -f clustersecretstore.yaml`（需 VPN/IOA） | 是 |
| 创建 ExternalSecret | `kubectl apply -f externalsecret.yaml`（需 VPN/IOA） | 是 |
| 查看同步状态 | `kubectl get externalsecret ES_NAME -n NAMESPACE`（需 VPN/IOA） | 是 |
| 强制同步 | `kubectl annotate es ES_NAME -n NAMESPACE force-sync=$(date +%s) --overwrite`（需 VPN/IOA） | 是 |

## 操作步骤

### 步骤 1：在 SSM 中创建凭据

#### 选择依据

- **凭据类型**：`SecretString`（键值对 JSON）适合数据库密码；`SecretBinary` 适合证书文件
- **凭据版本**：每次更新凭据值会生成新版本号，ESO 可按版本同步
- **加密方式**：SSM 默认使用腾讯云 KMS 加密，也可指定自定义 KMS 密钥

参数说明：

| 参数 | 说明 | 示例值 |
|------|------|--------|
| SecretName | SSM 凭据名称，64 字符内 | `prod-db-password` |
| VersionId | 版本号 | `v1` |
| SecretString | 凭据值，JSON 字符串 | `{"username":"admin","password":"SECRET_VALUE"}` |
| Description | 凭据描述 | `生产数据库密码` |

#### 最小创建

```bash
cat > create-ssm-secret.json <<'EOF'
{
    "SecretName": "prod-db-password",
    "VersionId": "v1",
    "SecretString": "{\"username\":\"admin\",\"password\":\"SECRET_VALUE\",\"host\":\"DB_HOST\",\"port\":\"3306\"}",
    "Description": "生产数据库连接凭据"
}
EOF
tccli ssm CreateSecret --region <Region> \
    --cli-input-json file://create-ssm-secret.json
# expected: 返回 SecretName: prod-db-password, VersionId: v1
```

#### 增强配置

```bash
# 创建多个凭据
tccli ssm CreateSecret --region <Region> \
    --SecretName "prod-api-key" \
    --VersionId "v1" \
    --SecretString "{\"api_key\":\"API_KEY_VALUE\",\"endpoint\":\"https://api.example.com\"}" \
    --Description "生产API密钥"
# expected: SecretName: prod-api-key

# 更新凭据版本（模拟密码轮换）
tccli ssm PutSecretValue --region <Region> \
    --SecretName "prod-db-password" \
    --VersionId "v2" \
    --SecretString "{\"username\":\"admin\",\"password\":\"NEW_SECRET_VALUE\",\"host\":\"DB_HOST\",\"port\":\"3306\"}"
# expected: 返回新 VersionId: v2

# 查看凭据所有版本
tccli ssm DescribeSecret --region <Region> \
    --SecretName prod-db-password
# expected: VersionIds 列表，含 v1, v2
```

### 步骤 2：安装 External Secrets Operator

数据面（需 VPN/IOA）：

```bash
helm repo add external-secrets https://charts.external-secrets.io
helm repo update
# expected: external-secrets 仓库更新成功

helm install external-secrets external-secrets/external-secrets \
    --namespace external-secrets-system \
    --create-namespace \
    --set installCRDs=true \
    --set serviceAccount.create=true
# expected: NAME: external-secrets, STATUS: deployed

kubectl get pods -n external-secrets-system
# expected: 3 个 Pod Running（cert-controller, webhook, external-secrets）
```

```text
NAME  STATUS  AGE
...
```

### 步骤 3：配置 SecretStore 或 ClusterSecretStore

#### 选择依据

- **SecretStore** vs **ClusterSecretStore**：SecretStore 仅在指定命名空间可用，适合隔离敏感凭据；ClusterSecretStore 全集群可用，适合公共凭据
- **认证方式**：AK/SK（直接配置在 Secret 中）、CAM 角色（通过 IRSA 或工作负载身份，生产推荐）

```bash
# 创建包含 AK/SK 的 Secret（用于 SSM 认证）
kubectl create secret generic tc-credentials -n NAMESPACE \
    --from-literal=secret-id=AKID_PLACEHOLDER \
    --from-literal=secret-key=SECRET_KEY_PLACEHOLDER
# expected: secret/tc-credentials created
```

SecretStore YAML（命名空间级）：

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: tencent-ssm-store
  namespace: NAMESPACE
spec:
  provider:
    tencentcloudsm:
      region: REGION
      secretId:
        name: tc-credentials
        key: secret-id
      secretKey:
        name: tc-credentials
        key: secret-key
```

ClusterSecretStore YAML（集群级）：

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: tencent-ssm-cluster
spec:
  provider:
    tencentcloudsm:
      region: REGION
      secretId:
        name: tc-credentials
        key: secret-id
        namespace: NAMESPACE
      secretKey:
        name: tc-credentials
        key: secret-key
        namespace: NAMESPACE
```

```bash
kubectl apply -f secretstore.yaml
# expected: secretstore.external-secrets.io/tencent-ssm-store created

kubectl get secretstore -n NAMESPACE
# expected: STATUS Valid, READY True
```

### 步骤 4：创建 ExternalSecret 同步凭据

#### 选择依据

- **refreshInterval**：凭据同步间隔，建议 1h（生产）或 5m（测试）
- **remoteRef.property**：从 JSON 凭据中提取特定字段（如 `password`）
- **target.creationPolicy**：`Owner`（ESO 管理生命周期）、`Orphan`（仅创建不删除）

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-password-es
  namespace: NAMESPACE
spec:
  refreshInterval: "1h"
  secretStoreRef:
    name: tencent-ssm-store
    kind: SecretStore
  target:
    name: db-password-k8s
    creationPolicy: Owner
  data:
  - secretKey: username
    remoteRef:
      key: prod-db-password
      property: username
      version: "v1"
  - secretKey: password
    remoteRef:
      key: prod-db-password
      property: password
      version: "v1"
  - secretKey: host
    remoteRef:
      key: prod-db-password
      property: host
      version: "v1"
  - secretKey: port
    remoteRef:
      key: prod-db-password
      property: port
      version: "v1"
```

```bash
kubectl apply -f externalsecret.yaml
# expected: externalsecret.external-secrets.io/db-password-es created

# 查看同步状态
kubectl get externalsecret -n NAMESPACE
# expected: STATUS SecretSynced, READY True

# 查看生成的 K8s Secret
kubectl get secret db-password-k8s -n NAMESPACE -o yaml
# expected: data 包含 username, password, host, port
```

```text
NAME  STATUS  AGE
...
```

## 验证

### 控制面（tccli）

```bash
# 验证 SSM 凭据值
tccli ssm GetSecretValue --region <Region> \
    --SecretName prod-db-password \
    --VersionId v1 | jq -r '.SecretString' | jq .
# expected: {"username":"admin","password":"SECRET_VALUE","host":"DB_HOST","port":"3306"}

# 验证版本列表
tccli ssm DescribeSecret --region <Region> \
    --SecretName prod-db-password \
    | jq '.VersionIds'
# expected: ["v1", "v2"]
```

### 数据面（需 VPN/IOA）

```bash
# 验证 ESO 同步状态
kubectl get externalsecret db-password-es -n NAMESPACE
# expected: STORE tencent-ssm-store, REFRESH INTERVAL 1h, STATUS SecretSynced

# 解码并验证 K8s Secret 值
kubectl get secret db-password-k8s -n NAMESPACE \
    -o jsonpath='{.data.username}' | base64 -d && echo
# expected: admin

kubectl get secret db-password-k8s -n NAMESPACE \
    -o jsonpath='{.data.password}' | base64 -d && echo
# expected: SECRET_VALUE

# 手动触发同步并验证
kubectl annotate externalsecret db-password-es -n NAMESPACE \
    force-sync=$(date +%s) --overwrite
# expected: externalsecret annotated

kubectl get externalsecret db-password-es -n NAMESPACE -o jsonpath='{.status.refreshTime}'
# expected: 最新同步时间戳

# 验证 ESO Pod 日志无错误
kubectl logs -n external-secrets-system -l app.kubernetes.io/name=external-secrets --tail=20 \
    | grep -i error
# expected: （无输出或仅有 info 日志）
```

```text
NAME  STATUS  AGE
...
```

## 清理

### 数据面（需 VPN/IOA）

```bash
# 删除 ExternalSecret（同时删除关联的 K8s Secret，因 creationPolicy: Owner）
kubectl delete externalsecret db-password-es -n NAMESPACE
# expected: externalsecret "db-password-es" deleted
# ⚠️ 警告：creationPolicy=Owner 时删除 ExternalSecret 会级联删除目标 Secret

# 确认 Secret 已删除
kubectl get secret db-password-k8s -n NAMESPACE 2>&1
# expected: Error from server (NotFound): secrets "db-password-k8s" not found

# 删除 SecretStore
kubectl delete secretstore tencent-ssm-store -n NAMESPACE
# expected: secretstore "tencent-ssm-store" deleted

# 删除 AK/SK Secret
kubectl delete secret tc-credentials -n NAMESPACE
# expected: secret "tc-credentials" deleted

# 卸载 ESO
helm uninstall external-secrets -n external-secrets-system
# expected: release "external-secrets" uninstalled

kubectl delete ns external-secrets-system
# ⚠️ 警告：删除命名空间会清除所有 ESO 配置和 CRD 实例
```

```text
NAME  STATUS  AGE
...
```

### 控制面（tccli）

```bash
# 删除 SSM 凭据
tccli ssm DeleteSecret --region <Region> \
    --SecretName prod-db-password
# expected: exit 0
# ⚠️ 警告：删除 SSM 凭据不可逆，关联的 K8s ExternalSecret 将无法同步

tccli ssm DeleteSecret --region <Region> \
    --SecretName prod-api-key
# expected: exit 0
```

```bash
# 验证删除
tccli ssm DescribeSecret --region <Region> --SecretName prod-db-password 2>&1
# expected: ResourceNotFound: secret not found
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| ExternalSecret STATUS SecretSyncedError | `kubectl describe externalsecret ES_NAME -n NAMESPACE` 查看 Events | AK/SK 无 SSM 读取权限或 Secret 不存在 | 确认 AK/SK 权限包含 `ssm:GetSecretValue`，`tccli ssm GetSecretValue --SecretName SECRET_NAME` 测试 |
| ExternalSecret 显示 SecretAlreadyExists | `kubectl get secret TARGET_NAME -n NAMESPACE` | 目标 Secret 已存在且 creationPolicy=Owner 或未设置 | 删除已有 Secret 或设置 `creationPolicy: Orphan` |
| Secret 未自动轮换 | `kubectl get externalsecret ES_NAME -n NAMESPACE -o jsonpath='{.status.refreshTime}'` | refreshInterval 未到或 ESO controller 异常 | 手动触发 `kubectl annotate es ES_NAME force-sync=$(date +%s) --overwrite` |
| SecretStore STATUS Invalid | `kubectl describe secretstore STORE_NAME -n NAMESPACE` | AK/SK Secret 不存在或引用 key 名不匹配 | `kubectl get secret tc-credentials -n NAMESPACE` 确认 Secret 存在且 key 名正确 |

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| ESO Webhook 启动失败 | `kubectl get pods -n external-secrets-system` 和 `kubectl logs` | CRD 未安装或 cert-manager 证书未就绪 | `helm install external-secrets --set installCRDs=true`，或等待 cert-manager 签发证书 |
| SSM PutSecretValue 返回 InvalidParameter | `tccli ssm DescribeSecret --SecretName SECRET_NAME` | SecretName 不存在 | 先执行 `tccli ssm CreateSecret` 创建凭据再更新 |
| remoteRef.property 返回空值 | `tccli ssm GetSecretValue --SecretName SECRET_NAME --VersionId VERSION_ID` 解析 JSON | SecretString 不包含该属性名 | 检查 JSON key 拼写是否与 remoteRef.property 一致 |
| ClusterSecretStore 跨命名空间失败 | `kubectl describe clustersecretstore STORE_NAME` | AK/SK Secret 引用的 namespace 不存在或无权限 | 创建 tc-credentials Secret 在指定 namespace，或改用 SecretStore |

## 下一步

- [Pod 使用 CAM 对数据库身份验证](../Pod%20使用%20CAM%20对数据库身份验证/tccli%20操作.md) -- page_id `81989`
- [容器镜像签名及验证](../容器镜像签名及验证/tccli%20操作.md) -- page_id `80909`
- [在 TKE 中自定义 RBAC 授权](../../运维/在%20TKE%20中自定义%20RBAC%20授权/tccli%20操作.md) -- page_id `51683`

## 控制台替代

[SSM 控制台](https://console.cloud.tencent.com/ssm) -> 凭据列表 -> 新建凭据（输入凭据名称、类型和值）。
ESO 安装和 CRD 管理需通过 kubectl/helm CLI 操作，控制台不提供 External Secrets Operator 管理功能。
