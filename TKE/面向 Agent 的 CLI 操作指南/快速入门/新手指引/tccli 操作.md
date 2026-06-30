# 新手指引

> 对照官方：[新手指引](https://cloud.tencent.com/document/product/457/54224) · page_id `54224`

## 概述

TKE 新手指引覆盖从零到部署第一个容器应用的完整流程。本文聚焦 CLI 可执行部分：验证环境、查询可用地域和 K8s 版本、确认已有集群、获取集群访问凭证。创建集群和部署应用的操作由对应专项页面覆盖。

| 阶段 | CLI 支持 | 说明 |
|------|:--:|------|
| 账号注册与实名认证 | 否 | 控制台或账号中心操作 |
| 充值 | 否 | 计费中心操作 |
| CAM 服务授权 | 否 | 首次进入 TKE 控制台时授权 |
| 环境验证 | 是 | `tccli --version`、`tccli configure list` |
| 查询可用地域和版本 | 是 | `DescribeRegions`、`DescribeVersions` |
| 确认已有集群 | 是 | `DescribeClusters` |
| 创建集群 | 是 | 跳转至 [快速创建一个标准集群](../快速创建一个标准集群/tccli%20操作.md) |
| 部署应用（kubectl） | 部分 | 需 kubectl 可达环境 |

## 前置条件

- [环境准备](../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置，region 为目标地域

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:DescribeRegions, tke:DescribeVersions, tke:DescribeClusters
#    tke:DescribeClusterKubeconfig
# 验证：执行 DescribeRegions 确认权限
tccli tke DescribeRegions
# expected: exit 0，返回地域列表
```

```json
{
  "TotalCount": "<TotalCount>",
  "Regions": "<Regions>",
  "Alias": "<Alias>",
  "RegionId": "<RegionId>",
  "RegionName": "<RegionName>",
  "Status": "<Status>"
}
```

### 资源检查

```bash
# 4. 查询已有集群（确认是否需要新建）
tccli tke DescribeClusters --region <Region>
# expected: exit 0，返回集群列表（可为空，表示尚无集群）
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

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 进入 TKE 控制台 | `tccli tke DescribeRegions` | 是 |
| 选择地域 | `--region <Region>` | — |
| 查看 K8s 版本列表 | `DescribeVersions` | 是 |
| 查看集群列表 | `DescribeClusters` | 是 |
| 点击集群名查看详情 | `DescribeClusters --ClusterIds` | 是 |
| 获取连接凭证 | `DescribeClusterKubeconfig` | 是 |
| 新建集群 | 跳转至 `CreateCluster` 专页 | 否 |

## 操作步骤

### 步骤 1：验证 tccli 环境

```bash
# 确认 tccli 已安装且版本满足要求
tccli --version
# expected: tccli version >= 1.0.0

# 确认凭据和默认地域已配置
tccli configure list
# expected: secretId, secretKey, region 均有值
```

**预期输出**：

```
secretId     = AKIDxxxxxxxxxxxxxxxxxxxxxxxxxx
secretKey    = xxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
region       = ap-guangzhou
```

### 步骤 2：查询可用地域

```bash
tccli tke DescribeRegions
# expected: exit 0，返回所有 TKE 支持的地域
```

**预期输出**：

```json
{
    "Regions": [
        {
            "RegionName": "ap-guangzhou",
            "RegionRemark": "华南地区（广州）"
        },
        {
            "RegionName": "ap-shanghai",
            "RegionRemark": "华东地区（上海）"
        },
        {
            "RegionName": "ap-beijing",
            "RegionRemark": "华北地区（北京）"
        }
    ],
    "RequestId": "..."
}
```

选择一个地域作为后续操作的目标地域，替换文档中的 `REGION` 占位符。

### 步骤 3：查询可用 K8s 版本

```bash
tccli tke DescribeVersions --region <Region>
# expected: exit 0，返回可用 K8s 版本列表
```

**预期输出**：

```json
{
    "TotalCount": 3,
    "Versions": [
        {
            "Version": "1.32.0",
            "Name": "1.32.0",
            "Status": "ENABLED"
        },
        {
            "Version": "1.30.0",
            "Name": "1.30.0",
            "Status": "ENABLED"
        },
        {
            "Version": "1.28.0",
            "Name": "1.28.0",
            "Status": "ENABLED"
        }
    ],
    "RequestId": "..."
}
```

推荐选择 `Status` 为 `ENABLED` 的最新版本。

### 步骤 4：确认已有集群

```bash
tccli tke DescribeClusters --region <Region>
# expected: exit 0，返回集群列表
```

**预期输出**（无集群时）：

```json
{
    "TotalCount": 0,
    "Clusters": [],
    "RequestId": "..."
}
```

**预期输出**（有集群时，以真实集群 `cls-xxxxxxxx` 为例）：

```json
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "cls-xxxxxxxx",
            "ClusterName": "my-tke-cluster",
            "ClusterType": "MANAGED_CLUSTER",
            "ClusterStatus": "Running",
            "ClusterVersion": "1.30.0",
            "ClusterNodeNum": 3,
            "DeletionProtection": true
        }
    ],
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

- 若 `TotalCount == 0`：尚无集群，继续步骤 5。
- 若已有集群：记录 `ClusterId`，可直接跳至步骤 6 获取 kubeconfig。

### 步骤 5：创建集群

如尚无集群，前往：

> [快速创建一个标准集群](../快速创建一个标准集群/tccli%20操作.md) — 通过 CLI 创建托管集群的完整流程

创建完成后，记录 `ClusterId` 并回到本文档继续步骤 6。

### 步骤 6：获取集群访问凭证

```bash
tccli tke DescribeClusterKubeconfig --region <Region> \
    --ClusterId CLUSTER_ID | jq -r '.Kubeconfig' > ~/.kube/config
# expected: 生成 ~/.kube/config 文件
```

```json
{
  "Kubeconfig": "<Kubeconfig>",
  "RequestId": "<RequestId>"
}
```

> 集群需处于 `Running` 状态才能获取有效的 kubeconfig。若状态非 `Running`，等待集群就绪后重试。

## 验证

### 控制面（tccli）

```bash
# 验证集群状态（以 cls-xxxxxxxx 为例）
tccli tke DescribeClusters --region ap-guangzhou \
    --ClusterIds '["cls-xxxxxxxx"]' \
    | jq '.Clusters[0].ClusterStatus'
# expected: "Running"

# 验证 kubeconfig 已生成
ls -la ~/.kube/config
# expected: 文件存在且非空
```

**预期输出**：

```json
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "cls-xxxxxxxx",
            "ClusterName": "my-tke-cluster",
            "ClusterStatus": "Running",
            "ClusterType": "MANAGED_CLUSTER",
            "ClusterVersion": "1.30.0"
        }
    ],
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

### 数据面

> **kubectl 数据面不可达**：CAM 拒绝公网端点（strategyId:240463971），内网需 VPN/IOA。以下 kubectl 命令需在内网/VPN 环境下执行。

```bash
# 验证集群连通性
kubectl cluster-info
# expected: Kubernetes control plane is running at ...

# 查看命名空间
kubectl get ns
# expected: 返回 default 等系统命名空间
```

**预期输出**：

```
NAME              STATUS   AGE
default           Active   1h
kube-node-lease   Active   1h
kube-public       Active   1h
kube-system       Active   1h
```

## 清理

本页为指导性页面，无资源需清理。

> 如之前创建了验证用集群且不再需要，请前往 [快速创建一个标准集群](../快速创建一个标准集群/tccli%20操作.md) 的「清理」节执行删除，避免持续计费。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `tccli --version` 返回 `command not found` | `which tccli` 无结果 | tccli 未安装或不在 PATH 中 | 安装 tccli：`pip install tccli` 或参考 [安装与配置 tccli](https://cloud.tencent.com/document/product/440/34066) |
| `tccli configure list` 输出为空 | `cat ~/.tccli/default.configure` 检查配置文件 | 未完成 tccli 配置 | `tccli configure set secretId SECRET_ID secretKey SECRET_KEY region REGION` |
| `DescribeRegions` 返回 `AuthFailure.SignatureFailure` | `tccli configure list` 检查 secretId/secretKey | 凭据错误或过期 | 重新获取 API 密钥并执行 `tccli configure` |
| `DescribeClusters` 返回 `UnauthorizedOperation` | `tccli configure list` 检查当前凭据 | 子账号缺少 `tke:DescribeClusters` 权限（此为环境限制，非命令错误） | 联系主账号授予 `QcloudTKEFullAccess` 或最小权限 `tke:DescribeClusters` |
| `DescribeClusterKubeconfig` 返回 `InvalidParameter.ClusterId` | 检查 `--ClusterId` 参数值 | 集群 ID 格式错误或集群不属于当前账号/地域 | `tccli tke DescribeClusters --region <Region>` 列出可用集群确认 ID |

### 操作成功但异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DescribeClusterKubeconfig` 成功但 kubectl 无法连接 | `cat ~/.kube/config` 检查 server 地址；`curl -k SERVER_ADDRESS` 测试连通性 | 集群 API Server 未开启公网访问或网络不通 | `tccli tke DescribeClusterEndpoints --region <Region> --ClusterId CLUSTER_ID` 检查端点配置；如未开启外网，前往控制台开启 |
| `CreateCluster` 返回了 ClusterId 但长时间非 Running | `tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'` 轮询状态 | 后端创建中，托管集群通常需数分钟 | 每 30 秒轮询一次；超过 10 分钟则保留 region、ClusterId、RequestId → 登录 [TKE 控制台](https://console.cloud.tencent.com/tke2/cluster) 查看详细状态 |

## 下一步

- [快速创建一个标准集群](https://cloud.tencent.com/document/product/457/54231) — 通过 CLI 创建第一个 TKE 托管集群
- [连接集群](https://cloud.tencent.com/document/product/457/32191) — 获取 kubeconfig 并验证连通性
- [创建简单的 Nginx 服务](https://cloud.tencent.com/document/product/457/7851) — 在集群中部署第一个应用
- [TKE API 概览](https://cloud.tencent.com/document/product/457/31853) — 完整 API 列表
- [容器服务计费概述](https://cloud.tencent.com/document/product/457/6776) — 了解托管集群费用构成

## 控制台替代

[TKE 控制台 → 新手指引](https://console.cloud.tencent.com/tke2/cluster)：进入控制台后按向导逐步完成集群创建和应用部署。
