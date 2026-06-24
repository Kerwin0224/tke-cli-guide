# 容器实例概述

> 对照官方：[容器实例概述](https://cloud.tencent.com/document/product/457/57339) · page_id `57339`

## 概述

弹性容器实例（EKS Container Instance, EKS CI）是 TKE 提供的 Serverless 容器运行方式。无需管理底层节点，直接指定镜像、CPU、内存即可运行容器。按实际使用量计费，适用于批处理任务、CI/CD、定时任务、微服务等场景。

**核心特性**：

| 特性 | 说明 |
|------|------|
| Serverless | 无需购买和管理 CVM 节点，按 vCPU/内存 + 运行时长计费 |
| 安全隔离 | 每个 Pod 拥有独立的内核和网络命名空间 |
| 弹性伸缩 | 秒级启动，按需创建销毁 |
| VPC 集成 | 分配 VPC 内网 IP，支持安全组、EIP 等网络能力 |
| 最小规格 | 0.25 核 / 0.5 GB 内存 |

**API 入口**：`tccli tke` 的 `EKSContainerInstances` 系列接口，包括 `Create`、`Describe`、`Update`、`Restart`、`Delete`。

**生命周期**：

```
Creating → Running → Succeeded / Failed → Terminating
```

| 状态 | 含义 | 容器级 State |
|------|------|------------|
| `Creating` | 调度中、拉镜像、创建容器中 | `waiting` |
| `Running` | 所有容器已启动，正常运行 | `running` |
| `Succeeded` | 容器正常退出（exit 0），仅 `RestartPolicy=Never` | `terminated` (exitCode: 0) |
| `Failed` | 容器异常退出（exit 非 0 或 OOMKilled） | `terminated` (exitCode 非 0) |
| `Terminating` | 正在删除 | — |

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
#    tke:DescribeEKSContainerInstances
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

# 5. 查询已有容器实例（确认 API 可访问）
tccli tke DescribeEKSContainerInstances --region <Region> --Limit 1
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

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看实例列表 | `DescribeEKSContainerInstances` | 是 |
| 查看实例详情 | `DescribeEKSContainerInstances --EksCiIds` | 是 |
| 创建实例 | `CreateEKSContainerInstances` | 否 |
| 更新实例 | `UpdateEKSContainerInstance` | 否 |
| 重启实例 | `RestartEKSContainerInstances` | 否 |
| 删除实例 | `DeleteEKSContainerInstances` | 否 |
| 查看事件 | `DescribeEKSContainerInstanceEvent` | 是 |
| 查看日志 | `DescribeEksContainerInstanceLog` | 是 |

## 操作步骤

### 查看已有实例

```bash
tccli tke DescribeEKSContainerInstances --region <Region>
# expected: exit 0，返回 TotalCount 和 EksCis 列表
```

**预期输出**：

```json
{
    "TotalCount": 0,
    "EksCis": [],
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

### 网络模型

- 每个实例分配 VPC 内网 IP，支持安全组
- 可通过 `AutoCreateEip` 或 `ExistedEipIds` 绑定公网 IP
- 可通过 `CamRoleName` 绑定 CAM 角色，通过 metadata 服务获取临时凭证

### 计费说明

按 vCPU 和内存的实际使用量及运行时长计费，实例删除后停止计费。使用 `EipAddress` 时 EIP 可能产生额外费用。

## 验证

```bash
# 确认可访问 EKS CI API
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

## 清理

本页为概念说明页，无资源需清理。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DescribeEKSContainerInstances` 返回 `UnauthorizedOperation.CamNoAuth` | `tccli configure list` 检查当前凭据和 region | 子账号缺少 `tke:DescribeEKSContainerInstances` 权限（此为环境限制，非命令错误） | 联系主账号授予 `QcloudTKEFullAccess` 或最小权限 `tke:DescribeEKSContainerInstances` |
| `DescribeEKSContainerInstances` 返回空列表 | `tccli configure list` 确认 region 与创建实例所在地域一致 | region 与实例所在地域不匹配 | 用 `tccli configure set region <正确地域>` 切换至正确地域 |

### 操作成功但异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 实例 `Status` 长时间为 `Creating` | `tccli tke DescribeEKSContainerInstanceEvent --region <Region> --EksCiId <EksCiId>` 查看事件 | 子网 IP 不足或镜像拉取失败 | 检查 Reason 字段：`FailedScheduling` → 子网扩容；`ErrImagePull` → 修正镜像名或配置凭证 |
| 实例 `Status` 为 `Failed` | 同上，查看事件和 `Containers[].CurrentState.ExitCode` | 容器启动命令失败或 OOM | 检查 Commands 参数、增加资源配额 |

## 下一步

- [快速创建一个容器实例](../快速创建一个容器实例/tccli%20操作.md) — 创建第一个 EKS CI
- [容器实例操作手册](../容器实例操作手册/tccli%20操作.md) — 完整 CRUD 操作
- [容器实例生命周期](../容器实例生命周期/tccli%20操作.md) — 状态详解与事件诊断
- [登录实例](../登录实例/tccli%20操作.md) — 通过日志和事件诊断容器
- [查看日志及事件](../查看日志及事件/tccli%20操作.md) — 日志与事件联合诊断

## 控制台替代

[TKE 控制台 → 弹性容器 → 容器实例](https://console.cloud.tencent.com/tke2/eks/ci)：查看实例列表、创建实例。
