# 其他资源管理

> 对照官方：[其他资源管理](https://cloud.tencent.com/document/product/457/39818) · page_id `39818`

## 概述

TKE Serverless 集群中除工作负载和 Service 外，还需要管理 ConfigMap、Secret、Namespace、ResourceQuota 等 Kubernetes 资源。ConfigMap 和 Secret 用于配置与敏感数据管理，Namespace 用于资源隔离，ResourceQuota 在 Serverless 集群中尤为重要——它限制命名空间内的总资源使用量，防止单个命名空间耗尽集群资源。

**重要**：Serverless 集群按 Pod 实际使用的 CPU/内存计费，通过 ResourceQuota 可设置命名空间级别的资源上限，控制成本。

## 前置条件

- [环境准备](../../../../环境准备.md)

### 集群可达性检查（Before you begin）

资源管理为 kubectl-only 操作。确认集群连接正常后方可继续。

```bash
# 1. 确认集群处于 Running 状态
tccli tke DescribeEKSClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterStatus: Running

# 2. 验证 kubectl 连接
kubectl --kubeconfig ~/.kube/config-eks-CLUSTER_ID cluster-info
# expected: Kubernetes control plane is running at https://...

# 3. 查看已有命名空间
kubectl --kubeconfig ~/.kube/config-eks-CLUSTER_ID get ns
# expected: 列出已有命名空间（default, kube-system 等）
```

```json
{
  "TotalCount": "<TotalCount>",
  "Clusters": "<Clusters>",
  "ClusterId": "<ClusterId>",
  "ClusterName": "<ClusterName>",
  "VpcId": "<VpcId>",
  "SubnetIds": "<SubnetIds>"
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | kubectl 命令 | 幂等 |
|-----------|-----------|:--:|
| 创建 Namespace | `kubectl create ns NAMESPACE` | 否 |
| 创建 ConfigMap | `kubectl create configmap NAME --from-file=FILE -n NAMESPACE` | 是 |
| 创建 Secret | `kubectl create secret generic NAME --from-literal=KEY=VALUE -n NAMESPACE` | 是 |
| 创建 ResourceQuota | `kubectl apply -f quota.yaml` | 是 |
| 查看资源 | `kubectl get configmaps/secrets/resourcequotas -n NAMESPACE` | 是 |
| 删除资源 | `kubectl delete configmap/secret/ns NAME -n NAMESPACE` | 是 |

## 操作步骤

### 步骤 1: 管理 Namespace

Namespace 提供逻辑隔离和资源配额边界：

```bash
# 创建命名空间
kubectl --kubeconfig ~/.kube/config-eks-CLUSTER_ID create ns NAMESPACE
# expected: namespace/NAMESPACE created

# 查看命名空间
kubectl --kubeconfig ~/.kube/config-eks-CLUSTER_ID get ns
# expected: NAMESPACE 出现在列表中

# 查看命名空间详情
kubectl --kubeconfig ~/.kube/config-eks-CLUSTER_ID describe ns NAMESPACE
# expected: 显示 Labels、Annotations、ResourceQuota 等信息
```

### 步骤 2: 管理 ConfigMap

ConfigMap 存储非敏感的配置数据，如环境变量、配置文件：

```bash
# 从字面值创建
kubectl --kubeconfig ~/.kube/config-eks-CLUSTER_ID create configmap app-config \
    --from-literal=APP_ENV=production \
    --from-literal=LOG_LEVEL=info \
    -n NAMESPACE
# expected: configmap/app-config created

# 从文件创建
cat > /tmp/app.properties << 'EOF'
app.name=my-service
app.port=8080
EOF

kubectl --kubeconfig ~/.kube/config-eks-CLUSTER_ID create configmap app-properties \
    --from-file=/tmp/app.properties \
    -n NAMESPACE
# expected: configmap/app-properties created

# 查看 ConfigMap
kubectl --kubeconfig ~/.kube/config-eks-CLUSTER_ID get configmap app-config -n NAMESPACE -o yaml
# expected: 显示 data 字段包含 APP_ENV 和 LOG_LEVEL
```

或使用 YAML 声明式创建：

`app-configmap.yaml`：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: NAMESPACE
data:
  APP_ENV: "production"
  LOG_LEVEL: "info"
  app.properties: |
    app.name=my-service
    app.port=8080
```

```bash
kubectl --kubeconfig ~/.kube/config-eks-CLUSTER_ID apply -f app-configmap.yaml
# expected: configmap/app-config configured 或 created
```

### 步骤 3: 管理 Secret

Secret 存储敏感数据，支持 Opaque（通用）、docker-registry（镜像拉取）、tls 等类型：

```bash
# 创建 Opaque Secret
kubectl --kubeconfig ~/.kube/config-eks-CLUSTER_ID create secret generic db-credentials \
    --from-literal=username=admin \
    --from-literal=password='S3cureP@ss!' \
    -n NAMESPACE
# expected: secret/db-credentials created

# 创建 docker-registry Secret（用于拉取私有镜像）
kubectl --kubeconfig ~/.kube/config-eks-CLUSTER_ID create secret docker-registry tcr-secret \
    --docker-server=ccr.ccs.tencentyun.com \
    --docker-username=YOUR_TCR_USERNAME \
    --docker-password=YOUR_TCR_PASSWORD \
    -n NAMESPACE
# expected: secret/tcr-secret created

# 查看 Secret（值经过 base64 编码）
kubectl --kubeconfig ~/.kube/config-eks-CLUSTER_ID get secret db-credentials -n NAMESPACE -o yaml
# expected: data 字段包含 base64 编码的值
```

### 步骤 4: 管理 ResourceQuota

ResourceQuota 在 Serverless 集群中至关重要，可限制命名空间内的总资源使用：

`namespace-quota.yaml`：

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: NAMESPACE
spec:
  hard:
    requests.cpu: "16"
    requests.memory: "32Gi"
    limits.cpu: "32"
    limits.memory: "64Gi"
    pods: "20"
    services: "10"
    configmaps: "20"
    secrets: "20"
```

```bash
kubectl --kubeconfig ~/.kube/config-eks-CLUSTER_ID apply -f namespace-quota.yaml
# expected: resourcequota/compute-quota created

# 查看配额使用情况
kubectl --kubeconfig ~/.kube/config-eks-CLUSTER_ID describe resourcequota compute-quota -n NAMESPACE
# expected: 显示 Used 和 Hard 列
```

预期输出：

```
Name:       compute-quota
Namespace:  NAMESPACE
Resource    Used    Hard
--------    ----    ----
pods        2       20
requests.cpu 2      16
requests.memory 4Gi 32Gi
...
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `NAMESPACE` | Kubernetes 命名空间 | 需预先存在或通过 `kubectl create ns` 创建 | `kubectl get ns` |
| `CLUSTER_ID` | 集群 ID | `cls-xxxxxxxx` 格式 | `tccli tke DescribeEKSClusters --region <Region>` |
| `REGION` | 地域 | 如 `ap-guangzhou` | `tccli configure list` |
| `YOUR_TCR_USERNAME` | TCR 用户名 | 腾讯云 TCR 长期访问凭证用户名 | TCR 控制台 → 访问凭证 |
| `YOUR_TCR_PASSWORD` | TCR 密码 | 腾讯云 TCR 长期访问凭证密码 | TCR 控制台 → 访问凭证 |

## 验证

| 维度 | 命令 | 预期 |
|------|------|------|
| Namespace 存在 | `kubectl get ns NAMESPACE` | NAME 和 STATUS Active |
| ConfigMap 数据 | `kubectl get configmap app-config -n NAMESPACE -o jsonpath='{.data.APP_ENV}'` | `production` |
| Secret 解码 | `kubectl get secret db-credentials -n NAMESPACE -o jsonpath='{.data.username}' \| base64 -d` | `admin` |
| ResourceQuota 生效 | `kubectl describe resourcequota compute-quota -n NAMESPACE` | Used/Hard 值正确 |
| Pod 引用 ConfigMap | 在 Pod 中引用后 `kubectl exec POD_NAME -n NAMESPACE -- env \| grep APP_ENV` | `APP_ENV=production` |

```bash
# 综合验证
kubectl --kubeconfig ~/.kube/config-eks-CLUSTER_ID get ns,configmap,secret,resourcequota -n NAMESPACE
# expected: 列出命名空间内所有资源
```

## 清理

> **警告**：删除 Namespace 会**级联删除**其下所有资源（Pod、Service、ConfigMap、Secret 等），不可逆。删除 Secret 可能影响依赖该 Secret 的工作负载（如图像拉取、环境变量注入）。

```bash
# 删除 ConfigMap
kubectl --kubeconfig ~/.kube/config-eks-CLUSTER_ID delete configmap/app-config -n NAMESPACE
# expected: configmap "app-config" deleted

# 删除 Secret
kubectl --kubeconfig ~/.kube/config-eks-CLUSTER_ID delete secret/db-credentials -n NAMESPACE
# expected: secret "db-credentials" deleted

# 删除 ResourceQuota
kubectl --kubeconfig ~/.kube/config-eks-CLUSTER_ID delete resourcequota/compute-quota -n NAMESPACE
# expected: resourcequota "compute-quota" deleted

# 删除 Namespace（不可逆！确认命名空间内无重要资源）
kubectl --kubeconfig ~/.kube/config-eks-CLUSTER_ID delete ns NAMESPACE
# expected: namespace "NAMESPACE" deleted（Terminating 状态持续数秒）
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl create ns` 返回 `AlreadyExists` | `kubectl get ns NAMESPACE` 检查 | 命名空间已存在（此为操作错误） | 使用已有 Namespace 或创建不同名称 |
| `kubectl create configmap` 返回 `must specify one of -f and -k` | 检查命令格式 | 缺少 `--from-literal` 或 `--from-file` 参数 | 补充参数来源 |
| Secret 创建后 Pod 仍报 `ImagePullBackOff` | `kubectl describe pod POD_NAME -n NAMESPACE` 检查 Events | Pod 未引用 imagePullSecrets | 在 Pod spec 中添加 `imagePullSecrets: [{name: tcr-secret}]` |
| `kubectl apply -f quota.yaml` 返回 `insufficient quota` | `kubectl describe resourcequota -n NAMESPACE` 检查当前用量 | 新配额低于已使用量（此为操作错误） | 调整配额不小于当前使用量 |
| 删除 Namespace 卡在 `Terminating` | `kubectl get ns NAMESPACE -o json \| jq '.spec.finalizers'` | 存在 Finalizer 阻止删除 | 手动移除 Finalizer：`kubectl patch ns NAMESPACE -p '{"spec":{"finalizers":[]}}' --type=merge` |

### 操作成功但异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| ConfigMap 更新后 Pod 未自动刷新 | `kubectl exec POD_NAME -n NAMESPACE -- env \| grep APP_ENV` 检查环境变量 | 环境变量只在 Pod 启动时注入 | 重启 Pod（`kubectl rollout restart deployment/NAME`） |
| ResourceQuota 不限制 Pod 数量 | 检查 Pod 是否通过 annotation 指定资源 | 带 `eks.tke.cloud.tencent.com/cpu|mem` annotation 的 Pod 不计入 `requests.cpu/memory` | 同时使用 `limits.cpu/memory` 配额或限制 Annotation 资源范围 |
| Secret 的 data 值为空 | `kubectl get secret NAME -n NAMESPACE -o jsonpath='{.data}'` | 创建时 `--from-literal=KEY=` value 为空 | 检查 value 是否提供；base64 编码的空字符串在 data 中仍会显示 |

## 下一步

- [工作负载管理](../工作负载管理/tccli%20操作.md) — 使用 ConfigMap/Secret 部署应用
- [服务管理](../服务管理/tccli%20操作.md) — 暴露工作负载
- [监控和告警](../../运维中心/监控和告警/tccli%20操作.md) — 监控资源使用

## 控制台替代

控制台：容器服务控制台 → 集群 → 选择目标 Serverless 集群 → 配置管理 → ConfigMap/Secret/ResourceQuota → 新建。
