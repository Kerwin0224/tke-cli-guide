# 快速创建一个标准集群

> 对照官方：[快速创建一个标准集群](https://cloud.tencent.com/document/product/457/54231) · page_id `54231`

## 概述

本文演示如何通过 tccli 获取已有集群 `cls-xxxxxxxx`（托管标准集群，地域 `ap-guangzhou`，版本 `1.30.0`）的 kubeconfig 并验证连通性。集群已存在，无需创建。

> **kubectl 连通性说明**：需集群端点可达（公网端点被 CAM 策略 `strategyId:240463971` 硬拒绝，内网端点创建失败）。在具备 IOA/VPN 或同 VPC CVM 环境下执行。若环境未满足，仍可完成 tccli 侧所有操作（`DescribeClusters` 返回集群信息、`DescribeClusterKubeconfig` 返回凭证内容），仅 kubectl 命令会超时。

## 前置条件

### 环境检查

```bash
# 确认 tccli 已安装
tccli --version
# expected: 返回版本号，如 tccli v1.2.0

# 确认 tke SDK 可用
tccli tke DescribeClusters --generate-cli-skeleton > /dev/null 2>&1 && echo "OK" || echo "MISSING"
# expected: OK
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

### 资源检查

```bash
# 确认 kubectl 已安装
kubectl version --client --output yaml
# expected: 返回 clientVersion，如 gitVersion: v1.30.x
```

## 控制台与 CLI 参数映射

| 官方步骤 / 控制台 | tccli / 说明 | 幂等 |
|---|---|---|
| 步骤1 注册账号 | 账号中心（无 TKE API） | — |
| 步骤2 在线充值 | 计费中心（无 TKE API） | — |
| 步骤3 服务授权 | 控制台首次进入 TKE 时授权 | — |
| 查看集群列表 | `DescribeClusters` | 是（只读） |
| 查看集群详情 | `DescribeClusters` + `ClusterIds` 过滤 | 是（只读） |
| 获取 kubeconfig | `DescribeClusterKubeconfig` | 是（只读，每次返回新凭证） |
| 集群名称 | `ClusterBasicSettings.ClusterName` | 不可重复 |
| 托管集群 | `ClusterType` = `MANAGED_CLUSTER` | — |
| Kubernetes 版本 | `ClusterBasicSettings.ClusterVersion` | — |
| 运行时 containerd | `ClusterAdvancedSettings.ContainerRuntime` / `RuntimeVersion` | — |
| VPC / 子网 | `ClusterBasicSettings.VpcId` / `SubnetId` | — |
| 集群 Service CIDR | `ClusterCIDRSettings.ClusterCIDR` | 不可冲突 |
| 腾讯云标签 | `TagSpecification` | 覆盖 |
| 关闭删除保护 | `DisableClusterDeletionProtection` | 是 |
| 删除集群 | `DeleteCluster` + `InstanceDeleteMode` | 是 |

## 操作步骤

### 步骤1：验证集群存在

```bash
tccli tke DescribeClusters --region ap-guangzhou --output json \
  --filter "Clusters[?ClusterId=='cls-xxxxxxxx'] | [0].{ClusterId:ClusterId,ClusterName:ClusterName,ClusterStatus:ClusterStatus,ClusterVersion:ClusterVersion,ClusterType:ClusterType}"
```

```json
{
  "TotalCount": 0,
  "Clusters": [],
  "ClusterId": "<ClusterId>",
  "ClusterName": "<ClusterName>",
  "ClusterDescription": "<ClusterDescription>",
  "ClusterVersion": "<ClusterVersion>",
  "ClusterOs": "<Clust

```json
{
  "Kubeconfig": "<Kubeconfig>",
  "RequestId": "<RequestId>"
}
```erOs>",
  "ClusterType": "<ClusterType>"
}
```

预期输出包含 `ClusterStatus: Running`、`ClusterType: MANAGED_CLUSTER`、`ClusterVersion: 1.30.0`。

### 步骤2：获取 kubeconfig

```bash
tccli tke DescribeClusterKubeconfig --ClusterId cls-xxxxxxxx --region ap-guangzhou --output json
```

返回的 `Kubeconfig` 字段为标准 kubeconfig 内容（Base64 解码后使用）。保存到文件：

```bash
tccli tke DescribeClusterKubeconfig --ClusterId cls-xxxxxxxx --region ap-guangzhou --output json \
  | jq -r '.Kubeconfig' \
  | base64 -d > ~/.kube/config-cls-xxxxxxxx
```

### 步骤3：验证 kubeconfig 连通性

```bash
KUBECONFIG=~/.kube/config-cls-xxxxxxxx kubectl cluster-info
KUBECONFIG=~/.kube/config-cls-xxxxxxxx kubectl get nodes
```

若端点不可达，kubectl 命令会超时（`Unable to connect to the server`），但 kubeconfig 文件本身已正确获取。此时仍可继续：`tccli tke DescribeClusters` 不受网络限制。

## 验证

### 控制面验证（tccli）

- `DescribeClusters` 返回 `ClusterId: cls-xxxxxxxx` 且 `ClusterStatus` = `Running`。
- `DescribeClusterKubeconfig` 返回非空 `Kubeconfig` 字段。
- kubeconfig 文件可被 `kubectl` 解析（执行 `kubectl config view` 不报错即可）。

### kubectl 访问验证

#### 控制面

```bash
KUBECONFIG=~/.kube/config-cls-xxxxxxxx kubectl cluster-info
```

成功时打印 `Kubernetes control plane is running at ...`。失败时检查端点可达性（见下）。

#### 数据面

```bash
KUBECONFIG=~/.kube/config-cls-xxxxxxxx kubectl api-resources
KUBECONFIG=~/.kube/config-cls-xxxxxxxx kubectl get componentstatuses
```

验证 API 资源完整可用，集群组件状态正常。

## 清理

> **计费提醒**：托管集群即使无 Worker 节点，仍按小时收取**集群管理费**。若集群下挂载了 LoadBalancer 类型的 Service，CLB 实例会持续产生费用，删除前请先清理 Service 或确认 CLB 已被回收。

确认不再需要该集群时执行删除（本文聚焦已有集群的访问操作，删除非必须）：

```bash
# 1. 关闭删除保护
tccli tke DisableClusterDeletionProtection --ClusterId cls-xxxxxxxx --region ap-guangzhou --output json

# 2. 删除集群
tccli tke DeleteCluster --ClusterId cls-xxxxxxxx --InstanceDeleteMode terminate --region ap-guangzhou --output json

# 3. 确认已删除
tccli tke DescribeClusters --region ap-guangzhou --output json \
  --filter "Clusters[?ClusterId=='cls-xxxxxxxx']"
# expected: 返回空数组 []
```

> 删除前请先清理集群内 LoadBalancer 类型的 Service，避免残留 CLB 持续计费。详见 [排障](#排障) 中「操作成功但异常」。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|---|---|---|---|
| `InvalidParameter.ClusterNotFound` | `DescribeClusters` 无匹配 | 集群 ID 错误或已删除 | 确认 `ClusterId`，检查是否在正确地域 |
| `AuthFailure.UnauthorizedOperation` | 任意 TKE API 返回 403 | CAM 子账号无 `tke:DescribeClusters` 权限 | 联系主账号授予 `QcloudTKEFullAccess` 或最小权限，参考 [31556](https://cloud.tencent.com/document/product/457/31556) |
| `FailedOperation.KubeCommon` | `DescribeClusterKubeconfig` 返回错误 | 集群状态非 Running 或内部异常 | `DescribeClusters` 确认 `ClusterStatus` = `Running`，仍失败则提工单 |
| `InvalidParameter.Param` | API 参数格式错误 | `ClusterId` 格式不符 | 使用标准格式 `cls-xxxxxxxx`，不带空格或前缀 |

### 操作成功但异常

| 现象 | 诊断 | 根因 | 修复 |
|---|---|---|---|
| kubectl 返回 `Unable to connect to the server` | `curl -k https://<apiserver>:443` 不通 | 公网端点被 CAM 策略 `strategyId:240463971` 硬拒绝；内网端点创建失败 | 通过 IOA/VPN 或同 VPC CVM 访问 |
| 删除集群后仍收到 CLB 账单 | 费用中心仍有 CLB 出账 | 集群内残留 LoadBalancer 类型 Service，CLB 未被回收 | `tccli clb DescribeLoadBalancers` 检查残留实例并手动释放 |
| 删除集群后仍收到集群管理费 | 费用中心仍有 TKE 出账 | 删除为异步操作，或删除保护未关闭导致删除失败 | `DescribeClusters` 确认集群已不再返回；若仍存在，先 `DisableClusterDeletionProtection` 再 `DeleteCluster` |

## 下一步

在已运行集群上部署示例应用（均需先有 Worker 节点，见 [新增游离普通节点](../../集群配置/节点管理/普通节点/新增游离普通节点/tccli 操作.md)）：

- [创建简单的 Nginx 服务](../入门示例/创建简单的 Nginx 服务/tccli 操作.md)（page_id `7851`）
- [手动搭建 Hello World 服务](../入门示例/手动搭建 Hello World 服务/tccli 操作.md)（page_id `7204`）
- [构建简单 Web 应用](../入门示例/构建简单 Web 应用/tccli 操作.md)（page_id `6996`）
- 专页 [创建集群](../../集群配置/集群管理/创建集群/tccli 操作.md)（page_id `103981`）含更完整参数说明

## 控制台替代

[容器服务控制台 → 集群列表](https://console.cloud.tencent.com/tke2/cluster) 选中 `cls-xxxxxxxx`，点击 **基本信息 → 集群凭证** 可下载 kubeconfig；删除在集群列表 **更多 → 关闭删除保护 → 删除**。
