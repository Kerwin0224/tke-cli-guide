# 注册节点常见问题

> 对照官方：[注册节点常见问题](https://cloud.tencent.com/document/product/457/79750) · page_id `79750`

## 概述

汇总注册节点在**开启支持、创建节点池、执行注册脚本、节点运行**各阶段中的常见问题及诊断方法。覆盖网络、代理组件、权限、版本兼容四类问题。

> 本文档中所有 ID（集群 ID、节点池 ID、子网 ID 等）均为占位符，请替换为实际值。

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

# 3. 检查 kubectl（部分诊断需要）
kubectl version --client --short
# expected: 显示 kubectl 客户端版本
```

### 资源检查

```bash
# 4. 确认集群存在并获取 kubeconfig
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterStatus 为 Running

tccli tke DescribeClusterKubeconfig --region <Region> \
    --ClusterId CLUSTER_ID | jq -r '.Kubeconfig' > ~/.kube/config
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
| 查看注册节点配置 | `DescribeExternalNodeSupportConfig` | 是 |
| 查看注册节点池 | `DescribeExternalNodePools` | 是 |
| 查看注册节点详情 | `DescribeExternalNode` | 是 |
| 查看 Addon 状态（公网版） | `DescribeAddon --AddonName externaledge` | 是 |

### API 参数对照

诊断相关 API 的输出关键字段：

| API | 关键输出字段 | 用途 |
|-----|------------|------|
| `DescribeExternalNodeSupportConfig` | `Enabled`, `Status`, `FailedReason`, `NetworkType`, `ClusterCIDR` | 诊断注册功能开启状态 |
| `DescribeExternalNodePools` | `LifeState`, `RuntimeConfig`, `DeletionProtection` | 诊断节点池状态 |
| `DescribeExternalNode` | `Status`, `Reason`, `IP`, `Unschedulable` | 诊断单个节点状态 |
| `DescribeAddon` | Addon 状态信息 | 诊断 externaledge 组件（公网版） |

## 操作步骤

本页为排障说明页，不包含写操作。以下为各常见问题的诊断命令。

### 诊断 1：注册节点功能开启失败

```bash
tccli tke DescribeExternalNodeSupportConfig --region <Region> \
    --ClusterId CLUSTER_ID
# expected: 查看 Status 和 FailedReason 定位失败原因
```

```json
{
  "ClusterCIDR": "<ClusterCIDR>",
  "NetworkType": "<NetworkType>",
  "SubnetId": "<SubnetId>",
  "Enabled": "<Enabled>",
  "AS": "<AS>",
  "SwitchIP": "<SwitchIP>"
}
```

```bash
# 检查集群整体状态
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]' | jq '.Clusters[0].ClusterStatus'
# expected: "Running"
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

### 诊断 2：节点注册脚本执行失败

在 IDC 主机上执行以下诊断：

```bash
# 检查主机网络连通性
curl -I https://cloud.tencent.com
# expected: HTTP/2 200（公网版）或其他正常响应

# 检查 DNS 解析
nslookup cloud.tencent.com
# expected: 返回 IP 地址

# 检查代理组件的连接（公网版重点）
# 检查 externaledge 代理是否在节点上正常运行
```

### 诊断 3：externaledge Addon 状态异常（公网版）

```bash
tccli tke DescribeAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName externaledge
# expected: 检查 Addon 状态是否正常
```

```json
{
  "Addons": "<Addons>",
  "AddonName": "<AddonName>",
  "AddonVersion": "<AddonVersion>",
  "RawValues": "<RawValues>",
  "Phase": "<Phase>",
  "Reason": "<Reason>"
}
```

### 诊断 4：节点注册后状态异常

```bash
# 查看节点详情
tccli tke DescribeExternalNode --region <Region> \
    --ClusterId CLUSTER_ID \
    --NodePoolId NODE_POOL_ID
# expected: 查看 Status 和 Reason 字段

# 如节点 Status 不为 Normal，获取 kubeconfig 后：
kubectl describe node NODE_NAME
# expected: 查看 Conditions 部分定位异常原因

kubectl get events --field-selector involvedObject.name=NODE_NAME
# expected: 查看相关事件
```

```json
{
  "Nodes": "<Nodes>",
  "Name": "<Name>",
  "NodePoolId": "<NodePoolId>",
  "IP": "<IP>",
  "Location": "<Location>",
  "Status": "<Status>"
}
```

### 诊断 5：删除保护导致无法删除

```bash
# 查看节点池是否开启删除保护
tccli tke DescribeExternalNodePools --region <Region> \
    --ClusterId CLUSTER_ID \
    | jq '.NodePoolSet[] | {NodePoolId, Name, DeletionProtection}'
# expected: DeletionProtection 为 true 时需先关闭
```

```json
{
  "TotalCount": "<TotalCount>",
  "NodePoolSet": "<NodePoolSet>",
  "NodePoolId": "<NodePoolId>",
  "Name": "<Name>",
  "LifeState": "<LifeState>",
  "ClusterCIDR": "<ClusterCIDR>"
}
```

## 验证

本页为排障说明页，无需要验证的操作。

## 清理

本页为排障说明页，无资源需清理。

## 排障

### 网络类问题

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 专线版开启失败，`FailedReason` 提示子网不可达 | `tccli vpc DescribeSubnets --region <Region> --SubnetIds '["SUBNET_ID"]'` 检查子网状态 | 子网未关联路由表或专线/VPN 未配置 | 确认专线/VPN 已建立且配置了正确路由；确认子网路由表有指向 IDC 的路由 |
| 公网版脚本执行时 `curl` 超时 | `curl -v https://cloud.tencent.com` 从 IDC 主机测试 | IDC 主机无法访问公网或防火墙阻断了出站 HTTPS 流量 | 配置 IDC 主机公网访问权限；添加 `*.cloud.tencent.com` 到防火墙白名单 |
| 节点注册成功后间歇性断连 | `tccli tke DescribeExternalNode --region <Region> --ClusterId CLUSTER_ID` 查看 Reason | 公网质量波动或专线不稳定 | 专线版：检查专线监控指标；公网版：考虑升级到专线版 |
| 节点 `Status=Abnormal`，`Reason` 含 "network" | `kubectl get node NODE_NAME -o yaml` 查看 conditions | 代理组件与集群控制面网络中断 | 检查 IDC 到 VPC 的网络路径；专线版确认路由；公网版确认 externaledge 状态 |

### 代理组件问题

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `externaledge` Addon 安装失败 | `tccli tke DescribeAddon --region <Region> --ClusterId CLUSTER_ID --AddonName externaledge` | 集群版本不兼容或资源不足 | 确认集群为托管集群（`MANAGED_CLUSTER`）；检查集群资源配额 |
| 节点上代理进程未运行 | 在 IDC 主机上执行 `ps aux \| grep edge` 或 `systemctl status tke-edge` | 注册脚本执行不完整或进程被手动停止 | 重新执行 `DescribeExternalNodeScript` 返回的脚本 |
| 代理日志报 TLS 证书错误 | 检查代理日志（通常位于 `/var/log/tke-edge/`） | 系统时间不同步导致证书验证失败 | 同步 IDC 主机时间：`ntpdate ntp.tencent.com` 或 `timedatectl set-ntp true` |

### 权限类问题

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `EnableExternalNodeSupport` 返回 `UnauthorizedOperation` | `tccli configure list` 检查当前凭据 | 子账号缺少 `tke:EnableExternalNodeSupport` 权限（此为环境限制，非命令错误） | 联系主账号授予 `QcloudTKEFullAccess` 或 `tke:EnableExternalNodeSupport` |
| `CreateExternalNodePool` 返回 `UnauthorizedOperation` | `tccli configure list` 检查当前凭据 | 缺少节点池创建权限 | 联系主账号授予 `tke:CreateExternalNodePool` |
| `DescribeExternalNodeScript` 返回空 Token | 确认 `NodePoolId` 正确且用户有该节点池的访问权限 | 节点池可能属于不同账号或子账号权限不足 | 用主账号或有完整 TKE 权限的子账号操作 |

### 版本与兼容性问题

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `CreateExternalNodePool` 返回版本错误 | `tccli tke DescribeVersions --region <Region>` 查询支持版本 | `RuntimeVersion` 与集群 K8s 版本不匹配 | 选择与集群版本兼容的 RuntimeVersion（如 `1.6.9` 兼容 K8s 1.30） |
| 节点注册后 `kubectl get nodes` 显示 `NotReady` | `kubectl describe node NODE_NAME` 查看详细原因 | 节点上 kubelet 版本与集群 API Server 版本不兼容 | 确保注册脚本使用的代理组件版本与集群兼容；检查节点操作系统内核版本 |
| 独立集群无法使用注册节点功能 | `tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'` 检查 `ClusterType` | 注册节点功能仅支持托管集群（`MANAGED_CLUSTER`） | 使用托管集群；如需独立集群管理外部节点，考虑其他方案 |

### 通用诊断信息收集

遇到无法定位的问题时，收集以下信息以备工单提交：

```bash
# 1. 集群信息
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]' > /tmp/cluster-info.json

# 2. 注册节点配置
tccli tke DescribeExternalNodeSupportConfig --region <Region> \
    --ClusterId CLUSTER_ID > /tmp/support-config.json

# 3. 节点池信息
tccli tke DescribeExternalNodePools --region <Region> \
    --ClusterId CLUSTER_ID > /tmp/node-pools.json

# 4. 节点详情
tccli tke DescribeExternalNode --region <Region> \
    --ClusterId CLUSTER_ID > /tmp/nodes.json

# 5. kubectl 节点状态
kubectl get nodes -o wide > /tmp/kubectl-nodes.txt
kubectl describe nodes > /tmp/kubectl-nodes-desc.txt

# 6. 保留最近失败的 RequestId
# 从错误输出中记录 RequestId 字段
```

```json
{
  "TotalCount": 0,
  "Clusters": [],
  "ClusterId": "<ClusterId>",
  "ClusterName": "<ClusterName>",
  "ClusterDescription": "<ClusterDescription>",
  "ClusterVersion": "<ClusterVersion>",
  "ClusterOs": "<ClusterOs>",
  "ClusterType": "<ClusterType>"
}
```

## 下一步

- [注册节点概述](https://cloud.tencent.com/document/product/457/57916) — 专线版与公网版对比
- [创建注册节点（专线版）](https://cloud.tencent.com/document/product/457/57917) — 专线版创建注册节点
- [创建注册节点（公网版）](https://cloud.tencent.com/document/product/457/101532) — 公网版创建注册节点
- [移除注册节点](https://cloud.tencent.com/document/product/457/79767) — 删除注册节点或节点池
- [流量接入](https://cloud.tencent.com/document/product/457/79749) — 注册节点上的流量接入方式

## 控制台替代

[TKE 控制台 → 集群 → 节点管理 → 注册节点](https://console.cloud.tencent.com/tke2/cluster)：查看节点状态和异常信息。部分诊断（如日志查看）需通过 CLI 完成。
