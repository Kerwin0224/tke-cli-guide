# 常见问题

> 对照官方：[常见问题](https://cloud.tencent.com/document/product/457/54778) · page_id `54778`

## 概述

本文汇总 TKE Serverless (EKS) 集群的常见问题，涵盖 Pod 调度、网络、计费、资源限制、存储等方面。通过 tccli 和 kubectl 命令可自助排查大多数问题，避免频繁登录控制台。所有问题均提供命令行诊断方法和修复建议。

**核心排查工具**：
- `tccli tke DescribeEKSClusters` — 集群状态
- `kubectl describe pod` — Pod 详情和 Events
- `kubectl get events` — 集群事件
- `kubectl logs` — Pod 日志

## 前置条件

- [环境准备](../../../环境准备.md)

### 环境检查

```bash
tccli --version
# expected: tccli version >= 1.0.0

tccli configure list
# expected: secretId, secretKey, region 均已配置

tccli tke DescribeEKSClusters --region <Region>
# expected: exit 0，返回集群列表
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

### 通用排障流程

```bash
# Step 1: 检查集群状态
tccli tke DescribeEKSClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterStatus: Running

# Step 2: 检查 Pod 状态和 Events
kubectl --kubeconfig ~/.kube/config-eks-CLUSTER_ID get pods -n NAMESPACE
kubectl --kubeconfig ~/.kube/config-eks-CLUSTER_ID describe pod POD_NAME -n NAMESPACE

# Step 3: 查看集群 Events
kubectl --kubeconfig ~/.kube/config-eks-CLUSTER_ID get events -n NAMESPACE --sort-by='.lastTimestamp'
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

| 控制台操作 | tccli / kubectl 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看集群状态 | `tccli tke DescribeEKSClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'` | 是 |
| 查看 Pod 详情 | `kubectl describe pod POD_NAME -n NAMESPACE` | 是 |
| 查看集群事件 | `tccli tke DescribeEKSClusterEvent --region <Region> --ClusterId CLUSTER_ID` | 是 |
| 查看 Pod 日志 | `kubectl logs POD_NAME -n NAMESPACE` | 是 |
| 进入容器调试 | `kubectl exec -it POD_NAME -n NAMESPACE -- /bin/sh` | 否 |

## 操作步骤

### 问题 1: Pod 一直处于 Pending 状态

#### 现象

`kubectl get pods` 显示 Pod STATUS 为 Pending，且持续超过 2 分钟。

#### 诊断

```bash
# 查看 Pod Events，定位 Pending 原因
kubectl --kubeconfig ~/.kube/config-eks-CLUSTER_ID describe pod POD_NAME -n NAMESPACE | tail -20
# expected: Events 区域显示调度失败原因
```

#### 常见原因与修复

| 原因 | Events 关键字 | 修复方式 |
|------|-------------|---------|
| 资源配置不合法 | `Invalid cpu/mem annotation` | 检查 `eks.tke.cloud.tencent.com/cpu` 和 `eks.tke.cloud.tencent.com/mem` annotation 规格是否在支持范围内 |
| 资源配额不足 | `exceeded quota` | `kubectl describe resourcequota -n NAMESPACE` 检查配额；调整配额或减少 Pod 数 |
| 子网 IP 耗尽 | `no available ip in subnet` | `tccli vpc DescribeSubnets --region <Region> --SubnetIds '["SUBNET_ID"]'` 检查子网可用 IP 数 |
| 镜像拉取失败 | `Failed to pull image` | 参见问题 2 |

```bash
# 快速验证：确认注解 CPU/内存规格
kubectl --kubeconfig ~/.kube/config-eks-CLUSTER_ID get pod POD_NAME -n NAMESPACE -o jsonpath='{.metadata.annotations}'
# expected: 包含 eks.tke.cloud.tencent.com/cpu 和 eks.tke.cloud.tencent.com/mem
```

### 问题 2: Pod 处于 ImagePullBackOff 或 ErrImagePull

#### 现象

Pod STATUS 显示 `ImagePullBackOff` 或 `ErrImagePull`。

#### 诊断

```bash
kubectl --kubeconfig ~/.kube/config-eks-CLUSTER_ID describe pod POD_NAME -n NAMESPACE | grep -A10 Events
# expected: Events 显示具体的拉取错误
```

#### 常见原因与修复

| 原因 | 诊断命令 | 修复方式 |
|------|---------|---------|
| 镜像名称错误 | `kubectl describe pod POD_NAME -n NAMESPACE \| grep Image:` | 纠正镜像名称和 tag |
| 私有镜像缺少 imagePullSecret | `kubectl get pod POD_NAME -n NAMESPACE -o jsonpath='{.spec.imagePullSecrets}'` | 创建 docker-registry Secret 并在 Pod spec 中引用 |
| Registry 不可达 | `kubectl run -it --rm debug --image=busybox:1.36 -n NAMESPACE -- nslookup REGISTRY_HOST` | 检查网络连通性；确认 registry 域名可解析 |
| TCR 凭证过期 | 在 TCR 控制台检查凭证有效期 | 刷新 TCR 长期访问凭证，更新 Secret |

```bash
# 创建 TCR 拉取凭证 Secret（若缺少）
kubectl --kubeconfig ~/.kube/config-eks-CLUSTER_ID create secret docker-registry tcr-secret \
    --docker-server=ccr.ccs.tencentyun.com \
    --docker-username=YOUR_TCR_USERNAME \
    --docker-password=YOUR_TCR_PASSWORD \
    -n NAMESPACE
# expected: secret/tcr-secret created
```

### 问题 3: Pod 频繁重启（CrashLoopBackOff）

#### 现象

Pod STATUS 显示 `CrashLoopBackOff`，RESTARTS 列数值持续增加。

#### 诊断

```bash
# 查看 Pod 日志（当前和上一次）
kubectl --kubeconfig ~/.kube/config-eks-CLUSTER_ID logs POD_NAME -n NAMESPACE
kubectl --kubeconfig ~/.kube/config-eks-CLUSTER_ID logs POD_NAME -n NAMESPACE --previous
# expected: 日志显示应用退出原因

# 查看退出码
kubectl --kubeconfig ~/.kube/config-eks-CLUSTER_ID describe pod POD_NAME -n NAMESPACE | grep -E 'Exit Code|Reason'
```

#### 常见原因与修复

| 退出码 | 常见原因 | 修复方式 |
|--------|---------|---------|
| `0` | 应用正常退出（非持续运行） | 检查启动命令；主进程不应退出 |
| `1` | 应用错误退出 | 查看日志定位应用层错误 |
| `137` | OOMKilled — 内存不足 | 增加 `eks.tke.cloud.tencent.com/mem` annotation 值 |
| `143` | SIGTERM — 被优雅终止 | 检查 readiness/liveness probe 配置，调整 `terminationGracePeriodSeconds` |

### 问题 4: Service LoadBalancer EXTERNAL-IP 持续 Pending

#### 现象

`kubectl get svc` 显示 LoadBalancer 类型 Service 的 EXTERNAL-IP 为 `<pending>` 超过 3 分钟。

#### 诊断

```bash
kubectl --kubeconfig ~/.kube/config-eks-CLUSTER_ID describe svc SERVICE_NAME -n NAMESPACE | grep -A15 Events
# expected: Events 显示 CLB 创建失败的具体原因
```

#### 常见原因与修复

| Events 关键字 | 原因 | 修复方式 |
|-------------|------|---------|
| `insufficient ip in subnet` | 内网 CLB 子网 IP 耗尽 | 选择 IP 充足的子网，或释放不再使用的内网 CLB |
| `quota exceeded` | CLB 配额不足 | `tccli clb DescribeQuota --region <Region>` 检查配额；释放无用的 CLB |
| `subnet not found` | annotation 中的子网 ID 不存在 | `tccli vpc DescribeSubnets --region <Region>` 验证子网 ID |
| `VpcId not match` | 子网所属 VPC 与集群 VPC 不一致 | 确认子网属于集群 VPC：`tccli tke DescribeEKSClusters --region <Region> --ClusterIds '["CLUSTER_ID"]' \| jq '.Clusters[0].VpcId'` |

### 问题 5: HPA 不工作或扩缩容异常

#### 现象

HPA 配置了但 `TARGETS` 显示 `<unknown>`，或副本数不按预期变化。

#### 诊断

```bash
# 检查 HPA 状态
kubectl --kubeconfig ~/.kube/config-eks-CLUSTER_ID describe hpa HPA_NAME -n NAMESPACE | grep -A10 Conditions
# expected: AbleToScale: True, ScalingActive: True

# 检查 metrics-server
kubectl --kubeconfig ~/.kube/config-eks-CLUSTER_ID get deployment metrics-server -n kube-system
# expected: READY 1/1

# 检查 Pod 资源使用是否可采集
kubectl --kubeconfig ~/.kube/config-eks-CLUSTER_ID top pods -n NAMESPACE
# expected: 显示 CPU 和 MEMORY 数值
```

#### 修复

```bash
# 若 metrics-server 异常，重启
kubectl --kubeconfig ~/.kube/config-eks-CLUSTER_ID rollout restart deployment/metrics-server -n kube-system
# expected: deployment.apps/metrics-server restarted

# 确认 Pod 有资源注解
kubectl --kubeconfig ~/.kube/config-eks-CLUSTER_ID get pods -n NAMESPACE -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.metadata.annotations.eks\.tke\.cloud\.tencent\.com/cpu}{"\n"}{end}'
# expected: 每个 Pod 显示对应的 CPU 规格
```

### 问题 6: 集群状态异常（Abnormal）

#### 诊断

```bash
tccli tke DescribeEKSClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]' | jq '.Clusters[0] | {ClusterStatus, ClusterName, CreatedTime}'
# expected: ClusterStatus: Running 为正常，Abnormal 需进一步排查

# 查看集群事件
tccli tke DescribeEKSClusterEvent --region <Region> \
    --ClusterId CLUSTER_ID | jq '.Events[:10]'
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

#### 修复

| 状态 | 含义 | 操作 |
|------|------|------|
| `Abnormal` | 集群异常，可能是底层组件故障 | 登录控制台查看详细状态；若持续 Abnormal，联系技术支持 |
| `Creating` | 创建中 | 正常，等待 2-5 分钟 |
| `Deleting` | 删除中 | 正常，等待完成 |

### 问题 7: 计费相关

#### 如何查看 Pod 计费规格

Serverless 集群按 Pod 声明的 CPU/内存 annotation 计费，而非实际使用量：

```bash
# 查看 Pod 计费规格
kubectl --kubeconfig ~/.kube/config-eks-CLUSTER_ID get pod POD_NAME -n NAMESPACE -o json | \
    jq '{name: .metadata.name, cpu: .metadata.annotations["eks.tke.cloud.tencent.com/cpu"], mem: .metadata.annotations["eks.tke.cloud.tencent.com/mem"]}'
# expected: 显示声明规格（非实际使用量）
```

#### 如何降低 Serverless 成本

1. 使用适当规格的 CPU/内存 annotation，避免过度分配
2. 使用 CronJob 替代常驻 Pod 处理定时任务
3. 配置 ResourceQuota 限制命名空间总资源
4. 设置 HPA minReplicas=0（需支持 scale to zero 的场景）

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `NAMESPACE` | Kubernetes 命名空间 | 需预先存在 | `kubectl get ns` |
| `CLUSTER_ID` | 集群 ID | `cls-xxxxxxxx` 格式 | `tccli tke DescribeEKSClusters --region <Region>` |
| `REGION` | 地域 | 如 `ap-guangzhou` | `tccli configure list` |
| `POD_NAME` | Pod 名称 | 从 `kubectl get pods` 获取 | `kubectl get pods -n NAMESPACE` |
| `SUBNET_ID` | 子网 ID | 集群创建时指定的子网 | `tccli tke DescribeEKSClusters --region <Region> --ClusterIds '["CLUSTER_ID"]' \| jq '.Clusters[0].SubnetId'` |

## 验证

```bash
# 验证集群健康状态
tccli tke DescribeEKSClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]' | jq '.Clusters[0].ClusterStatus'
# expected: "Running"

# 验证所有 Pod 健康
kubectl --kubeconfig ~/.kube/config-eks-CLUSTER_ID get pods -A \
    --field-selector=status.phase!=Running,status.phase!=Succeeded
# expected: No resources found（仅显示非 Running 和非 Completed 的 Pod）
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

> **警告**：本文档为 FAQ 诊断指南，无独立资源需清理。但在排障过程中创建的临时调试 Pod（如 `kubectl run debug`）应及时清理，避免产生费用。

```bash
# 清理临时调试 Pod
kubectl --kubeconfig ~/.kube/config-eks-CLUSTER_ID delete pod debug -n NAMESPACE --ignore-not-found
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DescribeEKSClusters` 返回 `UnauthorizedOperation` | 检查 CAM 策略 | 缺少 `tke:DescribeEKSClusters` 权限 | 联系管理员授权 |
| `DescribeEKSClusterEvent` 返回空 | 检查集群是否为 Running 状态 | 非 Running 集群可能不产生新事件 | 确认集群状态；最近无操作则无事件为正常 |
| `kubectl logs` 返回 `previous terminated container not found` | 查看 Pod RESTARTS 次数 | Pod 尚未重启，无上一次日志 | 使用不带 `--previous` 的 kubectl logs |
| `kubectl exec` 返回 `unable to upgrade connection` | 检查 `kubectl cluster-info` | API Server 连接异常 | 检查网络和 kubeconfig 有效性 |

### 操作成功但异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 所有 Pod Running 但服务不可达 | `kubectl get endpoints -n NAMESPACE` 检查 Endpoints | Endpoints 为空或不匹配 | 检查 Service selector 与 Pod labels 匹配 |
| Pod 日志持续输出错误 | `kubectl logs POD_NAME -n NAMESPACE --tail=50` | 应用层错误或配置问题 | 根据日志定位应用问题；ConfigMap 更新需重启 Pod |
| `kubectl describe pod` 无 Events | 检查 Pod 创建时间 | Events 可能已过期被清理 | 使用 `kubectl get events -n NAMESPACE` 查看全量 Events |

## 下一步

- [监控和告警](../运维中心/监控和告警/tccli%20操作.md) — 配置集群监控和 HPA
- [日志采集](../运维中心/日志采集/tccli%20操作.md) — 将日志采集到 CLS
- [工作负载管理](../Kubernetes%20对象管理/工作负载管理/tccli%20操作.md) — 管理工作负载
- [服务管理](../Kubernetes%20对象管理/服务管理/tccli%20操作.md) — 管理 Service

## 控制台替代

控制台：容器服务控制台 → 集群 → 选择目标 Serverless 集群 → 事件/监控/日志 页面查看详情。
