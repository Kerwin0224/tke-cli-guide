---
doc_type: Reference
subtype: 8D
---
# TKE 错误码

> 错误码按 API 返回原样写出。双版本相关错误码优先列出，TKE 特有错误码次之，通用错误码垫底。退出码仅三级（0 成功 / 252 参数解析错误 / 255 其他），定位问题须解析响应 JSON 的 `Response.Error.Code`。

## 双版本相关错误码

| 错误码 | 含义 | 可重试 | 诊断 | 修复 |
|------|---------|:---------:|----------|-----|
| `UnknownParameter` | 跨版本参数不兼容（如旧版 `InstanceIds` 用到新版 `DescribeClusterInstances`） | 否 | `tccli tke <Action> --version <VERSION> --generate-cli-skeleton` 逐字段核契约 | 改用目标版本的参数名（旧 `InstanceIds`/`InstanceRole` vs 新 `SortBy`/`NeedTags`） |
| `UnknownAction` | 新版 Action 不带 `--version` 误走旧版，旧版无此 Action | 否 | `tccli tke help --version 2022-05-01 \| grep <Action>` 确认 Action 归属版本 | 显式带 `--version 2022-05-01` |

## TKE 特有错误码

> 来源：腾讯云 TKE API 错误码文档。`InvalidParameter.Param` / `ResourceNotFound` 为常见错误码。

| 错误码 | 含义 | 可重试 | 诊断 | 修复 |
|------|---------|:---------:|----------|-----|
| `InvalidParameter.Param` | 参数错误（含资源不存在） | 否 | 核对参数值与资源 ID | 确认资源 ID 存在、参数格式正确；删不存在的集群返回 `[E404000 ResourceNotFound] record not found` |
| `InternalError.QuotaMaxClsLimit` | 集群数超配额 | 否 | `tccli tke DescribeClusters` 看集群存量 | 删除闲置集群或提工单提额（默认 50/地域） |
| `InternalError.QuotaMaxNodLimit` | 节点数超配额 | 否 | `tccli tke DescribeClusterStatus` 看节点数 | 删除闲置节点或提工单提额 |
| `LimitExceeded` | 超过配额限制（节点池等） | 否 | `tccli tke DescribeClusterNodePools` 看节点池数 | 删除闲置节点池（单集群建议 ≤20） |
| `InvalidParameter.CidrConflictWithOtherCluster` | 容器 CIDR 与同 VPC 其他集群冲突 | 否 | `tccli tke DescribeClusters --ClusterIds '["<ID>"]' --filter "Clusters[].ClusterCIDRSettings.ClusterCIDR"` 看已用 CIDR | 换不重叠的 CIDR 段重建（不得与 VPC CIDR 或同 VPC 其他集群 CIDR 重叠，消息含 `CIDR_CONFLICT_WITH_OTHER_CLUSTER[cidr X is conflict with cluster id: Y]`） |
| `FailedOperation.QuotaExceeded` | 操作超配额 | 否 | 查对应资源存量 | 提工单提额 |
| `FailedOperation.UnexpectedError` | 不可预知的错误 | 否 | 收集 RequestId | 提工单附 RequestId |
| `InternalError.CamNoAuth` | CAM 权限不足 | 否 | 查 CAM 策略 | 授予 `tke:<Action>` 权限 |
| `UnauthorizedOperation.CamNoAuth` | 无该接口 CAM 权限 | 否 | 查 CAM 策略 | 授予对应 Action 权限 |
| `UnsupportedOperation` | 操作不支持（如集群状态不允许） | 否 | `tccli tke DescribeClusterStatus` 看集群状态 | 等集群 `Running` 后重试 |

## 通用错误码

| 错误码 | 含义 | 可重试 | 诊断 | 修复 |
|------|---------|:---------:|----------|-----|
| `AuthFailure.SecretIdNotFound` | 凭证无效或已过期 | 否 | `tccli tke DescribeRegions` 验证凭证 | 见 [配置凭证](../../getting-started/credentials.md) 重新配置 |
| `InvalidParameterValue` | 参数值不合法 | 否 | 查 `--generate-cli-skeleton` 的字段约束 | 检查参数格式和取值范围 |
| `ResourceNotFound` | 资源不存在 | 否 | `tccli tke DescribeClusters` 核对 ID/Region | 确认 ID 格式与地域一致 |
| `RequestLimitExceeded` | API 限频 | yes (退避) | 观察请求频率 | 等待后重试；TKE 多数接口 20/s，`DeleteCluster` 50/s，`DescribeClusterSecurity` 100/s |
| `InternalError` | 内部错误 | yes (退避) | 收集 RequestId 重试 | 重试 2-3 次，仍失败提工单 |

## 诊断命令

```bash
# 验证凭证
tccli cvm DescribeRegions
# expected: RegionSet 列表返回

# 核对集群 ID 与地域
tccli tke DescribeClusters --region <REGION> --ClusterIds '["<CLUSTER_ID>"]'
# expected: TotalCount ≥ 1

# 核对 Action 参数契约 (双版本切换前必做)
tccli tke <Action> --version <VERSION> --generate-cli-skeleton
# expected: 入参骨架 JSON
```

> 错误响应结构：`{"Response":{"Error":{"Code":"...","Message":"..."},"RequestId":"..."}}`。用 `--language en-US` 锁定英文错误消息，便于脚本匹配。如遇未列出的错误码，查 [腾讯云 TKE 错误码文档](https://cloud.tencent.com/document/product/457)。`ResourceNotFound` 删不存在的集群时消息为 `[E404000 ResourceNotFound] record not found`。

## 相关文档

- [状态机](states.md) — `UnsupportedOperation` 常因集群状态不允许
- [配额和限制](quotas.md) — `LimitExceeded` / `QuotaMaxClsLimit` 的配额阈值
- [故障排查](../troubleshooting.md) — 错误码对应的诊断路径
