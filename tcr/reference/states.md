---
doc_type: Reference
subtype: 8B
---
# TCR 实例状态机

> 企业版实例的状态机。状态值来自 `DescribeInstanceStatus` 响应的 `Status` 字段，以实测 + 官方文档为准。个人版无状态机（共享服务，无独立实例生命周期）。

## 查询命令

```bash
# 企业版实例状态 (Status + Conditions)
tccli tcr DescribeInstanceStatus --region <REGION> --RegistryIds '["<REGISTRY_ID>"]'
# expected: RegistryStatusSet[0].Status = "Running"
```

```json
{
    "RegistryStatusSet": [
        {
            "RegistryId": "tcr-example",
            "Status": "Running",
            "Conditions": [
                {
                    "Type": "",
                    "Status": "Running",
                    "Reason": ""
                }
            ]
        }
    ],
    "RequestId": "xxx"
}
```

> 实测：`Status` 是实例整体状态，`Conditions` 是过程明细（`Type`/`Status`/`Reason`）。稳定态下 `Conditions[0].Status` 与顶层 `Status` 一致；过渡态下 `Conditions` 反映正在进行的子流程。

## 企业版实例状态 (Status)

> 来源：`DescribeInstanceStatus` 响应。实测值 `Running`；枚举来自腾讯云 TCR 官方文档。

| 状态 | 含义 | 触发条件 | 用户可执行操作 | 终态 |
|:-----|:-----|:---------|:--------------|:----:|
| `Creating` | 实例创建中 | `CreateInstance` | 等待（通常 3-5 分钟） | 否 |
| `Running` | 实例正常运行 | 创建完成 | 全部操作（推送 / 拉取 / 配置访问 / 生命周期） | 否 |
| `Deleting` | 实例删除中 | `DeleteInstance` | 等待 | 是 |
| `Isolated` | 实例已隔离 | 欠费隔离 | 续费恢复 | 否 |
| `Abnormal` | 实例异常 | 后端故障 | 提工单 | 否 |

> 常驻工作状态：`Running`。企业版实例创建是异步操作，`CreateInstance` 返回 `RegistryId` 后须轮询 `DescribeInstanceStatus` 直到 `Running`。个人版无此状态机——个人版是共享服务，无独立实例生命周期，直接用命名空间/仓库操作。

## Conditions 字段

`DescribeInstanceStatus` 响应的 `Conditions` 数组给出实例的过程信息：

| 字段 | 含义 | 示例 |
|:-----|:-----|:-----|
| `Type` | 条件类型（过渡态下非空） | `""`（稳定态） |
| `Status` | 条件状态 | `Running` |
| `Reason` | 原因（异常态下非空） | `""`（正常） |

> 稳定态下 `Reason` 为空；若 `Status` 非 `Running` 且 `Reason` 非空，按 `Reason` 文本诊断。

## 相关文档

- [创建实例](../instances/create.md) — 触发 `Creating → Running`
- [访问管理](../instances/manage-access.md) — `Running` 后配置访问凭证
- [推送拉取镜像](../images/push-pull.md) — `Running` 后 docker login
- [故障排查](../troubleshooting.md) — 状态异常的诊断路径
- [错误码](error-codes.md) — 状态查询失败时的错误码
