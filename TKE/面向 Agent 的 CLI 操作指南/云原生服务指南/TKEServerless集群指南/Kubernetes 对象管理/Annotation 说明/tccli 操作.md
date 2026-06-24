# Annotation 说明

> 对照官方：[Annotation 说明](https://cloud.tencent.com/document/product/457/60413) · page_id `60413`

## 概述

TKE Serverless 集群通过 Pod Annotations 实现对 Pod 行为的精细控制，包括资源规格声明、计费规格、镜像缓存、网络配置等。这些 Annotation 是 Serverless 集群特有的扩展，在标准 TKE 中无对应功能。

使用 Annotation 控制的核心领域：
- **资源规格**：显式声明 Pod 的 vCPU/内存/GPU 规格和计费模式。
- **镜像缓存**：启用镜像缓存加速 Pod 启动。
- **网络配置**：指定子网、安全组、固定 IP 等。
- **调度策略**：指定可用区节点亲和性。
- **日志采集**：配置 CRD 日志采集（无需 DaemonSet）。

## 前置条件

- [环境准备](../../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查 kubectl 并连接集群
kubectl cluster-info
# expected: Kubernetes control plane is running at ...

# 3. 确认集群为 Serverless 类型
tccli tke DescribeEKSClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]' \
    | jq '.Clusters[0].ClusterType'
# expected: "SERVERLESS_CLUSTER"
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
| 创建工作负载时配置资源规格 | kubectl（通过 Pod Annotation） | — |
| 启用镜像缓存 | kubectl（通过 Pod Annotation） | — |
| 绑定安全组 | kubectl（通过 Pod Annotation） | 是 |
| 指定子网 | kubectl（通过 Pod Annotation） | 是 |
| 查看已有 Pod 的 Annotation | `kubectl describe pod` / `kubectl get pod -o yaml` | 是 |

## 操作步骤

### 步骤 1：查看当前 Pod 的 Annotation

> **需 kubectl 可达环境**

```bash
# 查看指定 Pod 的所有 Annotation
kubectl get pod POD_NAME -n NAMESPACE -o jsonpath='{.metadata.annotations}' | jq .
# expected: 返回该 Pod 的所有 Annotation

# 查看集群中所有 Pod 的特定 Annotation
kubectl get pods -A -o json | jq '.items[] | {name: .metadata.name, annotations: .metadata.annotations}'
# expected: 列出每个 Pod 的 Annotation
```

### 步骤 2：常用 Serverless Annotation 参考

#### 资源规格类

| Annotation Key | 说明 | 示例值 |
|---------------|------|--------|
| `eks.tke.cloud.tencent.com/cpu` | 显式声明 Pod vCPU 核数 | `"2"`（2 核） |
| `eks.tke.cloud.tencent.com/mem` | 显式声明 Pod 内存（Gi） | `"4Gi"`（4 GB） |
| `eks.tke.cloud.tencent.com/gpu-type` | GPU 类型 | `"V100"`、`"T4"`、`"A10"` |
| `eks.tke.cloud.tencent.com/gpu-count` | GPU 卡数 | `"1"` |
| `eks.tke.cloud.tencent.com/cpu-type` | CPU 类型（如 Intel / AMD / 鲲鹏） | `"intel"`、`"amd"`、`"kunpeng"` |

#### 网络类

| Annotation Key | 说明 | 示例值 |
|---------------|------|--------|
| `eks.tke.cloud.tencent.com/subnet-id` | 指定 Pod 分配 IP 的子网 | `"subnet-xxxxxxxx"` |
| `eks.tke.cloud.tencent.com/security-group-id` | 绑定安全组（Pod 级） | `"sg-xxxxxxxx"` |
| `eks.tke.cloud.tencent.com/static-ip` | 固定 Pod IP | `"true"` |
| `eks.tke.cloud.tencent.com/eip-attributes` | 自动绑定 EIP | JSON 格式 EIP 配置 |

#### 存储与镜像类

| Annotation Key | 说明 | 示例值 |
|---------------|------|--------|
| `eks.tke.cloud.tencent.com/use-image-cache` | 启用镜像缓存加速 | `"auto"` 或 `"imc-xxxxxxxx"` |
| `eks.tke.cloud.tencent.com/root-cbs-size` | 系统盘大小（GB） | `"50"` |
| `eks.tke.cloud.tencent.com/disk-type` | CBS 系统盘类型 | `"CLOUD_PREMIUM"`、`"CLOUD_SSD"` |

#### 调度类

| Annotation Key | 说明 | 示例值 |
|---------------|------|--------|
| `eks.tke.cloud.tencent.com/zone` | 指定可用区调度 | `"ap-guangzhou-3"` |
| `eks.tke.cloud.tencent.com/cross-zone` | 是否跨可用区调度 | `"true"` |
| `eks.tke.cloud.tencent.com/retain-ip-hours` | 固定 IP 保留时长（小时） | `"24"` |

### 步骤 3：创建带 Annotation 的 Pod 示例

```yaml title="eks-pod-with-annotations.yaml"
apiVersion: v1
kind: Pod
metadata:
  name: eks-annotated-pod
  namespace: default
  annotations:
    eks.tke.cloud.tencent.com/cpu: "2"
    eks.tke.cloud.tencent.com/mem: "4Gi"
    eks.tke.cloud.tencent.com/cpu-type: "intel"
    eks.tke.cloud.tencent.com/security-group-id: "sg-xxxxxxxx"
    eks.tke.cloud.tencent.com/use-image-cache: "auto"
spec:
  containers:
  - name: app
    image: nginx:latest
    resources:
      requests:
        cpu: "2"
        memory: "4Gi"
      limits:
        cpu: "2"
        memory: "4Gi"
  imagePullSecrets:
  - name: image-pull-secret
```

```bash
kubectl apply -f eks-pod-with-annotations.yaml
# expected: pod/eks-annotated-pod created
```

### 步骤 4：验证 Annotation 生效

```bash
# 查看 Pod Annotation 是否已正确应用
kubectl get pod eks-annotated-pod -n default -o jsonpath='{.metadata.annotations}' | jq .
# expected: 返回与创建时一致的 Annotation

# 验证 Pod 运行正常
kubectl get pod eks-annotated-pod -n default
# expected: STATUS Running
```

## 验证

```bash
# 验证 Annotation 已应用
kubectl get pod eks-annotated-pod -n default -o json | jq '.metadata.annotations'
# expected: 包含 eks.tke.cloud.tencent.com/* 相关 Annotation

# 验证 Pod 按 Annotation 调度到指定子网（检查 Pod IP）
kubectl get pod eks-annotated-pod -n default -o jsonpath='{.status.podIP}'
# expected: 返回 Pod IP（确认 IP 属于指定子网 CIDR）

# 验证镜像缓存生效（查看 Pod Events）
kubectl describe pod eks-annotated-pod -n default | grep -i cache
# expected: 如有日志显示镜像缓存命中或创建
```

## 清理

```bash
# 删除测试 Pod
kubectl delete pod eks-annotated-pod -n default
# expected: pod "eks-annotated-pod" deleted

# 验证已删除
kubectl get pod eks-annotated-pod -n default
# expected: Error from server (NotFound)
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Pod 创建后 Annotation 消失或值异常 | `kubectl apply -f` 后 `kubectl get pod -o json` 检查 | Annotation Key 拼写错误或值格式不正确 | 对照本文 Annotation 参考表修正 Key 和值格式 |
| 指定 `subnet-id` 后 Pod Pending | `kubectl describe pod` 查看 Events | 指定子网不存在、不可用或 IP 耗尽 | `tccli vpc DescribeSubnets --SubnetIds '["subnet-xxxxxxxx"]'` 确认子网可用性 |
| 指定 `gpu-count` 后 Pod Pending | `kubectl describe pod` 查看 Events | GPU 资源不足或 GPU 类型不支持 | 更换 GPU 类型或可用区，或提工单确认 GPU 库存 |
| `use-image-cache: "auto"` 不生效 | `kubectl describe pod` 查看 Events | 未创建镜像缓存或缓存中无该镜像 | 先创建镜像缓存或使用 `"imc-xxxxxxxx"` 显式指定已有缓存 |
| Pod 固定 IP Annotation 不生效 | 删除 Pod 后重新创建，检查 IP 是否保留 | `retain-ip-hours` 设置的保留期已过 | 增加 `retain-ip-hours` 值或确保在保留期内重建 Pod |

### 操作成功但异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Annotation 声明了 `cpu: "4"` 但实际分配规格不符 | `kubectl get pod -o json` 检查 `resources.requests.cpu` | Annotation 仅在底层调度使用，Pod spec 中 `resources.requests` 必须与 Annotation 一致 | 确保 `resources.requests` 与 Annotation 声明的规格一致 |
| 镜像缓存命中但 Pod 启动仍慢 | 检查 Pod Events 和镜像缓存命中日志 | 镜像缓存中仅缓存了基础层，部分层未命中 | 确认镜像缓存创建时指定的镜像 Tag 与实际拉取的一致 |

## 下一步

- [Serverless 集群全局配置说明](../Serverless%20集群全局配置说明/tccli%20操作.md) — 了解全局 ConfigMap 配置
- [注意事项](../../TKE%20Serverless%20集群管理/注意事项/tccli%20操作.md) — Serverless 集群限制和兼容性
- [TKE Serverless 集群概述](../../TKEServerless集群概述/tccli%20操作.md) — 返回概述页
- [连接集群](../../TKE%20Serverless%20集群管理/连接集群/tccli%20操作.md) — 获取 kubeconfig

## 控制台替代

[TKE 控制台 → Serverless 集群 → 工作负载 → 新建](https://console.cloud.tencent.com/tke2/ecluster)：在创建工作负载页面，控制台将 Annotation 配置项转为可视化表单（CPU/内存规格、镜像缓存开关、安全组等），无需手动编写 YAML Annotation。
