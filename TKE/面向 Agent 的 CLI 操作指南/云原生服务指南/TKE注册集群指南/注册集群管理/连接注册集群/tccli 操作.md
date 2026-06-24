# 连接注册集群

> 对照官方：[连接注册集群](https://cloud.tencent.com/document/product/457/63216) · page_id `63216`

## 概述

通过 `tccli tke DescribeExternalClusterCredential` 获取已注册的外部集群的 kubeconfig 凭证，配置到本地 kubectl，实现从管理节点直接访问外部集群。注册集群的 kubeconfig 指向外部集群自身的 API Server（非 TKE 托管控制面），因此需确保管理节点与外部集群 API Server 网络可达。

操作分为两步：**tccli** 从 TKE 管理面获取凭证，**kubectl** 在本地管理节点验证连通性。

## 前置条件

- [环境准备](../../../../环境准备.md)
- 已完成 [创建注册集群](../创建注册集群/tccli%20操作.md)，TKE 管理面有可用的注册集群条目

### 环境检查

```bash
tccli --version
# expected: tccli version >= 1.0.0

tccli configure list
# expected: secretId, secretKey, region 均已配置

tccli tke DescribeExternalClusters --region <Region>
# expected: exit 0，返回至少 1 个注册集群
```

```json
{"RequestId": "..."}
```

### 资源检查

```bash
# 确认 kubectl 已安装
kubectl version --client --output yaml
# expected: 返回 clientVersion

# 确认当前 kubeconfig 状态
kubectl config view
# expected: 显示当前已有上下文（clusters / contexts / users）
```

### 连通性检查

```bash
# 在外部集群 API Server 可达的前提下，才能验证 kubectl 连接
# 若外部集群 API Server 在 IDC 内网，需先建立 VPN/专线
# TKE 管理面的凭证获取不依赖外部集群可达性
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看注册集群列表 | `tccli tke DescribeExternalClusters --region <Region>` | 是（只读） |
| 获取集群凭证 | `tccli tke DescribeExternalClusterCredential --region <Region> --ClusterId CLUSTER_ID` | 是（每次返回当前有效凭证） |
| 配置 kubeconfig | `kubectl config set-cluster / set-credentials / set-context` | 是（覆盖） |
| 验证连接 | `kubectl cluster-info` | 是（只读） |

## 操作步骤

### 步骤1：确认目标注册集群

```bash
tccli tke DescribeExternalClusters --region <Region> --output json \
  --filter "ClusterSet[?ClusterId=='CLUSTER_ID'] | [0].{ClusterId:ClusterId,ClusterName:ClusterName,ClusterStatus:ClusterStatus}"
# expected: ClusterStatus 为 Normal 或 Running
```

```json
{"RequestId": "..."}
```

### 步骤2：获取注册集群 kubeconfig 凭证

```bash
tccli tke DescribeExternalClusterCredential --region <Region> \
  --ClusterId CLUSTER_ID --output json
# expected: exit 0，返回 Kubeconfig 字段（Base64 编码）
```

预期输出：

```json
{
    "Kubeconfig": "YXBpVmVyc2lvbjogdjEKY2x1c3RlcnM6Ci0gY2x1c3RlcjoKICAgIGNlcnRpZmljYXRlLWF1...",
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

### 步骤3：解码并保存 kubeconfig

```bash
tccli tke DescribeExternalClusterCredential --region <Region> \
  --ClusterId CLUSTER_ID --output json \
  | jq -r '.Kubeconfig' \
  | base64 -d > ~/.kube/config-CLUSTER_ID
# expected: 生成 kubeconfig 文件（无报错即成功）
```

```json
{"RequestId": "..."}
```

验证 kubeconfig 文件可解析：

```bash
KUBECONFIG=~/.kube/config-CLUSTER_ID kubectl config view
# expected: 显示 clusters、contexts、users 配置
```

### 步骤4：合并到默认 kubeconfig（可选）

```bash
# 备份当前 kubeconfig
cp ~/.kube/config ~/.kube/config.bak.$(date +%Y%m%d)

# 合并新集群配置
KUBECONFIG=~/.kube/config:~/.kube/config-CLUSTER_ID \
  kubectl config view --flatten > ~/.kube/config.merged

# 替换原配置（确认内容正确后）
mv ~/.kube/config.merged ~/.kube/config
# expected: 合并完成
```

### 步骤5：验证 kubectl 连接外部集群

```bash
# 方式1：使用独立 kubeconfig 文件
KUBECONFIG=~/.kube/config-CLUSTER_ID kubectl cluster-info
# expected: Kubernetes control plane is running at https://<external-apiserver>:6443

KUBECONFIG=~/.kube/config-CLUSTER_ID kubectl get nodes
# expected: 返回外部集群节点列表
```

```text
NAME  STATUS  AGE
...
```

```bash
# 方式2：切换 kubectl context（若已合并到默认 config）
kubectl config get-contexts
# expected: 列表中包含新集群的 context

kubectl config use-context <external-cluster-context>
# expected: Switched to context "<external-cluster-context>"

kubectl cluster-info
# expected: 显示外部集群控制面地址
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `REGION` | 地域 | 如 `ap-guangzhou` | `tccli configure list` |
| `CLUSTER_ID` | 注册集群 ID | 格式 `cls-xxxxxxxx` | `tccli tke DescribeExternalClusters --region <Region>` |

## 验证

### 凭证有效性验证

```bash
# 确认 kubeconfig 文件非空且可解析
KUBECONFIG=~/.kube/config-CLUSTER_ID kubectl config view --minify
# expected: 返回单集群配置，含 server 地址和证书信息
```

### 控制面连接验证（需外部集群 API Server 可达）

```bash
KUBECONFIG=~/.kube/config-CLUSTER_ID kubectl cluster-info
# expected: Kubernetes control plane is running at https://...
```

成功返回集群信息。若返回 `Unable to connect to the server`，检查管理节点到外部集群 API Server 的网络可达性（VPN/专线/公网）。

### 数据面访问验证（需外部集群 API Server 可达）

```bash
KUBECONFIG=~/.kube/config-CLUSTER_ID kubectl api-resources
# expected: 返回外部集群全部 API 资源列表

KUBECONFIG=~/.kube/config-CLUSTER_ID kubectl get namespaces
# expected: 返回命名空间列表，含 tke-xxx（agent 所在命名空间）
```

```text
NAME  STATUS  AGE
...
```

## 清理

> **注意**：清理仅移除本地 kubeconfig 文件，不影响外部集群和 TKE 管理面关联关系。

### 删除独立 kubeconfig 文件

```bash
rm ~/.kube/config-CLUSTER_ID
# expected: 文件已删除
```

### 从合并后的 config 中移除集群（若执行了步骤4）

```bash
# 删除 context
kubectl config delete-context <external-cluster-context>

# 删除 cluster 条目
kubectl config delete-cluster <external-cluster-name>

# 删除 user 条目
kubectl config unset users.<external-user-name>
# expected: 各项删除成功
```

> **提醒**：如需完全解除 TKE 注册关联，请参见 [解除注册集群](../解除注册集群/tccli%20操作.md)。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DescribeExternalClusterCredential` 返回 `ResourceNotFound` | `DescribeExternalClusters` 确认集群存在 | 集群 ID 错误或已解除注册 | 确认正确的 `ClusterId` 和 `--region` |
| `DescribeExternalClusterCredential` 返回 `AuthFailure` | `tccli configure list` 检查凭据 | CAM 权限不足 | 授予 `tke:DescribeExternalClusterCredential` 权限 |
| `kubectl config view` 报错 `error: no configuration` | `cat ~/.kube/config-CLUSTER_ID` 检查文件内容 | kubeconfig 文件为空或格式异常 | 重新获取凭证，确认 Base64 解码正确 |
| `jq` 报 `null` | `tccli tke DescribeExternalClusterCredential ... --output json` 查看原始输出 | `Kubeconfig` 字段名可能不是 `Kubeconfig` | 查看原始 JSON 输出，确认正确的字段名 |
| `kubectl config use-context` 报 `no context exists` | `kubectl config get-contexts` 确认 context 名 | context 名输入错误 | 复制确切的 context 名称 |

### 操作成功但异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| kubectl 返回 `Unable to connect to the server` | `curl -k https://<apiserver>:6443` 测试连通性 | 管理节点到外部集群 API Server 网络不通 | 建立 VPN/专线，或将管理节点部署在同一网络 |
| kubectl 返回 `certificate signed by unknown authority` | `kubectl config view --minify` 检查证书 | kubeconfig 中 CA 证书不匹配或过期 | 重新获取凭证（`DescribeExternalClusterCredential`），替换 kubeconfig |
| kubectl 返回 `x509: certificate has expired` | 检查外部集群 API Server 证书有效期 | 外部集群自身证书过期 | 在外部集群续期 API Server 证书，然后重新获取 kubeconfig |
| kubeconfig 中 server 地址为内网 IP，管理节点无法访问 | 检查 kubeconfig 中的 `server` 字段 | 外部集群 API Server 仅绑定内网地址 | 确认管理节点可通过 VPN/专线访问该内网地址 |

## 下一步

- [解除注册集群](../解除注册集群/tccli%20操作.md) — 解除 TKE 管理面关联（page_id `63219`）
- [日志采集](../../运维指南/日志采集/tccli%20操作.md) — 配置外部集群日志采集（page_id `72148`）
- [集群审计](../../运维指南/集群审计/tccli%20操作.md) — 启用审计日志（page_id `72149`）

## 控制台替代

[容器服务控制台 → 集群列表](https://console.cloud.tencent.com/tke2/external) 选中目标注册集群，点击 **基本信息 → 集群凭证 → 复制 kubeconfig** 或 **下载**。
