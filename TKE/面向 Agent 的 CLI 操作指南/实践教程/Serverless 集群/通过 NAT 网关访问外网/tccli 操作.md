# 通过 NAT 网关访问外网（tccli）
> 对照官方：[通过 NAT 网关访问外网](https://cloud.tencent.com/document/product/457/48710) · page_id `48710`
## 概述
为 TKE Serverless（EKS）集群的子网配置 NAT 网关，使得 VPC 内无公网 IP 的 Pod 通过 NAT 网关的统一 SNAT 出口访问外网（如拉取公网镜像、pip install、apt update）。本文使用 EKS API 管理集群上下文，VPC API 管理 NAT 网关与路由表。

**数据流**: Pod (VPC 内网) -> 子网路由表 -> NAT 网关 (SNAT) -> Internet

**核心 API**:
- 控制面: `tccli tke CreateEKSCluster`, `tccli tke DescribeEKSClusters`, `tccli tke DeleteEKSCluster`
- NAT 网关: `tccli vpc CreateNatGateway`, `tccli vpc DescribeNatGateways`, `tccli vpc DeleteNatGateway`
- 路由: `tccli vpc CreateRoutes`, `tccli vpc DeleteRoutes`, `tccli vpc DescribeRouteTables`
- 数据面: `kubectl exec` (外网连通性测试)

## 前置条件

### 环境检查

```bash
# 1. 验证 tccli 版本
tccli --version
# expected: tccli version 1.x.x

# 2. 验证凭证有效性
tccli sts GetCallerIdentity
# expected:
# {
#     "AccountId": "1xxxxxxxxx",
#     "Arn": "arn:aws:sts::1xxxxxxxxx:assumed-role/...",
#     "UserId": "1xxxxxxxxx",
#     "RequestId": "xxx"
# }

# 3. 验证 CAM Action — VPC 读权限
tccli vpc DescribeVpcs \
    --region <Region> \
    --Limit 1
# expected: (含有 RequestId，无 UnauthorizedOperation 错误)

# 4. 验证 CAM Action — EKS 读权限
tccli tke DescribeEKSClusters \
    --region <Region> \
    --Limit 1
# expected: (含有 RequestId，无 UnauthorizedOperation 错误)
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

### 资源检查

```bash
# 5. 查询 EKS 集群列表
tccli tke DescribeEKSClusters \
    --region <Region> \
    --Limit 20
# expected:
# {
#     "Clusters": [
#         {
#             "ClusterId": "CLUSTER_ID",
#             "ClusterName": "eks-demo",
#             "VpcId": "VPC_ID",
#             "SubnetIds": ["SUBNET_ID"],
#             "K8SVersion": "1.30.0",
#             "Status": "Running"
#         }
#     ],
#     "TotalCount": 1,
#     "RequestId": "xxx"
# }

# 6. 查询 VPC 列表
tccli vpc DescribeVpcs \
    --region <Region> \
    --Limit 20
# expected:
# {
#     "VpcSet": [
#         {
#             "VpcId": "VPC_ID",
#             "VpcName": "...",
#             "CidrBlock": "10.0.0.0/16"
#         }
#     ],
#     "TotalCount": N,
#     "RequestId": "xxx"
# }

# 7. 查询子网列表
tccli vpc DescribeSubnets \
    --region <Region> \
    --Filters '[{"Name":"vpc-id","Values":["VPC_ID"]}]' \
    --Limit 20
# expected:
# {
#     "SubnetSet": [
#         {
#             "SubnetId": "SUBNET_ID",
#             "SubnetName": "...",
#             "VpcId": "VPC_ID",
#             "CidrBlock": "10.0.1.0/24",
#             "Zone": "REGION-3"
#         }
#     ],
#     "TotalCount": 1,
#     "RequestId": "xxx"
# }

# 8. 检查 NAT 网关配额（每 VPC 默认上限 5）
tccli vpc DescribeNatGateways \
    --region <Region> \
    --Filters '[{"Name":"vpc-id","Values":["VPC_ID"]}]' \
    --Limit 20
# expected:
# {
#     "NatGatewaySet": [],
#     "TotalCount": 0,
#     "RequestId": "xxx"
# }
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

| 控制台参数 | CLI 参数 | Idempotency 行为 | 约束 |
|---|---|---|---|
| 网关名称 | `NatGatewayName` | 同名可重复创建（不同 NatGatewayId） | 1-60 字符 |
| 所属网络 | `VpcId` | 同 VPC 可建多个 NAT 网关 | 格式 vpc-xxxxxxxx |
| 弹性 IP | `AddressCount` / `AddressId` | 绑定关系与 EIP 状态耦合 | 需先申请 EIP 或指定已有 EIP ID |
| 网关类型 | `NatProductVersion` | 创建后不可更改，需重建 | 1 (标准型) / 2 (传统型) |
| 出带宽上限 | `InternetMaxBandwidthOut` | 可后续 ModifyNatGatewayAttribute 修改 | 0-5000 Mbps |
| 并发连接数 | `MaxConcurrentConnection` | 可后续 Modify 修改 | 1000000-10000000 |
| Region | `--region` | 必填，网关须与 VPC 同地域 | REGION |
| EKS 集群名 | `ClusterName` (CreateEKSCluster) | 同名不可重复创建 | 1-63 字符 |

## 操作步骤

### 占位符说明

| 占位符 | 说明 | 约束 | 获取方式 |
|---|---|---|---|
| `CLUSTER_ID` | EKS 集群 ID | 格式 cls-xxxxxxxx | `DescribeEKSClusters` |
| `VPC_ID` | VPC ID | 格式 vpc-xxxxxxxx | `DescribeVpcs` |
| `SUBNET_ID` | 子网 ID | 格式 subnet-xxxxxxxx | `DescribeSubnets` |
| `REGION` | 地域 | ap-guangzhou | 固定值 |
| `EIP_ID` | 弹性公网 IP ID | 格式 eip-xxxxxxxx | `AllocateAddresses` 返回值 |

#### 选择依据

NAT 网关适用于**多 Pod 共享同一出口 IP**、需要集中管理出向流量和安全策略的生产场景。若仅单个 Pod 需要外网访问，优先考虑 EIP 绑定方案（见 page_id 60354）。

1. **计费模式**: NAT 网关按小时 + 按流量计费，持续运行成本较高，测试环境建议随建随删
2. **网关类型**: `NatProductVersion` — 标准型 (1) 支持 DDoS 高防、精细化流量控制；传统型 (2) 功能较少但成本更低
3. **EIP 前置**: NAT 网关创建时需绑定 EIP 作为 SNAT 源地址，EIP 闲置于未绑定状态按小时收取闲置费
4. **子网范围**: 路由表 NAT 条目的 `DestinationCidrBlock: 0.0.0.0/0` 会作用到该路由表关联的所有子网

#### 最小创建

```bash
# 步骤 1: 为 NAT 网关申请弹性公网 IP
# >=4 参数，使用 --cli-input-json
cat > nat-allocate-eip.json <<'EOF'
{
    "AddressCount": 1,
    "InternetChargeType": "TRAFFIC_POSTPAID_BY_HOUR",
    "AddressName": "nat-gateway-eip",
    "Tags": [
        {"Key": "purpose", "Value": "eks-nat"}
    ]
}
EOF
tccli vpc AllocateAddresses \
    --region <Region> \
    --cli-input-json file://nat-allocate-eip.json
# expected:
# {
#     "AddressSet": ["EIP_ID"],
#     "TaskId": "xxx",
#     "RequestId": "xxx"
# }

# 步骤 2: 创建 NAT 网关（标准型）
cat > create-nat-gateway.json <<'EOF'
{
    "NatGatewayName": "eks-nat-gateway",
    "VpcId": "VPC_ID",
    "InternetMaxBandwidthOut": 100,
    "MaxConcurrentConnection": 1000000,
    "AddressCount": 1,
    "NatProductVersion": 1
}
EOF
tccli vpc CreateNatGateway \
    --region <Region> \
    --cli-input-json file://create-nat-gateway.json
# expected:
# {
#     "NatGatewaySet": [
#         {
#             "NatGatewayId": "nat-xxxxxxxx",
#             "NatGatewayName": "eks-nat-gateway",
#             "VpcId": "VPC_ID",
#             "State": "PENDING",
#             "CreatedTime": "2026-06-17T00:00:00Z"
#         }
#     ],
#     "TotalCount": 1,
#     "RequestId": "xxx"
# }

# 步骤 3: 轮询 NAT 网关状态直到 AVAILABLE
tccli vpc DescribeNatGateways \
    --region <Region> \
    --NatGatewayIds '["nat-xxxxxxxx"]'
# expected:
# {
#     "NatGatewaySet": [
#         {
#             "NatGatewayId": "nat-xxxxxxxx",
#             "NatGatewayName": "eks-nat-gateway",
#             "State": "AVAILABLE",
#             "PublicIpAddressSet": ["x.x.x.x"],
#             "CreatedTime": "..."
#         }
#     ],
#     "TotalCount": 1,
#     "RequestId": "xxx"
# }

# 步骤 4: 查询子网关联的路由表
tccli vpc DescribeRouteTables \
    --region <Region> \
    --Filters '[{"Name":"vpc-id","Values":["VPC_ID"]},{"Name":"association.subnet-id","Values":["SUBNET_ID"]}]'
# expected:
# {
#     "RouteTableSet": [
#         {
#             "RouteTableId": "rtb-xxxxxxxx",
#             "RouteTableName": "default",
#             "VpcId": "VPC_ID",
#             "RouteSet": [
#                 {
#                     "DestinationCidrBlock": "10.0.0.0/16",
#                     "GatewayType": "LOCAL",
#                     "GatewayId": "local",
#                     "Enabled": true
#                 }
#             ]
#         }
#     ],
#     "RequestId": "xxx"
# }
```

#### 增强配置

```bash
# 步骤 5: 在路由表中添加 0.0.0.0/0 -> NAT 网关路由条目
cat > create-nat-route.json <<'EOF'
{
    "RouteTableId": "rtb-xxxxxxxx",
    "Routes": [
        {
            "DestinationCidrBlock": "0.0.0.0/0",
            "GatewayType": "NAT",
            "GatewayId": "nat-xxxxxxxx",
            "RouteDescription": "eks-default-nat-route"
        }
    ]
}
EOF
tccli vpc CreateRoutes \
    --region <Region> \
    --cli-input-json file://create-nat-route.json
# expected:
# {
#     "TotalCount": 1,
#     "RouteTableSet": [
#         {
#             "RouteTableId": "rtb-xxxxxxxx",
#             "RouteSet": [
#                 {
#                     "DestinationCidrBlock": "0.0.0.0/0",
#                     "GatewayType": "NAT",
#                     "GatewayId": "nat-xxxxxxxx",
#                     "RouteDescription": "eks-default-nat-route",
#                     "Enabled": true
#                 }
#             ]
#         }
#     ],
#     "RequestId": "xxx"
# }

# 步骤 6: (可选) 调整 NAT 网关出带宽上限
tccli vpc ModifyNatGatewayAttribute \
    --region <Region> \
    --NatGatewayId nat-xxxxxxxx \
    --InternetMaxBandwidthOut 500
# expected:
# {
#     "RequestId": "xxx"
# }

# 步骤 7: (可选) 调整 NAT 网关最大并发连接数
tccli vpc ModifyNatGatewayAttribute \
    --region <Region> \
    --NatGatewayId nat-xxxxxxxx \
    --MaxConcurrentConnection 3000000
# expected:
# {
#     "RequestId": "xxx"
# }

# 步骤 8: 验证最终路由配置
tccli vpc DescribeRouteTables \
    --region <Region> \
    --RouteTableIds '["rtb-xxxxxxxx"]'
# expected:
# {
#     "RouteTableSet": [
#         {
#             "RouteTableId": "rtb-xxxxxxxx",
#             "RouteSet": [
#                 {
#                     "DestinationCidrBlock": "10.0.0.0/16",
#                     "GatewayType": "LOCAL",
#                     "GatewayId": "local",
#                     "Enabled": true
#                 },
#                 {
#                     "DestinationCidrBlock": "0.0.0.0/0",
#                     "GatewayType": "NAT",
#                     "GatewayId": "nat-xxxxxxxx",
#                     "Enabled": true
#                 }
#             ]
#         }
#     ],
#     "RequestId": "xxx"
# }
```

## 验证

### 控制面（tccli）

```bash
# 1. 验证 NAT 网关 AVAILABLE 且绑定 EIP
tccli vpc DescribeNatGateways \
    --region <Region> \
    --NatGatewayIds '["nat-xxxxxxxx"]'
# expected:
# {
#     "NatGatewaySet": [
#         {
#             "NatGatewayId": "nat-xxxxxxxx",
#             "State": "AVAILABLE",
#             "PublicIpAddressSet": ["x.x.x.x"],
#             "MaxConcurrentConnection": 3000000,
#             "InternetMaxBandwidthOut": 500
#         }
#     ],
#     "TotalCount": 1,
#     "RequestId": "xxx"
# }

# 2. 验证路由条目 Enabled
tccli vpc DescribeRouteTables \
    --region <Region> \
    --Filters '[{"Name":"route.gateway-id","Values":["nat-xxxxxxxx"]}]'
# expected:
# {
#     "RouteTableSet": [
#         {
#             "RouteTableId": "rtb-xxxxxxxx",
#             "RouteSet": [
#                 {
#                     "DestinationCidrBlock": "0.0.0.0/0",
#                     "GatewayType": "NAT",
#                     "GatewayId": "nat-xxxxxxxx",
#                     "Enabled": true
#                 }
#             ]
#         }
#     ],
#     "RequestId": "xxx"
# }
```

### 数据面（需 VPN/IOA）

> **kubectl 数据面不可达**：CAM 拒绝公网端点 (strategyId:240463971)，内网需 VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

```bash
# 3. 创建测试 Pod（无公网 IP，调度到 EKS 子网）
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
    name: nat-egress-test
    namespace: default
    labels:
        test: nat-gateway
spec:
    containers:
        - name: busybox
          image: busybox:1.36
          command: ["sleep", "3600"]
    restartPolicy: Never
EOF
# expected: pod/nat-egress-test created

# 4. 等待 Pod Running
kubectl wait --for=condition=Ready pod/nat-egress-test --timeout=120s
# expected: pod/nat-egress-test condition met

# 5. 从 Pod 内测试外网连通性，验证出口 IP 为 NAT 网关 EIP
kubectl exec nat-egress-test -- wget -q -O- --timeout=10 http://httpbin.org/ip
# expected:
# {
#     "origin": "x.x.x.x"
# }
# 注: origin 应与步骤 3 中 PublicIpAddressSet 的 EIP 一致

# 6. 测试 DNS 解析
kubectl exec nat-egress-test -- nslookup httpbin.org
# expected: (返回 DNS 解析结果，无超时)
```

## 清理

> **计费警告**: NAT 网关按小时计费（标准型约 0.45 元/小时），EIP 闲置于未绑定状态按小时收取闲置费（约 0.20 元/小时）。完成测试后请立即清理全部资源，避免持续扣费。

> **副作用警告**: 删除 NAT 网关后，子网内所有通过该网关出公网的 Pod 将失去外网访问能力。请确认无业务依赖后再执行清理。

### 控制面（tccli）

```bash
# 1. 删除路由表中指向 NAT 网关的 0.0.0.0/0 路由条目
tccli vpc DeleteRoutes \
    --region <Region> \
    --RouteTableId rtb-xxxxxxxx \
    --Routes '[{"DestinationCidrBlock":"0.0.0.0/0","GatewayType":"NAT","GatewayId":"nat-xxxxxxxx"}]'
# expected:
# {
#     "RequestId": "xxx"
# }

# 2. 删除 NAT 网关
tccli vpc DeleteNatGateway \
    --region <Region> \
    --NatGatewayId nat-xxxxxxxx
# expected:
# {
#     "RequestId": "xxx"
# }

# 3. 释放弹性公网 IP
tccli vpc ReleaseAddresses \
    --region <Region> \
    --AddressIds '["EIP_ID"]'
# expected:
# {
#     "RequestId": "xxx"
# }

# 4. 验证 NAT 网关已删除（Describe 返回空或 State DELETING）
tccli vpc DescribeNatGateways \
    --region <Region> \
    --NatGatewayIds '["nat-xxxxxxxx"]'
# expected:
# {
#     "NatGatewaySet": [],
#     "TotalCount": 0,
#     "RequestId": "xxx"
# }

# 5. 验证 EIP 已释放
tccli vpc DescribeAddresses \
    --region <Region> \
    --AddressIds '["EIP_ID"]'
# expected: (空列表或 EIP 状态为 RELEASED，或 AddressSet 为空)
```

### 数据面（需 VPN/IOA）

```bash
# 6. 删除测试 Pod
kubectl delete pod nat-egress-test --namespace default
# expected: pod "nat-egress-test" deleted

# 7. 确认 Pod 已删除
kubectl get pod nat-egress-test --namespace default
# expected: Error from server (NotFound): pods "nat-egress-test" not found
```

```text
NAME  STATUS  AGE
...
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|---|---|---|---|
| `CreateNatGateway` 返回 `LimitExceeded` | `DescribeNatGateways` 确认 VPC 下 NAT 网关数量 | 超出每 VPC 默认配额（5 个） | 删除闲置 NAT 网关或提交工单提升配额 |
| `CreateNatGateway` 返回 `InvalidParameterValue.AddressNotAvailable` | `DescribeAddresses` 确认 EIP 绑定状态 | 指定 EIP 已绑定到其他资源 | 释放并重新申请 EIP，或使用 `AddressCount` 自动分配 |
| `CreateRoutes` 返回 `InvalidParameterValue.RouteConflict` | `DescribeRouteTables` 检查现有 0.0.0.0/0 路由 | 已存在默认路由条目（优先级冲突） | 删除冲突路由后重试；或指定替换已有路由 |
| `DeleteNatGateway` 返回 `ResourceInUse` | `DescribeRouteTables` 检查关联路由条目 | 路由表中仍存在指向该 NAT 网关的路由 | 先执行 `DeleteRoutes` 删除关联条目，再删除 NAT 网关 |
| `ReleaseAddresses` 返回 `InvalidParameterValue.AddressNotFound` | EIP 可能已被其他操作释放 | EIP 状态不一致（已解绑/已释放） | 通过 `DescribeAddresses` 确认 EIP 当前状态和绑定关系 |

### 创建成功但状态异常

| 现象 | 诊断 | 根因 | 修复 |
|---|---|---|---|
| NAT 网关创建成功但 Pod 无法访问外网 | `DescribeRouteTables` 检查子网关联路由表 | 路由表未关联到 Pod 所在子网，或 0.0.0.0/0 路由未 Enable | 确认子网关联的路由表包含 `Enabled: true` 的 NAT 路由条目 |
| NAT 网关状态长时间 PENDING (>5 分钟) | `DescribeNatGateways` 轮询状态，检查 `PublicIpAddressSet` | EIP 分配延迟或账号余额不足 | 等待超时后删除重建；检查 EIP 配额和账户状态 |
| Pod 外网连接间歇性失败 | `kubectl exec` 重复外网测试，查看 NAT 网关监控 | 并发连接数达到 `MaxConcurrentConnection` 上限 | 通过 `ModifyNatGatewayAttribute` 上调并发连接数上限 |
| Pod 出口 IP 非 NAT 网关 EIP | `kubectl exec dnstools -- wget -q -O- httpbin.org/ip` | Pod 所在 ENI 可能单独绑定了 EIP，或存在其他出口路由 | 排查 Pod 安全组/ENI 的 EIP 绑定，检查路由表优先级 |
| 已有 Pod 仍使用旧路由 | `kubectl describe pod` 查看 Pod 网络配置 | 路由表变更后，已有 Pod 的路由缓存未刷新 | 重建受影响 Pod 或等待路由收敛（通常 1-2 分钟） |

## 下一步

- [通过弹性公网 IP 访问外网](../通过弹性公网%20IP%20访问外网/tccli%20操作.md)：单 Pod EIP 直通方案，适用于固定出口 IP 场景
- [使用 Cloud NAT 网关](https://cloud.tencent.com/document/product/552)：NAT 网关产品完整文档，含高级安全策略
- [管理 EKS 集群](https://cloud.tencent.com/document/product/457/xxxxx)：EKS 集群网络模型、超级节点与安全组配置

## 控制台替代
控制台完成本节操作需在 **私有网络 > NAT 网关** 页面创建网关，再切换到 **路由表** 页面添加路由，最后返回 NAT 网关页面确认状态。tccli 将 NAT 网关创建、路由配置、状态校验编排为顺序命令流，通过 `DescribeNatGateways` / `DescribeRouteTables` 闭环验证，适合 Terraform/Pulumi 等 IaC 工具集成和 CI/CD 自动化。
