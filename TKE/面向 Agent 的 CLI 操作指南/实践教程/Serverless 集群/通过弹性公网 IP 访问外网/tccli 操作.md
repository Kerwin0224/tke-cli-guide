# 通过弹性公网 IP 访问外网（tccli）
> 对照官方：[通过弹性公网 IP 访问外网](https://cloud.tencent.com/document/product/457/60354) · page_id `60354`
## 概述
在 TKE Serverless（EKS）集群中，通过为 Pod 的 ENI 绑定弹性公网 IP（EIP），实现特定 Pod 拥有独立的公网访问能力。与 NAT 网关方案不同，EIP 方案为每个 Pod 提供**独占公网 IP**，适用于需要对外暴露固定 IP 的场景（如游戏服务器、API 网关出口、需要 IP 白名单的第三方服务对接）。

**数据流**: Pod ENI -> EIP -> Internet（每个 Pod 独立 IP）

**核心 API**:
- 控制面: `tccli tke CreateEKSCluster`, `tccli tke DescribeEKSClusters`
- EIP 管理: `tccli vpc AllocateAddresses`, `tccli vpc AssociateAddress`, `tccli vpc DisassociateAddress`, `tccli vpc ReleaseAddresses`, `tccli vpc DescribeAddresses`
- 数据面: `kubectl apply -f` (Pod YAML 声明式绑定), `kubectl exec` (外网连通性验证)

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

# 3. 验证 CAM Action — VPC EIP 读权限
tccli vpc DescribeAddresses \
    --region <Region> \
    --Limit 1
# expected: (含有 RequestId，无 UnauthorizedOperation)

# 4. 验证 CAM Action — EKS 读权限
tccli tke DescribeEKSClusters \
    --region <Region> \
    --ClusterIds '["CLUSTER_ID"]'
# expected:
# {
#     "Clusters": [
#         {
#             "ClusterId": "CLUSTER_ID",
#             "ClusterName": "eks-demo",
#             "Status": "Running",
#             "K8SVersion": "1.30.0"
#         }
#     ],
#     "TotalCount": 1,
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

### 资源检查

```bash
# 5. 查询 EKS 集群子网信息（EIP 需绑定 Pod 所在子网的 ENI）
tccli tke DescribeEKSClusters \
    --region <Region> \
    --ClusterIds '["CLUSTER_ID"]'
# expected:
# {
#     "Clusters": [
#         {
#             "ClusterId": "CLUSTER_ID",
#             "VpcId": "VPC_ID",
#             "SubnetIds": ["SUBNET_ID"],
#             "Status": "Running"
#         }
#     ],
#     "TotalCount": 1,
#     "RequestId": "xxx"
# }

# 6. 检查已有 EIP 数量（每地域默认配额 20 个）
tccli vpc DescribeAddresses \
    --region <Region> \
    --Limit 100
# expected:
# {
#     "AddressSet": [],
#     "TotalCount": 0,
#     "RequestId": "xxx"
# }

# 7. 检查 EIP 配额
tccli vpc DescribeAddressQuota \
    --region <Region>
# expected:
# {
#     "QuotaSet": [
#         {
#             "QuotaId": "TOTAL_EIP_QUOTA",
#             "QuotaCurrent": 0,
#             "QuotaLimit": 20
#         }
#     ],
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
| 计费模式 | `InternetChargeType` | 创建后可通过 Modify 变更 | TRAFFIC_POSTPAID_BY_HOUR / BANDWIDTH_POSTPAID_BY_HOUR |
| 带宽上限 | `InternetMaxBandwidthOut` | 可 ModifyAddressAttribute 调整 | 1-2000 Mbps |
| EIP 名称 | `AddressName` | 同名可重复（不同 AddressId） | 1-60 字符 |
| 绑定资源 | `AssociateAddress.InstanceId` | 已绑定则先解绑 | ENI ID 或 NAT 网关 ID |
| 地域 | `--region` | 必填，EIP 与资源同地域 | REGION |
| Pod EIP 注解 | `tke.cloud.tencent.com/eip-attributes` | 声明式，apply 即为期望状态 | JSON 字符串，包含 Bandwidth/ChargeType |

## 操作步骤

### 占位符说明

| 占位符 | 说明 | 约束 | 获取方式 |
|---|---|---|---|
| `CLUSTER_ID` | EKS 集群 ID | 格式 cls-xxxxxxxx | `DescribeEKSClusters` |
| `EIP_ID` | 弹性公网 IP ID | 格式 eip-xxxxxxxx | `AllocateAddresses` 返回值 |
| `NAMESPACE` | Kubernetes 命名空间 | 不超过 63 字符 | 自定义或使用 default |
| `REGION` | 地域 | ap-guangzhou | 固定值 |
| `INSTANCE_ID` | 绑定目标实例 ID (ENI) | 格式 eni-xxxxxxxx | `DescribeAddresses` 绑定后返回 |

#### 选择依据

EIP 方案与 NAT 网关方案的选择取决于使用场景和成本模型:

1. **EIP 适用**: 单 Pod 需要固定公网 IP、需要被外部主动连接（入站）、需要 IP 白名单、Pod 数量少（<10 个）
2. **NAT 网关适用**: 多 Pod 仅需出站访问、不需独立 IP、需要统一出口管理和流量审计
3. **计费对比**: EIP 未绑定时收取闲置费（约 0.20 元/小时），绑定后按流量/带宽计费。NAT 网关按小时固定收费 + 流量费
4. **计费模式选择**: 测试环境建议 `TRAFFIC_POSTPAID_BY_HOUR`（按流量），生产固定带宽需求选 `BANDWIDTH_POSTPAID_BY_HOUR`

#### 最小创建

```bash
# 步骤 1: 申请弹性公网 IP
# >=4 参数，使用 --cli-input-json
cat > allocate-eip.json <<'EOF'
{
    "AddressCount": 1,
    "InternetChargeType": "TRAFFIC_POSTPAID_BY_HOUR",
    "InternetMaxBandwidthOut": 10,
    "AddressName": "eks-pod-eip"
}
EOF
tccli vpc AllocateAddresses \
    --region <Region> \
    --cli-input-json file://allocate-eip.json
# expected:
# {
#     "AddressSet": ["EIP_ID"],
#     "TaskId": "xxx",
#     "RequestId": "xxx"
# }

# 步骤 2: 查询 EIP 详情获取公网地址
tccli vpc DescribeAddresses \
    --region <Region> \
    --AddressIds '["EIP_ID"]'
# expected:
# {
#     "AddressSet": [
#         {
#             "AddressId": "EIP_ID",
#             "AddressName": "eks-pod-eip",
#             "AddressIp": "x.x.x.x",
#             "AddressStatus": "UNBIND",
#             "InternetChargeType": "TRAFFIC_POSTPAID_BY_HOUR",
#             "CreatedTime": "..."
#         }
#     ],
#     "TotalCount": 1,
#     "RequestId": "xxx"
# }
```

#### 增强配置

```bash
# 步骤 3: （可选）调整 EIP 带宽上限
tccli vpc ModifyAddressAttribute \
    --region <Region> \
    --AddressId EIP_ID \
    --AddressName "eks-pod-eip-upgraded" \
    --InternetMaxBandwidthOut 50
# expected:
# {
#     "RequestId": "xxx"
# }

# 步骤 4: 创建带 EIP 注解的 Pod（数据面操作，需 VPN/IOA）
# 以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行
cat <<'EOF' > pod-with-eip.yaml
apiVersion: v1
kind: Pod
metadata:
    name: app-with-eip
    namespace: NAMESPACE
    annotations:
        tke.cloud.tencent.com/eip-attributes: '{"Bandwidth":"10","ChargeType":"TRAFFIC_POSTPAID_BY_HOUR"}'
        tke.cloud.tencent.com/eip-id: "EIP_ID"
    labels:
        app: eip-demo
spec:
    containers:
        - name: app
          image: nginx:1.25
          ports:
              - containerPort: 80
                protocol: TCP
    restartPolicy: Always
EOF

kubectl apply -f pod-with-eip.yaml
# expected: pod/app-with-eip created

# 步骤 5: 等待 Pod Running 并确认 EIP 已绑定（ENI 自动分配）
kubectl wait --for=condition=Ready pod/app-with-eip --namespace NAMESPACE --timeout=120s
# expected: pod/app-with-eip condition met

# 步骤 6: 通过控制面查询 EIP 绑定状态
tccli vpc DescribeAddresses \
    --region <Region> \
    --AddressIds '["EIP_ID"]'
# expected:
# {
#     "AddressSet": [
#         {
#             "AddressId": "EIP_ID",
#             "AddressName": "eks-pod-eip-upgraded",
#             "AddressIp": "x.x.x.x",
#             "AddressStatus": "BIND",
#             "InstanceId": "INSTANCE_ID",
#             "NetworkInterfaceId": "eni-xxxxxxxx",
#             "BindResourceType": "ENI",
#             "CreatedTime": "..."
#         }
#     ],
#     "TotalCount": 1,
#     "RequestId": "xxx"
# }
```

## 验证

### 控制面（tccli）

```bash
# 1. 验证 EIP 为 BIND 状态且关联到正确 ENI
tccli vpc DescribeAddresses \
    --region <Region> \
    --AddressIds '["EIP_ID"]'
# expected:
# {
#     "AddressSet": [
#         {
#             "AddressId": "EIP_ID",
#             "AddressStatus": "BIND",
#             "InstanceId": "INSTANCE_ID",
#             "NetworkInterfaceId": "eni-xxxxxxxx"
#         }
#     ],
#     "TotalCount": 1,
#     "RequestId": "xxx"
# }

# 2. 验证 Pod Running 且含 EIP 注解
# 需 VPN/IOA — 以下为参考格式
kubectl describe pod app-with-eip --namespace NAMESPACE | grep -A2 "eip"
# expected:
# Annotations:  tke.cloud.tencent.com/eip-attributes: ...
#               tke.cloud.tencent.com/eip-id: EIP_ID
```

```text
NAME  STATUS  AGE
...
```

### 数据面（需 VPN/IOA）

> **kubectl 数据面不可达**：CAM 拒绝公网端点 (strategyId:240463971)，内网需 VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

```bash
# 3. 从 Pod 内部验证出口 IP 即为绑定的 EIP
kubectl exec app-with-eip --namespace NAMESPACE -- \
    curl -s --connect-timeout 5 http://httpbin.org/ip
# expected:
# {
#     "origin": "x.x.x.x"
# }
# 注: origin 应与步骤 2 中 DescribeAddresses 返回的 AddressIp 一致

# 4. 验证外网可访问 Pod 的 nginx（入站连通性）
curl -s --connect-timeout 5 http://x.x.x.x:80
# expected: (nginx 默认欢迎页 HTML)
# 注: 如安全组未开放 80 端口，此步会超时 — 详见排障

# 5. 从 Pod 内测试外网 DNS 解析
kubectl exec app-with-eip --namespace NAMESPACE -- \
    nslookup httpbin.org
# expected: (返回 DNS 解析结果，无超时)

# 6. 从 Pod 内下载文件验证外网带宽
kubectl exec app-with-eip --namespace NAMESPACE -- \
    wget -q -O /dev/null --timeout=10 http://speedtest.tele2.net/1MB.zip
# expected: (文件下载成功，无超时)
```

## 清理

> **计费警告**: EIP 闲置（UNBIND 状态）按小时收取闲置费（约 0.20 元/小时）。Pod 删除后 EIP 自动解绑但**不会自动释放**，持续产生闲置费。请务必手动释放 EIP。

> **副作用警告**: 释放 EIP 会永久丢失该公网 IP 地址。释放后无法恢复，需重新申请（获得新 IP）。如 IP 已配置到外部白名单，请先更新白名单。

### 控制面（tccli）

```bash
# 1. 解绑 EIP（Pod 删除后 EIP 通常自动解绑，此步为兜底）
tccli vpc DisassociateAddress \
    --region <Region> \
    --AddressId EIP_ID
# expected:
# {
#     "TaskId": "xxx",
#     "RequestId": "xxx"
# }

# 2. 释放弹性公网 IP
tccli vpc ReleaseAddresses \
    --region <Region> \
    --AddressIds '["EIP_ID"]'
# expected:
# {
#     "RequestId": "xxx"
# }

# 3. 验证 EIP 已释放
tccli vpc DescribeAddresses \
    --region <Region> \
    --AddressIds '["EIP_ID"]'
# expected:
# {
#     "AddressSet": [],
#     "TotalCount": 0,
#     "RequestId": "xxx"
# }
```

### 数据面（需 VPN/IOA）

```bash
# 4. 删除测试 Pod
kubectl delete pod app-with-eip --namespace NAMESPACE
# expected: pod "app-with-eip" deleted

# 5. 确认 Pod 已删除
kubectl get pod app-with-eip --namespace NAMESPACE
# expected: Error from server (NotFound): pods "app-with-eip" not found
```

```text
NAME  STATUS  AGE
...
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|---|---|---|---|
| `AllocateAddresses` 返回 `QuotaExceeded` | `DescribeAddressQuota` 检查配额使用量 | 每地域 EIP 配额已满（默认 20） | 释放闲置 EIP 或提交工单提升配额 |
| `AllocateAddresses` 返回 `InvalidParameterValue` | 参数拼写或值域检查 | `InternetChargeType` 值非法或 `AddressCount` 超限 | 确认 `InternetChargeType` 为 `TRAFFIC_POSTPAID_BY_HOUR` 或 `BANDWIDTH_POSTPAID_BY_HOUR` |
| `AssociateAddress` 返回 `InvalidInstanceId.NotFound` | `DescribeAddresses` 确认目标实例 | ENI ID 不存在或已删除（Pod 已终止） | 先确认 Pod Running，ENI 存在后再绑定 |
| `AssociateAddress` 返回 `ResourceInUse` | EIP 已绑定到其他资源 | EIP 状态为 BIND，不允许同时绑定 | 先执行 `DisassociateAddress` 解绑，再重新绑定 |
| `ReleaseAddresses` 返回 `ResourceInUse.AddressCascadeRelease` | EIP 仍绑定中 | 未先解绑即尝试释放 | 先执行 `DisassociateAddress`，轮询确认 UNBIND 后再释放 |
| Pod YAML apply 无 EIP 注解生效 | `kubectl describe pod` 查看 Annotations | EIP ID 不存在或注解 JSON 格式错误 | 验证 `eip-id` 为有效 AddressId，`eip-attributes` 为合法 JSON |

### 创建成功但状态异常

| 现象 | 诊断 | 根因 | 修复 |
|---|---|---|---|
| EIP BIND 但外网无法访问 Pod | `curl --connect-timeout 5 http://EIP_IP:PORT` | 安全组未放通入站规则 | 在 EKS 集群安全组中添加入站规则，放行所需端口和协议 |
| Pod 内无法访问外网（EIP BIND） | `kubectl exec POD -- curl httpbin.org/ip` | EKS 安全组出站规则限制 | 检查安全组出站规则，确认允许 0.0.0.0/0 或目标 CIDR |
| EIP 绑定在 Pod 创建后未自动 Associate | `DescribeAddresses` 查看 AddressStatus | EIP 注解中的 `eip-id` 与已申请 EIP 不匹配 | 确认 YAML 中 `eip-id` 与 `AllocateAddresses` 返回的 AddressId 一致 |
| Pod 删除后 EIP 持续计费 | `DescribeAddresses` 查看计费状态 | EIP 未被释放，仍为 UNBIND 闲置状态 | 执行 `ReleaseAddresses` 释放 EIP |
| curl 返回 Connection Refused | `kubectl logs POD` 查看容器日志 | 容器内应用未监听目标端口或未启动 | 确认容器内服务正常启动，监听在配置的 containerPort |

## 下一步

- [通过 NAT 网关访问外网](../通过%20NAT%20网关访问外网/tccli%20操作.md)：多 Pod 共享出口 IP 方案，适合批量出站场景
- [在 TKE Serverless 上运行深度学习](../在%20Serverless%20集群上玩转深度学习/在%20TKE%20Serverless%20上运行深度学习/tccli%20操作.md)：使用 GPU Pod + EIP 运行分布式训练
- [EIP 产品文档](https://cloud.tencent.com/document/product/1199)：弹性公网 IP 完整文档，含带宽包、精品 BGP IP

## 控制台替代
控制台完成本节操作需先在 **私有网络 > 弹性公网 IP** 页面申请 EIP，然后切换到 TKE 控制台的 Workload 页面创建 Pod 并配置 EIP 注解，最后返回 VPC 控制台验证绑定状态。tccli 将 EIP 申请、查询、Pod 声明式绑定编排为流水线，DescribeAddresses 提供状态闭环验证，适合基础设施即代码 (IaC) 管理和自动化运维脚本。
