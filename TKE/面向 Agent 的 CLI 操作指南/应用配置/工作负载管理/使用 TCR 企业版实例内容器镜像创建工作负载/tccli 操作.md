# 使用 TCR 企业版实例内容器镜像创建工作负载（tccli）

> 对照官方：[使用 TCR 企业版实例内容器镜像创建工作负载](https://cloud.tencent.com/document/product/457/45624) · page_id `45624`

## 概述

使用腾讯云容器镜像服务（TCR）企业版中的容器镜像在 TKE 集群中创建工作负载。需要配置镜像拉取凭证（imagePullSecrets），使集群节点能够从 TCR 企业版实例拉取私有镜像。本页为 tccli + kubectl 混合操作：tccli 完成 TCR 凭证获取和集群凭证获取（控制面），kubectl 完成 Secret 创建和工作负载部署（数据面）。

## 前置条件

- [环境准备](../../../环境准备.md)：`tccli` 已配置，`kubectl` >= v1.28
- 已创建 TCR 企业版实例
- 已将镜像推送至 TCR 企业版实例
- 需要 CAM 权限：`tke:DescribeClusters`、`tke:DescribeClusterKubeconfig`、`tcr:DescribeInstances`、`tcr:CreateInstanceToken`

```bash
# 检查集群状态
tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'
# expected: ClusterStatus "Running"
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

> **注意**：kubectl 数据面当前因公网端点被组织级 CAM 策略拒绝（strategyId:240463971）而不可达。外网端点被 `tke:clusterExtranetEndpoint=true` 条件拦截，内网端点需通过 IOA/VPN 或同 VPC 内 CVM 访问。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 获取集群凭证 | `tccli tke DescribeClusterKubeconfig --region <Region> --ClusterId <ClusterId>` | 是 |
| 查询 TCR 实例 | `tccli tcr DescribeInstances --region <Region>` | 是 |
| 获取 TCR 长期访问凭证 | `tccli tcr CreateInstanceToken --region <Region> --RegistryId <RegistryId>` | 否 |
| 创建镜像拉取 Secret | `kubectl create secret docker-registry` | 否 |
| 创建工作负载 | `kubectl apply -f deployment-tcr.yaml` | 是 |
| 删除工作负载 | `kubectl delete deployment/<name>` | 是 |
| 删除 Secret | `kubectl delete secret/<name>` | 是 |

## 操作步骤

### 步骤 1：获取 TCR 访问凭证（控制面）

```bash
# 查询 TCR 企业版实例
tccli tcr DescribeInstances --region <Region>
# expected: 返回实例列表，含 RegistryId
```

```json
{
  "TotalCount": "<TotalCount>",
  "Registries": "<Registries>",
  "RegistryId": "<RegistryId>",
  "RegistryName": "<RegistryName>",
  "RegistryType": "<RegistryType>",
  "Status": "<Status>"
}
```

```bash
# 创建长期访问令牌
tccli tcr CreateInstanceToken --region <Region> --RegistryId <RegistryId> --Desc "tke-pull-token"
# expected: 返回 Username 和 Token（仅此次显示，妥善保管）
```

### 步骤 2：获取集群 kubeconfig（控制面）

```bash
tccli tke DescribeClusterKubeconfig --region <Region> --ClusterId <ClusterId> \
    | jq -r '.Kubeconfig' > kubeconfig.yaml
export KUBECONFIG=$(pwd)/kubeconfig.yaml
# expected: kubeconfig 写入本地文件
```

```json
{
  "Kubeconfig": "<Kubeconfig>",
  "RequestId": "<RequestId>"
}
```

### 步骤 3：创建镜像拉取 Secret（数据面）

```bash
kubectl create secret docker-registry tcr-secret \
  --docker-server=<RegistryId>.tencentcloudcr.com \
  --docker-username=<TCR_USERNAME> \
  --docker-password=<TCR_TOKEN>
# expected: secret/tcr-secret created
```

### 步骤 4：使用 TCR 镜像创建工作负载（数据面）

YAML 清单：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tcr-app
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: tcr-app
  template:
    metadata:
      labels:
        app: tcr-app
    spec:
      imagePullSecrets:
        - name: tcr-secret
      containers:
        - name: app
          image: <RegistryId>.tencentcloudcr.com/<Namespace>/<Image>:<Tag>
          ports:
            - containerPort: 80
```

```bash
kubectl apply -f deployment-tcr.yaml
# expected: deployment.apps/tcr-app created
```

**预期输出**：

```text
deployment.apps/tcr-app created
```

### 步骤 5：验证镜像拉取

```bash
kubectl get pods -l app=tcr-app
# expected: STATUS Running（非 ImagePullBackOff）
```

```text
NAME  STATUS  AGE
...
```

```bash
kubectl describe pod -l app=tcr-app | grep -A2 "Pulled\|Pull"
# expected: Successfully pulled image
```

```text
NAME  STATUS  AGE
...
```

## 验证

### 控制面（tccli）

```bash
tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'
# expected: ClusterStatus "Running"

tccli tcr DescribeInstances --region <Region>
# expected: 实例状态 Running
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

### 数据面（kubectl）

```bash
kubectl get deployment tcr-app
# expected: READY 2/2

kubectl get pods -l app=tcr-app
# expected: STATUS Running

kubectl get secret tcr-secret
# expected: NAME tcr-secret
```

```text
NAME  STATUS  AGE
...
```

## 清理

```bash
kubectl delete deployment tcr-app
# expected: deployment.apps "tcr-app" deleted

kubectl delete secret tcr-secret
# expected: secret "tcr-secret" deleted
```

> **注意**：TCR 长期访问令牌需在 TCR 控制台手动删除。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `Unable to connect to the server` | `kubectl cluster-info` 检查连通性 | 公网端点被 CAM 策略 strategyId:240463971 以 `tke:clusterExtranetEndpoint=true` 条件拒绝 | 通过 IOA/VPN 连接内网，或使用同 VPC CVM 执行 kubectl 命令 |
| `unauthorized` | `kubectl describe pod <pod>` 查看事件 | TCR 令牌无效或过期 | 重新创建 TCR 长期访问令牌并更新 Secret |
| `dial tcp: lookup` | `kubectl describe pod <pod>` 查看事件 | 节点无法解析 TCR 域名 | 检查节点 VPC DNS 配置 |

### 资源创建成功但状态异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Pod `ImagePullBackOff` | `kubectl describe pod <pod>` 查看事件 | Secret 未配置或凭证错误 | 确认 Secret 所在命名空间和凭证正确 |
| `repository does not exist` | `kubectl describe pod <pod>` 查看事件 | 镜像路径错误 | 检查 `<RegistryId>/<Namespace>/<Image>:<Tag>` |

## 下一步

- [构建简单 Web 应用](../../../快速入门/入门示例/构建简单%20Web%20应用/tccli%20操作.md) — page_id `6996`
- [Secret 管理](../../服务和配置管理/Secret/tccli%20操作.md) — page_id `31718`

## 控制台替代

[容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) → 集群 → 工作负载 → Deployment → 新建，选择 TCR 镜像。
