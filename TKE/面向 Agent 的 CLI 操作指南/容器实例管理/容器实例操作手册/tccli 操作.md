# 容器实例操作手册

> 对照官方：[容器实例操作手册](https://cloud.tencent.com/document/product/457/57341) · page_id `57341`

## 概述

弹性容器实例全生命周期管理：创建、查询、更新、重启、删除。对应控制台弹性容器 → 容器实例页的所有操作。所有写操作（创建、更新、删除）均为非幂等，读操作（查询、事件、日志）为幂等。

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
#    tke:UpdateEKSContainerInstance, tke:RestartEKSContainerInstances
#    tke:DeleteEKSContainerInstances, tke:DescribeEKSContainerInstanceEvent
#    tke:DescribeEksContainerInstanceLog
# 验证：执行 DescribeEKSContainerInstances 确认权限
tccli tke DescribeEKSContainerInstances --region <Region>
# expected: exit 0，返回 TotalCount（可为 0）
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
# 4. 查询目标实例存在（如操作已有实例）
tccli tke DescribeEKSContainerInstances --region <Region> \
    --EksCiIds '["<EksCiId>"]'
# expected: TotalCount >= 1，目标实例信息可查

# 5. 查询 VPC 和子网（如需创建新实例）
tccli vpc DescribeVpcs --region <Region>
# expected: 至少返回 1 个 VPC

tccli vpc DescribeSubnets --region <Region> \
    --Filters '[{"Name":"vpc-id","Values":["<VpcId>"]}]'
# expected: 至少返回 1 个子网，AvailableIpCount >= 1
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 创建实例 | `CreateEKSContainerInstances` | 否 |
| 查询实例 | `DescribeEKSContainerInstances --EksCiIds` | 是 |
| 更新实例 | `UpdateEKSContainerInstance` | 否 |
| 重启实例 | `RestartEKSContainerInstances` | 否 |
| 查看事件 | `DescribeEKSContainerInstanceEvent` | 是 |
| 查看日志 | `DescribeEksContainerInstanceLog` | 是 |
| 删除实例 | `DeleteEKSContainerInstances` | 否 |

## 操作步骤

### 步骤 1：创建实例

#### 选择依据

- **最小规格**：0.25 核 / 0.5 GB 内存，验证 API 最低成本。
- **镜像**：`nginx:alpine` 轻量启动快，适合快速验证。
- **重启策略**：`Never` 适用于一次性任务；`Always` 适用于持续运行的服务。

`create.json`：

```json
{
    "EksCiName": "eksci-example",
    "VpcId": "<VpcId>",
    "SubnetId": "<SubnetId>",
    "SecurityGroupIds": ["<SecurityGroupId>"],
    "Cpu": 0.25,
    "Memory": 0.5,
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
    --cli-input-json file://create.json
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

### 步骤 2：查询实例

**按实例 ID 查询**：

```bash
tccli tke DescribeEKSContainerInstances --region <Region> \
    --EksCiIds '["<EksCiId>"]'
# expected: TotalCount: 1
```

**预期输出**：

```json
{
    "TotalCount": 1,
    "EksCis": [
        {
            "EksCiId": "eksci-example",
            "EksCiName": "eksci-example",
            "Status": "Running",
            "Cpu": 0.25,
            "Memory": 0.5,
            "PrivateIp": "10.0.x.x",
            "VpcId": "<VpcId>",
            "SubnetId": "<SubnetId>"
        }
    ],
    "RequestId": "..."
}
```

**列表查询（分页）**：

```bash
tccli tke DescribeEKSContainerInstances --region <Region> \
    --Offset 0 \
    --Limit 20
# expected: TotalCount 含实例总数，EksCis 列表最多 20 条
```

### 步骤 3：更新实例

更新实例名称。实例需在 `Running` 状态方可更新。

`update.json`：

```json
{
    "EksCiId": "<EksCiId>",
    "Name": "eksci-updated"
}
```

```bash
tccli tke UpdateEKSContainerInstance --region <Region> \
    --cli-input-json file://update.json
# expected: exit 0，返回 RequestId
```

**预期输出**：

```json
{
    "RequestId": "..."
}
```

> **约束**：实例需在 `Running` 状态方可更新。若实例处于 `Creating`、`Failed`、`Terminating` 等状态，更新会失败。

### 步骤 4：重启实例

```bash
tccli tke RestartEKSContainerInstances --region <Region> \
    --EksCiIds '["<EksCiId>"]'
# expected: exit 0，返回 RequestId
```

**预期输出**：

```json
{
    "RequestId": "..."
}
```

重启操作幂等：重启成功后再次重启不会报错。

### 步骤 5：查看事件

```bash
tccli tke DescribeEKSContainerInstanceEvent --region <Region> \
    --EksCiId <EksCiId>
# expected: Events 数组含生命周期事件
```

**预期输出**：

```json
{
    "EksCiId": "eksci-example",
    "Events": [
        {"Reason": "Scheduled", "Message": "Successfully assigned default/eksci-example", "Type": "Normal"},
        {"Reason": "Pulling", "Message": "Pulling image nginx:alpine", "Type": "Normal"},
        {"Reason": "Pulled", "Message": "Successfully pulled image", "Type": "Normal"},
        {"Reason": "Created", "Message": "Created container nginx", "Type": "Normal"},
        {"Reason": "Started", "Message": "Started container nginx", "Type": "Normal"}
    ]
}
```

### 步骤 6：查看日志

```bash
tccli tke DescribeEksContainerInstanceLog --region <Region> \
    --EksCiId <EksCiId> \
    --ContainerName nginx
# expected: LogContent 含应用日志
```

**预期输出**：

```json
{
    "ContainerName": "nginx",
    "LogContent": "/docker-entrypoint.sh: Configuration complete; ready for start up\nnginx: start worker processes\n"
}
```

### 步骤 7：删除实例

#### 选择依据

- 删除实例会释放分配的 CPU/内存资源，实例数据（容器文件系统）不可恢复。
- 若创建时使用 `AutoCreateEip` 且 `DeletePolicy=Delete`，EIP 随实例一同释放。若使用 `ExistedEipIds` 或 `DeletePolicy=Retain`，需手动释放 EIP。

`delete.json`：

```json
{
    "EksCiIds": ["<EksCiId>"]
}
```

```bash
tccli tke DeleteEKSContainerInstances --region <Region> \
    --cli-input-json file://delete.json
# expected: exit 0，返回 RequestId
```

**预期输出**：

```json
{
    "RequestId": "..."
}
```

## 验证

```bash
# 验证实例状态（更新/重启后）
tccli tke DescribeEKSContainerInstances --region <Region> \
    --EksCiIds '["<EksCiId>"]'
# expected: Status: "Running"
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

| 验证维度 | 命令 | 预期 |
|------|------|------|
| 状态 | `DescribeEKSContainerInstances --EksCiIds '["<EksCiId>"]'` | `Status: "Running"` |
| 更新生效 | 同上，检查 `EksCiName` | 与更新后的名称一致 |
| 容器状态 | 同上，检查 `Containers[].CurrentState.State` | `"running"` |
| 日志 | `DescribeEksContainerInstanceLog --EksCiId <EksCiId> --ContainerName nginx` | `LogContent` 非空 |

## 清理

> **警告**：`DeleteEKSContainerInstances` 会释放实例及其关联资源。实例容器内数据（文件系统）不可恢复。
> 若通过 `AutoCreateEip` + `DeletePolicy=Retain` 或 `ExistedEipIds` 绑定了 EIP，EIP 不会随实例自动释放，将持续计费。

### 1. 清理前状态检查

```bash
tccli tke DescribeEKSContainerInstances --region <Region> \
    --EksCiIds '["<EksCiId>"]'
# 确认是待删除的目标实例，记录 EksCiId、EksCiName、EipAddress
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

### 3. 确认已删除

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
tccli vpc ReleaseAddresses --region <Region> \
    --AddressIds '["<EipId>"]'
# expected: exit 0
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `CreateEKSContainerInstances` 返回 `InvalidParameterValue.VpcId` | `tccli vpc DescribeVpcs --region <Region> --VpcIds '["<VpcId>"]'` 确认 VPC 存在 | VPC ID 格式错误或所属地域不对 | 用 `tccli vpc DescribeVpcs --region <Region>` 获取正确 VPC ID |
| `CreateEKSContainerInstances` 返回 `LimitExceeded` | `tccli tke DescribeEKSContainerInstances --region <Region>` 查看已有实例数 | 容器实例配额用尽（此为环境限制，非命令错误） | 删除不再使用的实例后重试 |
| `UpdateEKSContainerInstance` 失败 | `tccli tke DescribeEKSContainerInstances --region <Region> --EksCiIds '["<EksCiId>"]'` 查看 Status | 实例状态非 `Running`，处于 `Creating`、`Failed` 或 `Terminating` | 等待实例进入 `Running` 状态后再更新 |
| `DeleteEKSContainerInstances` 返回 `InvalidParameterValue.EksCiId` | 检查 EksCiId 格式 | 实例 ID 格式错误或实例不存在 | `tccli tke DescribeEKSContainerInstances --region <Region>` 列出全部实例确认 ID |

### 操作成功但异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 创建后长时间处于 `Creating` 状态 | `tccli tke DescribeEKSContainerInstanceEvent --region <Region> --EksCiId <EksCiId>` 查看事件 | 子网 IP 不足（`FailedScheduling`）或镜像拉取失败（`ErrImagePull`） | `FailedScheduling` → 扩容子网；`ErrImagePull` → 修正镜像名或配置凭证 |
| 删除后 EIP 仍计费 | `tccli vpc DescribeAddresses --region <Region>` 查看 EIP 状态 | 创建时未设置 `DeletePolicy: "Delete"` 或使用了 `ExistedEipIds` | `tccli vpc ReleaseAddresses --region <Region> --AddressIds '["<EipId>"]'` 手动释放 |
| 重启后容器未恢复 | `tccli tke DescribeEKSContainerInstances --region <Region> --EksCiIds '["<EksCiId>"]'` 查看 Status 和 CurrentState | 容器启动命令执行失败或镜像变量变化 | 查看日志和事件定位具体原因，必要时重新创建实例 |
| 日志为空 | `tccli tke DescribeEKSContainerInstances --EksCiIds '["<EksCiId>"]'` 查看 CurrentState | Web 服务无持续 stdout 输出 | 使用 `--Tail` 增大行数或 `--SinceSeconds` 扩大时间范围 |

## 下一步

- [容器实例生命周期](../容器实例生命周期/tccli%20操作.md) — 状态流转与事件诊断
- [登录实例](../登录实例/tccli%20操作.md) — 通过日志和事件诊断容器
- [查看日志及事件](../查看日志及事件/tccli%20操作.md) — 日志与事件联合诊断
- [通过绑定弹性公网 IP 访问外网](../通过绑定弹性公网%20IP%20访问外网/tccli%20操作.md) — 为实例配置公网访问

## 控制台替代

[TKE 控制台 → 弹性容器 → 容器实例](https://console.cloud.tencent.com/tke2/eks/ci)：操作列提供查看、更新、重启、删除操作。
