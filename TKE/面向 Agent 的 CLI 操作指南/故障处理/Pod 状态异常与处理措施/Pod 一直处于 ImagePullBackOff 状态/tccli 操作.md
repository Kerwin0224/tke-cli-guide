# Pod 一直处于 ImagePullBackOff 状态（tccli）

> 对照官方：[Pod 一直处于 ImagePullBackOff 状态](https://cloud.tencent.com/document/product/457/42947) · page_id `42947`

## 概述

`ImagePullBackOff` 表示 kubelet 无法拉取指定镜像。常见原因：镜像名错误、仓库认证失败、网络不通、私有仓库未配置 imagePullSecrets。通过 tccli 检查集群状态，配合 kubectl 查看拉取失败的错误信息。

> **kubectl 数据面不可达**：CAM拒绝公网端点(strategyId:240463971)，内网需VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

## 前置条件

- [环境准备](../../../环境准备.md)
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
| 查看 Pod Events | `kubectl describe pod <pod> \| grep -A10 Events`（需 VPN/IOA） | 是 |
| 查看镜像名 | `kubectl get pod <pod> -o jsonpath='{.spec.containers[*].image}'`（需 VPN/IOA） | 是 |
| 配置拉取凭证 | `kubectl create secret docker-registry <name> --docker-server=...`（需 VPN/IOA） | 否 |

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
kubectl describe pod <pod-name> -n <ns> | grep -A10 Events
# 预期: Failed 事件包含具体错误信息

kubectl get pod <pod-name> -n <ns> -o jsonpath='{.spec.containers[*].image}'
# 预期: 确认 Pod 引用的完整镜像名
```

```text
NAME  STATUS  AGE
...
```

### 步骤3：按错误信息分类诊断

| 错误信息 | 根因 | 修复 |
|---|---|---|
| `manifest unknown` | 镜像名或 Tag 拼写错误 | 确认镜像仓库中的正确路径和 Tag |
| `unauthorized: authentication required` | 私有仓库未配置拉取凭证 | 创建 imagePullSecrets 并关联到 Pod 或 ServiceAccount |
| `dial tcp: i/o timeout` | 节点无法访问镜像仓库网络 | 检查节点到仓库的网络连通性和安全组出站规则 |
| `x509: certificate signed by unknown authority` | 自签仓库证书不受信任 | 在节点上安装 CA 证书或配置 insecure-registries |

### 步骤4：配置 TCR 私有仓库认证（需 VPN/IOA）

```bash
kubectl create secret docker-registry tcr-secret \
  --docker-server=<RegistryId>.tencentcloudcr.com \
  --docker-username=<TCR 用户名> \
  --docker-password=<长期访问令牌> \
  -n <ns>
```

在 Deployment 中关联：

```yaml
spec:
  template:
    spec:
      imagePullSecrets:
      - name: tcr-secret
```

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
# 预期: STATUS Running，不再显示 ImagePullBackOff

kubectl describe pod <pod> -n <ns> | grep "Pulled"
# 预期: Normal 事件 "Successfully pulled image"
```

```text
NAME  STATUS  AGE
...
```

## 清理

无需特殊清理（排障页）。修复后 Pod 自动恢复。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|---|---|---|---|
| 创建 Secret 后仍失败 | `kubectl get pod <pod> -o yaml` 确认已关联 | Secret 名称或命名空间错误 | 确认 Secret 在 Pod 所在命名空间，名称拼写正确 |
| TCR 令牌过期 | TCR 控制台检查令牌状态 | 长期令牌设置了过期时间 | 重新生成 TCR 长期访问令牌并更新 Secret |
| 节点拉取速度极慢 | SSH 到节点 `time crictl pull <image>` | 镜像过大或网络带宽限制 | 使用 TCR 内网加速；镜像分层优化 |

### 操作成功但异常

| 现象 | 诊断 | 根因 | 修复 |
|---|---|---|---|
| 部分节点可拉取但部分失败 | 检查问题节点的网络和 DNS | 特定节点安全组或 DNS 配置异常 | 对比正常节点和异常节点的网络配置 |
| Pod 运行正常但 Events 仍显示历史 BackOff | 查看 Events 时间戳 | 历史记录，修复前的旧事件 | 无需处理 |

## 下一步

- [容器进程主动退出](../容器进程主动退出/tccli%20操作.md) -- page_id `43148`
- [Pod 健康检查失败](../Pod%20健康检查失败/tccli%20操作.md) -- page_id `43129`

## 控制台替代

[容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) -> 工作负载 -> Pod -> 事件 Tab。
