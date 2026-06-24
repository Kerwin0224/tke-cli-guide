# 登录实例

> 对照官方：[登录实例](https://cloud.tencent.com/document/product/457/57344) · page_id `57344`

## 概述

弹性容器实例（EKS CI）是 Serverless Pod，不支持传统的 SSH 登录或 `kubectl exec`。诊断依赖两个 CLI 接口：`DescribeEKSContainerInstanceEvent`（事件）和 `DescribeEksContainerInstanceLog`（日志）。两种手段结合使用可覆盖绝大多数诊断场景。

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
# 4. 确认目标实例存在且处于 Running 状态
tccli tke DescribeEKSContainerInstances --region <Region> \
    --EksCiIds '["<EksCiId>"]'
# expected: TotalCount >= 1，Status: "Running"
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
| 查看实例详情 | `DescribeEKSContainerInstances --EksCiIds` | 是 |
| 查看标准输出日志 | `DescribeEksContainerInstanceLog --EksCiId --ContainerName` | 是 |
| 查看最近 N 行日志 | `DescribeEksContainerInstanceLog --Tail` | 是 |
| 查看控制台事件 | `DescribeEKSContainerInstanceEvent --EksCiId` | 是 |

## 操作步骤

### 步骤 1：确认实例在运行

```bash
tccli tke DescribeEKSContainerInstances --region <Region> \
    --EksCiIds '["<EksCiId>"]'
# expected: TotalCount: 1，Status: "Running"
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
            "PrivateIp": "10.0.x.x",
            "Containers": [
                {
                    "Name": "nginx",
                    "CurrentState": {
                        "State": "running",
                        "RestartCount": 0
                    }
                }
            ]
        }
    ],
    "RequestId": "..."
}
```

### 步骤 2：查看容器日志（等效登录方式）

EKS CI 不支持 SSH 登录，日志是主要诊断手段。通过 `DescribeEksContainerInstanceLog` 获取容器的 stdout/stderr 输出。

**查看最近日志**：

```bash
tccli tke DescribeEksContainerInstanceLog --region <Region> \
    --EksCiId <EksCiId> \
    --ContainerName nginx \
    --Tail 20
# expected: LogContent 不为空
```

**预期输出**：

```json
{
    "ContainerName": "nginx",
    "LogContent": "/docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration\n/docker-entrypoint.sh: Looking for shell scripts in /docker-entrypoint.d/\n/docker-entrypoint.sh: Configuration complete; ready for start up\nnginx: start worker processes\n"
}
```

### 步骤 3：查看实例事件

事件展示实例的底层调度和执行过程：

```bash
tccli tke DescribeEKSContainerInstanceEvent --region <Region> \
    --EksCiId <EksCiId>
# expected: Events 含生命周期事件（Scheduled → Pulling → Pulled → Created → Started）
```

**预期输出**：

```json
{
    "EksCiId": "eksci-example",
    "Events": [
        {"Reason": "Scheduled", "Message": "Successfully assigned default/eksci-example", "Type": "Normal"},
        {"Reason": "Pulling", "Message": "Pulling image nginx:alpine", "Type": "Normal"},
        {"Reason": "Pulled", "Message": "Successfully pulled image nginx:alpine", "Type": "Normal"},
        {"Reason": "Created", "Message": "Created container nginx", "Type": "Normal"},
        {"Reason": "Started", "Message": "Started container nginx", "Type": "Normal"}
    ]
}
```

### 诊断决策树

```
Status=Running?
  ├─ 是 → DescribeEksContainerInstanceLog（查日志）
  │       ├─ 日志非空 → 分析业务日志定位问题
  │       └─ 日志为空 → 应用可能无 stdout 输出，检查 CurrentState
  └─ 否 → DescribeEKSContainerInstanceEvent（查事件）
          ├─ FailedScheduling → 子网 IP 不足
          ├─ ErrImagePull     → 镜像/凭证错误
          └─ OOMKilled        → 内存不足
```

## 验证

### 控制面（tccli）

```bash
# 确认可获取到日志
tccli tke DescribeEksContainerInstanceLog --region <Region> \
    --EksCiId <EksCiId> \
    --ContainerName nginx
# expected: LogContent 字段存在且非空

# 确认可获取事件
tccli tke DescribeEKSContainerInstanceEvent --region <Region> \
    --EksCiId <EksCiId>
# expected: Events 数组非空
```

```json
{
  "ContainerName": "<ContainerName>",
  "LogContent": "<LogContent>",
  "RequestId": "<RequestId>"
}
```

### 数据面

EKS CI 为 Serverless 容器，无 SSH 或 kubectl exec 能力。数据面验证无需额外操作——日志和事件即数据面诊断入口。

## 清理

本页为读操作页面，无资源需清理。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DescribeEksContainerInstanceLog` 返回 `InvalidParameterValue.ContainerName` | `tccli tke DescribeEKSContainerInstances --region <Region> --EksCiIds '["<EksCiId>"]'` 查看 `Containers[].Name` | ContainerName 与创建时 `Containers[].Name` 不一致 | 用正确的容器名重新执行命令 |
| `DescribeEKSContainerInstanceLog` 返回 `InvalidParameterValue.EksCiId` | 检查 EksCiId 格式 | 实例 ID 格式错误或实例不存在 | `tccli tke DescribeEKSContainerInstances --region <Region>` 列出全部实例确认 ID |
| `DescribeEKSContainerInstanceLog` / `DescribeEKSContainerInstanceEvent` 返回 `UnauthorizedOperation.CamNoAuth` | `tccli configure list` 检查凭据 | 子账号缺少对应接口权限（此为环境限制，非命令错误） | 联系主账号授予 `QcloudTKEFullAccess` 或对应 Action 权限 |

### 操作成功但日志/事件异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 日志为空 | `tccli tke DescribeEKSContainerInstances --region <Region> --EksCiIds '["<EksCiId>"]'` 查看 `CurrentState` | Web 服务（如 nginx）无持续 stdout 输出，或容器已退出 | 使用 `--Tail` 增大行数或 `--SinceSeconds` 扩大时间范围；检查 `CurrentState.State` |
| 实例 `Failed` | `tccli tke DescribeEKSContainerInstanceEvent --region <Region> --EksCiId <EksCiId>` 查看退出原因 | 容器启动命令失败（`ExitCode` 非 0）或 OOM（`OOMKilled`） | 修正 Commands 参数，增加 Memory 配额 |
| 日志只有启动输出而无后续业务日志 | 正常现象 | 日志只采集 stdout/stderr，业务写文件日志不在采集范围 | 如需采集文件日志，配置 [开启日志采集](../开启日志采集/tccli%20操作.md) 到 CLS |
| 事件中 `FailedScheduling` | `tccli vpc DescribeSubnets --region <Region> --SubnetIds '["<SubnetId>"]'` 查看 AvailableIpCount | 子网可用 IP 用尽 | 清理无用实例释放 IP，或扩容子网 CIDR |
| 事件中 `ErrImagePull` | 检查镜像名拼写和仓库权限 | 镜像名拼写错误，或私有仓库未配置凭证 | 修正镜像名，或创建实例时传入 `ImageRegistryCredentials` |

## 下一步

- [查看日志及事件](../查看日志及事件/tccli%20操作.md) — 日志与事件联合诊断
- [开启日志采集](../开启日志采集/tccli%20操作.md) — 配置持久化日志采集到 CLS
- [容器实例生命周期](../容器实例生命周期/tccli%20操作.md) — 状态含义与事件解读
- [容器实例操作手册](../容器实例操作手册/tccli%20操作.md) — 完整 CRUD 操作

## 控制台替代

[TKE 控制台 → 弹性容器 → 容器实例 → 点击实例名](https://console.cloud.tencent.com/tke2/eks/ci)：日志 Tab / 事件 Tab。
