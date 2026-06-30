# Pod 一直处于 ImagePullBackOff 状态（tccli）

> 对照官方：[Pod 一直处于 ImagePullBackOff 状态](https://cloud.tencent.com/document/product/457/42947) · page_id `42947`

## 概述

排查 Pod ImagePullBackOff 错误：镜像名错误、镜像不存在、仓库凭证无效、网络不通等常见原因。通过 tccli 检查集群状态，配合 kubectl 查看拉取失败的详细错误信息和节点网络状况。

> **kubectl 数据面不可达**：CAM拒绝公网端点(strategyId:240463971)，内网需VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

## 前置条件

- [环境准备](../../环境准备.md)
- 已获取集群 kubeconfig（如需 kubectl 操作）

### 环境检查

```bash
tccli --version
# 预期输出: tccli version X.X.X
```

### 资源检查

```bash
tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]'
```

**预期输出**：

```json
{
  "TotalCount": 1,
  "Clusters": [
    {
      "ClusterId": "cls-xxxxxxxx",
      "ClusterName": "my-test-cluster",
      "ClusterStatus": "Running",
      "ClusterVersion": "1.30.0",
      "ClusterType": "MANAGED_CLUSTER"
    }
  ]
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli / CLI | 幂等 |
|---|---|---|
| 查看集群状态 | `tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]'` | 是 |
| 查看节点实例 | `tccli tke DescribeClusterInstances --region ap-guangzhou --ClusterId cls-xxxxxxxx` | 是 |
| 查看 Pod Events | `kubectl describe pod <pod>`（需 VPN/IOA） | 是 |
| 查看 Pod 镜像 | `kubectl get pod <pod> -o jsonpath='{.spec.containers[*].image}'`（需 VPN/IOA） | 是 |

## 操作步骤

### 步骤1：控制面检查集群

```bash
tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]'
```

**预期输出**：

```json
{
  "Clusters": [
    {
      "ClusterId": "cls-xxxxxxxx",
      "ClusterStatus": "Running",
      "ClusterVersion": "1.30.0"
    }
  ]
}
```

### 步骤2：数据面查看拉取失败详情（需 VPN/IOA）

```bash
kubectl describe pod <pod> -n <ns> | grep -A10 Events
# 预期: Failed 事件包含具体拉取错误信息

kubectl get events -n <ns> --field-selector involvedObject.name=<pod> --sort-by='.lastTimestamp'
# 预期: 按时间排列的相关事件，BackOff 表示指数退避重试中
```

```text
NAME  STATUS  AGE
...
```

### 步骤3：按错误信息分类诊断

| Events 错误信息 | 根因 | 修复 |
|---|---|---|
| `manifest unknown` 或 `not found` | 镜像名或 Tag 拼写错误 | 确认镜像仓库中的完整路径和 Tag |
| `unauthorized: authentication required` | 私有仓库未配置拉取凭证 | 创建 imagePullSecrets 并关联到 Pod |
| `dial tcp: i/o timeout` | 节点无法访问镜像仓库网络 | 检查节点网络和安全组出站规则 |
| `x509: certificate signed by unknown authority` | 自签镜像仓库证书不受信任 | 将 CA 证书挂载到节点或配置 insecure-registries |

### 步骤4：配置私有仓库认证（需 VPN/IOA）

**创建 TCR 拉取凭证：**

```bash
kubectl create secret docker-registry tcr-secret \
  --docker-server=<RegistryId>.tencentcloudcr.com \
  --docker-username=<TCR 实例 ID 或临时账号> \
  --docker-password=<长期访问令牌> \
  -n <ns>
```

**关联到 Pod：**

```yaml
spec:
  imagePullSecrets:
  - name: tcr-secret
```

关联后删除旧 Pod 让 Deployment 重建新 Pod 触发重新拉取。

## 验证

### 控制面（tccli）

```bash
tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]'
```

**预期输出**：

```json
{
  "Clusters": [
    {
      "ClusterId": "cls-xxxxxxxx",
      "ClusterStatus": "Running"
    }
  ]
}
```

### 数据面（需 VPN/IOA）

```bash
kubectl get pod <pod> -n <ns>
# 预期: STATUS 变为 Running，不再显示 ImagePullBackOff

kubectl describe pod <pod> -n <ns> | grep -A3 "Pulled"
# 预期: Normal 事件 "Successfully pulled image"
```

```text
NAME  STATUS  AGE
...
```

## 清理

无需特殊清理（排障页）。修复后 Pod 自动恢复正常。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|---|---|---|---|
| 创建 Secret 后仍 ImagePullBackOff | `kubectl get pod <pod> -o yaml \| grep imagePullSecrets` 确认已关联 | Secret 名称写错或未在正确的命名空间 | 确认 Secret 名称和命名空间与 Pod 一致 |
| TCR 凭证提示登录失败 | 登录 TCR 控制台检查令牌状态 | 长期访问令牌已过期或被禁用 | 在 TCR 控制台重新创建访问令牌 |

### 操作成功但异常

| 现象 | 诊断 | 根因 | 修复 |
|---|---|---|---|
| 公共镜像偶尔快偶尔慢 | `kubectl describe pod` 在不同节点表现不一致 | 部分节点到 Docker Hub 的网络不稳定 | 将镜像同步到 TCR（`ccr.ccs.tencentyun.com`）使用内网加速 |
| Pod 运行正常但 Events 仍有 BackOff 记录 | 查看 Events 时间戳 | 历史记录，不影响当前状态 | 无需处理，这是修复前的旧 Events |

## 下一步

- [节点常见报错与处理](../节点常见报错与处理/tccli%20操作.md) -- page_id `89869`
- [Pod 一直处于 ContainerCreating 或 Waiting 状态](../Pod%20一直处于%20ContainerCreating%20或%20Waiting%20状态/tccli%20操作.md) -- page_id `42946`

## 控制台替代

[容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) -> 工作负载 -> Pod -> 事件 Tab。
