# 公网访问相关（tccli）
> 对照官方：[公网访问相关](https://cloud.tencent.com/document/product/457/60222) · page_id `60222`

## 概述

在 TKE Serverless（EKS）集群中，Pod 默认运行于 VPC 内网环境。深度学习任务（镜像拉取、训练数据下载、模型上传）依赖公网访问能力。公网出口需通过 **NAT 网关**（批量共享出口）或 **弹性公网 IP / EIP**（单 Pod 独占出口）实现。

本文档覆盖 EKS 集群公网访问的常见问题诊断与排障流程。核心诊断链路：**EKS 集群状态 → VPC NAT 网关/路由表 → Pod EIP 绑定 → Pod 内网络连通性**。

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
| tke | `DescribeEKSContainerInstances` | 查询容器实例/Pod 信息 |
| vpc | `DescribeNatGateways` | 查询 NAT 网关 |
| vpc | `DescribeRouteTables` | 查询路由表配置 |
| vpc | `DescribeAddresses` | 查询弹性公网 IP |
| vpc | `DescribeSubnets` | 查询子网信息 |
| vpc | `DescribeVpcs` | 查询 VPC 信息 |

### 资源检查

```bash
# 确认 EKS 集群存在且运行正常
tccli tke DescribeEKSClusters \
    --region <Region> \
    --ClusterIds '["EKS_CLUSTER_ID"]'
# expected:
{
    "Response": {
        "Clusters": [
            {
                "ClusterId": "EKS_CLUSTER_ID",
                "ClusterName": "eks-deep-learning",
                "ClusterDesc": "EKS deep learning cluster",
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
# 确认 VPC 存在
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
| 地域 | `--region` / `Region` | String | 是 | 地域标识，如 `ap-guangzhou` | 无影响 |
| 集群 ID | `--ClusterId` / `ClusterIds` | String | 是 | EKS 集群 ID | 查询操作天然幂等 |
| EIP ID | `--AddressId` | String | 是 | 弹性公网 IP 的 ID | 查询操作天然幂等 |
| NAT 网关 ID | `--NatGatewayId` | String | 是 | NAT 网关资源 ID | 查询操作天然幂等 |
| VPC ID | `--VpcId` | String | 是 | VPC 网络 ID | 查询操作天然幂等 |
| 子网 ID | `--SubnetId` | String | 是 | 子网 ID | 查询操作天然幂等 |
| 路由表 ID | `--RouteTableId` | String | 是 | 路由表资源 ID | 查询操作天然幂等 |
| Pod 名称 | kubectl `POD_NAME` | String | 是 | 目标 Pod | 查询操作天然幂等 |

## 操作步骤

### 选择依据

EKS 公网访问问题按症状分为三类诊断路径：

1. **镜像拉取超时 / DNS 解析失败**：跳至「场景一：NAT 网关与路由诊断」。
2. **Pod 无公网 IP 但需固定出口 IP**：跳至「场景二：EIP 绑定诊断」。
3. **基础设施正常但 Pod 内无法访问公网**：跳至「场景三：Pod 内网络诊断」。

### 最小创建（诊断基础）

#### 场景一：NAT 网关与路由诊断

**步骤 1** — 查询 EKS 集群所在 VPC 的 NAT 网关。

| 参数 | 值 | 说明 |
|------|----|------|
| `REGION` | 地域 | `ap-guangzhou` |
| `VPC_ID` | VPC 标识 | EKS 集群所在 VPC |

```bash
tccli vpc DescribeNatGateways \
    --region <Region> \
    --Filters '[{"Name":"vpc-id","Values":["VPC_ID"]}]'
# expected:
{
    "Response": {
        "NatGatewaySet": [
            {
                "NatGatewayId": "nat-xxxxxxxx",
                "NatGatewayName": "eks-nat-gateway",
                "State": "AVAILABLE",
                "InternetMaxBandwidthOut": 100,
                "PublicIpAddressSet": ["203.0.113.10"],
                "VpcId": "VPC_ID"
            }
        ],
        "TotalCount": 1,
        "RequestId": "..."
    }
}
```

**步骤 2** — 查询 EKS 容器实例所在子网的路由表。

```bash
# 先获取子网所属路由表
tccli vpc DescribeSubnets \
    --region <Region> \
    --SubnetIds '["SUBNET_ID"]'
# expected:
{
    "Response": {
        "SubnetSet": [
            {
                "SubnetId": "SUBNET_ID",
                "VpcId": "VPC_ID",
                "CidrBlock": "10.0.1.0/24",
                "RouteTableId": "rtb-xxxxxxxx"
            }
        ],
        "RequestId": "..."
    }
}
```

```bash
# 查询路由表详细路由策略
tccli vpc DescribeRouteTables \
    --region <Region> \
    --RouteTableIds '["rtb-xxxxxxxx"]'
# expected:
{
    "Response": {
        "RouteTableSet": [
            {
                "RouteTableId": "rtb-xxxxxxxx",
                "RouteSet": [
                    {
                        "DestinationCidrBlock": "0.0.0.0/0",
                        "GatewayType": "NAT",
                        "GatewayId": "nat-xxxxxxxx",
                        "Enabled": true
                    },
                    {
                        "DestinationCidrBlock": "10.0.0.0/8",
                        "GatewayType": "LOCAL",
                        "Enabled": true
                    }
                ]
            }
        ],
        "RequestId": "..."
    }
}
```

**诊断结论**：确认路由表中存在 `DestinationCidrBlock=0.0.0.0/0, GatewayType=NAT, Enabled=true` 条目。若无此路由，Pod 流量不经 NAT 网关，无法访问公网。

#### 场景二：EIP 绑定诊断

```bash
# 查询账号下所有 EIP 及其绑定状态
tccli vpc DescribeAddresses \
    --region <Region>
# expected:
{
    "Response": {
        "AddressSet": [
            {
                "AddressId": "EIP_ID",
                "AddressIp": "203.0.113.10",
                "AddressStatus": "BIND",
                "InstanceId": "eksci-xxxxxxxx",
                "IsEip": true,
                "InternetMaxBandwidthOut": 10
            }
        ],
        "TotalCount": 1,
        "RequestId": "..."
    }
}
```

**诊断结论**：
- `AddressStatus=BIND` 表示 EIP 已绑定到实例，`UNBIND` 表示未绑定。
- `InstanceId` 指向当前绑定的 EKS 容器实例 ID。
- 若 EIP 数量为 0，检查账户 EIP 配额。

### 增强配置

#### 场景三：Pod 内网络连通性诊断

**步骤 1** — 查询 EKS 容器实例详情。

```bash
tccli tke DescribeEKSContainerInstances \
    --region <Region> \
    --EksCiId EKS_CLUSTER_ID \
    --Limit 20
# expected:
{
    "Response": {
        "EksCiSet": [
            {
                "EksCiId": "eksci-xxxxxxxx",
                "Name": "deep-learning-trainer",
                "Status": "Running",
                "EipAddress": "",
                "VpcId": "VPC_ID",
                "SubnetId": "SUBNET_ID",
                "Cpu": 8.0,
                "Memory": 32.0
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

**步骤 2** — 从 Pod 内测试外网连通性。

```bash
# DNS 解析测试
kubectl exec -n NAMESPACE deep-learning-trainer -- nslookup www.baidu.com
# expected (通):
# Server:     10.0.0.2
# Address:    10.0.0.2#53
# Non-authoritative answer:
# Name:   www.baidu.com
# Address: 110.242.68.66

# expected (不通):
# ;; connection timed out; no servers could be reached
```

```bash
# HTTP 连通性测试
kubectl exec -n NAMESPACE deep-learning-trainer -- \
    curl -v -m 5 https://www.baidu.com
# expected (通):
# * Connected to www.baidu.com (110.242.68.66) port 443
# < HTTP/1.1 200 OK

# expected (不通):
# curl: (28) Connection timed out after 5000 milliseconds
```

```bash
# 镜像仓库连通性测试（Docker Hub）
kubectl exec -n NAMESPACE deep-learning-trainer -- \
    wget -qO- --timeout=5 https://registry-1.docker.io/v2/
# expected (通):
# {}  或 401 Unauthorized（表示连通，仅未认证）

# expected (不通):
# wget: bad address 'registry-1.docker.io'
```

## 验证

### 控制面（tccli）

```bash
# 验证 NAT 网关状态为 AVAILABLE
tccli vpc DescribeNatGateways \
    --region <Region> \
    --NatGatewayIds '["nat-xxxxxxxx"]' \
    | jq '.Response.NatGatewaySet[0].State'
# expected: "AVAILABLE"

# 验证 EIP 绑定状态为 BIND
tccli vpc DescribeAddresses \
    --region <Region> \
    --AddressIds '["EIP_ID"]' \
    | jq '.Response.AddressSet[0].AddressStatus'
# expected: "BIND"

# 验证路由表含公网路由
tccli vpc DescribeRouteTables \
    --region <Region> \
    --RouteTableIds '["rtb-xxxxxxxx"]' \
    | jq '.Response.RouteTableSet[0].RouteSet[] | select(.DestinationCidrBlock=="0.0.0.0/0")'
# expected: GatewayType="NAT", Enabled=true
```

### 数据面（需 VPN/IOA）

```bash
# 验证 Pod 可访问公网
kubectl exec -n NAMESPACE deep-learning-trainer -- \
    wget -qO- --timeout=5 http://httpbin.org/ip
# expected: {"origin": "203.0.113.10"}

# 验证可拉取公网镜像
kubectl run test-pull --image=nginx:alpine --restart=Never --rm -it -n NAMESPACE -- \
    echo "image pull success"
# expected: pod "test-pull" deleted ... image pull success
```

## 清理

> **账单警告**：NAT 网关按小时计费（约 0.36 元/小时起），EIP 闲置按小时计费（约 0.06 元/小时起），请确认无业务使用后再执行释放。

### 控制面（tccli）

NAT 网关和 EIP 为共享基础设施，清理前需确认无其他业务依赖。

**验证无其他依赖后再释放：**

```bash
# 查看 EIP 绑定状态
tccli vpc DescribeAddresses \
    --region <Region> \
    --AddressIds '["EIP_ID"]'
# 确认 AddressStatus=UNBIND 且无 InstanceId 关联后可释放

# 查看 NAT 网关下 SNAT 规则关联的子网
tccli vpc DescribeNatGatewayDestinationIpPortTranslationNatRules \
    --region <Region> \
    --NatGatewayId nat-xxxxxxxx
# 确认无业务子网关联后可删除
```

> **副作用警告**：释放 EIP 后公网 IP 地址立即回收且不可恢复。删除 NAT 网关前请确保关联路由表中已移除指向该 NAT 的 `0.0.0.0/0` 路由。

### 数据面（需 VPN/IOA）

```bash
# 删除调试用测试 Pod
kubectl delete pod -n NAMESPACE test-pull --grace-period=30 --ignore-not-found
# expected: pod "test-pull" deleted
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `UnauthorizedOperation` | `tccli vpc DescribeNatGateways --region <Region>` | CAM 缺少 `vpc:DescribeNatGateways` 权限 | `tccli cam AttachUserPolicy --Uin <uin> --PolicyId <id>` 添加 VPC 只读策略 |
| `ResourceNotFound.NatGatewayNotFound` | `tccli vpc DescribeNatGateways --region <Region> --NatGatewayIds '["nat-xxxxxxxx"]'` | NAT 网关 ID 不存在或已被删除 | 检查 ID 是否正确，或通过控制台重建 NAT 网关 |
| `LimitExceeded.AddressQuota` | `tccli vpc DescribeAddressQuota --region <Region>` | EIP 配额已满（默认 20 个） | 释放闲置 EIP 或提交工单提升配额 |
| `InvalidParameterValue.SubnetNotExist` | `tccli vpc DescribeSubnets --region <Region> --SubnetIds '["SUBNET_ID"]'` | 子网 ID 无效或子网已被删除 | 检查子网 ID 是否正确，确认子网仍在目标 VPC 内 |
| `RequestLimitExceeded` | 查看错误响应中的 `RequestId` | API 调用频率超限（默认 20 次/秒） | 添加 `sleep 1` 重试间隔，批量查询时合并过滤器 |
| `InvalidParameterValue.AddressIdMalformed` | 参数中 `AddressId` 格式 | EIP ID 格式错误，应为 `eip-xxxxxxxx` | 使用 `tccli vpc DescribeAddresses` 查询正确的 EIP 列表 |

### 创建成功但状态异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Pod 无法解析域名 | `kubectl exec -n NAMESPACE POD_NAME -- nslookup www.baidu.com` | CoreDNS 不可达或 kube-dns Service 异常 | 检查 `kube-system` 下 CoreDNS Pod 状态，确保 Service ClusterIP 可达 |
| NAT 已绑定但 Pod 无公网 | `tccli vpc DescribeRouteTables --RouteTableIds '["rtb-xxxxxxxx"]'` | 路由表缺少 `0.0.0.0/0 -> NAT` 路由条目 | `tccli vpc CreateRoutes --RouteTableId rtb-xxxxxxxx --Routes '[{"DestinationCidrBlock":"0.0.0.0/0","GatewayType":"NAT","GatewayId":"nat-xxxxxxxx"}]'` |
| EIP 已绑定但 Pod 无法外访 | `tccli vpc DescribeAddresses --AddressIds '["EIP_ID"]'` | EIP 带宽为 0 Mbps 或流量被限速 | 调整 EIP 带宽：`tccli vpc ModifyAddressAttribute --AddressId EIP_ID --InternetMaxBandwidthOut 50` |
| curl 超时但 DNS 解析正常 | `kubectl exec -n NAMESPACE POD_NAME -- curl -v -m 5 https://example.com` | 安全组出站规则拦截特定端口 | 检查 Pod 所在子网关联的安全组出站规则，放通 TCP 80/443/8080 |
| NAT 网关状态为 PENDING | `tccli vpc DescribeNatGateways --NatGatewayIds '["nat-xxxxxxxx"]'` | NAT 网关创建中或配置变更中 | 等待 1-2 分钟，状态变更为 `AVAILABLE` 后重试 |
| EIP 状态为 BIND 但无法 ping | `tccli vpc DescribeAddresses --AddressIds '["EIP_ID"]'` | EIP 被 DDoS 封堵（黑洞） | 前往 [DDoS 防护控制台](https://console.cloud.tencent.com/ddos) 查看封堵状态，解封后恢复 |
| 同一 NAT 下部分 Pod 通部分不通 | `kubectl get pod -n NAMESPACE -o wide` 对比 Pod 所在子网 | 不通 Pod 的子网路由表未指向该 NAT 网关 | 为每个子网路由表添加 `0.0.0.0/0 -> NAT` 路由 |
| 深度学习镜像拉取极慢 | `kubectl describe pod -n NAMESPACE POD_NAME` 查看 Events | NAT 网关出带宽不足，大镜像（> 10 GB）拉取受阻 | 升级 NAT 网关带宽规格或改用 TCR 内网拉取 |
| 偶发性超时 | 连续执行 10 次：`kubectl exec -n NAMESPACE POD_NAME -- curl -m 3 https://www.baidu.com` | NAT 网关并发连接数达上限（默认 100 万 SNAT 端口） | 增加 NAT 网关数量做负载分担，或使用多个 EIP |

## 下一步

- [通过 NAT 网关访问外网](../../../通过%20NAT%20网关访问外网/tccli%20操作.md)：NAT 网关的创建与配置操作指南
- [通过弹性公网 IP 访问外网](../../../通过弹性公网%20IP%20访问外网/tccli%20操作.md)：EIP 绑定 Pod 的操作指南
- [NAT 网关产品文档](https://cloud.tencent.com/document/product/552)：NAT 网关的详细产品说明
- [弹性公网 IP 产品文档](https://cloud.tencent.com/document/product/1199)：EIP 的详细产品说明
- [VPC 路由表配置](https://cloud.tencent.com/document/product/215/39406)：子网路由策略管理
- [安全组规则配置](https://cloud.tencent.com/document/product/215/39793)：安全组出入站规则管理

## 控制台替代

| 操作 | 控制台路径 | 等效 CLI |
|------|-----------|----------|
| 查看 NAT 网关 | VPC 控制台 > NAT 网关 | `tccli vpc DescribeNatGateways` |
| 查看路由表 | VPC 控制台 > 路由表 | `tccli vpc DescribeRouteTables` |
| 查看 EIP | VPC 控制台 > 弹性公网 IP | `tccli vpc DescribeAddresses` |
| 查看 EKS 容器实例 | TKE 控制台 > Serverless 集群 > 工作负载 | `tccli tke DescribeEKSContainerInstances` |
| 添加路由策略 | VPC 控制台 > 路由表 > 新增路由策略 | `tccli vpc CreateRoutes` |
| 绑定 EIP | VPC 控制台 > 弹性公网 IP > 更多 > 绑定 | `tccli vpc AssociateAddress` |
| 解绑 EIP | VPC 控制台 > 弹性公网 IP > 更多 > 解绑 | `tccli vpc DisassociateAddress` |
| 调整 EIP 带宽 | VPC 控制台 > 弹性公网 IP > 调整网络 | `tccli vpc ModifyAddressAttribute` |
| 查看安全组 | VPC 控制台 > 安全组 | `tccli vpc DescribeSecurityGroups` |
