# Pod 使用 CAM 对数据库身份验证（tccli）

> 对照官方：[Pod 使用 CAM 对数据库身份验证](https://cloud.tencent.com/document/product/457/81989) · page_id `81989`

## 概述

在 TKE 托管集群中运行的 Pod 借助 CAM OIDC 提供商 + SSM 凭据管理系统 + CAM 角色，安全连接到腾讯云数据库（MySQL），无需在 Pod 中硬编码数据库用户名和密码。SSM 自动轮转凭据，消除密钥泄漏风险和手工轮转负担。

## 前置条件

- **托管集群**（集群版本 ≥ v1.20.6-tke.27 或 v1.22.5-tke.1）。
- 已创建 [腾讯云 MySQL 实例](https://console.cloud.tencent.com/cdb)（开启公网，记录地址 `$db_address` 和端口 `$db_port`）。
- 已开通 [凭据管理系统 SSM](https://console.cloud.tencent.com/ssm)。
- 业务 Pod 可访问外网。
- 已 [连接集群](../../../集群配置/集群管理/连接集群/tccli%20操作.md)。

```bash
tccli tke DescribeClusters --region ap-guangzhou --output json \
  --filter "Clusters[?ClusterId=='<ClusterId>'] | [0].{ClusterType:ClusterType,ClusterStatus:ClusterStatus,ClusterVersion:ClusterVersion}"
```

```
{
  "ClusterType": "MANAGED_CLUSTER",
  "ClusterStatus": "Running",
  "ClusterVersion": "1.28.3"
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 开启 OIDC | 控制台 APIServer 信息页 | 否 |
| 查看 OIDC 提供商 | CAM 控制台 | 是 |
| 创建数据库 | MySQL 客户端 `CREATE DATABASE` | 否 |
| 创建 SSM 凭据 | SSM 控制台 | 否 |
| 创建 CAM 角色 | CAM 控制台 | 否 |
| 部署 ServiceAccount | `kubectl apply -f my-serviceaccount.yaml` | 否 |
| 部署示例应用 | `kubectl apply -f sample-application.yaml` | 是 |

## 操作步骤

### 步骤1：开启 OIDC 资源访问控制

1. 进入集群详情页 **基本信息 → APIServer 信息**。
2. 单击 ServiceAccountIssuerDiscovery 旁的编辑。
3. 授权 `QcloudAccessForTKERoleInOIDCConfig` 策略。
4. 勾选"创建 CAM OIDC 提供商"和"创建 webhook 组件"，客户端 ID 使用默认值 `sts.cloud.tencent.com`。

### 步骤2：确认 OIDC 提供商和 Webhook 组件

```bash
kubectl get pod -n kube-system | grep pod-identity-webhook
```

```
NAMESPACE     NAME                                    READY   STATUS
kube-system   pod-identity-webhook-78c76xxx-9qrpj     1/1     Running
```

### 步骤3：准备数据库

```bash
mysql -h $db_address -P $db_port -u root -p

CREATE DATABASE mydb;
CREATE TABLE mydb.user (Id VARCHAR(120), Name VARCHAR(120));
INSERT INTO mydb.user (Id, Name) VALUES ('123', 'tke-oidc');
SELECT * FROM mydb.user;
```

```
+------+----------+
| Id   | Name     |
+------+----------+
| 123  | tke-oidc |
+------+----------+
```

更新数据库安全组，允许 Pod 所在 VPC 的入站 TCP:3306 流量。

### 步骤4：创建 SSM 数据库凭据

在 [SSM 控制台](https://console.cloud.tencent.com/ssm) 创建两个数据库凭据（一个有 SELECT 权限，一个有全部权限），开启凭据轮转。

记录 `$ssm_name` 和 `$ssm_region_name`。

### 步骤5：创建 CAM 角色

1. 登录 [访问管理控制台](https://console.cloud.tencent.com/cam/role)。
2. **新建角色 → 身份提供商**。
3. 身份提供商选 CAM OIDC 提供商，客户端 ID 填 `sts.cloud.tencent.com`。
4. 关联策略：`QcloudSSMReadOnlyAccess` + `QcloudCDBReadOnlyAccess`。
5. 记录 **RoleArn**：`qcs::cam::uin/<UIN>:roleName/<role_name>`。

### 步骤6：部署 ServiceAccount

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-serviceaccount
  namespace: my-namespace
  annotations:
    tke.cloud.tencent.com/role-arn: <RoleArn>
    tke.cloud.tencent.com/audience: sts.cloud.tencent.com
    tke.cloud.tencent.com/token-expiration: "86400"
```

```bash
kubectl apply -f my-serviceaccount.yaml
```

### 步骤7：部署示例应用

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: my-namespace
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      serviceAccountName: my-serviceaccount
      containers:
      - name: nginx
        image: ccr.ccs.tencentyun.com/tkeimages/sample-application:latest
        ports:
        - containerPort: 80
```

```bash
kubectl apply -f sample-application.yaml
kubectl get pods -n my-namespace
```

```
NAME                                READY   STATUS    AGE
nginx-deployment-6bfd845f47-9zxld   1/1     Running   67s
```

### 步骤8：查看环境变量

```bash
kubectl describe pod nginx-deployment-6bfd845f47-9zxld -n my-namespace
```

```text
Name:         ...
Status:       Running
...
```

可看到 `AWS_ROLE_ARN`、`AWS_WEB_IDENTITY_TOKEN_FILE` 等环境变量已自动注入。

### 步骤9：测试 demo

```bash
kubectl exec -ti nginx-deployment-6bfd845f47-9zxld -n my-namespace -- /bin/bash
cd /root
./demo --ssmName=$ssm_name --ssmRegionName=$ssm_region_name \
  --dbAddress=$db_address --dbName=$db_name --dbPort=$db_port
```

当 SSM 凭据具有 SELECT 权限时，日志输出 `succeed to query db`；无 SELECT 权限时返回 `failed to query db`。

## 验证

```bash
kubectl get pods -n my-namespace
kubectl logs nginx-deployment-xxx -n my-namespace
```

```text
NAME  STATUS  AGE
...
```

demo 输出需显示数据库连接成功和查询结果。

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl` 连接超时/拒绝 | `tccli tke DescribeClusters` 返回集群 Running 但 kubectl 不可达 | CAM 策略 240463971 拒绝公网访问 APIServer 端点 | 通过 VPN/IOA 接入内网后执行 kubectl 命令 |
| OIDC 开启失败 | 集群详情页 APIServer 信息查看 OIDC 状态 | 集群非托管集群或未完成服务授权 | 确认集群为托管集群；先进行服务授权 |
| pod-identity-webhook Pod 未运行 | `kubectl get pod -n kube-system | grep pod-identity-webhook` | 组件安装未完成或集群版本不满足 | 等待组件安装完成；确认集群版本 ≥ v1.20.6-tke.27 或 v1.22.5-tke.1 |
| `AssumeRoleWithWebIdentity` 权限拒绝 | `kubectl describe pod <pod>` 查看 Events 中注入的环境变量 | CAM 角色信任策略的 oidc:aud 与 ServiceAccount annotation 不一致 | 确认客户端 ID 为 `sts.cloud.tencent.com`；检查 RoleArn 正确 |
| 数据库连接失败 | Pod 内执行 `mysql -h $db_address -P $db_port -u root -p` 测试连通性 | 安全组未放通 Pod IP 或 MySQL 公网未开启 | 更新数据库安全组放通 Pod 所在 VPC 的入站 TCP:3306 |

## 清理

```bash
kubectl delete namespace my-namespace
# 在 SSM 控制台删除凭据
# 在 CAM 控制台删除角色
# 在 CDB 控制台删除数据库/实例
```

## 下一步

- [容器镜像签名及验证](../容器镜像签名及验证/tccli 操作.md)（page_id `80909`）
- [TKE ExternalSecretOperator 实践指南](https://cloud.tencent.com/document/product/457/132413)

## 控制台替代

控制台 **集群 → 基本信息 → ServiceAccountIssuerDiscovery 编辑**开启 OIDC；**CAM 控制台 → 角色**创建 OIDC 角色；**SSM 控制台 → 数据库凭据**创建轮转凭据；**集群 → 工作负载**部署应用。
