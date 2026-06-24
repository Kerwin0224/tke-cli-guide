# Serverless 集群全局配置说明

> 对照官方：[Serverless 集群全局配置说明](https://cloud.tencent.com/document/product/457/71915) · page_id `71915`

## 概述

TKE Serverless 集群通过 Kubernetes ConfigMap 实现集群级全局配置。这些配置以 ConfigMap 的形式存储在 `kube-system` 命名空间中，影响集群中所有 Pod 的默认行为。

全局配置管理的内容：
- **默认资源规格**：未声明 `resources.requests` 时 Pod 的默认 vCPU 和内存。
- **镜像缓存策略**：全局启用/禁用镜像缓存，设置默认镜像缓存规则。
- **日志采集配置**：CRD 日志采集的全局设置。
- **固定 IP 策略**：全局固定 IP 保留策略。
- **资源限制**：单 Pod 的资源上下限。

## 前置条件

- [环境准备](../../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 kubectl 并连接集群
kubectl cluster-info
# expected: Kubernetes control plane is running at ...

# 2. 确认集群为 Serverless 类型
tccli tke DescribeEKSClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]' \
    | jq '.Clusters[0].ClusterType'
# expected: "SERVERLESS_CLUSTER"

# 3. 确认有集群管理权限
kubectl auth can-i get configmap -n kube-system
# expected: yes
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

| 控制台操作 | tccli / kubectl | 幂等 |
|-----------|-----------|:--:|
| 查看全局配置 | `kubectl get configmap -n kube-system` | 是 |
| 编辑全局配置 | `kubectl edit configmap` / `kubectl apply` | 是 |
| 重置默认配置 | `kubectl delete configmap`（系统自动重建默认） | — |
| 查看配置生效状态 | kubectl（创建 Pod 验证） | 是 |

## 操作步骤

### 步骤 1：查看当前全局配置

```bash
# 列出 kube-system 下与 EKS 相关的 ConfigMap
kubectl get configmap -n kube-system | grep -E 'eks|config'
# expected: 返回 EKS 相关的 ConfigMap 列表

# 查看默认资源规格配置
kubectl get configmap eks-config -n kube-system -o yaml
# expected: 返回 EKS 全局配置（如果存在）
```

**预期输出示例**：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: eks-config
  namespace: kube-system
data:
  default-cpu: "0.5"
  default-mem: "1Gi"
  max-cpu: "32"
  max-mem: "128Gi"
  image-cache-enabled: "true"
  image-cache-default: "auto"
```

### 步骤 2：查看和修改默认资源规格

```bash
# 查看默认 CPU 和内存配置
kubectl get configmap eks-config -n kube-system -o jsonpath='{.data.default-cpu}'
# expected: 默认 CPU 值，如 "0.5"

kubectl get configmap eks-config -n kube-system -o jsonpath='{.data.default-mem}'
# expected: 默认内存值，如 "1Gi"
```

修改默认资源规格（当 Pod spec 未声明 `resources.requests` 时生效）：

```yaml title="eks-config-patch.yaml"
apiVersion: v1
kind: ConfigMap
metadata:
  name: eks-config
  namespace: kube-system
data:
  default-cpu: "1"
  default-mem: "2Gi"
```

```bash
kubectl apply -f eks-config-patch.yaml
# expected: configmap/eks-config configured
```

### 步骤 3：配置全局镜像缓存策略

```bash
# 查看当前镜像缓存配置
kubectl get configmap eks-config -n kube-system -o jsonpath='{.data.image-cache-enabled}'
# expected: "true" 或 "false"
```

修改镜像缓存策略：

```yaml title="eks-config-cache-patch.yaml"
apiVersion: v1
kind: ConfigMap
metadata:
  name: eks-config
  namespace: kube-system
data:
  image-cache-enabled: "true"
  image-cache-default: "auto"
```

```bash
kubectl apply -f eks-config-cache-patch.yaml
# expected: configmap/eks-config configured
```

### 步骤 4：配置资源限制

```bash
# 查看 Pod 资源上限
kubectl get configmap eks-config -n kube-system -o jsonpath='{.data.max-cpu}'
# expected: 最大 CPU 核数

kubectl get configmap eks-config -n kube-system -o jsonpath='{.data.max-mem}'
# expected: 最大内存
```

### 步骤 5：验证配置生效

创建测试 Pod 验证默认资源规格是否生效：

```yaml title="eks-test-default-resources.yaml"
apiVersion: v1
kind: Pod
metadata:
  name: test-default-resources
  namespace: default
spec:
  containers:
  - name: app
    image: nginx:latest
    # 故意不设置 resources，验证默认配置是否生效
```

```bash
kubectl apply -f eks-test-default-resources.yaml
# expected: pod/test-default-resources created

kubectl get pod test-default-resources -n default -o jsonpath='{.spec.containers[0].resources}'
# expected: 返回与全局默认配置一致的 resources（由 webhook 注入）
```

### 步骤 6：清理测试 Pod

```bash
kubectl delete pod test-default-resources -n default
# expected: pod "test-default-resources" deleted
```

## 验证

```bash
# 验证 ConfigMap 已更新
kubectl get configmap eks-config -n kube-system -o yaml
# expected: data 字段包含更新后的配置值

# 验证配置对新建 Pod 生效
kubectl run test-pod --image=nginx:latest --restart=Never && \
    kubectl get pod test-pod -o jsonpath='{.spec.containers[0].resources}' && \
    kubectl delete pod test-pod
# expected: resources.requests 符合全局默认值
```

## 清理

```bash
# 删除测试 Pod（如有遗留）
kubectl delete pod test-default-resources test-pod -n default --ignore-not-found
# expected: pod deleted

# 如需恢复默认配置，删除自定义 ConfigMap（系统自动重建默认值）
kubectl delete configmap eks-config -n kube-system
# expected: configmap "eks-config" deleted（系统自动重建默认 ConfigMap）
```

> **注意**：删除全局 ConfigMap 后系统会自动重建默认配置。若想保留自定义配置，不要执行此删除操作。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl get configmap eks-config` 返回 `NotFound` | `kubectl get configmap -n kube-system` 列出所有 ConfigMap | 集群中尚未创建 `eks-config` ConfigMap（可能使用不同名称） | 检查是否使用其他 ConfigMap 名称，或此集群版本不需要显式配置 |
| `kubectl edit configmap` 保存后报错 | 检查 YAML 格式是否正确 | `data` 字段值必须为字符串格式 | 确保所有 ConfigMap 的 `data` 字段值为字符串（如 `cpu: "1"` 而非 `cpu: 1`） |
| `kubectl apply` ConfigMap 报 `Unauthorized` | `kubectl auth can-i update configmap -n kube-system` 检查权限 | 当前 kubectl 凭据缺少 kube-system 命名空间的写权限 | 使用集群管理员凭据或联系管理员授予权限 |
| 修改 ConfigMap 后 Pod 仍未使用新默认值 | 检查 Pod 是否在修改 ConfigMap 前创建 | ConfigMap 修改仅对新创建的 Pod 生效，已有 Pod 不受影响 | 重建 Pod（Delete + Create）使新配置生效 |

### 操作成功但异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 删除 ConfigMap 后系统未自动重建 | 等待 1-2 分钟后重试 | 控制器重建有延迟 | 等待不超过 3 分钟；若仍未重建，联系售后 |
| `image-cache-enabled: "true"` 但部分 Pod 未使用镜像缓存 | `kubectl describe pod` 查看 Events | 镜像缓存未创建或不包含该镜像 | 先创建镜像缓存；或检查 Pod Annotation 是否覆盖了全局配置 |
| 默认 CPU 设置为 `"0.25"` 但 Pod 实际被分配了 `"0.5"` | 检查底层最小规格限制 | 底层资源池的最小规格为 0.5 核 | 最小值受底层限制，设置为低于 0.5 的值会自动向上取整 |

## 下一步

- [Annotation 说明](../Annotation%20说明/tccli%20操作.md) — 了解 Pod 级别的 Annotation 配置（覆盖全局配置）
- [注意事项](../../TKE%20Serverless%20集群管理/注意事项/tccli%20操作.md) — Serverless 集群限制和兼容性
- [创建 Serverless 集群](../../TKE%20Serverless%20集群管理/创建集群/tccli%20操作.md) — 新建集群

## 控制台替代

[TKE 控制台 → Serverless 集群 → 集群详情 → 全局配置](https://console.cloud.tencent.com/tke2/ecluster)：控制台提供可视化界面查看和修改全局配置，无需手动编辑 ConfigMap YAML。
