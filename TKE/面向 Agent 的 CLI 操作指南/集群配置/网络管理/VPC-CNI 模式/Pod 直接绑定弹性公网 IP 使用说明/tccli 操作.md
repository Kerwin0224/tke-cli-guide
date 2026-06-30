# Pod 直接绑定弹性公网 IP 使用说明

> 对照官方：[Pod 直接绑定弹性公网 IP 使用说明](https://cloud.tencent.com/document/product/457/64886) · page_id `64886`

## 概述

VPC-CNI 模式下，Pod 可直接绑定弹性公网 IP（EIP）实现公网访问。Pod 通过注解声明 EIP 配置，eniipamd 组件自动创建并绑定 EIP 到 Pod 的弹性网卡（ENI），或绑定已有 EIP。该功能仅支持 VPC-CNI 网络模式，GlobalRouter 模式不支持。

### EIP 获取方式对比

| 方式 | 注解 | 适用场景 | 典型用法 |
|------|------|---------|---------|
| 自动创建 EIP | `tke.cloud.tencent.com/eip-attributes` | 无需预购，Pod 创建时按量自动分配 | 临时任务、动态扩缩容 |
| 指定已有 EIP | `tke.cloud.tencent.com/eip-attributes`（含 `AddressId`） | 使用预购 EIP，固定公网地址 | 固定出口 IP、外部服务白名单 |

### 关键能力一览

| 能力 | 说明 | 最低组件版本 |
|------|------|:--:|
| 共享网卡多 IP 模式（固定 IP）| Pod 绑定 EIP + 固定 VPC IP | 任意 |
| 共享网卡多 IP 模式（非固定 IP）| Pod 绑定 EIP，VPC IP 可变更 | 任意 |
| 独立网卡模式 | Pod 独占 ENI 绑定 EIP | v3.3.9 |
| 级联回收（ownerRef）| Pod 删除时自动清理 EIPClaim | v3.5.0 |

## 前置条件

- [环境准备](../../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId、secretKey、region 均已配置，region 为目标地域

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:DescribeClusters, tke:DescribeAddon
#    vpc:AllocateAddresses, vpc:DescribeAddresses, vpc:ReleaseAddresses, vpc:DescribeAddressQuota
# 验证 TKE 权限
tccli tke DescribeClusters --region REGION
# expected: exit 0，返回集群列表（可为空）

# 验证 VPC EIP 权限
tccli vpc DescribeAddresses --region REGION
# expected: exit 0，返回 EIP 列表（可为空）
```

```json
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "cls-example",
            "ClusterName": "example-cluster",
            "ClusterStatus": "Running",
            "ClusterVersion": "1.32.2"
        }
    ]
}
```

**`tccli vpc DescribeAddresses` 预期输出**：

```json
{
    "TotalCount": 0,
    "AddressSet": [],
    "RequestId": "..."
}
```

### 资源检查

```bash
# 4. 确认集群为 VPC-CNI 模式
tccli tke DescribeClusters --region REGION \
    --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterNetworkSettings 中 Cni=true, Property 中 VpcCniType="tke-route-eni"
```

**预期输出**：

```json
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "cls-example",
            "ClusterName": "example-cluster",
            "ClusterStatus": "Running",
            "ClusterNetworkSettings": {
                "VpcId": "vpc-example",
                "ClusterCIDR": "10.0.0.0/16",
                "ServiceCIDR": "10.1.0.0/20",
                "Cni": true,
                "MaxNodePodNum": 64,
                "MaxClusterServiceNum": 4096,
                "Property": "{\"VpcCniType\":\"tke-route-eni\",\"NetworkType\":\"GR\"}"
            }
        }
    ],
    "RequestId": "..."
}
```

```bash
# 5. 确认 eniipamd 组件版本满足要求
tccli tke DescribeAddon --region REGION \
    --ClusterId CLUSTER_ID --AddonName eniipamd
# expected: 版本 ≥ 3.5.0（EIP 功能最低要求），Phase 为 Succeeded 或 Upgrading
```

**预期输出**：

```json
{
    "AddonName": "eniipamd",
    "AddonVersion": "3.11.0",
    "Phase": "Succeeded",
    "CreateTime": "2026-06-23T08:11:19Z"
}
```

```bash
# 6. 查询 EIP 配额
tccli vpc DescribeAddressQuota --region REGION
# expected: QUOTA_ID 各有剩余（QuotaCurrent < QuotaLimit）
```

**预期输出**：

```json
{
    "QuotaSet": [
        {
            "QuotaId": "TOTAL_EIP_QUOTA",
            "QuotaCurrent": 52,
            "QuotaLimit": 100,
            "QuotaGroup": "ap-guangzhou"
        },
        {
            "QuotaId": "DAILY_EIP_APPLY",
            "QuotaCurrent": 3,
            "QuotaLimit": 200,
            "QuotaGroup": "ap-guangzhou"
        }
    ],
    "RequestId": "..."
}
```

```bash
# 7. 查询已有 EIP（如使用指定已有 EIP 模式）
tccli vpc DescribeAddresses --region REGION
# expected: 如有已购 EIP，AddressStatus 为 UNBIND
```

**预期输出**：

```json
{
    "TotalCount": 1,
    "AddressSet": [
        {
            "AddressId": "eip-example",
            "AddressName": "my-eip",
            "AddressIp": "1.2.3.4",
            "AddressStatus": "UNBIND",
            "AddressType": "EIP",
            "InternetChargeType": "TRAFFIC_POSTPAID_BY_HOUR",
            "Bandwidth": 1
        }
    ]
}
```

### kubectl 连接准备（数据面操作需要）

EIP 绑定操作通过 kubectl 在集群内完成。需先在集群 VPC 可达环境中获取 kubeconfig：

```bash
# 获取内网端点 kubeconfig
tccli tke DescribeClusterKubeconfig --region REGION \
    --ClusterId CLUSTER_ID --IsExtranet false \
    | jq -r '.Kubeconfig' > ~/.kube/config
```

**预期输出**（`jq` 提取前，API 原始返回）：

```json
{
    "Kubeconfig": "apiVersion: v1\nclusters:\n- cluster:\n    ...",
    "RequestId": "..."
}
```

```bash
# 或通过 DescribeClusterSecurity 获取（使用域名）
tccli tke DescribeClusterSecurity --region REGION \
    --ClusterId CLUSTER_ID \
    | jq -r '.Kubeconfig' > ~/.kube/config
```

**预期输出**（`jq` 提取前，API 原始返回）：

```json
{
    "Kubeconfig": "apiVersion: v1\nclusters:\n- cluster:\n    ...",
    "RequestId": "..."
}
```

> **注意**：kubectl 需在集群 VPC 可达环境中执行——通过 IOA/VPN/专线 或同 VPC CVM。本地环境可能无法直连内网端点（内网 IP 不可达）。公网端点可能受 CAM 策略（如 strategyId:240463971）限制。

## 控制台与 CLI 参数映射

| 控制台操作 | CLI 命令 | 幂等 |
|-----------|----------|:--:|
| 查看集群网络配置 | `tke DescribeClusters` | 是 |
| 查看 eniipamd 组件版本 | `tke DescribeAddon` | 是 |
| 创建/申请 EIP | `vpc AllocateAddresses` | 否 |
| 查询已有 EIP | `vpc DescribeAddresses` | 是 |
| 查询 EIP 配额 | `vpc DescribeAddressQuota` | 是 |
| 释放/退还 EIP | `vpc ReleaseAddresses` | 是 |
| 自动创建 EIP 绑定 Pod | `kubectl apply`（注解 `eip-attributes`） | 否 |
| 指定已有 EIP 绑定 Pod | `kubectl apply`（注解 `eip-attributes` 含 `AddressId`） | 否 |
| 设置 EIP 保留策略 | `kubectl apply`（注解 `eip-claim-delete-policy`） | 否 |
| 出站流量走 EIP | `kubectl edit cm ip-masq-agent-config` | 否 |
| 查看 EIPClaim | `kubectl get eipc` | 是 |
| 手动回收 EIPClaim | `kubectl delete eipc` | 是 |

## 关键字段说明

### Pod EIP 注解

完整参数定义通过 `kubectl explain pod.metadata.annotations` 查看。以下为 EIP 功能核心注解：

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `tke.cloud.tencent.com/networks` | String | 是 | `tke-route-eni`（共享网卡）/`tke-direct-eni`（独立网卡） | 缺少 → Pod 不使用 VPC-CNI，EIP 不分配 |
| `tke.cloud.tencent.com/eip-attributes` | JSON String | 是（自动创建） | `{"InternetMaxBandwidthOut":10,"InternetChargeType":"TRAFFIC_POSTPAID_BY_HOUR"}`。可含 `AddressId` 指定已有 EIP | JSON 格式错误 → EIP 创建失败（Pod 正常运行但无公网 IP） |
| `tke.cloud.tencent.com/eip-claim-delete-policy` | String | 否 | `Never`（Pod 删除后保留 EIP）/ 省略或 `Immediate`（Pod 删除即释放） | 忘记设置 `Never` → Pod 删除时 EIP 意外释放，公网 IP 变更 |
| `tke.cloud.tencent.com/eip-public-ip` | String | 只读 | 由 eniipamd 自动写入，绑定的公网 IP 地址 | 只读，手动修改无效 |

### eip-attributes JSON 子字段

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `InternetMaxBandwidthOut` | Integer | 是 | 带宽上限（Mbps），范围 1-2000 | 超限 → EIP 创建失败 |
| `InternetChargeType` | String | 是 | `TRAFFIC_POSTPAID_BY_HOUR`（按流量）/ `BANDWIDTH_POSTPAID_BY_HOUR`（按带宽）/ `BANDWIDTH_PREPAID`（包年包月） | 填错枚举值 → EIP 创建失败 |
| `AddressId` | String | 否 | 已有 EIP ID，格式 `eip-xxxxxxxx`。指定后不再自动创建 | 指定不存在的 EIP → Pod 创建后 EIP 绑定失败 |
| `ISP` | String | 否 | `BGP`（默认）/ `CMCC` / `CTCC` / `CUCC` | 填错枚举 → EIP 创建失败 |

### AllocateAddresses 关键参数（tccli 创建 EIP）

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `AddressCount` | Integer | 是 | 申请数量，≥ 1 | 超过配额 → `LimitExceeded.AddressQuota` |
| `InternetChargeType` | String | 否 | `TRAFFIC_POSTPAID_BY_HOUR`（默认）/ `BANDWIDTH_POSTPAID_BY_HOUR` / `BANDWIDTH_PREPAID`，默认按流量 | 混淆 `TRAFFIC_POSTPAID_BY_HOUR`（按流量）与 `BANDWIDTH_POSTPAID_BY_HOUR`（按带宽）→ 计费模式偏差 |
| `AddressName` | String | 否 | EIP 名称，长度 1-60 | 无 |
| `InternetMaxBandwidthOut` | Integer | 否 | 带宽上限（Mbps），默认 1 | 超出限制 → `InvalidParameterValue.Bandwidth` |
| `Tags` | Array | 否 | 标签列表 `[{"Key":"key","Value":"value"}]`。组织 CAM 策略可能强制要求特定标签 | 缺必需标签 → `UnauthorizedOperation`（strategyId:274251632） |
| `AddressType` | String | 否 | `EIP`（默认）/ `AnycastEIP` / `HighQualityEIP` | 填错枚举 → `InvalidParameter` |

## 操作步骤

### 使用限制

- Pod 不能使用 `hostNetwork: true`
- 单节点 EIP 上限 = CVM 绑定限制 - 1（弹性网卡本身占用一个绑定）
- 集群删除不会自动回收集群创建的 EIP，需手动清理
- 自动创建的 EIP 绑定后免 IP 资源费，出站默认按流量计费
- 仅 VPC-CNI 模式支持；GlobalRouter 模式集群 Pod 无法绑定 EIP

### 步骤 1：确认集群 VPC-CNI 模式与组件版本

```bash
# 查看集群网络配置
tccli tke DescribeClusters --region REGION \
    --ClusterIds '["CLUSTER_ID"]'
# expected: Cni=true, Property 中 VpcCniType="tke-route-eni"
```

**预期输出**：

```json
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "cls-example",
            "ClusterName": "example-cluster",
            "ClusterNetworkSettings": {
                "VpcId": "vpc-example",
                "Cni": true,
                "Property": "{\"VpcCniType\":\"tke-route-eni\",\"NetworkType\":\"GR\"}"
            }
        }
    ],
    "RequestId": "..."
}
```

```bash
# 查看 eniipamd 组件版本
tccli tke DescribeAddon --region REGION \
    --ClusterId CLUSTER_ID --AddonName eniipamd
# expected: AddonVersion >= 3.5.0, Phase 为 Succeeded 或 Upgrading
```

**预期输出**：

```json
{
    "AddonName": "eniipamd",
    "AddonVersion": "3.11.0",
    "Phase": "Succeeded"
}
```

### 步骤 2：创建 EIP（为指定已有 EIP 模式准备）

如使用自动创建 EIP 模式（Pod 注解自动分配），可跳过此步骤。如使用指定已有 EIP 模式，需先通过 tccli 创建 EIP。

#### 选择依据

- **计费类型**：选择 `TRAFFIC_POSTPAID_BY_HOUR`（按流量计费），适合测试和验证场景。生产环境可根据流量模式选择 `BANDWIDTH_POSTPAID_BY_HOUR`（按带宽计费）或 `BANDWIDTH_PREPAID`（带宽包年包月）。注意区分：`TRAFFIC_POSTPAID_BY_HOUR` 按实际流量收费，`BANDWIDTH_POSTPAID_BY_HOUR` 按带宽峰值收费——两者容易混淆。
- **标签要求**：组织 CAM 策略（strategyId:274251632）可能要求 EIP 资源必须携带特定标签（如 `"billing"` 标签值为你的用户名），否则创建被拒。执行前确认你的环境是否有类似策略约束。
- **配额检查**：已通过前置条件中的 `DescribeAddressQuota` 确认配额充足。

#### 最小创建（仅必填字段）

```bash
tccli vpc AllocateAddresses --region REGION \
    --AddressCount 1
# expected: exit 0，返回 AddressSet 含 EIP ID
```

**预期输出**：

```json
{
    "AddressSet": [
        "eip-example"
    ],
    "TaskId": "218058863",
    "RequestId": "8d9fa0a1-a96a-475f-b406-5ef50b9db8bb"
}
```

#### 增强配置（含计费类型、名称、标签、带宽）

`eip-allocate.json`：

```json
{
    "AddressCount": 1,
    "InternetChargeType": "TRAFFIC_POSTPAID_BY_HOUR",
    "AddressName": "EIP_NAME",
    "InternetMaxBandwidthOut": 10,
    "Tags": [
        {
            "Key": "billing",
            "Value": "USER_NAME"
        }
    ]
}
```

```bash
tccli vpc AllocateAddresses --region REGION \
    --cli-input-json file://eip-allocate.json
# expected: exit 0，返回 AddressSet 含 EIP ID
```

**预期输出**：

```json
{
    "AddressSet": [
        "eip-example"
    ],
    "TaskId": "218058863",
    "RequestId": "8d9fa0a1-a96a-475f-b406-5ef50b9db8bb"
}
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `REGION` | 地域 | 如 `ap-guangzhou` | `tccli tke DescribeRegions` |
| `EIP_NAME` | EIP 名称 | 长度 1-60 字符 | 自定义 |
| `USER_NAME` | CAM 标签用户名 | 按组织策略要求填写 | `tccli configure list` 查看当前用户 |
| `CLUSTER_ID` | 集群 ID | 格式 `cls-xxxxxxxx` | `tccli tke DescribeClusters` |

```bash
# 确认 EIP 已创建
tccli vpc DescribeAddresses --region REGION \
    --Filters '[{"Name":"address-name","Values":["EIP_NAME"]}]'
# expected: TotalCount >= 1, AddressStatus 为 UNBIND
```

**预期输出**：

```json
{
    "TotalCount": 1,
    "AddressSet": [
        {
            "AddressId": "eip-example",
            "AddressName": "my-eip",
            "AddressIp": "1.2.3.4",
            "AddressStatus": "UNBIND",
            "AddressType": "EIP",
            "InternetChargeType": "TRAFFIC_POSTPAID_BY_HOUR",
            "Bandwidth": 10
        }
    ]
}
```

### 步骤 3：自动创建 EIP 绑定 Pod（数据面，kubectl）

> **可达性**：kubectl 命令需在集群 VPC 可达环境中执行——通过 IOA/VPN/专线 或同 VPC CVM。本地环境内网端点不可达。

#### 选择依据

- **自动创建模式**：不需要预购 EIP，Pod 创建时 eniipamd 自动调用 VPC API 分配 EIP 并绑定到 Pod 的弹性网卡。适合临时任务、动态扩缩容等场景。
- **EIP 保留策略**：默认 Pod 删除时 EIP 立即释放（`eip-claim-delete-policy` 未设置）。如需保留 EIP（固定出口 IP 场景），设置 `tke.cloud.tencent.com/eip-claim-delete-policy: Never`。忘记设置可能导致公网 IP 意外变更，影响依赖固定 IP 的外部服务。

#### 最小创建（自动创建 EIP，仅必填注解）

`pod-eip-auto-minimal.yaml`：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: eip-test-pod
  namespace: default
  annotations:
    tke.cloud.tencent.com/networks: tke-route-eni
    tke.cloud.tencent.com/eip-attributes: |
      {"InternetMaxBandwidthOut":10,"InternetChargeType":"TRAFFIC_POSTPAID_BY_HOUR"}
spec:
  containers:
  - name: nginx
    image: nginx:latest
    ports:
    - containerPort: 80
```

```bash
kubectl apply -f pod-eip-auto-minimal.yaml
# expected: pod/eip-test-pod created
```

#### 增强配置（含 EIP 保留策略、资源请求）

`pod-eip-auto-enhanced.yaml`：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: eip-retain-pod
  namespace: default
  annotations:
    tke.cloud.tencent.com/networks: tke-route-eni
    tke.cloud.tencent.com/eip-attributes: |
      {"InternetMaxBandwidthOut":10,"InternetChargeType":"TRAFFIC_POSTPAID_BY_HOUR"}
    tke.cloud.tencent.com/eip-claim-delete-policy: Never
spec:
  containers:
  - name: nginx
    image: nginx:latest
    ports:
    - containerPort: 80
    resources:
      requests:
        tke.cloud.tencent.com/eni-ip: "1"
        tke.cloud.tencent.com/eip: "1"
      limits:
        tke.cloud.tencent.com/eni-ip: "1"
        tke.cloud.tencent.com/eip: "1"
```

```bash
kubectl apply -f pod-eip-auto-enhanced.yaml
# expected: pod/eip-retain-pod created
```

```bash
# 查看 Pod 绑定的 EIP 公网地址
kubectl describe pod eip-retain-pod -n default | grep -E "eip-public-ip|eip-attributes"
# expected: tke.cloud.tencent.com/eip-public-ip 显示公网 IP，tke.cloud.tencent.com/eip-attributes 显示属性 JSON
```

**预期输出**（部分）：

```text
Annotations:  tke.cloud.tencent.com/eip-attributes: {"InternetMaxBandwidthOut":10,"InternetChargeType":"TRAFFIC_POSTPAID_BY_HOUR","AddressId":"eip-example","InternetMaxBandwidthOut":10}
              tke.cloud.tencent.com/eip-public-ip: 1.2.3.4
              tke.cloud.tencent.com/eip-claim-delete-policy: Never
              tke.cloud.tencent.com/networks: tke-route-eni
```

### 步骤 4：指定已有 EIP 绑定 Pod（数据面，kubectl）

> **可达性**：kubectl 命令需在集群 VPC 可达环境中执行——通过 IOA/VPN/专线 或同 VPC CVM。本地环境内网端点不可达。

#### 选择依据

- **指定已有 EIP 模式**：适用于需要固定公网 IP 的场景（如外部服务 IP 白名单）。需先在步骤 2 中通过 tccli 创建 EIP，然后在 Pod 注解的 `eip-attributes` JSON 中指定 `AddressId`。
- **副本限制**：StatefulSet Pod 按序号匹配 EIP 列表，Deployment Pod（无数字后缀）只能指定单个 EIP。多 EIP 场景使用 StatefulSet。

`pod-eip-specified.yaml`：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: eip-specified-pod
  namespace: default
  annotations:
    tke.cloud.tencent.com/networks: tke-route-eni
    tke.cloud.tencent.com/eip-attributes: |
      {"AddressId":"EIP_ID","InternetMaxBandwidthOut":10,"InternetChargeType":"TRAFFIC_POSTPAID_BY_HOUR"}
spec:
  containers:
  - name: nginx
    image: nginx:latest
    resources:
      requests:
        tke.cloud.tencent.com/eni-ip: "1"
        tke.cloud.tencent.com/eip: "1"
      limits:
        tke.cloud.tencent.com/eni-ip: "1"
        tke.cloud.tencent.com/eip: "1"
```

```bash
kubectl apply -f pod-eip-specified.yaml
# expected: pod/eip-specified-pod created
```

| 占位符 | 说明 | 获取方式 |
|--------|------|---------|
| `EIP_ID` | 已有 EIP ID，格式 `eip-xxxxxxxx` | `tccli vpc DescribeAddresses --region REGION` |

```bash
# 验证 EIP 绑定
kubectl describe pod eip-specified-pod -n default | grep "eip-public-ip"
# expected: tke.cloud.tencent.com/eip-public-ip 为指定 EIP 的公网 IP
```

**预期输出**（部分）：

```text
tke.cloud.tencent.com/eip-public-ip: 1.2.3.4
```

### 步骤 5：确保出站流量走 EIP（数据面，kubectl）

> **可达性**：kubectl 命令需在集群 VPC 可达环境中执行。

集群默认部署 `ip-masq-agent`，Pod 出站流量被 SNAT 为节点 IP。需将 Pod IP 加入白名单以绕过 SNAT，使出站流量直接走 EIP。

```bash
# 查看 Pod IP
kubectl get pod eip-test-pod -n default -o jsonpath='{.status.podIP}'
# expected: 返回 Pod 的 VPC 子网 IP，如 172.26.0.x
```

**预期输出**：

```text
172.26.0.10
```

```bash
# 编辑 ip-masq-agent ConfigMap
kubectl -n kube-system edit cm ip-masq-agent-config
# expected: configmap/ip-masq-agent-config edited
```

在 `data.config` JSON 的 `NonMasqueradeSrcCIDRs` 中加入 Pod IP（如 `172.26.0.10/32`）：

```json
{
    "NonMasqueradeCIDRs": ["10.0.0.0/16", "10.1.0.0/20"],
    "NonMasqueradeSrcCIDRs": ["172.26.0.10/32"],
    "MasqLinkLocal": true,
    "ResyncInterval": "1m0s",
    "MasqLinkLocalIPv6": false
}
```

> **注意**：`NonMasqueradeSrcCIDRs` 需 ip-masq-agent ≥ v2.6.2。修改后 1 分钟内热生效。谨慎使用宽泛 CIDR，避免范围内所有 Pod 绕开 SNAT。

## 验证

### 控制面（tccli）

```bash
# 1. 确认 eniipamd 组件运行正常
tccli tke DescribeAddon --region REGION \
    --ClusterId CLUSTER_ID --AddonName eniipamd
# expected: AddonVersion >= 3.5.0, Phase "Succeeded"
```

**预期输出**：

```json
{
    "AddonName": "eniipamd",
    "AddonVersion": "3.11.0",
    "Phase": "Succeeded"
}
```

```bash
# 2. 确认 EIP 状态（如使用指定已有 EIP 模式）
tccli vpc DescribeAddresses --region REGION \
    --AddressIds '["EIP_ID"]'
# expected: AddressStatus 为 BIND（已绑定）或 UNBIND（未绑定但存在）
```

**预期输出**：

```json
{
    "TotalCount": 1,
    "AddressSet": [
        {
            "AddressId": "eip-example",
            "AddressName": "my-eip",
            "AddressIp": "1.2.3.4",
            "AddressStatus": "BIND",
            "Bandwidth": 10
        }
    ]
}
```

### 数据面（kubectl，需 IOA/VPN/专线）

```bash
# 3. 确认 Pod 运行且 EIP 已绑定
kubectl get pod -n default -l app=eip-test
# expected: STATUS Running, AGE 列显示启动时间
```

```text
NAME            READY   STATUS    RESTARTS   AGE
eip-test-pod    1/1     Running   0          2m
```

```bash
# 4. 查看 EIP 绑定详情
kubectl describe pod eip-test-pod -n default | grep -E "eip-public-ip|eip-attributes"
# expected: eip-public-ip 注解存在，值为公网 IP
```

**预期输出**（部分）：

```text
Annotations:  tke.cloud.tencent.com/eip-public-ip: 1.2.3.4
              tke.cloud.tencent.com/eip-attributes: {"InternetMaxBandwidthOut":10,...}
```

```bash
# 5. 查看 EIPClaim CRD
kubectl get eipc -n default
# expected: 存在 eip-test-pod，状态 Bound
```

**预期输出**：

```text
NAMESPACE   NAME           EIP        STATUS
default     eip-test-pod   1.2.3.4    Bound
```

```bash
# 6. 确认出站流量走 EIP（Pod 内执行）
kubectl exec eip-test-pod -n default -- wget -qO- ifconfig.me
# expected: 返回 EIP 公网地址，非节点 IP
```

**预期输出**：

```text
1.2.3.4
```

### 验证维度

| 维度 | 命令 | 预期 |
|------|------|------|
| 组件状态 | `tccli tke DescribeAddon` | eniipamd Phase "Succeeded" |
| EIP 状态 | `tccli vpc DescribeAddresses` | AddressStatus "BIND" |
| Pod 状态 | `kubectl get pod` | STATUS "Running" |
| EIP 绑定 | `kubectl describe pod \| grep eip-public-ip` | 注解值为公网 IP |
| EIPClaim | `kubectl get eipc` | STATUS "Bound" |
| 出站 IP | `kubectl exec ... wget ifconfig.me` | 返回 EIP 地址 |

## 清理

> **警告**：EIP 按量计费，创建后即使未绑定也会按小时收取 IP 资源费。验证完成后必须立即释放，避免持续产生费用。
> EIP 释放后 IP 地址不可恢复。如后续需要固定 IP，应使用 `eip-claim-delete-policy: Never` 保留 EIP。
> Pod 删除时默认释放关联 EIP（`eip-claim-delete-policy` 未设置或为 `Immediate`）。如需保留 EIP，务必检查 annotation 设置。
> 集群删除不自动回收 EIP。残留 EIP 需手动在 VPC 控制台或通过 `ReleaseAddresses` 释放。

### 数据面（kubectl，需 IOA/VPN/专线）

按依赖倒序清理：先删 Pod，再删 EIPClaim。

```bash
# 1. 清理前状态检查
kubectl get pod,eipc -n default | grep eip
# 确认是待清理的目标资源
```

**预期输出**：

```text
pod/eip-test-pod       1/1     Running   0          10m
pod/eip-retain-pod     1/1     Running   0          8m
pod/eip-specified-pod  1/1     Running   0          5m
eipclaim/eip-test-pod           1.2.3.4    Bound
```

```bash
# 2. 删除 Pod（自动模式：EIP 随 Pod 释放；Never 策略：EIP 保留）
kubectl delete pod eip-test-pod eip-retain-pod eip-specified-pod -n default
# expected: pod "eip-test-pod" deleted, pod "eip-retain-pod" deleted, pod "eip-specified-pod" deleted
```

```bash
# 3. 手动释放 EIPClaim（如 eip-claim-delete-policy 为 Never 或 EIPClaim 残留）
kubectl get eipc -n default --no-headers 2>/dev/null | awk '{print $1}' | xargs -r -I {} kubectl delete eipc {} -n default
# expected: 各 eipclaim 被删除（如存在）
```

```bash
# 4. 清理 ip-masq-agent 白名单（如修改过）
kubectl -n kube-system edit cm ip-masq-agent-config
# 从 NonMasqueradeSrcCIDRs 中移除已删除的 Pod IP
```

### 控制面（tccli）

```bash
# 5. 确认 EIP 状态（清理前）
tccli vpc DescribeAddresses --region REGION \
    --Filters '[{"Name":"address-name","Values":["EIP_NAME"]}]'
# expected: 返回待释放的 EIP 列表
```

**预期输出**：

```json
{
    "TotalCount": 1,
    "AddressSet": [
        {
            "AddressId": "eip-example",
            "AddressName": "EIP_NAME",
            "AddressIp": "1.2.3.4",
            "AddressStatus": "UNBIND"
        }
    ],
    "RequestId": "..."
}
```

```bash
# 6. 释放 EIP
tccli vpc ReleaseAddresses --region REGION \
    --AddressIds '["EIP_ID"]'
# expected: exit 0，返回 TaskId
```

**预期输出**：

```json
{
    "TaskId": "218058864",
    "RequestId": "9e0fb1b2-b07b-4860-c724-6f60af0c9cc9"
}
```

```bash
# 7. 验证 EIP 已释放
tccli vpc DescribeAddresses --region REGION \
    --AddressIds '["EIP_ID"]'
# expected: TotalCount 为 0 或 AddressSet 为空
```

**预期输出**：

```json
{
    "TotalCount": 0,
    "AddressSet": []
}
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `AllocateAddresses` 返回 `UnauthorizedOperation` | 检查请求 Tags 参数：是否携带必需标签 | 组织 CAM 策略 strategyId:274251632 要求 EIP 资源必须携带 `"billing"` 标签（`cvm:AllocateAddresses` 操作的条件约束） | 创建 EIP 时添加 `--Tags '[{"Key":"billing","Value":"USER_NAME"}]'`。或使用 `--cli-input-json file://` 格式在 JSON 中指定 Tags |
| `AllocateAddresses` 返回 `LimitExceeded.AddressQuota` | `tccli vpc DescribeAddressQuota --region REGION` 检查各配额项 | EIP 配额不足（此为环境限制，非命令错误） | 释放不用的 EIP 后重试，或提工单申请扩容 |
| Pod 处于 Pending | `kubectl describe pod POD_NAME -n NAMESPACE` 查看 Events | 节点 `tke.cloud.tencent.com/eip` 或 `eni-ip` 资源不足 | 换到资源充足的节点，或减少副本数。可先 `kubectl describe node NODE_NAME` 查看节点资源分配情况 |
| Pod 创建成功但 EIP 未分配 | `kubectl describe pod POD_NAME -n NAMESPACE \| grep "eip"` 检查注解；`kubectl logs -n kube-system -l app=eniipamd` 查看组件日志 | eniipamd 处理延迟，或注解 JSON 格式错误 | 等待 30-60 秒（eniipamd 轮询周期）；检查 `eip-attributes` 值是否为合法 JSON |
| `kubectl apply` 创建 Pod 成功但无公网 IP | `kubectl get eipc -n NAMESPACE` 检查是否创建了 EIPClaim | eniipamd 组件版本过低（< 3.5.0）或集群不是 VPC-CNI 模式 | 升级 eniipamd：`tccli tke UpdateAddon --ClusterId CLUSTER_ID --AddonName eniipamd --AddonVersion "3.11.0"`。确认集群为 VPC-CNI：`tccli tke DescribeClusters --region REGION --ClusterIds '["CLUSTER_ID"]'` |
| `DescribeAddon eniipamd` 返回空或组件未安装 | 检查集群网络模式 | VPC-CNI 未启用时 eniipamd 不安装 | 启用 VPC-CNI：在控制台或通过 `EnableVpcCniNetworkType` 开启 |
| EIPClaim 删除失败 | `kubectl describe eipc EIP_NAME -n NAMESPACE` | Pod 尚在运行，EIP 在被使用中 | 先删除 Pod，再删除 EIPClaim：`kubectl delete pod POD_NAME -n NAMESPACE && kubectl delete eipc EIP_NAME -n NAMESPACE` |

### 功能异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 出站 IP 不是 EIP（而是节点 IP） | `kubectl get cm ip-masq-agent-config -n kube-system -o json \| jq '.data.config'` 检查 NonMasqueradeSrcCIDRs | `NonMasqueradeSrcCIDRs` 未包含 Pod IP | 将 Pod IP 加入白名单（编辑 ConfigMap，1 分钟内生效）。获取 Pod IP：`kubectl get pod POD_NAME -n NAMESPACE -o jsonpath='{.status.podIP}'` |
| 指定已有 EIP 但 Pod 未绑定 | `kubectl describe pod POD_NAME -n NAMESPACE \| grep "eip"` | EIP ID 不存在，或 EIP 不在同一地域，或 EIP 已被其他资源绑定 | `tccli vpc DescribeAddresses --region REGION --AddressIds '["EIP_ID"]'` 确认 EIP 存在且状态为 UNBIND。确保 EIP 与集群在同一地域 |
| Pod 删除后 EIP 意外释放 | `kubectl describe pod POD_NAME -n NAMESPACE \| grep "eip-claim-delete-policy"` | 未设置 `eip-claim-delete-policy: Never`（默认 Immediate） | 重新创建 Pod 并添加注解 `tke.cloud.tencent.com/eip-claim-delete-policy: Never` |
| 集群删除后 EIP 残留 | `tccli vpc DescribeAddresses --region REGION \| jq '.AddressSet[] \| select(.AddressStatus=="BIND")'` | 官方已知限制：集群删除不自动回收 EIP | 在 VPC 控制台手动释放残留 EIP，或通过 `tccli vpc ReleaseAddresses --region REGION --AddressIds '["EIP_ID"]'` 释放 |
| kubectl 连接超时（`dial tcp 172.x.x.x:443: no route to host`） | `tccli tke DescribeClusterEndpoints --region REGION --ClusterId CLUSTER_ID` 查看端点 | 本地环境无法直连集群内网端点（172.x 内网 IP 不可达）。公网端点可能被 CAM 策略 strategyId:240463971（`tke:clusterExtranetEndpoint=true` 条件）拒绝 | 需在腾讯云内网环境（同 VPC CVM）或通过 IOA/VPN/专线 连接集群。`DescribeClusterSecurity` 可获取域名形式的 kubeconfig，但本地 DNS 可能无法解析。保留 RequestId 以备工单查询 |
| `CreateClusterEndpoint` (extranet) 返回 `InvalidParameter` | 检查 SecurityGroup 参数 | `SecurityGroup can not be empty`——公网端点创建必须指定安全组 | 传入有效的 SecurityGroup ID。若仍被 CAM 策略 strategyId:240463971 以 `tke:clusterExtranetEndpoint=true` 条件拒绝，回退到内网端点（`--IsExtranet false`），需通过 IOA/VPN/专线 或同 VPC CVM 访问 |

## 下一步

- [VPC-CNI 模式安全组使用说明](../VPC-CNI%20模式安全组使用说明/tccli%20操作.md) — 配置 Pod 级别的安全组规则
- [固定 IP 使用方法](../固定%20IP%20使用方法/tccli%20操作.md) — VPC-CNI 下 Pod 固定 IP
- [VPC-CNI 模式介绍](../VPC-CNI%20模式介绍/tccli%20操作.md) — VPC-CNI 网络模式概述
- [弹性公网 IP 文档](https://cloud.tencent.com/document/product/1199) — VPC EIP 产品文档
- [CAM 角色与策略](https://cloud.tencent.com/document/product/598) — IPAMD 角色授权参考

## 控制台替代

[弹性公网 IP 控制台](https://console.cloud.tencent.com/cvm/eip) → 申请 EIP；[容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) → 工作负载 → StatefulSet/Deployment → 数据卷/网络 → VPC-CNI → EIP 配置。
