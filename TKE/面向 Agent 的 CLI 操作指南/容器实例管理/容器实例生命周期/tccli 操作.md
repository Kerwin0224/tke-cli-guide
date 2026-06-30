# 容器实例生命周期

> 对照官方：[容器实例生命周期](https://cloud.tencent.com/document/product/457/57343) · page_id `57343`

## 概述

弹性容器实例从创建到销毁经历多个状态阶段。通过 `DescribeEKSContainerInstances` 查询当前 `Status`，通过 `DescribeEKSContainerInstanceEvent` 查看底层事件的完整链路（调度 → 拉镜像 → 创建 → 启动），快速判断容器处于哪个阶段以及是否正常。

**状态流转**：

```
Creating → Running → Succeeded / Failed → Terminating
```

| 状态 | 含义 | 典型耗时 | 容器级 State |
|------|------|---------|------------|
| `Creating` | 调度中、拉镜像、创建容器中 | < 2 分钟 | `waiting` |
| `Running` | 所有容器已启动，正常运行 | 持续 | `running` |
| `Succeeded` | 所有容器正常退出（exit 0），仅 `RestartPolicy=Never` 时出现 | — | `terminated` (exitCode: 0) |
| `Failed` | 容器异常退出（exit 非 0 或 OOMKilled） | — | `terminated` (exitCode 非 0) |
| `Terminating` | 正在删除，释放资源 | < 30 秒 | `terminated` 或 `running` |

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
#    tke:DescribeEKSContainerInstances, tke:DescribeEKSContainerInstanceEvent
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
# 4. 确认目标实例存在
tccli tke DescribeEKSContainerInstances --region <Region> \
    --EksCiIds '["<EksCiId>"]'
# expected: TotalCount >= 1，目标实例信息可查
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

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看实例状态 | `DescribeEKSContainerInstances --EksCiIds` | 是 |
| 查看状态变更事件 | `DescribeEKSContainerInstanceEvent` | 是 |
| 查看容器级状态 | `DescribeEKSContainerInstances` 返回中 `Containers[].CurrentState` | 是 |

## 操作步骤

### 步骤 1：查看实例当前状态

```bash
tccli tke DescribeEKSContainerInstances --region <Region> \
    --EksCiIds '["<EksCiId>"]'
# expected: TotalCount: 1，Status 为生命周期状态之一
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
            "CreationTime": "2026-06-18 10:00:00 +0000 UTC",
            "SucceededTime": "",
            "Containers": [
                {
                    "Name": "nginx",
                    "CurrentState": {
                        "State": "running",
                        "StartTime": "2026-06-18T10:00:05Z",
                        "RestartCount": 0
                    }
                }
            ]
        }
    ],
    "RequestId": "..."
}
```

### 步骤 2：查看状态变更事件

事件链揭示实例从创建到运行的完整过程，是定位 `Creating` 或 `Failed` 的首选手段。

```bash
tccli tke DescribeEKSContainerInstanceEvent --region <Region> \
    --EksCiId <EksCiId>
# expected: Events 含 Scheduled → Pulling → Pulled → Created → Started 事件流
```

**预期输出**（正常流程）：

```json
{
    "EksCiId": "eksci-example",
    "Events": [
        {"Type": "Normal", "Reason": "Scheduled", "Message": "Successfully assigned default/eksci-example", "Count": 1},
        {"Type": "Normal", "Reason": "Pulling", "Message": "Pulling image \"nginx:alpine\"", "Count": 1},
        {"Type": "Normal", "Reason": "Pulled", "Message": "Successfully pulled image \"nginx:alpine\"", "Count": 1},
        {"Type": "Normal", "Reason": "Created", "Message": "Created container nginx", "Count": 1},
        {"Type": "Normal", "Reason": "Started", "Message": "Started container nginx", "Count": 1}
    ]
}
```

### 步骤 3：解读容器级 CurrentState

`DescribeEKSContainerInstances` 返回中 `Containers[].CurrentState` 提供容器内更细粒度的状态：

| 字段 | 类型 | 说明 |
|------|------|------|
| `State` | String | `waiting`（等待启动）、`running`（运行中）、`terminated`（已退出） |
| `StartTime` | String | 容器启动时间（`running` 时有值） |
| `RestartCount` | Integer | 容器重启次数 |
| `ExitCode` | Integer | 退出码（`terminated` 时有值），0 表示正常退出 |
| `Reason` | String | 退出原因，如 `OOMKilled`（内存溢出）、`Error`（启动失败）、`Completed`（正常完成） |

## 验证

```bash
# 验证实例状态和容器运行状态
tccli tke DescribeEKSContainerInstances --region <Region> \
    --EksCiIds '["<EksCiId>"]'
# expected: Status: "Running"，Containers[].CurrentState.State: "running"
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
| 实例状态 | `DescribeEKSContainerInstances --EksCiIds '["<EksCiId>"]'` | `Status: "Running"` |
| 容器状态 | 同上，查看 `Containers[].CurrentState.State` | `"running"` |
| 重启次数 | 同上，查看 `Containers[].CurrentState.RestartCount` | `0` |
| 事件完整性 | `DescribeEKSContainerInstanceEvent --EksCiId <EksCiId>` | 含 Scheduled、Pulling、Pulled、Created、Started 事件 |

## 清理

本页为概念说明页，无资源需清理。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DescribeEKSContainerInstances` 返回空列表 | `tccli configure list` 确认 region 与实例所在地域一致 | region 与实例所在地域不匹配 | 用 `tccli configure set region <正确地域>` 切换 region |
| `DescribeEKSContainerInstances` 返回 `InvalidParameterValue.EksCiId` | 检查 EksCiId 格式 | EksCiId 格式错误或不属于当前账号 | `tccli tke DescribeEKSContainerInstances --region <Region>` 列出全部实例确认 ID |
| `DescribeEKSContainerInstances` 返回 `UnauthorizedOperation.CamNoAuth` | `tccli configure list` 检查凭据 | 子账号缺少 `tke:DescribeEKSContainerInstances` 权限（此为环境限制，非命令错误） | 联系主账号授予 `QcloudTKEFullAccess` 或 `tke:DescribeEKSContainerInstances` |

### 状态异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 实例长时间处于 `Creating` | `tccli tke DescribeEKSContainerInstanceEvent --region <Region> --EksCiId <EksCiId>` 查看事件 | 子网 IP 不足（`FailedScheduling`）或镜像拉取失败（`ErrImagePull`） | `FailedScheduling` → 扩容子网或清理无用实例；`ErrImagePull` → 检查镜像名拼写和仓库权限 |
| `RestartCount` 持续增长 | `tccli tke DescribeEKSContainerInstances --region <Region> --EksCiIds '["<EksCiId>"]'` 查看 `CurrentState.Reason` | OOM（`OOMKilled`）、应用崩溃（`Error`）或健康检查失败 | 增加 Memory 配额或修复应用代码 |
| 事件中 `ErrImagePull` | 检查 `DescribeEKSContainerInstanceEvent` 事件详情 | 镜像名拼写错误或私有仓库未配置凭证 | 修正镜像名，或通过 `ImageRegistryCredentials` 配置私有仓库凭证 |
| 实例从 `Running` 变为 `Failed` | 查看 `ExitCode` 和 `Reason`，结合日志 | 容器内应用异常退出 | 查看日志定位应用层问题：`tccli tke DescribeEksContainerInstanceLog --region <Region> --EksCiId <EksCiId> --ContainerName <Name>` |
| 实例进入 `Succeeded` 但预期是 `Running` | 检查创建时的 `RestartPolicy` | `RestartPolicy=Never` 下容器正常退出即进入 `Succeeded`，不再保持运行 | 如需持续运行，创建时设置 `RestartPolicy: "Always"` |

## 下一步

- [快速创建一个容器实例](../快速创建一个容器实例/tccli%20操作.md) — 创建第一个 EKS CI
- [容器实例操作手册](../容器实例操作手册/tccli%20操作.md) — 完整 CRUD 操作
- [登录实例](../登录实例/tccli%20操作.md) — 查看容器内运行状态
- [查看日志及事件](../查看日志及事件/tccli%20操作.md) — 日志与事件联合诊断

## 控制台替代

[TKE 控制台 → 弹性容器 → 容器实例 → 点击实例名](https://console.cloud.tencent.com/tke2/eks/ci)：详情页查看状态、事件和容器信息。
