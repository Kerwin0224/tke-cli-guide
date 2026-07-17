---
doc_type: How-to
subtype: 6A
fused: true
---
# 删除集群

> 控制台: [容器服务控制台 - 集群列表](https://console.cloud.tencent.com/tke2/cluster)
> ⚠️ **不可逆操作。** 删除集群会销毁所有工作节点 (CVM)，数据无法恢复。
>
> **删除前勿做**（高危，部分不可恢复）：在节点上重装 OS、删 `/etc/kubernetes`、自行换 Master 证书、改 Master 节点 IP、在 LB 控制台改 TKE 管理的监听器/后端——详见 [故障排查 — 高危操作后果速查](../troubleshooting.md#高危操作后果速查)。删集群本身不恢复已误毁的 Master 数据；CBS/EIP/CLB 默认**保留并持续计费**，须按本文副作用表清理。

> 本文档 Action 属 **TKE 2018-05-25**（`DeleteCluster`/`Enable/DisableClusterDeletionProtection` 均旧版独有）。文中 `DescribeClusters`/`DescribeClusterStatus` 为辅助查询，走默认旧版；`DescribeClusters` 是两版同名 Action，见 [查询集群](query.md#两版同名-actiondescribeclusters)。

> 官方文档：[删除集群](https://cloud.tencent.com/document/product/457/44808) · [集群生命周期](https://cloud.tencent.com/document/product/457/32188) · [常见高危操作](https://cloud.tencent.com/document/product/457/39539)

> 配额：删除集群无额外配额限制，但删除后会释放集群配额占用。[配额说明](https://cloud.tencent.com/document/product/457/9087)

## 触发条件

- 集群已废弃/迁移完毕，确认不再需要（`DescribeClusters` 核对无业务 Pod）— 用本文销毁
- 集群创建失败卡 `Creating` > 30 分钟，需删除重建 — 先删再建（见 [创建集群](create.md)）
- 测试集群用毕即删，避免空集群持续计管理费 — 用本文清理

## 副作用 {#副作用}

删除集群时会影响以下资源:

| 资源 | 是否自动删除 | 需手动清理 |
|------|:----------:|------|
| 工作节点 (CVM) | ✅ 自动销毁 | — |
| 托管 Master | ✅ 自动回收 | — |
| CBS 云硬盘 | ❌ **保留** | `tccli cbs TerminateDisks` |
| 弹性公网 IP (EIP) | ❌ **保留** | `tccli vpc ReleaseAddresses` |
| CLB 负载均衡 | ❌ **保留** | 控制台或 CLB API |
| 集群内创建的 VPC 子网 | ❌ **保留** | `tccli vpc DeleteSubnet` |

> **计费提示**: 集群管理费在删除后立即停止。但保留的 CBS/EIP/CLB 会**持续扣费**。

## 决策依据 {#决策依据}

### InstanceDeleteMode: terminate vs retain

`InstanceDeleteMode` 决定工作节点 CVM 的去留（必填，不传报 `the following arguments are required: --InstanceDeleteMode`）：

| 模式 | 节点 CVM | 适用场景 | 计费影响 |
|:-----|:---------|:---------|:---------|
| `terminate` | 销毁（仅按量计费可销毁） | 默认推荐，彻底清理 | CVM 立即停止计费 |
| `retain` | 保留为孤立实例，仅移出集群 | 误删回退、节点复用 | CVM **持续计费**，需手动 `cvm:TerminateInstances` 清理 |

> **默认推荐 `terminate`**——`retain` 会留下无人管理的 CVM 持续扣费，是常见成本泄漏。包年包月 CVM 只能 `retain`（销毁需到 CVM 侧退费）。

### 是否级联删除关联资源

| 资源 | 默认（最小化） | 级联（Enhanced） | 说明 |
|:-----|:------------|:---------------|:-----|
| 工作节点 CVM | 按 `InstanceDeleteMode` | 同左 | 受 InstanceDeleteMode 控制 |
| CBS 云硬盘 | 保留 | 可级联删（`ResourceType:CBS`） | 级联删避免残留计费 |
| CLB 负载均衡 | 保留 | 可级联删（`ResourceType:CLB`） | 级联删避免残留计费 |
| 弹性公网 IP | 保留 | **不可级联删**（EIP 非合法 ResourceType） | 必须用 `vpc:ReleaseAddresses` 单独清理 |

> **生产清理用 Enhanced**：`InstanceDeleteMode=terminate` + `ResourceDeleteOptions` 级联删 CBS/CLB，再手动 `vpc:ReleaseAddresses` 清 EIP，确保零残留计费。`ResourceType` 合法值仅 `CBS`/`CLB`/`CVM`（传 `EIP` 报 `FailedOperation.Param`）。

## 准备工作

```bash
# 确认要删除的集群
tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["<CLUSTER_ID>"]'
# expected: 确认 ClusterName 与预期一致

# 检查删除保护状态
tccli tke DescribeClusterStatus --region ap-guangzhou --filter "ClusterStatusSet[?ClusterId=='<CLUSTER_ID>'] | [0].ClusterDeletionProtection" --output text
# expected: True（已开启删除保护，需先关闭）或 False（已关闭，可直接删）
```

## 操作步骤

> ⚠️ **高危操作**：集群删除**不可逆**！`InstanceDeleteMode` 决定是否连带删除 CVM/CBS/CLB；`retain` 模式下 CVM 保留但持续计费。[常见高危操作](https://cloud.tencent.com/document/product/457/39539)

### 步骤 1：关闭删除保护 {#步骤-1关闭删除保护}

如果创建时开启了删除保护，需要先关闭:

```bash
tccli tke DisableClusterDeletionProtection \
  --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>"
# expected: { "RequestId": "..." }
```

### 步骤 2：删除集群

`DeleteCluster` 必传 `ClusterId` + `InstanceDeleteMode`（决策见 [§决策依据](#决策依据)）。按场景**二选一**：A 最小化（销毁节点，CBS/CLB/EIP 保留手动清理）或 B 级联删除（连 CBS/CLB 一起删，减少残留资源）。

> ⚠️ **A 与 B 是二选一变体，不是先做 A 再做 B**——两者各调一次 `DeleteCluster` 删的是同一个集群，第二次调用报集群已不存在。CBS/CLB 残留清理见 [§步骤 4](#步骤-4清理残留资源)。

#### 选项 A：最小化（销毁节点，CBS/CLB/EIP 保留）

```bash
tccli tke DeleteCluster \
  --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" \
  --InstanceDeleteMode terminate
# expected: exit 0
```

#### 选项 B：级联删除（连 CBS/CLB 一起删）

> **与 A 二选一，非在 A 之后执行**。连 CBS 盘、CLB 一起删除：

```bash
tccli tke DeleteCluster \
  --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" \
  --InstanceDeleteMode terminate \
  --ResourceDeleteOptions '[
    {"ResourceType":"CBS","DeleteMode":"terminate","SkipDeletionProtection":false},
    {"ResourceType":"CLB","DeleteMode":"terminate","SkipDeletionProtection":false}
  ]'
# expected: exit 0
```

> `ResourceType` 合法枚举：`CBS`（云硬盘）/ `CLB`（负载均衡）/ `CVM`（节点实例）。**`EIP` 不是合法值**——传 `EIP` 报 `FailedOperation.Param`，弹性公网 IP 需用 `vpc:ReleaseAddresses` 单独清理（见 [§步骤 4](#步骤-4清理残留资源)）。
>
> ⚠️ `SkipDeletionProtection: false` 表示不跳过资源的删除保护。`true` 时跳过开启了删除保护的资源（含 CLB 有终端节点的情况，亦视为开了删除保护）。如果 CBS 盘也开启了删除保护且 `SkipDeletionProtection=false`，删除会失败——需先手动关闭保护或改用 `true`。

### 步骤 3：验证

```bash
tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["<CLUSTER_ID>"]'
# expected: { "TotalCount": 0, "Clusters": [] }
```

### 步骤 4：清理残留资源 {#步骤-4清理残留资源}

即使用了 Enhanced 模式，也建议检查残留:

```bash
# 检查 CBS 盘（按标签或创建时间过滤）
tccli cbs DescribeDisks --region ap-guangzhou

# 检查 EIP
tccli vpc DescribeAddresses --region ap-guangzhou

# 检查 CLB（Service 类型 LoadBalancer 自动创建的 CLB）
tccli clb DescribeLoadBalancers --region ap-guangzhou --Forward 1
```

> 删集群不会自动清理 CBS/EIP/CLB/VPC 子网，残留资源持续计费。逐个确认后删除：
> - **CBS 盘**：`tccli cbs TerminateDisks --region <REGION> --DiskIds '["<DISK_ID>"]'`
> - **EIP**：`tccli vpc ReleaseAddresses --region <REGION> --AddressIds '["<EIP_ID>"]'`
> - **CLB**：`tccli clb DeleteLoadBalancer --region <REGION> --LoadBalancerIds '["<LBC_ID>"]'`
> - **VPC 子网**（集群内创建的，若不再用）：先 `tccli vpc DescribeSubnets --region <REGION> --Filters '[{"Name":"vpc-id","Values":["<VPC_ID>"]}]'` 确认无依赖，再 `tccli vpc DeleteSubnet --region <REGION> --SubnetId "<SUBNET_ID>"`（删子网前须无实例占用）

> ⚠️ VPC 子网本身不直接计费，但占用 VPC 配额且可能含计费 NAT/路由表；CLB 持续计费，须清理。删子网前确认无 CVM/CLB/NAT 占用，否则报 `ResourceInUse`。

## 故障恢复

> 删除相关错误码速查见 [错误码速查](../reference/error-codes.md)；集群状态机见 [状态机](../reference/states.md)。

### 命令返回错误（exit ≠ 0）

| 现象 | 诊断 | 根因 | 修复 |
|---------|----------|------------|-----|
| 删除保护阻止 | 错误码 `OperationDenied.ClusterInDeletionProtection`（消息含 `cluster in deletion protection`） | 未关闭删除保护 | `tccli tke DisableClusterDeletionProtection --ClusterId "<ID>"` 先关保护；若该接口被 CAM 拒（要求 `request_tag` 但接口无 Tags 入参），需控制台关闭或调整 CAM 策略 |
| `ResourceNotFound.ClusterId` | `tccli tke DescribeClusters --ClusterIds '["<ID>"]'` | 集群 ID 错误或已删除 | 检查集群 ID 是否正确 |
| `AuthFailure` | `tccli tke DescribeRegions` | 凭证过期 | 见 [配置凭证](../../getting-started/credentials.md) |
| `InternalError.ClusterDeletion` | 查看错误详情的 RequestId | 服务端删除异常 | 等待 5 分钟后重试；仍失败则提交工单附带 RequestId |
| CBS 盘未随集群销毁 | `tccli cbs DescribeDisks` → 检查 `DeleteWithInstance` | `DeleteWithInstance=false`（云盘设为不随实例销毁，非"删除保护"字段） | `tccli cbs ModifyDiskAttributes --DeleteWithInstance true` 后销毁，或 `tccli cbs TerminateDisks --DiskIds '["<ID>"]'` |
| `FailedOperation.Param` (ResourceDeleteOptions) | 核对 `ResourceType` 值 | 传了非法 `ResourceType`（如 `EIP`，合法值仅 `CBS`/`CLB`/`CVM`） | 改用合法枚举；EIP 用 `vpc:ReleaseAddresses` 单独清理 |

### 命令成功但状态不对（exit = 0）

| 现象 | 诊断 | 根因 | 修复 |
|---------|----------|------------|-----|
| 命令返回成功但集群仍在列表中 | `tccli tke DescribeClusters --ClusterIds '["<ID>"]'` | 异步删除进行中 | 等待 1-2 分钟后重试查询 |
| 集群删除后 CBS 盘仍存在 | `tccli cbs DescribeDisks` | 未使用 Enhanced 删除模式 | `tccli cbs TerminateDisks --DiskIds '["<ID>"]'` |
| 删除后 EIP 仍在扣费 | `tccli vpc DescribeAddresses` | EIP 未随集群释放 | `tccli vpc ReleaseAddresses --AddressIds '["<ID>"]'` |

## 开启删除保护（反操作）

> 删除保护的开启是 `DeleteCluster` 的逆操作——给已创建集群补开保护以防误删。与 [§步骤 1 关闭删除保护](#步骤-1关闭删除保护) 对称，入参同为 `ClusterId`。

```bash
tccli tke EnableClusterDeletionProtection \
  --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>"
# expected: { "RequestId": "..." }
```

验证保护已生效：

```bash
tccli tke DescribeClusterStatus --region ap-guangzhou \
  --ClusterIds '["<CLUSTER_ID>"]' \
  --filter "ClusterStatusSet[0].ClusterDeletionProtection" --output text
# expected: True（布尔值，开启后再次删除会被 OperationDenied.ClusterInDeletionProtection 拒绝）
```

> 幂等：对已开启保护的集群再次 `Enable` 仍返回 RequestId，不报错。

## 收尾确认

```bash
# 残留资源核查：CBS/EIP/CLB 是否真清零（删除完成标志——仅查集群 TotalCount=0 不够，还须查关联计费资源）
tccli cbs DescribeDisks --region ap-guangzhou \
  --filter "DiskSet[?DiskState=='UNATTACHED'].{id:DiskId,name:DiskName}" --output text
# expected: 核对未挂载的 CBS 盘（集群销毁后节点 CVM 已终止，DeleteWithInstance=false 的盘变为 UNATTACHED，须清理；TKE 创建的盘 DiskName 含集群 ID 前缀如 cls-xxx/pvc-...）

tccli vpc DescribeAddresses --region ap-guangzhou \
  --filter "AddressSet[?InstanceId==null].{id:AddressId,eip:AddressIp}" --output text
# expected: 核对未绑定的 EIP（集群销毁后节点 CVM 已终止，原绑定 EIP 变为未绑定状态，须释放）

tccli clb DescribeLoadBalancers --region ap-guangzhou --Forward 1 \
  --filter "TotalCount" --output text
# expected: 核对 CLB 数量，结合集群标签确认无该集群残留 CLB
```

> 集群 `TotalCount=0`（步骤 3 已核）+ CBS/EIP/CLB 残留为 0 = 删除完成，无持续计费。**残留资源是删除完成的真正标志**——集群销毁不自动清理 CBS/EIP/CLB，这些资源会持续扣费（见 [§副作用](#副作用) 表）。非 0 残留须回到 [§步骤 4](#步骤-4清理残留资源) 逐个清理。

## 下一步

- [创建集群](create.md) — 重新创建一个新集群
- [查询集群](query.md) — 确认其他集群状态
