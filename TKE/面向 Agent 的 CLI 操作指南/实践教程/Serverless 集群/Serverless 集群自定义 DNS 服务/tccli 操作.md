# Serverless 集群自定义 DNS 服务（tccli）
> 对照官方：[Serverless 集群自定义 DNS 服务](https://cloud.tencent.com/document/product/457/63735) · page_id `63735`

## 概述

TKE Serverless（EKS）集群默认使用 VPC 内置 DNS（`183.60.83.19`、`183.60.82.98`）和集群 CoreDNS 进行域名解析。在以下场景中需自定义 DNS：

- **混合云**：需要解析企业内网自定义域名（如 `*.mycompany.local`）。
- **Private DNS**：通过腾讯云 Private DNS（私有域解析）管理 VPC 内域名。
- **自定义上游 DNS**：使用自建 DNS 服务器替代默认解析。
- **性能优化**：调整 `ndots`、`timeout` 等 DNS 客户端参数。

本文档覆盖两种自定义 DNS 方式：**EKS 容器实例级 `DnsConfig`**（Pod 粒度）和 **Private DNS 私有域**（VPC 粒度，通过 kubectl 修改 CoreDNS ConfigMap 注入）。

环境约定：`REGION=ap-guangzhou`，Kubernetes 版本 `1.30.0`，集群类型 `MANAGED_CLUSTER`（EKS）。

> **kubectl 数据面不可达**：CAM拒绝公网端点(strategyId:240463971)，内网需VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

## 前置条件

### 环境检查

```bash
tccli --version
# expected: 3.0.x

tccli configure list
# expected: secretId/secretKey/region 均已配置
{
    "secretId": "AKID******************************",
    "secretKey": "********************************",
    "region": "ap-guangzhou"
}
```

CAM 权限检查（执行以下 Describe* 命令，确认无 `UnauthorizedOperation` 错误）：

| 服务 | CAM Action | 用途 |
|------|-----------|------|
| tke | `DescribeEKSClusters` | 查询 EKS 集群状态 |
| tke | `CreateEKSContainerInstances` | 创建含 DnsConfig 的容器实例 |
| tke | `DescribeEKSContainerInstances` | 查询容器实例 DNS 配置 |
| privatedns | `DescribePrivateZones` | 查询私有域列表 |
| privatedns | `CreatePrivateZone` | 创建私有域 |
| privatedns | `CreatePrivateZoneRecord` | 创建私有域解析记录 |
| privatedns | `ModifyPrivateZone` | 修改私有域配置 |

### 资源检查

```bash
# 确认 EKS 集群存在
tccli tke DescribeEKSClusters \
    --region <Region> \
    --ClusterIds '["EKS_CLUSTER_ID"]'
# expected:
{
    "Response": {
        "Clusters": [
            {
                "ClusterId": "EKS_CLUSTER_ID",
                "ClusterName": "eks-custom-dns",
                "Status": "Running",
                "VpcId": "VPC_ID"
            }
        ],
        "TotalCount": 1,
        "RequestId": "..."
    }
}
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

```bash
# 确认 VPC 信息
tccli vpc DescribeVpcs \
    --region <Region> \
    --VpcIds '["VPC_ID"]'
# expected:
{
    "Response": {
        "VpcSet": [
            {
                "VpcId": "VPC_ID",
                "VpcName": "eks-vpc",
                "CidrBlock": "10.0.0.0/16"
            }
        ],
        "RequestId": "..."
    }
}
```

## 控制台与 CLI 参数映射

| 控制台字段 | CLI 参数/路径 | 类型 | 必填 | 说明 | 幂等性 |
|-----------|-------------|------|------|------|--------|
| 地域 | `--region` / `Region` | String | 是 | 地域标识 | 无影响 |
| 集群 ID | `--ClusterId` / `ClusterIds` | String | 是 | EKS 集群 ID | 查询操作天然幂等 |
| 私有域名称 | `--Domain` / `Domain` | String | 是 | 私有域名，如 `mycompany.local` | 创建非幂等（同名报错） |
| 关联 VPC | `--VpcSet` / `VpcSet` | Array | 是 | 关联的 VPC 列表 | 修改幂等（覆盖写） |
| 子域名 | `--SubDomain` | String | 是 | 私有域解析记录的子域名 | 创建非幂等（同记录报错） |
| 记录类型 | `--RecordType` | String | 是 | A/AAAA/CNAME/MX/TXT | 创建非幂等 |
| 记录值 | `--RecordValue` | String | 是 | 解析目标地址 | 修改幂等 |
| DNS Nameserver | `DnsConfig.Nameservers` | Array | 否 | 自定义 DNS 服务器列表 | 创建时指定，不可事后修改 |
| DNS Search | `DnsConfig.Searches` | Array | 否 | DNS 搜索域列表 | 创建时指定，不可事后修改 |
| DNS Options | `DnsConfig.Options` | Array | 否 | DNS 解析器选项 | 创建时指定，不可事后修改 |
| VPC ID | `--VpcId` | String | 是 | VPC 网络 ID | 查询天然幂等 |
| 子网 ID | `--SubnetId` | String | 是 | 容器实例所在子网 | 查询天然幂等 |

## 操作步骤

### 选择依据

EKS 自定义 DNS 有两种粒度的实现方式：

1. **Pod 粒度**（`DnsConfig`）：通过 EKS API `CreateEKSContainerInstances` 的 `DnsConfig` 参数为单个容器实例指定 Nameservers、Searches 和 Options。适用于单个 Pod 需要特定 DNS 配置的场景。
2. **VPC 粒度**（Private DNS）：通过 `tccli privatedns` 创建私有域并关联 VPC，VPC 内所有 EKS Pod 自动解析该私有域。适用于企业内部域名统一管理的场景。

### 最小创建

#### 方式一：EKS 容器实例 DnsConfig（Pod 粒度）

通过 `CreateEKSContainerInstances` 在创建容器实例时传入 `DnsConfig` 参数。

| 参数 | 值 | 说明 |
|------|----|------|
| `REGION` | 地域 | `ap-guangzhou` |
| `EKS_CLUSTER_ID` | 集群 ID | EKS 集群标识 |
| `VPC_ID` | VPC ID | EKS 集群所在 VPC |
| `SUBNET_ID` | 子网 ID | 容器实例部署子网 |

```bash
cat > eks-dnsconfig.json <<'EOF'
{
    "EksCiName": "custom-dns-pod",
    "ClusterId": "EKS_CLUSTER_ID",
    "VpcId": "VPC_ID",
    "SubnetId": "SUBNET_ID",
    "SecurityGroupIds": ["SECURITY_GROUP_ID"],
    "Cpu": 0.25,
    "Memory": 0.5,
    "DnsConfig": {
        "Nameservers": ["8.8.8.8", "114.114.114.114"],
        "Searches": ["mycompany.local", "svc.cluster.local"],
        "Options": [
            {"Name": "ndots", "Value": "2"},
            {"Name": "timeout", "Value": "1"},
            {"Name": "attempts", "Value": "3"}
        ]
    },
    "Containers": [
        {
            "Name": "app",
            "Image": "nginx:alpine",
            "Cpu": 0.25,
            "Memory": 0.5
        }
    ]
}
EOF
tccli tke CreateEKSContainerInstances \
    --region <Region> \
    --cli-input-json file://eks-dnsconfig.json
# expected:
{
    "Response": {
        "EksCiIds": ["eksci-xxxxxxxx"],
        "RequestId": "..."
    }
}
```

#### 方式二：Private DNS 私有域（VPC 粒度）

**步骤 1** — 创建私有域。

```bash
tccli privatedns CreatePrivateZone \
    --region <Region> \
    --Domain mycompany.local
# expected:
{
    "Response": {
        "ZoneId": "zone-xxxxxxxx",
        "Domain": "mycompany.local",
        "RequestId": "..."
    }
}
```

**步骤 2** — 将私有域关联到 EKS 集群所在 VPC。

```bash
tccli privatedns ModifyPrivateZoneVpc \
    --region <Region> \
    --ZoneId zone-xxxxxxxx \
    --VpcSet '[{"UniqVpcId":"VPC_ID","Region":"REGION"}]'
# expected:
{
    "Response": {
        "ZoneId": "zone-xxxxxxxx",
        "RequestId": "..."
    }
}
```

**步骤 3** — 添加解析记录。

```bash
tccli privatedns CreatePrivateZoneRecord \
    --region <Region> \
    --ZoneId zone-xxxxxxxx \
    --RecordType A \
    --SubDomain api.internal \
    --RecordValue 10.0.1.100 \
    --TTL 300
# expected:
{
    "Response": {
        "RecordId": "record-xxxxxxxx",
        "RequestId": "..."
    }
}
```

```bash
# 添加通配符解析
tccli privatedns CreatePrivateZoneRecord \
    --region <Region> \
    --ZoneId zone-xxxxxxxx \
    --RecordType A \
    --SubDomain "*.app" \
    --RecordValue 10.0.1.200 \
    --TTL 300
# expected:
{
    "Response": {
        "RecordId": "record-xxxxxxxx",
        "RequestId": "..."
    }
}
```

### 增强配置

#### 方式三：CoreDNS ConfigMap 注入 Private DNS（VPC 粒度）

当 Private DNS 已创建并关联 VPC 后，可通过 kubectl 修改 CoreDNS ConfigMap 添加 stubDomain，使集群内 Pod 的 DNS 查询自动转发到 Private DNS。

```bash
# 查看当前 CoreDNS ConfigMap
kubectl get configmap coredns -n kube-system -o yaml
# expected:
# apiVersion: v1
# data:
#   Corefile: |
#     .:53 {
#         errors
#         health
#         kubernetes cluster.local in-addr.arpa ip6.arpa {
#             pods insecure
#             fallthrough in-addr.arpa ip6.arpa
#         }
#         prometheus :9153
#         forward . /etc/resolv.conf
#         cache 30
#         loop
#         reload
#         loadbalance
#     }
```

```text
NAME  STATUS  AGE
...
```

```bash
# 编辑 CoreDNS ConfigMap 添加自定义域名转发
kubectl edit configmap coredns -n kube-system
# 在 Corefile 中添加：
# mycompany.local:53 {
#     errors
#     cache 30
#     forward . 10.0.1.53 10.0.1.54
# }
```

配置后，集群内所有 Pod 对 `*.mycompany.local` 的 DNS 查询将被转发到指定的上游 DNS 服务器。

> **注意**：修改 CoreDNS ConfigMap 后，CoreDNS Pod 会自动重载配置（`reload` 插件），通常 30 秒内生效。若需立即生效，可执行 `kubectl rollout restart deployment coredns -n kube-system`。

## 验证

### 控制面（tccli）

```bash
# 验证容器实例 DNS 配置
tccli tke DescribeEKSContainerInstances \
    --region <Region> \
    --EksCiIds '["eksci-xxxxxxxx"]'
# expected: 返回实例详情，确认 DnsConfig 与创建时一致

# 验证 Private DNS 私有域已关联 VPC
tccli privatedns DescribePrivateZoneList \
    --region <Region>
# expected:
{
    "Response": {
        "PrivateZoneSet": [
            {
                "ZoneId": "zone-xxxxxxxx",
                "Domain": "mycompany.local",
                "VpcSet": [
                    {
                        "UniqVpcId": "VPC_ID",
                        "Region": "REGION"
                    }
                ]
            }
        ],
        "TotalCount": 1,
        "RequestId": "..."
    }
}
```

```json
{
  "TotalCount": "<TotalCount>",
  "EksCis": "<EksCis>",
  "AutoCreatedEipId": "<AutoCreatedEipId>",
  "CamRoleName": "<CamRoleName>",
  "Containers": "<Containers>",
  "Image": "<Image>"
}
```

```bash
# 验证私有域解析记录
tccli privatedns DescribePrivateZoneRecordList \
    --region <Region> \
    --ZoneId zone-xxxxxxxx
# expected:
{
    "Response": {
        "RecordSet": [
            {
                "RecordId": "record-xxxxxxxx",
                "SubDomain": "api.internal",
                "RecordType": "A",
                "RecordValue": "10.0.1.100",
                "TTL": 300
            },
            {
                "RecordId": "record-xxxxxxxx",
                "SubDomain": "*.app",
                "RecordType": "A",
                "RecordValue": "10.0.1.200",
                "TTL": 300
            }
        ],
        "TotalCount": 2,
        "RequestId": "..."
    }
}
```

### 数据面（需 VPN/IOA）

```bash
# 验证 Pod 内 resolv.conf 使用了自定义 DnsConfig
kubectl exec -n NAMESPACE custom-dns-pod -- cat /etc/resolv.conf
# expected (DnsConfig Pod):
# nameserver 8.8.8.8
# nameserver 114.114.114.114
# search mycompany.local svc.cluster.local
# options ndots:2 timeout:1 attempts:3

# expected (普通 Pod，默认配置):
# nameserver 183.60.83.19
# nameserver 183.60.82.98
# search default.svc.cluster.local svc.cluster.local
# options ndots:5
```

```bash
# 验证私有域解析（Private DNS 关联 VPC 后，在任意 Pod 内测试）
kubectl run dns-test \
    --image=busybox:1.28 \
    --restart=Never \
    --rm -it \
    -n NAMESPACE \
    -- nslookup api.internal.mycompany.local
# expected:
# Server:    10.0.0.2
# Address:   10.0.0.2#53
# Name:      api.internal.mycompany.local
# Address:   10.0.1.100

# 验证通配符解析
kubectl run dns-test-wildcard \
    --image=busybox:1.28 \
    --restart=Never \
    --rm -it \
    -n NAMESPACE \
    -- nslookup any.app.mycompany.local
# expected:
# Name:      any.app.mycompany.local
# Address:   10.0.1.200
```

## 清理

> **账单警告**：Private DNS 私有域按域名数量和解析查询量计费。闲置的私有域仍可能产生少量费用。请及时清理不再需要的私有域和解析记录。

### 控制面（tccli）

**清理 Private DNS（按顺序执行）：**

```bash
# 1. 查询所有私有域
tccli privatedns DescribePrivateZoneList \
    --region <Region>
# expected: 确认待删除的 ZoneId

# 2. 删除私有域下的所有解析记录（批量删除）
# 注意：删除私有域时会自动级联删除解析记录，此步可跳过

# 3. 解绑 VPC 关联（可选，删除私有域时自动解绑）
tccli privatedns ModifyPrivateZoneVpc \
    --region <Region> \
    --ZoneId zone-xxxxxxxx \
    --VpcSet '[]'
# expected: VpcSet 置空

# 4. 删除私有域
tccli privatedns DeletePrivateZone \
    --region <Region> \
    --ZoneId zone-xxxxxxxx
# expected:
{
    "Response": {
        "RequestId": "..."
    }
}

# 5. 验证已删除
tccli privatedns DescribePrivateZoneList \
    --region <Region> \
    --Filters '[{"Name":"Domain","Values":["mycompany.local"]}]'
# expected: TotalCount=0
```

> **副作用警告**：删除私有域后，VPC 内所有 EKS Pod 对该域名的解析请求将立即失败（NXDOMAIN）。请确认无在线业务依赖该域名解析后再执行删除。

### 数据面（需 VPN/IOA）

```bash
# 恢复 CoreDNS ConfigMap 至默认配置（若之前修改过）
kubectl edit configmap coredns -n kube-system
# 移除自定义 stubDomain 配置块

# 删除测试 Pod
kubectl delete pod dns-test -n NAMESPACE --ignore-not-found
kubectl delete pod dns-test-wildcard -n NAMESPACE --ignore-not-found
# expected: pod "dns-test" deleted
```

```bash
# 删除自定义 DNS 配置的容器实例
tccli tke DeleteEKSContainerInstances \
    --region <Region> \
    --EksCiIds '["eksci-xxxxxxxx"]'
# expected:
{
    "Response": {
        "RequestId": "..."
    }
}
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `UnauthorizedOperation` | `tccli privatedns DescribePrivateZoneList --region <Region>` | CAM 缺少 `privatedns:DescribePrivateZoneList` 权限 | 添加 Private DNS 只读策略：`tccli cam AttachUserPolicy --Uin <uin> --PolicyId <id>` |
| `InvalidParameter.DomainNotValid` | `tccli privatedns CreatePrivateZone --region <Region> --Domain "invalid_domain"` | 私有域名格式不合法（如含下划线、特殊字符） | 使用符合 RFC 标准的域名格式：`[a-zA-Z0-9][-a-zA-Z0-9]*.[a-zA-Z]{2,}` |
| `ResourceInUse.DomainExists` | `tccli privatedns CreatePrivateZone --region <Region> --Domain "mycompany.local"` | 该域名已创建私有域（可能属于其他 UIN） | 使用 `DescribePrivateZoneList` 确认域名归属，或使用不同域名 |
| `LimitExceeded.ZoneCount` | `tccli privatedns DescribePrivateZoneList --region <Region>` 查看计数 | 私有域数量达到配额上限（默认 20） | 删除不再使用的私有域，或提交工单提升配额 |
| `FailedOperation.VpcNotAssociated` | `tccli privatedns CreatePrivateZoneRecord` 后 Pod 内 nslookup 返回 NXDOMAIN | VPC 未关联到该私有域 | `tccli privatedns ModifyPrivateZoneVpc` 关联 VPC |
| `InvalidParameterValue.SubnetNotExist` | `CreateEKSContainerInstances` 时 `SubnetId` 无效 | 子网 ID 不存在或不在指定 VPC 中 | 使用 `tccli vpc DescribeSubnets` 查询有效的子网 ID |

### 创建成功但状态异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| DnsConfig 未生效 | `kubectl exec POD_NAME -- cat /etc/resolv.conf` | 容器实例创建时未设置 DnsConfig，或使用了旧镜像缓存 | 重新创建容器实例（EKS 容器实例不可原地更新 DnsConfig） |
| 私有域名解析失败（NXDOMAIN） | `kubectl exec POD_NAME -- nslookup api.internal.mycompany.local` | 私有域未关联到 Pod 所在 VPC | `tccli privatedns ModifyPrivateZoneVpc --VpcSet '[{"UniqVpcId":"VPC_ID","Region":"REGION"}]'` |
| 私有域名解析返回 SERVFAIL | `kubectl exec POD_NAME -- nslookup api.internal.mycompany.local` | Private DNS 服务异常或解析记录配置错误 | `tccli privatedns DescribePrivateZoneRecordList` 检查记录是否正确，确认上游 DNS 可达 |
| ndots 配置导致解析失败 | `kubectl exec POD_NAME -- nslookup kubernetes.default` 对比默认 ndots=5 和自定义 ndots=2 | ndots 过小时，短域名不会追加 search domain 查询 | 将 ndots 调整为 5（默认值），或始终使用 FQDN（末尾带 `.`） |
| CoreDNS ConfigMap 修改不生效 | `kubectl get configmap coredns -n kube-system -o yaml` 确认内容已更新 | CoreDNS reload 插件未加载或 ConfigMap 格式错误 | 检查 ConfigMap YAML 缩进（Corefile 数据块需正确缩进），执行 `kubectl rollout restart deployment coredns -n kube-system` |
| 自定义 Nameserver 无法访问 | `kubectl exec POD_NAME -- nslookup google.com` | DnsConfig 中指定的 Nameserver IP 在 VPC 内不可达 | 确认 Nameserver IP 在 Pod 所在子网的安全组规则中放通 UDP 53 端口 |
| 私有域名 TTL 未按预期生效 | `kubectl exec POD_NAME -- nslookup -debug api.internal.mycompany.local` 查看 TTL 值 | CoreDNS 缓存插件（cache 30）缓存了旧记录 | `kubectl delete pod -n kube-system -l k8s-app=kube-dns` 重启 CoreDNS Pod 清除缓存 |
| 混合 DNS 配置冲突 | `cat /etc/resolv.conf` 与预期不符 | EKS DnsConfig 与 CoreDNS stubDomain 同时配置，行为不确定 | 选择一种方式：Pod 粒度用 DnsConfig，VPC 粒度用 Private DNS + CoreDNS |

## 下一步

- [快速创建一个容器实例](../../../容器实例管理/快速创建一个容器实例/tccli%20操作.md)：EKS 容器实例的创建与 DnsConfig 配置
- [TKE DNS 最佳实践](../../网络/DNS%20相关/TKE%20DNS%20最佳实践/tccli%20操作.md)：TKE 集群 DNS 完整配置指南
- [Private DNS 产品文档](https://cloud.tencent.com/document/product/1338)：私有域解析的完整产品说明
- [CoreDNS 配置参考](https://coredns.io/manual/toc/)：CoreDNS 插件与配置官方文档

## 控制台替代

| 操作 | 控制台路径 | 等效 CLI |
|------|-----------|----------|
| 创建私有域 | Private DNS 控制台 > 私有域列表 > 新建私有域 | `tccli privatedns CreatePrivateZone` |
| 查看私有域列表 | Private DNS 控制台 > 私有域列表 | `tccli privatedns DescribePrivateZoneList` |
| 添加解析记录 | Private DNS 控制台 > 私有域详情 > 添加记录 | `tccli privatedns CreatePrivateZoneRecord` |
| 查看解析记录 | Private DNS 控制台 > 私有域详情 > 记录列表 | `tccli privatedns DescribePrivateZoneRecordList` |
| 关联/解绑 VPC | Private DNS 控制台 > 私有域详情 > 关联 VPC | `tccli privatedns ModifyPrivateZoneVpc` |
| 删除私有域 | Private DNS 控制台 > 私有域列表 > 删除 | `tccli privatedns DeletePrivateZone` |
| 创建含 DNS 配置的容器实例 | TKE 控制台 > Serverless 集群 > 新建 > 高级设置 > DNS 配置 | `tccli tke CreateEKSContainerInstances --cli-input-json`（含 DnsConfig） |
| 查看容器实例 DNS 配置 | TKE 控制台 > Serverless 集群 > 工作负载 > 详情 | `tccli tke DescribeEKSContainerInstances` |
| 编辑 CoreDNS ConfigMap | kubectl 命令行（需 VPN/IOA） | `kubectl edit configmap coredns -n kube-system` |
