# 通过绑定弹性公网 IP 访问外网

> 对照官方：[通过绑定弹性公网 IP 访问外网](https://cloud.tencent.com/document/product/457/57346) · page_id `57346`

## 概述

容器实例默认分配 VPC 内网 IP，无法直接访问外网。创建实例时通过 EIP 相关参数可在两种方式中选择：**自动创建新 EIP**（推荐）或**绑定已有 EIP**。两种方式互斥，不可同时使用。

**方式对比**：

| 维度 | 自动创建 EIP | 绑定已有 EIP |
|------|------------|------------|
| 配置复杂度 | 低，一行参数 `AutoCreateEip: true` | 需先 `AllocateAddresses` 创建 EIP |
| 释放管理 | `DeletePolicy: "Delete"` 时随实例自动释放 | 需手动 `ReleaseAddresses` 释放 |
| EIP 地址 | 自动分配，不可指定 | 可绑定特定 IP 地址 |
| 适用场景 | 大多数场景，随用随释放 | 需要固定公网 IP 的场景 |

## 前置条件

- [环境准备](../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:CreateEKSContainerInstances, tke:DescribeEKSContainerInstances
#    tke:DeleteEKSContainerInstances
#    vpc:AllocateAddresses, vpc:ReleaseAddresses, vpc:DescribeAddresses
# 验证：执行 DescribeEKSContainerInstances 确认 TKE 权限
tccli tke DescribeEKSContainerInstances --region <Region>
# expected: exit 0，返回 TotalCount（可为 0）

# 验证 VPC EIP 权限
tccli vpc DescribeAddresses --region <Region>
# expected: exit 0，返回 EIP 列表（可为空）
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

> **说明**：示例中 `<Region>` 替换为 `ap-guangzhou`。

### 资源检查

```bash
# 4. 查询 VPC 和子网
tccli vpc DescribeVpcs --region <Region>
# expected: 至少返回 1 个 VPC

tccli vpc DescribeSubnets --region <Region> \
    --Filters '[{"Name":"vpc-id","Values":["<VpcId>"]}]'
# expected: 至少返回 1 个子网，AvailableIpCount >= 1

# 5. 检查 EIP 配额（数量配额）
tccli vpc DescribeAddresses --region <Region>
# expected: 当前 EIP 数量，确认是否达到配额上限
```

## 关键字段说明

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `AutoCreateEip` | Boolean | 否 | `true` 时自动创建并绑定新 EIP。与 `ExistedEipIds` **互斥** | 与 `ExistedEipIds` 同时设置 → 参数冲突错误 |
| `AutoCreateEipAttribute.DeletePolicy` | String | 否 | `Delete`（随实例释放）或 `Retain`（保留）。默认值可能为 `Retain` | 设 `Retain` → 删除实例后 EIP 持续计费 |
| `AutoCreateEipAttribute.InternetServiceProvider` | String | 否 | `BGP`（普通 BGP IP）、`CTCC`（电信）、`CUCC`（联通）、`CMCC`（移动） | 线路类型不支持 → `InvalidParameterValue.InternetServiceProvider` |
| `AutoCreateEipAttribute.InternetMaxBandwidthOut` | Integer | 否 | 出带宽上限（Mbps），默认 1 | 超限 → `InvalidParameterValue.BandwidthOutOfRange` |
| `ExistedEipIds` | Array | 否 | 已有 EIP ID 列表，如 `["eip-xxxxxxxx"]`。与 `AutoCreateEip` **互斥** | 与 `AutoCreateEip` 同时设置 → 参数冲突错误；EIP 不存在 → `InvalidParameterValue.EipId` |
| `SecurityGroupIds` | Array | 是 | 安全组 ID 列表。EIP 绑定后，需确保安全组放通目标端口 | 未放通端口 → EIP 已绑定但访问不通 |


## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 |
|-----------|----------|
| 参见官方文档 | `tccli tke DescribeClusters` |

## 操作步骤

### 方式一：自动创建 EIP（推荐）

#### 选择依据

- **线路类型**：`BGP` 是普通 BGP IP，覆盖最广，为默认推荐。
- **带宽**：`10` Mbps 适合验证用途，生产环境按实际需求调整。
- **释放策略**：`Delete` 确保删除实例时 EIP 自动释放，避免产生额外费用。`Retain` 适用于需要保留 EIP 地址的场景。

#### 最小创建（自动创建 EIP）

`create-eip-minimal.json`：

```json
{
    "EksCiName": "eksci-eip-example",
    "VpcId": "<VpcId>",
    "SubnetId": "<SubnetId>",
    "SecurityGroupIds": ["<SecurityGroupId>"],
    "Cpu": 0.25,
    "Memory": 0.5,
    "AutoCreateEip": true,
    "AutoCreateEipAttribute": {
        "DeletePolicy": "Delete",
        "InternetServiceProvider": "BGP",
        "InternetMaxBandwidthOut": 10
    },
    "Containers": [
        {
            "Name": "nginx",
            "Image": "nginx:alpine",
            "Commands": ["nginx", "-g", "daemon off;"]
        }
    ],
    "RestartPolicy": "Never"
}
```

```bash
tccli tke CreateEKSContainerInstances --region <Region> \
    --cli-input-json file://create-eip-minimal.json
# expected: exit 0，返回 EksCiIds
```

**预期输出**：

```json
{
    "EksCiIds": ["eksci-example"],
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

| 占位符 | 说明 | 获取方式 |
|--------|------|---------|
| `<VpcId>` | VPC 实例 ID | `tccli vpc DescribeVpcs --region <Region>` |
| `<SubnetId>` | 子网 ID | `tccli vpc DescribeSubnets --region <Region> --Filters '[{"Name":"vpc-id","Values":["<VpcId>"]}]'` |
| `<SecurityGroupId>` | 安全组 ID | `tccli vpc DescribeSecurityGroups --region <Region>` |
| `<Region>` | 地域 | `tccli configure list` |

### 方式二：绑定已有 EIP

#### 选择依据

使用已有 EIP 适用于需要固定公网 IP 地址的场景（如白名单、DNS 绑定）。注意：`ExistedEipIds` 绑定的 EIP 不会随实例删除而释放，需手动管理。

#### 步骤 2a：创建 EIP

```bash
tccli vpc AllocateAddresses --region <Region> \
    --AddressCount 1
# expected: exit 0，返回 AddressSet
```

**预期输出**：

```json
{
    "AddressSet": ["eip-xxxxxxxx"],
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

#### 步骤 2b：创建实例绑定已有 EIP

`create-existing-eip.json`：

```json
{
    "EksCiName": "eksci-existing-eip",
    "VpcId": "<VpcId>",
    "SubnetId": "<SubnetId>",
    "SecurityGroupIds": ["<SecurityGroupId>"],
    "Cpu": 0.25,
    "Memory": 0.5,
    "ExistedEipIds": ["<EipId>"],
    "Containers": [
        {
            "Name": "nginx",
            "Image": "nginx:alpine",
            "Commands": ["nginx", "-g", "daemon off;"]
        }
    ],
    "RestartPolicy": "Never"
}
```

```bash
tccli tke CreateEKSContainerInstances --region <Region> \
    --cli-input-json file://create-existing-eip.json
# expected: exit 0，返回 EksCiIds
```

### 步骤 3：轮询直到实例 Running 并获取公网 IP

```bash
tccli tke DescribeEKSContainerInstances --region <Region> \
    --EksCiIds '["<EksCiId>"]'
# expected: Status: "Running"，EipAddress 字段非空
```

**预期输出**：

```json
{
    "TotalCount": 1,
    "EksCis": [
        {
            "EksCiId": "eksci-example",
            "EksCiName": "eksci-eip-example",
            "Status": "Running",
            "PrivateIp": "10.0.x.x",
            "EipAddress": "1.2.3.4"
        }
    ],
    "RequestId": "..."
}
```

### 步骤 4：验证外网连通性

确认安全组已放通目标端口后，从有外网的环境测试连通性。

## 验证

| 验证维度 | 命令 | 预期 |
|------|------|------|
| 状态 | `DescribeEKSContainerInstances --EksCiIds '["<EksCiId>"]'` | `Status: "Running"` |
| EIP 绑定 | 同上，检查 `EipAddress` | 非空字符串，格式为公网 IP |
| 外网可达 | `curl http://<公网IP>:<端口>`（从外网环境） | 能获取响应 |

```bash
# 确认 EIP 已绑定
tccli tke DescribeEKSContainerInstances --region <Region> \
    --EksCiIds '["<EksCiId>"]'
# expected: EipAddress 字段非空字符串
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

## 清理

> **警告**：删除实例时的 EIP 清理行为取决于创建时的配置：
> - 方式一 `DeletePolicy: "Delete"`：EIP 随实例自动释放，停止计费。
> - 方式一 `DeletePolicy: "Retain"`：EIP 保留，持续计费。需手动释放。
> - 方式二 `ExistedEipIds`：EIP 始终保留，需手动释放。

### 1. 清理前状态检查

```bash
tccli tke DescribeEKSContainerInstances --region <Region> \
    --EksCiIds '["<EksCiId>"]'
# 确认是待删除的目标实例，记录 EksCiId、EipAddress
```

```json
{
  "TotalCount": 0,
  "EksCis": [],
  "AutoCreatedEipId": "<AutoCreatedEipId>",
  "CamRoleName": "<CamRoleName>",
  "Containers": [],
  "Image": "<Image>",
  "Name": "<Name>",
  "Args": []
}
```

### 2. 删除实例

```bash
tccli tke DeleteEKSContainerInstances --region <Region> \
    --EksCiIds '["<EksCiId>"]'
# expected: exit 0，返回 RequestId
```

### 3. 验证已删除

```bash
tccli tke DescribeEKSContainerInstances --region <Region> \
    --EksCiIds '["<EksCiId>"]'
# expected: TotalCount: 0
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

### 4. 手动释放 EIP（如未自动释放）

```bash
# 先确认 EIP 状态
tccli vpc DescribeAddresses --region <Region> \
    --Filters '[{"Name":"address-ip","Values":["1.2.3.4"]}]'
# expected: AddressStatus: "UNBIND" 或 "BIND"

# 释放 EIP
tccli vpc ReleaseAddresses --region <Region> \
    --AddressIds '["<EipId>"]'
# expected: exit 0
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `CreateEKSContainerInstances` 返回 `AutoCreateEip` 与 `ExistedEipIds` 冲突 | 检查 JSON 中是否同时设置了两个字段 | `AutoCreateEip` 与 `ExistedEipIds` 互斥 | 只保留其中一个方式 |
| `AllocateAddresses` 返回 `LimitExceeded.AddressQuota` | `tccli vpc DescribeAddresses --region <Region>` 查看已有 EIP 数量 | 账号 EIP 配额用尽（此为环境限制，非命令错误） | 释放未使用的 EIP 或提交工单提升配额 |
| `CreateEKSContainerInstances` 返回 `InvalidParameterValue.EipId` | `tccli vpc DescribeAddresses --region <Region>` 确认 EIP 存在 | EIP ID 格式错误或 EIP 不存在 | 用 `tccli vpc DescribeAddresses --region <Region>` 获取正确的 EIP ID |
| `ReleaseAddresses` 返回 EIP 仍被绑定 | `tccli vpc DescribeAddresses --region <Region> --Filters '[{"Name":"address-ip","Values":["1.2.3.4"]}]'` 查看状态 | EIP 仍绑定在实例上 | 先删除实例，待 EIP 解绑后再释放 |

### 操作成功但异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| EIP 已绑定但端口不通 | `tccli vpc DescribeSecurityGroupPolicies --region <Region> --SecurityGroupId <SecurityGroupId>` 查看安全组规则 | 安全组未放通目标端口 | 在安全组中添加入站规则，放通目标端口（如 TCP 80） |
| 方式一删除实例后 EIP 仍计费 | `tccli vpc DescribeAddresses --region <Region>` 查看 EIP 状态 | `AutoCreateEipAttribute.DeletePolicy` 未设为 `Delete` 或默认值为 `Retain` | `tccli vpc ReleaseAddresses --region <Region> --AddressIds '["<EipId>"]'` 手动释放 |
| 方式二删除实例后 EIP 仍计费 | 正常行为 | `ExistedEipIds` 绑定的 EIP 不会随实例释放 | `tccli vpc ReleaseAddresses --region <Region> --AddressIds '["<EipId>"]'` 手动释放（确认不再需要后） |
| EIP 访问延迟高 | `tccli vpc DescribeAddresses --region <Region> --AddressIds '["<EipId>"]'` 查看 InternetServiceProvider | 线路类型限制了网络质量 | 重新创建实例，选择 `BGP` 线路类型 |

## 下一步

- [容器实例操作手册](../容器实例操作手册/tccli%20操作.md) — 完整 CRUD 操作
- [查看日志及事件](../查看日志及事件/tccli%20操作.md) — 日志与事件联合诊断
- [VPC 弹性公网 IP 文档](https://cloud.tencent.com/document/product/1199) — EIP 完整功能文档
- [登录实例](../登录实例/tccli%20操作.md) — 通过日志和事件诊断容器

## 控制台替代

[TKE 控制台 → 弹性容器 → 容器实例 → 新建 → 网络配置](https://console.cloud.tencent.com/tke2/eks/ci)：公网 IP 选项中可选择自动创建或使用已有 EIP。
