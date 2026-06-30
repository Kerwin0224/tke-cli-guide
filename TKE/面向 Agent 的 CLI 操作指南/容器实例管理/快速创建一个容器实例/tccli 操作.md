# 快速创建一个容器实例

> 对照官方：[快速创建一个容器实例](https://cloud.tencent.com/document/product/457/58026) · page_id `58026`

## 概述

通过 `CreateEKSContainerInstances` 创建弹性容器实例。最小规格 0.25 核 / 0.5 GB 内存，使用 `nginx:alpine` 镜像即可在 2 分钟内跑通完整周期（Create → Describe → 验证 → Delete）。

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
#    tke:CreateEKSContainerInstances, tke:DescribeEKSContainerInstances
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
# 4. 查询可用 VPC 和子网
tccli vpc DescribeVpcs --region <Region>
# expected: 至少返回 1 个 VPC

tccli vpc DescribeSubnets --region <Region> \
    --Filters '[{"Name":"vpc-id","Values":["<VpcId>"]}]'
# expected: 至少返回 1 个子网，AvailableIpCount >= 1

# 5. 查询安全组
tccli vpc DescribeSecurityGroups --region <Region>
# expected: 至少返回 1 个安全组
```

## 关键字段说明

`CreateEKSContainerInstances` 的主要参数。完整参数定义见 `tccli tke CreateEKSContainerInstances --generate-cli-skeleton`。

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `EksCiName` | String | 是 | 实例名称，长度 1-60 字符，以字母开头 | 命名格式不符 → `InvalidParameterValue.EksCiName` |
| `VpcId` | String | 是 | 已有 VPC ID，格式 `vpc-xxxxxxxx`。`tccli vpc DescribeVpcs` 获取 | VPC 不存在 → `InvalidParameterValue.VpcId` |
| `SubnetId` | String | 是 | 已有子网 ID，需与 VPC 在同一可用区。`tccli vpc DescribeSubnets` 获取 | 子网不可用 → 实例长时间 `Pending`，事件显示 `FailedScheduling` |
| `SecurityGroupIds` | Array | 是 | 安全组 ID 列表，如 `["sg-xxxxxxxx"]` | 安全组不存在 → `InvalidParameterValue.SecurityGroupId` |
| `Cpu` | Float | 是 | vCPU 核数，最小 0.25，步长 0.25 | 规格不支持 → `InvalidParameterValue.Cpu` |
| `Memory` | Float | 是 | 内存 GB，最小 0.5，步长 0.5。CPU:Memory 配比需符合规格表 | 配比不合法 → `InvalidParameterValue.Memory` |
| `Containers` | Array | 是 | 容器定义列表，每项含 `Name`（容器名）、`Image`（镜像地址）、`Commands`（启动命令） | 镜像不存在 → 实例 `Failed`，事件 `ErrImagePull` |
| `RestartPolicy` | String | 否 | `Always`（默认）、`OnFailure`、`Never` | — |
| `CamRoleName` | String | 否 | CAM 角色名，需预先在 CAM 控制台创建且载体为 CVM | 角色不存在 → `InvalidParameterValue.CamRoleName` |
| `AutoCreateEip` | Boolean | 否 | `true` 时自动创建并绑定新 EIP | 与 `ExistedEipIds` 互斥 |
| `AutoCreateEipAttribute` | Object | 否 | 自动创建 EIP 的属性：`DeletePolicy`（`Delete`/`Retain`）、`InternetServiceProvider`（`BGP`）、`InternetMaxBandwidthOut` | `DeletePolicy` 不设 `Delete` → 删除实例后 EIP 持续计费 |


## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 |
|-----------|----------|
| 参见官方文档 | `tccli tke DescribeClusters` |

## 操作步骤

### 步骤 1：创建容器实例

#### 选择依据

*为什么选这些值：*

- **CPU/内存**：选择 0.25 核 / 0.5 GB，最小规格，成本最低，适合验证 API。
- **镜像**：选择 `nginx:alpine`，轻量（约 5 MB），启动快，无依赖，验证阶段首选。
- **重启策略**：选择 `Never`，验证用途无需自动重启；生产环境选择 `Always` 保活。
- **容器启动命令**：`["nginx", "-g", "daemon off;"]` 确保 nginx 前台运行，容器不会立即退出。

#### 最小创建（仅含必填字段）

`create-minimal.json`：

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
    --cli-input-json file://create-minimal.json
# expected: exit 0，返回 EksCiIds
```

**预期输出**：

```json
{
    "EksCiIds": ["eksci-example"],
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<VpcId>` | VPC 实例 ID | 格式 `vpc-xxxxxxxx` | `tccli vpc DescribeVpcs --region <Region>` |
| `<SubnetId>` | 子网 ID | 格式 `subnet-xxxxxxxx`，需与 VPC 同可用区 | `tccli vpc DescribeSubnets --region <Region> --Filters '[{"Name":"vpc-id","Values":["<VpcId>"]}]'` |
| `<SecurityGroupId>` | 安全组 ID | 格式 `sg-xxxxxxxx` | `tccli vpc DescribeSecurityGroups --region <Region>` |
| `<Region>` | 地域 | 如 `ap-guangzhou` | `tccli tke DescribeRegions` |

#### 增强配置（加 EIP、CAM 角色、标签）

`create-enhanced.json`：

```json
{
    "EksCiName": "eksci-enhanced",
    "VpcId": "<VpcId>",
    "SubnetId": "<SubnetId>",
    "SecurityGroupIds": ["<SecurityGroupId>"],
    "Cpu": 1.0,
    "Memory": 2.0,
    "AutoCreateEip": true,
    "AutoCreateEipAttribute": {
        "DeletePolicy": "Delete",
        "InternetServiceProvider": "BGP",
        "InternetMaxBandwidthOut": 10
    },
    "CamRoleName": "TKE_QCSRole",
    "Containers": [
        {
            "Name": "app",
            "Image": "nginx:alpine",
            "Commands": ["nginx", "-g", "daemon off;"]
        }
    ],
    "RestartPolicy": "Always"
}
```

```bash
tccli tke CreateEKSContainerInstances --region <Region> \
    --cli-input-json file://create-enhanced.json
# expected: exit 0，返回 EksCiIds
```

| 层级 | 包含字段 | 目的 |
|------|---------|------|
| **最小创建** | EksCiName, VpcId, SubnetId, SecurityGroupIds, Cpu, Memory, Containers, RestartPolicy | 让读者 2 分钟内跑通 |
| **增强配置** | 最小创建 + AutoCreateEip, AutoCreateEipAttribute, CamRoleName, 更大规格 | 生产环境推荐配置 |

### 步骤 2：轮询直到状态为 Running

实例创建是异步操作。轮询直到状态为 `Running`：

```bash
tccli tke DescribeEKSContainerInstances --region <Region> \
    --EksCiIds '["<EksCiId>"]'
# expected: Status: "Running"
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
            "SubnetId": "<SubnetId>",
            "CreationTime": "2026-06-18 10:00:00 +0000 UTC",
            "Containers": [
                {
                    "Name": "nginx",
                    "Image": "nginx:alpine",
                    "CurrentState": {
                        "State": "running",
                        "StartTime": "2026-06-18T10:00:05Z",
                        "RestartCount": 0
                    },
                    "Commands": ["nginx", "-g", "daemon off;"]
                }
            ]
        }
    ],
    "RequestId": "..."
}
```

### 步骤 3：查看容器日志

```bash
tccli tke DescribeEksContainerInstanceLog --region <Region> \
    --EksCiId <EksCiId> \
    --ContainerName nginx
# expected: LogContent 不为空
```

**预期输出**：

```json
{
    "ContainerName": "nginx",
    "LogContent": "/docker-entrypoint.sh: Configuration complete; ready for start up\nnginx: start worker processes\n"
}
```

## 验证

实例创建后，从多个维度确认可用：

| 维度 | 命令 | 预期 |
|------|------|------|
| 状态 | `tccli tke DescribeEKSContainerInstances --region <Region> --EksCiIds '["<EksCiId>"]'` | `Status: "Running"` |
| 网络 | 同上，检查 `PrivateIp` | 已分配内网 IP |
| 容器状态 | 同上，检查 `Containers[].CurrentState.State` | `"running"` |
| 重启次数 | 同上，检查 `RestartCount` | `0`（新创建实例） |
| 日志 | `tccli tke DescribeEksContainerInstanceLog --region <Region> --EksCiId <EksCiId> --ContainerName nginx` | `LogContent` 含容器输出 |

```bash
# 综合验证
tccli tke DescribeEKSContainerInstances --region <Region> \
    --EksCiIds '["<EksCiId>"]'
# expected: Status: "Running"，PrivateIp 非空，CurrentState.State: "running"，RestartCount: 0
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

> **警告**：`DeleteEKSContainerInstances` 会释放实例及其分配的 CPU/内存资源。删除后将无法恢复实例数据。
> 若创建时使用了 `AutoCreateEip` 且 `DeletePolicy=Delete`，EIP 随实例一同释放；否则需手动释放 EIP 以避免持续计费。

### 1. 清理前状态检查

```bash
tccli tke DescribeEKSContainerInstances --region <Region> \
    --EksCiIds '["<EksCiId>"]'
# 确认是待删除的目标实例，记录 EksCiId、EksCiName、Status
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

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `CreateEKSContainerInstances` 返回 `InvalidParameterValue.VpcId` | `tccli vpc DescribeVpcs --region <Region> --VpcIds '["<VpcId>"]'` 确认 VPC 存在 | VPC ID 格式错误或 VPC 不属于当前地域 | 用 `tccli vpc DescribeVpcs --region <Region>` 获取正确的 VPC ID |
| `CreateEKSContainerInstances` 返回 `InvalidParameterValue.SubnetId` | `tccli vpc DescribeSubnets --region <Region> --SubnetIds '["<SubnetId>"]'` 确认子网存在且可用 | 子网与 VPC 不在同一可用区 | 查询子网：`tccli vpc DescribeSubnets --region <Region> --Filters '[{"Name":"vpc-id","Values":["<VpcId>"]}]'`，选择同地域子网 |
| `CreateEKSContainerInstances` 返回 `Invalid JSON received` | `python3 -m json.tool create-minimal.json` 验证 JSON 合法性 | JSON 格式错误，如多余逗号、编码问题 | 用 `jq . create-minimal.json` 检查并修正 JSON |
| `CreateEKSContainerInstances` 返回 `UnauthorizedOperation.CamNoAuth` | `tccli configure list` 检查当前凭据 | 子账号缺少 `tke:CreateEKSContainerInstances` 权限（此为环境限制，非命令错误） | 联系主账号授予 `QcloudTKEFullAccess` 或 `tke:CreateEKSContainerInstances` |
| `CreateEKSContainerInstances` 返回 `LimitExceeded` | `tccli tke DescribeEKSContainerInstances --region <Region>` 查看已有实例数 | 容器实例配额用尽（此为环境限制，非命令错误） | 删除不再使用的实例后重试，或联系管理员提升配额 |

### 创建已提交但长时间不 Running

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 返回 EksCiId 但 5 分钟后状态仍非 `Running` | `tccli tke DescribeEKSContainerInstanceEvent --region <Region> --EksCiId <EksCiId>` 查看事件 | 子网 IP 不足（`FailedScheduling`）或镜像拉取失败（`ErrImagePull`） | `FailedScheduling` → 扩容子网或清理无用实例；`ErrImagePull` → 检查镜像名拼写 |
| 状态为 `Failed` | `tccli tke DescribeEKSContainerInstances --region <Region> --EksCiIds '["<EksCiId>"]'` 查看 `Containers[].CurrentState.ExitCode` 和 `Reason` | 容器启动命令失败或 OOM | 检查 Commands 参数是否正确，增加 Memory 配额 |
| 状态为 `Succeeded` 但预期是 `Running` | 检查 `RestartPolicy` 值 | `RestartPolicy=Never` 时容器正常退出后进入 `Succeeded`，不会再保持运行 | 如需容器持续运行，设置 `RestartPolicy: "Always"` 并使用前台命令 |

## 下一步

- [容器实例操作手册](../容器实例操作手册/tccli%20操作.md) — 更新、重启等全生命周期操作
- [容器实例生命周期](../容器实例生命周期/tccli%20操作.md) — 状态枚举与事件诊断
- [通过绑定弹性公网 IP 访问外网](../通过绑定弹性公网%20IP%20访问外网/tccli%20操作.md) — 为实例配置公网访问
- [登录实例](../登录实例/tccli%20操作.md) — 通过日志和事件诊断容器
- [查看日志及事件](../查看日志及事件/tccli%20操作.md) — 日志与事件联合诊断

## 控制台替代

[TKE 控制台 → 弹性容器 → 容器实例 → 新建](https://console.cloud.tencent.com/tke2/eks/ci)：可视化创建容器实例。
