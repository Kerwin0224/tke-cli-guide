# 注意事项

> 对照官方：[注意事项](https://cloud.tencent.com/document/product/457/39815) · page_id `39815`

## 概述

使用 TKE Serverless 集群时需注意其与标准 TKE 的差异和限制。本文汇总关键注意事项，避免因不了解特性而导致工作负载异常。

核心差异领域：
- **无节点**：不支持 `DaemonSet`、`hostPath`、`hostNetwork`、`hostPID`、`hostIPC` 等与节点相关的特性。
- **VPC-CNI 专用**：仅支持 VPC-CNI 网络模式，不支持 GlobalRouter。Pod IP 直接来自 VPC 子网。
- **安全沙箱隔离**：每个 Pod 运行在独立的安全沙箱中，内核隔离，部分内核参数不可修改。
- **存储限制**：不支持 `hostPath` 和 `emptyDir` 挂载为内存盘。支持 CBS（块存储）、CFS（文件存储）、COS（对象存储）。
- **Pod 安全上下文**：部分 `securityContext` 选项受限（如 `privileged` 容器）。
- **镜像拉取**：不支持 TCR 插件免密拉取，必须显式配置 `imagePullSecret`。

## 前置条件

- [环境准备](../../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看集群配置信息 | `DescribeEKSClusters` | 是 |
| 查看集群网络配置 | `DescribeEKSClusters` 的 `ClusterNetworkSettings` | 是 |
| 查看 Pod 运行状态和 Events | kubectl（`kubectl describe pod`） | 是 |
| 查看集群版本 | `DescribeEKSClusters` 的 `K8SVersion` | 是 |
| 修改集群配置 | 不支持 CLI（控制台操作） | — |

## 操作步骤

### 步骤 1：检查集群配置确认当前能力边界

```bash
tccli tke DescribeEKSClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]'
# expected: exit 0，返回集群完整配置
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

**关键配置字段检查**：

```bash
# 检查集群类型（确认为 SERVERLESS）
tccli tke DescribeEKSClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]' \
    | jq '.Clusters[0].ClusterType'
# expected: "SERVERLESS_CLUSTER"

# 检查 K8s 版本（确认支持的特性范围）
tccli tke DescribeEKSClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]' \
    | jq '.Clusters[0].K8SVersion'
# expected: 版本号如 "1.30.0"

# 检查网络配置（确认 VPC-CNI）
tccli tke DescribeEKSClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]' \
    | jq '.Clusters[0].ClusterNetworkSettings'
# expected: 含 VpcId、SubnetIds
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

### 步骤 2：验证不支持的 workload 类型

> **需 kubectl 可达环境**

```bash
# 确认 DaemonSet 调度失败（Serverless 不支持 DaemonSet）
kubectl get ds -A
# expected: 不应有 DaemonSet（如有会调度失败）

# 确认无 hostPath/hostNetwork Pod
kubectl get pods -A -o json | jq '.items[] | select(.spec.hostNetwork == true)'
# expected: 无输出（不支持 hostNetwork）
```

### 步骤 3：关键差异速查

| 特性 | 标准 TKE | TKE Serverless | 注意事项 |
|------|---------|---------------|---------|
| DaemonSet | 支持 | **不支持** | 如需日志采集，使用 CRD 方案或 Sidecar |
| hostPath / hostNetwork / hostPID | 支持 | **不支持** | 使用 PVC/CFS 替代 hostPath；使用 Service 替代 hostNetwork |
| privileged 容器 | 支持 | **部分支持**（需申请加白） | 默认不允许，需提工单申请 |
| 网络模式 | GlobalRouter / VPC-CNI | **仅 VPC-CNI** | Pod IP 直接来自 VPC 子网，需确保子网 IP 充足 |
| TCR 免密拉取 | 支持插件 | **不支持** | 必须手动创建 `imagePullSecret` |
| DNS 配置 | 可修改 CoreDNS | **不可修改 CoreDNS 配置** | 通过 `dnsConfig` 或 `dnsPolicy` 在 Pod 级别调整 |
| Pod 安全上下文 | 完整支持 | 部分受限 | 不支持 `sysctl` 修改内核参数，不支持 `capabilities` 中的部分 CAP |
| 存储 | 全部类型 | CBS / CFS / COS | 不支持 `hostPath`、`emptyDir`（memory medium） |
| GPU | 支持 | 支持（部分机型） | GPU Pod 调度受资源池限制 |

### 步骤 4：Pod spec 编写注意事项

编写 TKE Serverless Pod spec 时应遵循：

```yaml
# 正确示例：Serverless 兼容的 Pod spec
apiVersion: v1
kind: Pod
metadata:
  name: serverless-compatible
spec:
  # ✅ 不设置 hostNetwork/hostPID/hostIPC
  # ✅ 不设置 privileged: true（除非已加白）
  containers:
  - name: app
    image: nginx:latest
    resources:
      requests:
        cpu: "500m"
        memory: "512Mi"
      limits:
        cpu: "1"
        memory: "1Gi"
    # ✅ volumeMounts 使用 PVC 而非 hostPath
  imagePullSecrets:
  - name: image-pull-secret  # ✅ 显式指定（TCR 私有镜像）
  # ✅ dnsPolicy: ClusterFirst（默认，推荐）
```

```yaml
# 错误示例：不兼容的配置
apiVersion: v1
kind: Pod
metadata:
  name: incompatible
spec:
  hostNetwork: true       # ❌ 不支持
  hostPID: true           # ❌ 不支持
  containers:
  - name: app
    securityContext:
      privileged: true    # ❌ 不支持（默认）
    volumeMounts:
    - name: local-storage
      mountPath: /data
  volumes:
  - name: local-storage
    hostPath:             # ❌ 不支持
      path: /tmp
  nodeSelector:
    kubernetes.io/hostname: node-1  # ❌ Serverless 无节点
```

## 验证

```bash
# 验证集群类型正确
tccli tke DescribeEKSClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]' \
    | jq '.Clusters[0].ClusterType'
# expected: "SERVERLESS_CLUSTER"

# 验证网络模式为 VPC-CNI（通过检查 SubnetIds）
tccli tke DescribeEKSClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]' \
    | jq '.Clusters[0].SubnetIds'
# expected: 包含至少一个子网 ID，确认 VPC-CNI 启用

# 验证 Pod 不包含不兼容字段（kubectl 环境）
kubectl get pods POD_NAME -n NAMESPACE -o json | jq '.spec | {hostNetwork, hostPID, hostIPC}'
# expected: 均为 null 或 false
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

## 清理

本页面为只读操作，无需清理。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Pod 创建后一直 `Pending` | `kubectl describe pod POD_NAME` 查看 Events | Pod spec 包含不支持的特性（如 `hostNetwork: true`） | 移除不兼容字段，参考步骤 4 的正确示例 |
| Pod `CrashLoopBackOff` 且日志提示 `Operation not permitted` | `kubectl logs POD_NAME` 查看日志 | Pod 使用了受限的 `securityContext`（如 `privileged: true` 或受限 `capabilities`） | 移除 `privileged: true` 和受限 `capabilities`；如确需使用提工单申请加白 |
| `kubectl apply -f daemonset.yaml` 创建成功但无 Pod | `kubectl get ds -o yaml` 查看 DaemonSet 状态 | Serverless 不支持 DaemonSet，无节点可调度 | 换用 Sidecar 模式或 CRD 方案 |
| `ImagePullBackOff`（TCR 私有镜像） | `kubectl describe pod POD_NAME` 查看 Events | 未配置 `imagePullSecret` 或 Secret 无效 | 创建 `imagePullSecret` 并在 Pod spec 中显式引用 `imagePullSecrets` |
| PVC 挂载后 Pod 一直 `ContainerCreating` | `kubectl describe pod POD_NAME` 查看 Events | 使用了不支持的卷类型（如 `hostPath`）或 PVC 未绑定 | 检查卷类型是否支持；确认 PVC 已 `Bound` |

### 操作成功但异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Pod 正常运行但网络不通 | `kubectl exec POD_NAME -- ping TARGET_IP` 测试 | VPC-CNI 模式下子网 IP 耗尽或安全组规则拦截 | 检查子网 IP 余量；检查安全组出站规则 |
| Pod 中手动修改内核参数无效 | `kubectl exec POD_NAME -- sysctl -w net.core.somaxconn=65535` | Serverless 安全沙箱禁用了 `sysctl` 部分参数 | 部分参数不可修改，此为设计限制。如需调整内核参数，改用标准 TKE 集群 |

## 下一步

- [创建 Serverless 集群](../创建集群/tccli%20操作.md) — 创建符合规范的集群
- [连接集群](../连接集群/tccli%20操作.md) — 获取 kubeconfig
- [Annotation 说明](../../Kubernetes%20对象管理/Annotation%20说明/tccli%20操作.md) — 了解 Serverless 特有的 Pod Annotation
- [Serverless 集群全局配置说明](../../Kubernetes%20对象管理/Serverless%20集群全局配置说明/tccli%20操作.md) — ConfigMap 配置

## 控制台替代

[TKE 控制台 → Serverless 集群](https://console.cloud.tencent.com/tke2/ecluster)：集群详情页展示集群配置，创建工作负载时控制台自动校验不兼容配置并给出提示。
