# 授权腾讯云售后运维排障（tccli）

> 对照官方：[授权腾讯云售后运维排障](https://cloud.tencent.com/document/product/457/61033) · page_id `61033`

## 概述

当需要腾讯云售后协助排查 TKE 集群问题时，通过控制台授权售后工程师访问集群。授权前通过 tccli 确认集群基本信息，授权后售后可查看集群日志和事件，问题解决后及时撤销。

> **kubectl 数据面不可达**：CAM拒绝公网端点(strategyId:240463971)，内网需VPN/IOA。售后获得授权后可通过内部通道访问集群数据面。

## 前置条件

- [环境准备](../../环境准备.md)
- 具备集群管理权限（主账号或具有 TKE 管理权限的子账号）

### 环境检查

```bash
tccli --version
# 预期输出: tccli version X.X.X
```

### 资源检查

```bash
tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]'
```

**预期输出**：

```json
{
  "TotalCount": 1,
  "Clusters": [
    {
      "ClusterId": "cls-xxxxxxxx",
      "ClusterName": "my-test-cluster",
      "ClusterStatus": "Running",
      "ClusterVersion": "1.30.0",
      "ClusterType": "MANAGED_CLUSTER",
      "ClusterDescription": "",
      "DeletionProtection": true
    }
  ]
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli / CLI | 幂等 |
|---|---|---|
| 查看集群属性 | `tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]'` | 是 |
| 查看集群状态 | `tccli tke DescribeClusterStatus --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]'` | 是 |
| 授权售后 | 控制台操作（无对应 tccli API） | 否 |
| 撤销授权 | 控制台操作（无对应 tccli API） | 否 |

## 操作步骤

### 步骤1：授权前确认集群信息

```bash
tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]'
```

**预期输出**：

```json
{
  "Clusters": [
    {
      "ClusterId": "cls-xxxxxxxx",
      "ClusterStatus": "Running",
      "ClusterVersion": "1.30.0",
      "ClusterType": "MANAGED_CLUSTER",
      "DeletionProtection": true
    }
  ]
}
```

确认集群 ID、状态和基本信息，供售后工单引用。

### 步骤2：控制台授权售后运维

1. 登录 [容器服务控制台](https://console.cloud.tencent.com/tke2/cluster)
2. 进入目标集群 -> 基本信息页
3. 找到"运维授权"模块
4. 点击"开启"授权售后工程师访问
5. 记录授权时间，设置授权时长（建议按需开启，默认 24 小时）

### 步骤3：授权前准备清单

- 确认集群 ID：`cls-xxxxxxxx`
- 确认地域：`ap-guangzhou`
- 明确授权范围：仅授权必要的集群，不授权整个账号
- 最小化授权时间：按需开启临时权限，问题解决后及时撤销
- 敏感资源标记：确保生产集群通过 Tag 标记（如 `env: production`）
- 记录当前集群状态：`tccli tke DescribeClusterStatus --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]'`

### 步骤4：提供售后的补充信息

授权的售后工程师可通过内部通道访问集群，但建议同时提供以下信息以加速排障：

- 集群 ID：`cls-xxxxxxxx`
- 问题发生时间点和持续时间
- 异常现象描述：Pod 状态、Service 不可达等
- 已尝试的排查步骤和结果
- 相关 tccli 命令和输出：
  ```bash
  tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]'
  tccli tke DescribeClusterInstances --region ap-guangzhou --ClusterId cls-xxxxxxxx
  ```

```json
{
  "TotalCount": 0,
  "Clusters": [],
  "ClusterId": "<ClusterId>",
  "ClusterName": "<ClusterName>",
  "ClusterDescription": "<ClusterDescription>",
  "ClusterVersion": "<ClusterVersion>"

```json
{
  "TotalCount": 0,
  "Clusters": [],
  "ClusterId": "<ClusterId>",
  "ClusterName": "<ClusterName>",
  "ClusterDescription": "<ClusterDescription>",
  "ClusterVersion": "<ClusterVersion>",
  "ClusterOs": "<ClusterOs>",
  "ClusterType": "<ClusterType>"
}
```,
  "ClusterOs": "<ClusterOs>",
  "ClusterType": "<ClusterType>"
}
```

### 步骤5：问题解决后撤销授权

问题解决后立即在控制台撤销授权：
1. 进入集群 -> 基本信息
2. 运维授权 -> 点击"关闭"
3. 确认授权已关闭

## 验证

### 控制面（tccli）

```bash
tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]'
```

**预期输出**：

```json
{
  "Clusters": [
    {
      "ClusterId": "cls-xxxxxxxx",
      "ClusterStatus": "Running",
      "DeletionProtection": true
    }
  ]
}
```

授权后售后可访问集群日志和事件。验证售后工单反馈排障结果。

## 清理

问题解决后在控制台撤销授权。无需其他清理操作。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|---|---|---|---|
| 控制台无"运维授权"选项 | 检查当前登录账号权限 | 子账号缺少集群管理权限 | 使用主账号或具有 TKE 管理员权限的子账号操作 |
| `DescribeClusters` 返回 `UnauthorizedOperation` | 检查 CAM 策略 | 当前账号缺少 TKE 集群查看权限 | 添加 `tke:DescribeClusters` 权限或 `QcloudTKEReadOnlyAccess` |

### 操作成功但异常

| 现象 | 诊断 | 根因 | 修复 |
|---|---|---|---|
| 售后反馈无法访问集群 | 确认控制台授权状态 | 授权可能仅对特定集群生效，售后可能访问了错误集群 | 确认提供给售后的集群 ID 正确 |
| 授权过期后售后仍需访问 | 检查授权时长设置 | 默认授权时长不足 | 在控制台延长授权时间或重新开启 |

## 下一步

- [集群 API Server 网络无法访问排障处理](../集群%20API%20Server%20网络无法访问排障处理/tccli%20操作.md) -- page_id `80912`
- [使用 TKE 审计和事件服务快速排查问题](../../实践教程/运维/使用%20TKE%20审计和事件服务快速排查问题/tccli%20操作.md)

## 控制台替代

[容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) -> 集群 -> 基本信息 -> 运维授权 -> 开启 / 关闭。
