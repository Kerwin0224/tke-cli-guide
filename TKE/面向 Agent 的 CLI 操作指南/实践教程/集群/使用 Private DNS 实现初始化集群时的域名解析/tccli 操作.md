# 使用 Private DNS 实现内网访问集群时的私有域名解析（tccli）

> 对照官方：[使用 Private DNS 实现内网访问集群时的私有域名解析](https://cloud.tencent.com/document/product/457/55348) · page_id `55348`

## 概述

当 TKE 集群开启内网访问并使用内网域名时，需在访问机上配置 Host 进行域名解析。若未配置，运行 `kubectl get nodes` 将报 "no such host"。使用腾讯云 **Private DNS（私有域解析）** 可替代手动 Host 配置，统一管理内网域名解析。

## 前置条件

- 已创建 TKE 集群并已开启内网访问（内网域名模式）。
- 已开通 **Private DNS** 服务（按量计费）。
- 已 [连接集群](../../../集群配置/集群管理/连接集群/tccli%20操作.md)。

```bash
tccli tke DescribeClusters --region ap-guangzhou --output json \
  --filter "Clusters[?ClusterId=='<ClusterId>'] | [0].{ClusterId:ClusterId,ClusterStatus:ClusterStatus}"

tccli tke DescribeClusterEndpoints --ClusterId <ClusterId> --region ap-guangzhou --output json \
  --filter "{IntranetEndpoint:ClusterIntranetEndpoint,IntranetDomain:ClusterIntranetDomain}"
```

```
{
  "ClusterId": "cls-example",
  "ClusterStatus": "Running"
}
{
  "IntranetEndpoint": "10.0.x.x",
  "IntranetDomain": "cls-example.css"
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 查看集群内网访问地址 | `tccli tke DescribeClusterEndpoints --ClusterId <ClusterId>` | 是 |
| 验证 kubectl 连通 | `kubectl get nodes` | 是 |

> **Private DNS 操作**（私有域创建、解析记录配置）目前仅通过控制台或 Private DNS API（`tccli privatedns`）操作，无 TKE API 覆盖。

## 操作步骤

### 1. 查看集群内网访问信息

```bash
tccli tke DescribeClusterEndpoints --ClusterId <ClusterId> --region ap-guangzhou --output json
```

```
{
  "ClusterIntranetEndpoint": "10.0.1.50",
  "ClusterIntranetDomain": "cls-example.css",
  "ClusterExternalEndpoint": "",
  "ClusterExternalDomain": ""
}
```

### 2. 开通 Private DNS

通过控制台 [Private DNS 开通页](https://console.cloud.tencent.com/privatedns) 开通服务。

### 3. 创建私有域

1. 登录 [Private DNS 控制台](https://console.cloud.tencent.com/privatedns/domains)。
2. 单击**新建私有域**：
   - **域名**：需要覆盖的域名，如 `css`（仅解析后缀）或完整域名。
   - **关联 VPC**：选择需要访问集群的节点所在 VPC。
3. 单击**确定**。

### 4. 配置解析记录

1. 进入私有域的**解析记录**页面。
2. 单击**添加记录**：
   - **主机记录**：TKE 集群内网域名中的子域部分，例如 `cls-example`。
   - **记录类型**：`A`。
   - **记录值**：TKE 集群内网访问 IP（从上一步 `DescribeClusterEndpoints` 的 `ClusterIntranetEndpoint` 获取）。
3. 保存配置。

### 5. 验证效果

```bash
kubectl get nodes
```

```
NAME         STATUS   ROLES    AGE   VERSION
10.0.4.144   Ready    <none>   24h   v1.28.3-tke.1
10.0.4.145   Ready    <none>   24h   v1.28.3-tke.1
10.0.4.146   Ready    <none>   24h   v1.28.3-tke.1
```

如返回节点列表，说明已通过 Private DNS 成功解析内网域名。

## 验证

```bash
kubectl cluster-info
```

```
Kubernetes control plane is running at https://cls-example.css:443
CoreDNS is running at https://cls-example.css:443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
```

## 清理

> 本文操作为只读/非持久化操作，无需清理。

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl get nodes` 报 `no such host` | `tccli tke DescribeClusters --region ap-guangzhou --ClusterId cls-xxxxxxxx --filter "Clusters[].ClusterStatus"` | 集群未 Running 或 Private DNS A 记录未配置/未关联 VPC | 确认集群 Running；在 Private DNS 控制台为集群内网域名添加 A 记录并关联访问机所在 VPC |
| `nslookup cls-example.css` 返回空 | `tccli tke DescribeClusterEndpoints --ClusterId cls-xxxxxxxx --region ap-guangzhou --filter "ClusterIntranetEndpoint"` | A 记录主机记录与集群子域不匹配，或地域不支持 Private DNS | 核对主机记录与 `DescribeClusterEndpoints` 的 `ClusterIntranetDomain` 子域；不支持地域改用 `/etc/hosts` |
| `kubectl get nodes` 报 `Unable to connect to the server`（IP 可解析） | `tccli tke DescribeClusters --region ap-guangzhou --ClusterId cls-xxxxxxxx --filter "Clusters[].ClusterStatus"` | CAM 拒绝公网端点（strategyId 240463971），数据面不可达 | 接入 VPN/IOA 后在内网执行 kubectl |

## 下一步

- [实现独立集群的 Master 容灾](../实现独立集群的 Master 容灾/tccli 操作.md)（page_id `48956`）
- [边缘集群迁移至 TKE 标准集群注册节点公网版](https://cloud.tencent.com/document/product/457/110447)

## 控制台替代

控制台 **Private DNS → 私有域 → 新建私有域**，输入域名并关联 VPC；**解析记录 → 添加记录**，配置 A 记录指向集群内网 IP。
