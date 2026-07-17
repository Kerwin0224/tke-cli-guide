---
doc_type: Reference
subtype: 8B
---
# TCR 实例状态机

> 企业版实例的状态机。状态值来自 `DescribeInstanceStatus` 响应的 `Status` 字段，以官方文档为准。个人版无状态机（共享服务，无独立实例生命周期）。规格与容量边界见 [产品服务层级与容量限制](https://cloud.tencent.com/document/product/1141/104731)。

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

> `Status` 是实例整体状态，`Conditions[]` 是过程明细（`Type`/`Status`/`Reason`）。契约未保证数组非空、元素顺序或首项与顶层状态一致；诊断时应遍历现有元素，并结合每项字段判断，不要固定读取 `Conditions[0]`。

## 企业版实例状态 (Status)

> `Status` 字段取自 `DescribeInstanceStatus` / `DescribeInstances` 响应。官方 `Registry.Status` 枚举如下（**无** `Creating` / `Isolated` / `Abnormal` 字面值；创建过渡态为 `Pending`→`Deploying`）。

| 状态 | 含义 | 触发条件 | 用户可执行操作 | 当前流程是否停止自动推进 |
|:-----|:-----|:---------|:--------------|:------------------------:|
| `Pending` | 初始化中 | `CreateInstance` 刚返回 | 等待 | 否 |
| `Deploying` | 创建中（部署） | 初始化后进入部署 | 等待（约 3–5 分钟） | 否 |
| `Running` | 运行中 | 创建完成 | 全部操作（推送 / 拉取 / 配置访问 / 生命周期） | 是（创建流程成功） |
| `Unhealthy` | 状态异常 | 后端健康检查失败 | 查 `Conditions[].Reason` / 提工单 | 不确定 |
| `FailedCreated` | 创建失败 | 创建流程失败 | 删实例重试或提工单 | 是（本次创建失败） |
| `FailedUpdated` | 更新失败 | 变配/更新失败 | 按 `Reason` 处理或提工单 | 是（本次更新失败） |
| `Bucket-Error` | 存储桶异常 | 底层 COS 异常 | 提工单 | 不确定 |
| `Isolate` | 待回收（欠费等） | 欠费/隔离策略 | 续费恢复（以控制台/账单为准） | 不确定 |
| `Deleting` | 删除中 | `DeleteInstance` | 等待；删除完成后实例从查询结果消失 | 否 |
| `DeleteBucketFailed` | 删实例时存储桶失败 | 删除流程 | 提工单后重试或恢复 | 是（本次删除失败） |
| `DeleteFailed` | 实例删除失败 | 删除流程 | 提工单后重试或恢复 | 是（本次删除失败） |

> “停止自动推进”只描述当前异步操作是否已经结束，不表示资源生命周期不可恢复。失败状态仍可能在修复原因后重试或由人工恢复。

> 常驻工作状态：`Running`。企业版实例创建是异步操作，`CreateInstance` 返回 `RegistryId` 后须轮询 `DescribeInstanceStatus` 直到 `Running`（过渡态是 `Pending`/`Deploying`，**不是** `Creating`）。个人版无此状态机——个人版是共享服务，无独立实例生命周期，直接用命名空间/仓库操作。

## Conditions 字段

`DescribeInstanceStatus` 响应的 `Conditions` 数组给出实例的过程信息：

| 字段 | 含义 | 示例 |
|:-----|:-----|:-----|
| `Type` | 条件类型 | 以实际返回为准 |
| `Status` | 条件状态 | 如 `Running` |
| `Reason` | 原因说明 | 异常时可提供诊断线索 |

> `Conditions` 可能为空，且契约未保证顺序。存在元素时遍历 `Conditions[]`：优先查看非空 `Reason`，并结合该元素的 `Type`/`Status` 与顶层 `Status` 诊断；不要假定稳定态 `Reason` 必为空或首元素必与顶层一致。

## 相关文档

- [创建实例](../instances/create.md) — 触发 `Pending` → `Deploying` → `Running`
- [访问管理](../instances/manage-access.md) — `Running` 后配置访问凭证
- [推送拉取镜像](../images/push-pull.md) — `Running` 后 docker login
- [故障排查](../troubleshooting.md) — 状态异常的诊断路径
- [错误码](error-codes.md) — 状态查询失败时的错误码
