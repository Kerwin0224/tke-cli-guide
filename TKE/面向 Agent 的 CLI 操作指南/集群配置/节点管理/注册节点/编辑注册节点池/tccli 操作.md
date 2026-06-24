# 编辑注册节点池

> 对照官方：[编辑注册节点池](https://cloud.tencent.com/document/product/457/79765) · page_id `79765` · tccli ≥3.1.107 · API 2018-05-25

## 概述

对已创建的注册节点池进行配置修改，支持更新节点池名称、标签（Labels）、污点（Taints）、删除保护开关（DeletionProtection）及自定义脚本（UserScript）。

**API 发现**：注册节点池使用**专用 API 体系**（`CreateExternalNodePool` / `DescribeExternalNodePools` / `ModifyExternalNodePool` / `DeleteExternalNodePool`），不能使用通用的节点池 API（`ModifyClusterNodePool` / `DescribeClusterNodePools` / `DescribeClusterNodePoolDetail`）。通用 API 对注册节点池返回 `FailedOperation.RecordNotFound` 或 `FailedOperation.NodePoolQueryFailed`，因为注册节点池不在通用节点池数据库中。

| 修改项 | 使用的 API | 不适用的通用 API | 通用 API 错误 |
|--------|-----------|----------------|--------------|
| 查询注册节点池 | `DescribeExternalNodePools` | `DescribeClusterNodePools` / `DescribeClusterNodePoolDetail` | `FailedOperation.NodePoolQueryFailed` |
| 修改注册节点池 | `ModifyExternalNodePool` | `ModifyClusterNodePool` | `FailedOperation.RecordNotFound` |
| 删除注册节点池 | `DeleteExternalNodePool` | `DeleteClusterNodePool` | `FailedOperation.RecordNotFound` |

修改标签和污点**仅影响后续加入节点池的新节点**，已在池中的现有节点不会自动同步这些变更。

> 本文档中所有 ID（集群 ID、节点池 ID 等）均为占位符，请替换为实际值。

## 前置条件

- [环境准备](../../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 3.1.107

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置，region 为目标地域（如 ap-guangzhou）
```

### 资源检查

```bash
# 3. 确认目标集群存在且状态正常（同时获取 ClusterId）
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["<ClusterId>"]'
# expected: TotalCount >= 1，ClusterStatus 为 Running，记录返回的 ClusterId
```

**预期输出**：

```json
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "cls-example",
            "ClusterName": "my-cluster",
            "ClusterStatus": "Running",
            "ClusterVersion": "1.32.2"
        }
    ]
}
```

```bash
# 4. 确认目标注册节点池存在
tccli tke DescribeExternalNodePools --region <Region> \
    --ClusterId <ClusterId>
# expected: 返回节点池列表，找到目标 NodePoolId，LifeState 为 normal
```

**预期输出**：

```json
{
    "TotalCount": 2,
    "NodePoolSet": [
        {
            "NodePoolId": "np-example-001",
            "Name": "my-external-pool-001",
            "LifeState": "normal",
            "RuntimeConfig": {
                "RuntimeType": "containerd",
                "RuntimeVersion": "1.7.28"
            },
            "Labels": [
                {"Name": "test-label", "Value": "original"},
                {"Name": "tke.cloud.tencent.com/nodepool-id", "Value": "np-example-001"},
                {"Name": "node.kubernetes.io/instance-type", "Value": "external"},
                {"Name": "tke.cloud.tencent.com/cbs-mountable", "Value": "false"}
            ],
            "Taints": [
                {"Key": "test-taint", "Value": "original", "Effect": "NoSchedule"}
            ],
            "DeletionProtection": false,
            "NodeType": "CPU"
        },
        {
            "NodePoolId": "np-example-002",
            "Name": "my-external-pool-002",
            "LifeState": "normal",
            "RuntimeConfig": {
                "RuntimeType": "containerd",
                "RuntimeVersion": "1.6.9"
            },
            "Labels": [],
            "Taints": [],
            "DeletionProtection": false,
            "NodeType": "CPU"
        }
    ],
    "RequestId": "..."
}
```

### CAM 权限检查

```bash
# 5. 检查 CAM 权限 — 需要以下 Action 名
#    tke:DescribeClusters
#    tke:DescribeExternalNodePools
#    tke:ModifyExternalNodePool
#    tke:DeleteExternalNodePool
# 验证：上一步 DescribeExternalNodePools 已执行（步骤 4），若 exit 0 则权限 OK
# 若非 0 且错误码为 UnauthorizedOperation / AuthFailure → CAM 权限不足，需授权
```

> **注意**：`Labels` 中的 `tke.cloud.tencent.com/nodepool-id`、`node.kubernetes.io/instance-type`、`tke.cloud.tencent.com/cbs-mountable` 为系统自动注入的标签，修改时**无需手动传入**，后端会自动保留。只需管理自定义标签（如 `test-label`）。

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看注册节点池列表 | `DescribeExternalNodePools` | 是 |
| 修改节点池名称 | `ModifyExternalNodePool --Name` | 是 |
| 修改节点池标签（覆盖写入） | `ModifyExternalNodePool --Labels` | 是（全量替换） |
| 修改节点池污点（覆盖写入） | `ModifyExternalNodePool --Taints` | 是（全量替换） |
| 启用/关闭删除保护 | `ModifyExternalNodePool --DeletionProtection` | 是 |
| 修改自定义脚本（base64） | `ModifyExternalNodePool --UserScript` | 是 |
| 删除注册节点池 | `DeleteExternalNodePool` | 是 |

### 关键字段说明

`ModifyExternalNodePool` 的可修改字段。完整参数定义见 `tccli tke ModifyExternalNodePool --generate-cli-skeleton`。

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `ClusterId` | String | 是 | 集群 ID，格式 `cls-xxxxxxxx` | 集群不存在 → `InvalidParameter.ClusterId` |
| `NodePoolId` | String | 是 | 注册节点池 ID，格式 `np-xxxxxxxx` | 节点池不存在 → `FailedOperation.RecordNotFound` |
| `Name` | String | 否 | 新节点池名称，1-60 字符 | 命名违规 → `InvalidParameter.NodePoolName` |
| `Labels` | Array | 否 | Label 对象数组 `[{"Name":"key","Value":"val"}]`。**覆盖写入**：传入新数组替换全部自定义标签；传 `[]` 清空所有自定义标签；不传表示不修改。系统标签自动保留 | 格式错误 → `InvalidParameter` |
| `Taints` | Array | 否 | Taint 对象数组 `[{"Key":"k","Value":"v","Effect":"NoSchedule"}]`。**覆盖写入**：传入新数组替换全部污点；传 `[]` 清空所有污点；不传表示不修改 | Effect 不合法 → `InvalidParameter.TaintEffect` |
| `DeletionProtection` | Boolean | 否 | `true` 或 `false`。开启后需先关闭才能删节点池 | 忘关删除保护 → `DeleteExternalNodePool` 返回 `OperationDenied` |
| `UserScript` | String | 否 | Base64 编码的自定义脚本，新节点注册后执行 | 未 base64 编码 → 脚本内容被截断；仅传 UserScript 不传其他字段 → `InvalidParameter.Param: PARAM_ERROR(nothing is updated)` |

> **关键语义**：`Labels` 和 `Taints` 都是**覆盖写入**（非增量 PATCH）。不传表示不修改；传入新数组替换全部；传入空数组 `[]` 清空全部。

## 操作步骤

### 步骤 1：查询当前节点池配置

修改前必须先获取当前配置，尤其是标签和污点——因为这两个字段是覆盖写入，需要拿到当前值才能正确合并。

```bash
tccli tke DescribeExternalNodePools --region <Region> \
    --ClusterId <ClusterId>
# expected: exit 0，返回所有注册节点池信息
```

**预期输出**：

```json
{
    "TotalCount": 2,
    "NodePoolSet": [
        {
            "NodePoolId": "np-example-001",
            "Name": "my-external-pool-001",
            "LifeState": "normal",
            "RuntimeConfig": {
                "RuntimeType": "containerd",
                "RuntimeVersion": "1.7.28"
            },
            "Labels": [
                {"Name": "test-label", "Value": "original"},
                {"Name": "tke.cloud.tencent.com/nodepool-id", "Value": "np-example-001"},
                {"Name": "node.kubernetes.io/instance-type", "Value": "external"},
                {"Name": "tke.cloud.tencent.com/cbs-mountable", "Value": "false"}
            ],
            "Taints": [
                {"Key": "test-taint", "Value": "original", "Effect": "NoSchedule"}
            ],
            "DeletionProtection": false,
            "NodeType": "CPU"
        }
    ],
    "RequestId": "..."
}
```

> 记录目标节点池的 `NodePoolId`、当前 `Labels`（自定义部分）、当前 `Taints`，后续修改步骤需要用到。

### 步骤 2：修改节点池名称

#### 选择依据

- **使用专用 API**：选择 `ModifyExternalNodePool` 而非通用的 `ModifyClusterNodePool`。注册节点池有独立的 CRUD API 体系，通用 API `ModifyClusterNodePool` 对注册节点池返回 `FailedOperation.RecordNotFound`，因为注册节点池不在通用节点池数据库中。
- **注册节点池编辑参数精简**：`ModifyExternalNodePool` 无 `MaxNodesNum` / `MinNodesNum` / `EnableAutoscale` / `OsName` 等普通节点池参数，仅支持 Name / Labels / Taints / DeletionProtection / UserScript。

```bash
tccli tke ModifyExternalNodePool --region <Region> \
    --ClusterId <ClusterId> \
    --NodePoolId <NodePoolId> \
    --Name <NewName>
# expected: exit 0，返回 RequestId
```

**预期输出**：

```json
{
    "RequestId": "..."
}
```

### 步骤 3：修改标签（覆盖写入）

#### 选择依据

- **Labels 是覆盖写入**：`ModifyExternalNodePool` 的 `Labels` 参数不是增量追加，而是用新值完全替换全部自定义标签。传入新数组会替换所有自定义标签；传入空数组 `[]` 清除所有自定义标签；不传表示不修改。
- **系统标签自动保留**：`tke.cloud.tencent.com/nodepool-id`、`node.kubernetes.io/instance-type` 等系统标签无需手动传入，后端自动保留。只需管理自定义标签。
- **修改前先获取当前标签**：因为覆盖写入，修改前先执行步骤 1 的 `DescribeExternalNodePools` 获取当前自定义标签，拼接后一并传入。

```bash
tccli tke ModifyExternalNodePool --region <Region> \
    --ClusterId <ClusterId> \
    --NodePoolId <NodePoolId> \
    --Labels '[{"Name":"app","Value":"demo"},{"Name":"team","Value":"backend"}]'
# expected: exit 0，返回 RequestId
```

**预期输出**：

```json
{
    "RequestId": "..."
}
```

> 此操作会将节点池的自定义标签替换为 `app=demo` 和 `team=backend`。原有的 `test-label=original` 等自定义标签会被覆盖丢失。系统标签不受影响。

### 步骤 4：修改污点（覆盖写入）

#### 选择依据

- **Taints 同样是覆盖写入**：不传表示不修改；传入新数组替换全部污点；传入空数组 `[]` 清空所有污点。
- **常见误区**：以为不传 `Taints` 会清空污点——实际上不传表示不修改，需显式传 `[]` 才能清空。

```bash
tccli tke ModifyExternalNodePool --region <Region> \
    --ClusterId <ClusterId> \
    --NodePoolId <NodePoolId> \
    --Taints '[{"Key":"dedicated","Value":"special","Effect":"NoSchedule"}]'
# expected: exit 0，返回 RequestId
```

**预期输出**：

```json
{
    "RequestId": "..."
}
```

### 步骤 5：清空标签（空数组用法）

#### 选择依据

- **显式传 `[]` 才能清空**：要清除所有自定义标签，必须显式传入空数组 `[]`。不传 `Labels` 参数只表示"不修改"，不会清空。
- **同时修改名称**：覆盖写入清空标签时，可同时传入 `--Name` 等其他字段，确保请求体包含有效变更。

```bash
cat > clear-labels.json <<'EOF'
{
  "ClusterId": "<ClusterId>",
  "NodePoolId": "<NodePoolId>",
  "Labels": [],
  "Name": "<NewName>"
}
EOF
tccli tke ModifyExternalNodePool --region <Region> \
    --cli-input-json file://clear-labels.json
# expected: exit 0，返回 RequestId
```

**预期输出**：

```json
{
    "RequestId": "..."
}
```

> 此操作清除所有自定义标签，仅保留系统标签。同样，要清空污点需传 `--Taints '[]'`。

### 步骤 6：启用删除保护

#### 选择依据

- **生产环境推荐开启**：`DeletionProtection=true` 防止误删节点池。删除节点池前需先设为 `false`。
- **删除保护是软防护**：开启后 `DeleteExternalNodePool` 会返回 `OperationDenied`，需先 `ModifyExternalNodePool --DeletionProtection false` 关闭后才能删除。

```bash
tccli tke ModifyExternalNodePool --region <Region> \
    --ClusterId <ClusterId> \
    --NodePoolId <NodePoolId> \
    --DeletionProtection true
# expected: exit 0，返回 RequestId
```

**预期输出**：

```json
{
    "RequestId": "..."
}
```

### 步骤 7：设置自定义脚本（UserScript）

#### 选择依据

- **UserScript 必须 base64 编码**：脚本内容需先 base64 编码后再传入，否则脚本内容会被截断。
- **避免 "nothing is updated" 错误**：如果只传 `--UserScript` 且节点池当前无其他变化时，可能返回 `InvalidParameter.Param: PARAM_ERROR(nothing is updated)`。解决方案：同时传 `--Name` 作为辅助变更，或使用 `--cli-input-json` 文件方式确保请求体包含有效变更。

```bash
# 1. 将脚本内容编码为 Base64
USERSCRIPT_BASE64=$(printf '#!/bin/bash\necho "Node init completed" > /tmp/node-init.log' | base64)

# 2. 构造 JSON 请求（同时传 Name 作为辅助变更，避免 nothing is updated）
cat > modify-extpool.json <<EOF
{
  "ClusterId": "<ClusterId>",
  "NodePoolId": "<NodePoolId>",
  "UserScript": "$USERSCRIPT_BASE64",
  "Name": "<NewName>"
}
EOF

# 3. 执行修改
tccli tke ModifyExternalNodePool --region <Region> \
    --cli-input-json file://modify-extpool.json
# expected: exit 0，返回 RequestId
```

**预期输出**：

```json
{
    "RequestId": "..."
}
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<ClusterId>` | 集群 ID | 格式 `cls-xxxxxxxx` | `tccli tke DescribeClusters --region <Region>` |
| `<NodePoolId>` | 注册节点池 ID | 格式 `np-xxxxxxxx` | `tccli tke DescribeExternalNodePools --region <Region> --ClusterId <ClusterId>` |
| `<NewName>` | 新节点池名称 | 1-60 字符，可与当前名称相同 | 自定义 |

## 验证

### 确认修改生效

```bash
tccli tke DescribeExternalNodePools --region <Region> \
    --ClusterId <ClusterId>
# expected: 返回的节点池配置与修改参数一致
```

**预期输出**：

```json
{
    "TotalCount": 1,
    "NodePoolSet": [
        {
            "NodePoolId": "np-example",
            "Name": "my-external-pool-final",
            "LifeState": "normal",
            "DeletionProtection": false,
            "Labels": [
                {"Name": "tke.cloud.tencent.com/nodepool-id", "Value": "np-example"},
                {"Name": "node.kubernetes.io/instance-type", "Value": "external"},
                {"Name": "tke.cloud.tencent.com/cbs-mountable", "Value": "false"}
            ],
            "Taints": null
        }
    ],
    "RequestId": "..."
}
```

> 上例展示了清空自定义标签和污点后的状态：`Labels` 仅保留系统标签，`Taints` 为 `null`。

```bash
# 精确验证关键字段
tccli tke DescribeExternalNodePools --region <Region> \
    --ClusterId <ClusterId> \
    | jq '.NodePoolSet[] | {Name, Labels, Taints, DeletionProtection}'
# expected: Name、Labels、Taints、DeletionProtection 与修改一致
```

| 验证维度 | 命令 | 预期 |
|------|------|------|
| 节点池名称 | `DescribeExternalNodePools` → `Name` | 与修改一致 |
| 自定义标签 | `DescribeExternalNodePools` → `Labels`（排除系统标签） | 与修改一致；清空后仅剩系统标签 |
| 污点 | `DescribeExternalNodePools` → `Taints` | 与修改一致；清空后为 `null` 或 `[]` |
| 删除保护 | `DescribeExternalNodePools` → `DeletionProtection` | 与修改一致 |
| 节点池状态 | `DescribeExternalNodePools` → `LifeState` | `normal`（修改不影响节点池运行状态） |

## 清理

> **⚠️ 警告**：
> - `DeleteExternalNodePool` 删除节点池本身，**不会删除已注册的外部节点**。如需移除已注册节点，需先执行 `DrainExternalNode` + `DeleteExternalNode`。
> - 删除操作**不可逆**。生产环境请先 `DescribeExternalNodePools` 确认目标 `NodePoolId`。
> - 如节点池已开启 `DeletionProtection`，需先关闭才能删除。

### 1. 清理前状态检查

```bash
tccli tke DescribeExternalNodePools --region <Region> \
    --ClusterId <ClusterId>
# 确认是待删除的目标节点池，记录 NodePoolId、Name、DeletionProtection
```

**预期输出**：

```json
{
    "TotalCount": 1,
    "NodePoolSet": [
        {
            "NodePoolId": "np-example",
            "Name": "my-external-pool",
            "LifeState": "normal",
            "DeletionProtection": false
        }
    ]
}
```

### 2. 关闭删除保护（如已启用）

```bash
tccli tke ModifyExternalNodePool --region <Region> \
    --ClusterId <ClusterId> \
    --NodePoolId <NodePoolId> \
    --DeletionProtection false
# expected: exit 0，返回 RequestId
```

### 3. 删除注册节点池

```bash
tccli tke DeleteExternalNodePool --region <Region> \
    --ClusterId <ClusterId> \
    --NodePoolIds '["<NodePoolId>"]'
# expected: exit 0，返回 RequestId
```

**预期输出**：

```json
{
    "RequestId": "..."
}
```

### 4. 验证已删除

```bash
tccli tke DescribeExternalNodePools --region <Region> \
    --ClusterId <ClusterId>
# expected: 目标 NodePoolId 不在列表中，或 TotalCount 减少
```

**预期输出**：

```json
{
    "TotalCount": 0,
    "NodePoolSet": [],
    "RequestId": "..."
}
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `ModifyClusterNodePool` 返回 `FailedOperation.RecordNotFound: Record not found` | 检查是否对注册节点池使用了通用 API | 注册节点池不在通用节点池数据库中，通用 API `ModifyClusterNodePool` 无法操作注册节点池 | 换用 `ModifyExternalNodePool`，参数更精简（无 `MaxNodesNum`/`MinNodesNum`/`EnableAutoscale`/`OsName` 等普通节点池参数） |
| `DescribeClusterNodePoolDetail` 返回 `FailedOperation.NodePoolQueryFailed: related node pool query err(get nodepool 'np-xxxxxxxx' failed: [E501001 DBRecordNotFound] record not found)` | 检查是否对注册节点池使用了通用查询 API | 注册节点池详情只能用 `DescribeExternalNodePools` 查询，通用 API 无法找到注册节点池 | 换用 `DescribeExternalNodePools`，只传 `--ClusterId`，不要使用 `DescribeClusterNodePoolDetail` 或 `DescribeClusterNodePools` |
| `ModifyExternalNodePool` 仅传 `--UserScript` 返回 `InvalidParameter.Param: PARAM_ERROR(nothing is updated)` | 检查请求是否只含 `UserScript` 一个变更字段 | 只传 `--UserScript` 且节点池当前无其他变化时，后端判定无有效变更 | 同时传 `--Name` 作为辅助变更，或使用 `--cli-input-json` 文件方式确保请求体包含有效变更 |
| `ModifyExternalNodePool` 返回 `InvalidParameter.TaintEffect` | 检查 JSON 中 `Effect` 值 | `Effect` 取值不在合法枚举中 | 修正 `Effect` 为 `NoSchedule`、`NoExecute` 或 `PreferNoSchedule` |
| `ModifyExternalNodePool` 返回 `InvalidParameter.UserScript` | 检查 `UserScript` 是否为有效 Base64 编码 | 脚本内容未 base64 编码，直接传入原始文本 | 用 `printf 'SCRIPT_CONTENT' \| base64` 重新编码后再传入 |
| 修改标签后原有自定义标签丢失 | `tccli tke DescribeExternalNodePools --region <Region> --ClusterId <ClusterId>` 对比修改前后 `Labels` | `Labels` 是覆盖写入（非增量追加），传入新数组会替换全部自定义标签 | 修改前先 `DescribeExternalNodePools` 获取当前自定义标签，拼接后一并传入 |
| 不传 `Taints` 期望清空污点但污点未变化 | `tccli tke DescribeExternalNodePools --region <Region> --ClusterId <ClusterId>` 检查 `Taints` | 不传 `Taints` 表示"不修改"，并非"清空" | 显式传入 `--Taints '[]'` 才能清空所有污点 |

### 修改后行为不符合预期

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 修改标签/污点后现有节点未同步 | `tccli tke DescribeExternalNode --region <Region> --ClusterId <ClusterId> --NodePoolId <NodePoolId>` 检查节点标签 | Labels/Taints 修改仅对后续加入节点池的新节点生效，现有节点不受影响 | 现有节点需手动修改或重建 |
| 修改后查询结果未立即更新 | `tccli tke DescribeExternalNodePools --region <Region> --ClusterId <ClusterId>` 重新查询 | 请求被接受但后端处理有延迟 | 等待 1-2 分钟后重新查询确认 |
| `DeleteExternalNodePool` 返回 `OperationDenied` | `tccli tke DescribeExternalNodePools --region <Region> --ClusterId <ClusterId>` 检查 `DeletionProtection` | 节点池已开启删除保护 | 先执行 `ModifyExternalNodePool --DeletionProtection false` 关闭后重试 |

> **排障提示**：遇到错误时保留 `RequestId`（响应中的 `RequestId` 字段），以备工单或日志查询。提交工单时附上 `region`、`ClusterId`、`NodePoolId`、`RequestId` 和请求参数。

## 下一步

- [创建注册节点（公网版）](https://cloud.tencent.com/document/product/457/101532) — 创建新的注册节点池
- [创建注册节点（专线版）](https://cloud.tencent.com/document/product/457/49681) — 通过专线接入 IDC 主机
- [移除注册节点](https://cloud.tencent.com/document/product/457/79767) — 删除注册节点或节点池
- [流量接入](https://cloud.tencent.com/document/product/457/79749) — 注册节点上的流量接入方式
- [注册节点常见问题](https://cloud.tencent.com/document/product/457/79750) — 排障与 FAQ

## 控制台替代

[TKE 控制台 → 集群 → 节点管理 → 注册节点 → 节点池详情](https://console.cloud.tencent.com/tke2/cluster)：编辑节点池配置。
