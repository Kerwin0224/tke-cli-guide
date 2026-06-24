# 连接注册集群（tccli）

> 对照官方：[连接注册集群](https://cloud.tencent.com/document/product/457/63216) · page_id `63216`

## 概述

获取已纳管注册集群的 kubeconfig 凭证，通过 kubectl 连接并操作集群。注册集群支持两种接入端点：**公网访问**（Internet）和**内网访问**（VPC 内网）。本指南覆盖端点管理、kubeconfig 获取及连接验证全流程。

## 前置条件

- 注册集群已成功创建并状态为 `Running`（参见 [创建注册集群](../创建注册集群/tccli%20操作.md)）。
- 本地已安装 `kubectl`（版本建议 >= 1.18 且与集群版本偏差 <= 1 个小版本）。
- 若需内网访问，客户端必须在注册集群所在 VPC 或已通过云联网/专线互通。
- 已配置 `tccli` 凭证与目标地域。

## 控制台与 CLI 参数映射

| 控制台 | tccli | 幂等 |
|--------|-------|------|
| 查看集群 kubeconfig | `DescribeClusterKubeconfig` | 是 |
| 开启公网访问端点 | `CreateClusterEndpoint` + `IsExtranet=true` | 否（重复调用视为更新） |
| 开启内网访问端点 | `CreateClusterEndpoint` + `IsExtranet=false` | 否 |
| 查看端点状态 | `DescribeClusterEndpointStatus` | 是 |
| 关闭端点 | `DeleteClusterEndpoint` | 否 |

## 操作步骤

### 1. 检查端点状态

确认注册集群的公网/内网访问端点是否已开启：

```bash
tccli tke DescribeClusterEndpointStatus \
  --ClusterId cls-registered-example \
  --IsExtranet true \
  --region ap-guangzhou \
  --output json
```

```json
{
  "Status": "Created",
  "ErrorMessage": "",
  "RequestId": "abc12345-6789-0abc-def1-234567890abc"
}
```

`Status` 取值：`Creating`（创建中）、`Created`（已就绪）、`Deleting`（删除中）、`NotFound`（未开启）。

同时检查内网端点：

```bash
tccli tke DescribeClusterEndpointStatus \
  --ClusterId cls-registered-example \
  --IsExtranet false \
  --region ap-guangzhou \
  --output json
```

```json
{
  "Status": "<Status>",
  "ErrorMsg": "<ErrorMsg>",
  "RequestId": "<RequestId>"
}
```

### 2. 开启访问端点（如未开启）

若端点状态为 `NotFound`，需先开启：

**公网访问**（适合开发/测试，通过互联网连接）：

```bash
tccli tke CreateClusterEndpoint \
  --ClusterId cls-registered-example \
  --IsExtranet true \
  --SubnetId subnet-hub-gz-01 \
  --SecurityGroup sg-hub-gz-001 \
  --region ap-guangzhou \
  --output json
```

```json
{
  "Response": {
    "RequestId": "def67890-1234-5678-90ab-cdef01234567"
  }
}
```

**内网访问**（适合生产，通过 VPC 内部连接）：

```bash
tccli tke CreateClusterEndpoint \
  --ClusterId cls-registered-example \
  --IsExtranet false \
  --SubnetId subnet-hub-gz-01 \
  --SecurityGroup sg-hub-gz-001 \
  --region ap-guangzhou \
  --output json
```

等待端点就绪后继续。

### 3. 获取 kubeconfig

```bash
tccli tke DescribeClusterKubeconfig \
  --Clus

```json
{
  "Kubeconfig": "<Kubeconfig>",
  "RequestId": "<RequestId>"
}
```terId cls-registered-example \
  --IsExtranet true \
  --region ap-guangzhou \
  --output json
```

```json
{
  "Kubeconfig": "apiVersion: v1\nclusters:\n- cluster:\n    certificate-authority-data: LS0tLS1CRUdJTi...\n    server: https://cls-registered-example.ccs.tencent-cloud.com\n  name: cls-registered-example\ncontexts:\n- context:\n    cluster: cls-registered-example\n    user: admin\n  name: cls-registered-example\ncurrent-context: cls-registered-example\nkind: Config\npreferences: {}\nusers:\n- name: admin\n  user:\n    token: eyJhbGciOiJSUzI1NiIsImtpZCI6...",
  "RequestId": "f6d0e2b8-3a19-4c7d-b5e1-c2f3a4b5d6e7"
}
```

将 kubeconfig 保存为文件并配置 kubectl：

```bash
tccli tke DescribeClusterKubeconfig \
  --ClusterId cls-registered-example \
  --IsExtranet true \
  --region ap-guangzhou \
  --output json \
  | jq -r '.Kubeconfig' \
  > ~/.kube/config-registered-cluster
```

### 4. 安装 kubectl（如未安装）

```bash
# Linux x86_64
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# macOS Apple Silicon
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/darwin/arm64/kubectl"
chmod +x kubectl && sudo mv kubectl /usr/local/bin/

# macOS Intel
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/darwin/amd64/kubectl"
chmod +x kubectl && sudo mv kubectl /usr/local/bin/
```

```bash
kubectl version --client --short
# Client Version: v1.28.2
```

### 5. 配置 kubectl 上下文

```bash
# 合并 kubeconfig 或设置环境变量
export KUBECONFIG=~/.kube/config-registered-cluster

# 查看当前上下文
kubectl config current-context
# 预期: cls-registered-example

# 查看所有上下文
kubectl config get-contexts
```

```
CURRENT   NAME                     CLUSTER                   AUTHINFO   NAMESPACE
*         cls-registered-example   cls-registered-example    admin
```

若你管理多个集群，用 `--context` 切换而避免覆盖默认 kubeconfig：

```bash
kubectl config use-context cls-registered-example
```

## 验证

### 连接性验证

```bash
kubectl get nodes
```

```
NAME           STATUS   ROLES    AGE   VERSION
node-uswest1   Ready    master   30d   v1.23.6
node-uswest2   Ready    <none>   30d   v1.23.6
node-uswest3   Ready    <none>   30d   v1.23.6
```

```bash
kubectl get ns
```

```
NAME               STATUS   AGE
clusternet-system  Active   1h
default            Active   90d
kube-node-lease    Active   90d
kube-public        Active   90d
kube-system        Active   90d
```

确认能看到你注册的命名空间与 Pod。特别注意 `clusternet-system` 命名空间存在且 agent Pod 运行正常。

### kubeconfig 有效期检查

```bash
# 解析证书过期时间
kubectl config view --raw -o json \
  | jq -r '.users[0].user["client-certificate-data"]' \
  | base64 -d \
  | openssl x509 -noout -enddate
# notAfter=Jan 15 12:00:00 2026 GMT
```

## 清理

### 关闭访问端点

若不再需要直连集群，关闭端点以增强安全：

```bash
# 关闭公网端点
tccli tke DeleteClusterEndpoint \
  --ClusterId cls-registered-example \
  --IsExtranet true \
  --region ap-guangzhou \
  --output json

# 关闭内网端点
tccli tke DeleteClusterEndpoint \
  --ClusterId cls-registered-example \
  --IsExtranet false \
  --region ap-guangzhou \
  --output json
```

验证端点已关闭：

```bash
tccli tke DescribeClusterEndpointStatus \
  --ClusterId cls-registered-example \
  --IsExtranet true \
  --region ap-guangzhou \
  --output json
# 预期 "Status": "NotFound"
```

### 移除本地 kubeconfig

```bash
rm ~/.kube/config-registered-cluster
unset KUBECONFIG
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DescribeClusterKubeconfig` 返回空 Kubeconfig | `tccli tke DescribeClusterEndpointStatus --ClusterId <ClusterId> --IsExtranet true/false --region <Region>` 检查端点状态 | 端点未开启，kubeconfig 尚未生成 | 先执行 `CreateClusterEndpoint` 开启公网/内网端点，再获取 kubeconfig |

### 获取 kubeconfig 后无法连接

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| kubectl 返回 `Unauthorized` | `kubectl auth can-i get nodes` 验证权限 | kubeconfig 中的 token 已过期 | 重新调用 `tccli tke DescribeClusterKubeconfig --ClusterId <ClusterId> --region <Region>` 获取新 kubeconfig |
| kubectl 返回 `dial tcp: i/o timeout` | `curl -k https://<API_SERVER_ENDPOINT>:443/version` 测试连通性 | 网络不可达：公网端点需客户端能访问公网，内网端点需在同一 VPC | 确认端点类型与网络路径匹配；内网端点需客户端与集群在同一 VPC 或通过专线/云联网互通 |
| `error: unable to connect to server` | `tccli tke DescribeClusterEndpointStatus --ClusterId <ClusterId> --region <Region>` | 端点状态非 `Created` 或安全组未放通 443 端口 | 检查端点 `Status` 是否为 `Created`；确认安全组入规则放通 TCP 443 |
| 证书校验失败 `x509: certificate signed by unknown authority` | `kubectl config view --raw` 检查 certificate-authority-data | kubeconfig 不完整，缺少 `certificate-authority-data` 或手动修改导致校验失败 | 重新获取完整 kubeconfig，勿手动拼接或删除任何字段 |
| `Current context is empty` | `kubectl config current-context` 检查 | kubeconfig 未正确设置 current-context 或 KUBECONFIG 环境变量未生效 | 用 `export KUBECONFIG=~/.kube/config-registered-cluster` 或用 `--kubeconfig` 参数显式指定路径 |

## 下一步

- [解除注册集群](../解除注册集群/tccli%20操作.md) — 从 TKE 中移除集群纳管
- [日志采集](../../运维指南/日志采集/tccli%20操作.md) — 配置注册集群日志采集
- [集群审计](../../运维指南/集群审计/tccli%20操作.md) — 配置 API Server 审计
- [事件存储](../../运维指南/事件存储/tccli%20操作.md) — 将集群事件持久化到 CLS

## 控制台替代

[控制台 → 注册集群 → 基本信息 → 连接信息](https://console.cloud.tencent.com/tke2/cluster?rid=1) — 提供一键复制 kubeconfig、开启/关闭公网内网访问端点。
