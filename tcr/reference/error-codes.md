---
doc_type: Reference
subtype: 8D
---
# TCR 错误码

> TCR 涉及 tccli 与 docker 两个工具，错误码分三层：docker CLI 错误（推送/拉取高频）→ TCR 特有错误码 → 通用错误码。错误码按 API/docker 返回原样写出。

## docker CLI 错误（推送 / 拉取高频）

> docker login / push / pull 的失败信号。这些不是 tccli 错误码，是 docker 侧的文本消息，按原样识别。

| 消息 | 含义 | 诊断 | 修复 |
|:-----|:-----|:-----|:-----|
| `unauthorized: authentication required` | Token 过期或未登录 | `tccli tcr DescribeInstanceToken` 查 Token 状态 | 重新 `CreateInstanceToken` 并 `docker login` |
| `denied: requested access to the resource is denied` | 无 push 权限（命名空间/仓库策略） | `tccli tcr DescribeNamespaces` 查权限 | 授予命名空间访问权限，或换有权限的账号 |
| `unknown: repository not found` | 仓库不存在或无访问权 | `tccli tcr DescribeRepositories` 查仓库 | 先 `CreateRepository`，或确认仓库可见性 |
| `manifest invalid` / `blob upload invalid` | manifest 损坏或镜像格式错 | 重新构建镜像 | `docker build` 重建后重推 |
| `pull access denied` | 拉取无权限（私有仓库未认证） | 确认 `docker login` 已执行 | 先 `docker login <endpoint>` |

## TCR 特有错误码

| 错误码 | 含义 | 可重试 | 诊断 | 修复 |
|------|---------|:---------:|----------|-----|
| `InvalidParameter.InstanceName` | 实例名不合法或已存在 | 否 | `tccli tcr CheckInstanceName` 检查名称 | 名称全局唯一，换名重试 |
| `LimitExceeded.Instance` | 实例数超配额（默认 10/地域） | 否 | `tccli tcr DescribeInstances` 看存量 | 删除闲置实例或提工单提额 |
| `LimitExceeded.Namespace` | 命名空间超配额 | 否 | `tccli tcr DescribeNamespaces` 看存量 | 删除闲置命名空间（basic=50/standard=100/premium=500） |
| `LimitExceeded.Repository` | 仓库数超配额 | 否 | `tccli tcr DescribeRepositories` 看存量 | 删除闲置仓库（basic=1000/standard=3000/premium=5000） |
| `ResourceNotFound` | 实例 / 命名空间 / 仓库不存在 | 否 | `tccli tcr DescribeInstances` 核对 ID/Region | 确认 ID 与地域一致 |
| `FailedOperation` | 操作失败（实例非 Running 等） | 否 | `tccli tcr DescribeInstanceStatus` 看状态 | 等实例 `Running` 后重试 |
| `UnauthorizedOperation` | CAM 权限不足 | 否 | 查 CAM 策略 | 授予 `tcr:<Action>` 权限 |

## 通用错误码

| 错误码 | 含义 | 可重试 | 诊断 | 修复 |
|------|---------|:---------:|----------|-----|
| `AuthFailure.SecretIdNotFound` | 凭证无效或已过期 | 否 | `tccli tcr DescribeRegions` 验证凭证 | 见 [配置凭证](../../getting-started/credentials.md) 重新配置 |
| `InvalidParameterValue` | 参数值不合法 | 否 | 查 `--generate-cli-skeleton` 字段约束 | 检查参数格式和取值范围 |
| `RequestLimitExceeded` | API 限频 | yes (退避) | 观察请求频率 | 等待后重试；`DescribeInstanceStatus` 限频 20/s |
| `InternalError` | 内部错误 | yes (退避) | 收集 RequestId 重试 | 重试 2-3 次，仍失败提工单 |

## 诊断命令

```bash
# 实例状态 (确认 Running 才能 push/pull)
tccli tcr DescribeInstanceStatus --region <REGION> --RegistryIds '["<REGISTRY_ID>"]'
# expected: RegistryStatusSet[0].Status = "Running"

# Token 状态 (docker login 失败时)
tccli tcr DescribeInstanceToken --region <REGION> --RegistryId "<REGISTRY_ID>"
# expected: Token 信息返回（未过期）

# 公网访问状态 (docker login 超时时)
tccli tcr DescribeExternalEndpointStatus --region <REGION> --RegistryId "<REGISTRY_ID>"
# expected: Status = "Opened"
```

> 错误响应结构：`{"Response":{"Error":{"Code":"...","Message":"..."},"RequestId":"..."}}`。docker 侧错误不是 JSON，是 stderr 文本——用 `--language en-US` 锁定 tccli 错误语言，docker 错误天然英文。如遇未列出的错误码，查 [腾讯云 TCR 错误码文档](https://cloud.tencent.com/document/product/1141)。

## 相关文档

- [实例状态机](states.md) — `FailedOperation` 常因实例非 Running
- [推送拉取镜像](../images/push-pull.md) — docker CLI 错误的完整诊断链路
- [访问控制](../access/manage.md) — `unauthorized` / `denied` 的权限配置
- [故障排查](../troubleshooting.md) — docker login / push 失败的诊断路径
