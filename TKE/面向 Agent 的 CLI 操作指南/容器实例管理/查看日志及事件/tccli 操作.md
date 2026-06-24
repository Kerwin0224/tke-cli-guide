# 查看日志及事件

> 对照官方：[查看日志及事件](https://cloud.tencent.com/document/product/457/57350) · page_id `57350`

## 概述

通过 `DescribeEKSContainerInstanceEvent` 查看容器实例生命周期事件（调度、拉镜像、启动等），通过 `DescribeEksContainerInstanceLog` 查看容器标准输出日志。两种手段结合使用，可快速定位容器实例的运行问题和异常退出原因。

**两种诊断手段的定位**：

| 手段 | CLI 接口 | 适用场景 |
|------|------|------|
| 事件 | `DescribeEKSContainerInstanceEvent` | 实例无法启动、状态卡住、调度失败、镜像问题 |
| 日志 | `DescribeEksContainerInstanceLog` | 应用运行日志、启动输出、错误诊断 |

**排查流程**：

```
DescribeEKSContainerInstances（查状态）
  ├─ Status=Running → DescribeEksContainerInstanceLog（查日志）
  ├─ Status=Creating → DescribeEKSContainerInstanceEvent（查事件）
  └─ Status=Failed   → 事件 + 日志联合诊断
```

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
| 查看实例事件 | `DescribeEKSContainerInstanceEvent --EksCiId` | 是 |
| 查看容器日志 | `DescribeEksContainerInstanceLog --EksCiId --ContainerName` | 是 |
| 查看最近 N 行日志 | `DescribeEksContainerInstanceLog --Tail` | 是 |
| 查看指定时间范围日志 | `DescribeEksContainerInstanceLog --SinceSeconds` | 是 |

## 操作步骤

### 步骤 1：查看实例事件

事件展示实例从创建到运行的完整过程，帮助定位 `Creating` 或 `Failed` 的原因：

```bash
tccli tke DescribeEKSContainerInstanceEvent --region <Region> \
    --EksCiId <EksCiId>
# expected: Events 数组非空
```

**预期输出**：

```json
{
    "EksCiId": "eksci-example",
    "Events": [
        {"Type": "Normal", "Reason": "Scheduled", "Message": "Successfully assigned default/eksci-example to eklet-subnet-xxx"},
        {"Type": "Normal", "Reason": "Pulling", "Message": "Pulling image \"nginx:alpine\""},
        {"Type": "Normal", "Reason": "Pulled", "Message": "Successfully pulled image \"nginx:alpine\" in 2.1s"},
        {"Type": "Normal", "Reason": "Created", "Message": "Created container nginx"},
        {"Type": "Normal", "Reason": "Started", "Message": "Started container nginx"}
    ]
}
```

| 占位符 | 说明 | 获取方式 |
|--------|------|---------|
| `<EksCiId>` | 容器实例 ID | `tccli tke DescribeEKSContainerInstances --region <Region>` |
| `<Region>` | 地域，如 `ap-guangzhou` | `tccli configure list` |

### 步骤 2：查看容器日志

**查看最近 N 行日志**：

```bash
tccli tke DescribeEksContainerInstanceLog --region <Region> \
    --EksCiId <EksCiId> \
    --ContainerName nginx \
    --Tail 10
# expected: LogContent 不为空
```

**预期输出**：

```json
{
    "ContainerName": "nginx",
    "LogContent": "/docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration\n/docker-entrypoint.sh: Looking for shell scripts in /docker-entrypoint.d/\n...\nnginx: start worker process 30\nnginx: start worker process 31\n"
}
```

**查看指定时间范围内的日志**（最近 1 小时）：

```bash
tccli tke DescribeEksContainerInstanceLog --region <Region> \
    --EksCiId <EksCiId> \
    --ContainerName nginx \
    --SinceSeconds 3600
# expected: LogContent 含最近 1 小时内日志
```

```json
{
  "ContainerName": "<ContainerName>",
  "LogContent": "<LogContent>",
  "RequestId": "<RequestId>"
}
```

### 关键字段说明

| 字段 | 类型 | 必填 | 取值与约束 | 说明 |
|------|------|:--:|------|------|
| `--EksCiId` | String | 是 | 实例 ID | `tccli tke DescribeEKSContainerInstances` 获取 |
| `--ContainerName` | String | 是（查日志时） | 容器名称，需与创建时 `Containers[].Name` 一致 | 不一致 → `InvalidParameterValue.ContainerName` |
| `--Tail` | Integer | 否 | 返回最近 N 行，默认全部 | 不指定时返回所有可获取的日志 |
| `--SinceSeconds` | Integer | 否 | 返回最近 N 秒内的日志 | 可与 `Tail` 组合使用 |

## 验证

```bash
# 确认日志可获取
tccli tke DescribeEksContainerInstanceLog --region <Region> \
    --EksCiId <EksCiId> \
    --ContainerName nginx \
    --Tail 1
# expected: LogContent 不为空字符串
```

```json
{
  "ContainerName": "<ContainerName>",
  "LogContent": "<LogContent>",
  "RequestId": "<RequestId>"
}
```

| 验证维度 | 命令 | 预期 |
|------|------|------|
| 事件可获取 | `DescribeEKSContainerInstanceEvent --region <Region> --EksCiId <EksCiId>` | `Events` 数组非空 |
| 日志可获取 | `DescribeEksContainerInstanceLog --region <Region> --EksCiId <EksCiId> --ContainerName nginx` | `LogContent` 非空字符串 |
| 容器名正确 | 同上 | 不返回 `InvalidParameterValue.ContainerName` |

## 清理

本页为读操作页面，无资源需清理。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DescribeEksContainerInstanceLog` 返回 `InvalidParameterValue.ContainerName` | `tccli tke DescribeEKSContainerInstances --region <Region> --EksCiIds '["<EksCiId>"]'` 查看 `Containers[].Name` | ContainerName 与创建时 `Containers[].Name` 不一致 | 用正确的容器名重新执行：`--ContainerName <正确名称>` |
| `DescribeEksContainerInstanceLog` 返回 `InvalidParameterValue.EksCiId` | 检查 EksCiId 格式 | 实例 ID 格式错误或实例不存在 | `tccli tke DescribeEKSContainerInstances --region <Region>` 列出全部实例确认 ID |
| `DescribeEKSContainerInstanceEvent` / `DescribeEksContainerInstanceLog` 返回 `UnauthorizedOperation.CamNoAuth` | `tccli configure list` 检查凭据 | 子账号缺少 `tke:DescribeEKSContainerInstanceEvent` 或 `tke:DescribeEksContainerInstanceLog` 权限（此为环境限制，非命令错误） | 联系主账号授予 `QcloudTKEFullAccess` 或对应 Action 权限 |

### 操作成功但日志/事件异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 日志为空（`LogContent: ""`） | 容器刚启动或已退出，见 `Containers[].CurrentState` | Web 服务（如 nginx）无持续 stdout 输出，或容器已正常退出 | 使用 `--SinceSeconds` 扩大时间范围，或检查 `CurrentState.State` 确认容器状态 |
| 日志只有启动输出而无业务日志 | 正常现象 | 日志只采集 stdout/stderr，文件日志不在采集范围 | 如需采集文件日志，配置 [开启日志采集](../开启日志采集/tccli%20操作.md) 到 CLS |
| 事件中 `FailedScheduling` | `tccli vpc DescribeSubnets --region <Region> --SubnetIds '["<SubnetId>"]'` 查看 AvailableIpCount | 子网可用 IP 用尽 | 清理无用实例释放 IP，或扩容子网 CIDR |
| 事件中 `ErrImagePull` | 检查镜像名和 `tccli tke DescribeEksContainerInstanceLog --EksCiId <EksCiId> --ContainerName <Name>` 查看详细错误 | 镜像名拼写错误或私有仓库未配置凭证 | 修正镜像名，或通过 `ImageRegistryCredentials` 配置私有仓库凭证 |
| 事件中 `ErrImageNeverPull` | 检查镜像仓库类型 | 私有仓库凭证未配置 | 在 `CreateEKSContainerInstances` 时传入 `ImageRegistryCredentials` 参数 |

## 下一步

- [容器实例生命周期](../容器实例生命周期/tccli%20操作.md) — 状态含义与事件解读
- [开启日志采集](../开启日志采集/tccli%20操作.md) — 配置持久化日志采集到 CLS
- [登录实例](../登录实例/tccli%20操作.md) — 通过日志和事件诊断容器
- [容器实例操作手册](../容器实例操作手册/tccli%20操作.md) — 完整 CRUD 操作

## 控制台替代

[TKE 控制台 → 弹性容器 → 容器实例 → 点击实例名](https://console.cloud.tencent.com/tke2/eks/ci)：事件 Tab / 日志 Tab。
