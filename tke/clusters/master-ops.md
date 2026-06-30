---
doc_type: How-to
subtype: 6A
fused: true
---
# 独立集群 Master 运维

> 扩缩容独立集群（INDEPENDENT_CLUSTER）的 Master/etcd 节点。仅独立集群适用——托管集群 Master 由腾讯云运维，无此操作。

> 本文档所有 Action 属 **TKE 2018-05-25（默认版本）**，无需显式 `--version`。入参骨架、枚举值、错误码均来自真机实测（P2）与 tccli 自带 `api.json` 权威定义（P7），非文档转述。

## 概述

独立集群与托管集群的本质差异：**你自己运维 Master 节点**。Master 与 etcd 通常同机部署（角色 `MASTER_ETCD`），数量 3～7 台（建议奇数，保证 etcd 选举多数派）。当集群规模增长或 Master 故障时，需用 tccli 扩缩容 Master。

| 操作 | 接口 | 用途 | 触发条件 |
|:-----|:-----|:-----|:-----|
| 扩容 Master | `ScaleOutClusterMaster` | 增加 Master/etcd 节点 | Master 负载高 / 需提升 HA 冗余 |
| 缩容 Master | `ScaleInClusterMaster` | 移除指定 Master/etcd 节点 | Master 过剩 / 下线故障节点 |

操作是**异步**的：命令返回即提交，Master 节点进入 `MasterScaling` 状态，需轮询直到 `Running`。

> ⚠️ **仅独立集群可用**：实测在托管集群调用稳定返回 `InvalidParameter: only independent cluster allowed to scale master or etcd`。集群类型创建后不可切换（见 [创建集群](create.md)）。

## 准备工作

### 环境检查

```bash
tccli --version
# expected: tccli 版本号

# 确认目标是独立集群（ClusterType=INDEPENDENT_CLUSTER）
tccli tke DescribeClusters --region <REGION> --ClusterIds '["<CLUSTER_ID>"]' \
  --filter "Clusters[0].{name:ClusterName,type:ClusterType}" --output text
# expected: INDEPENDENT_CLUSTER（托管集群显示 MANAGED_CLUSTER，不可执行本操作）
```

### 资源检查

```bash
# 1. 集群状态须 Running（MasterScaling 中不可再扩缩）
tccli tke DescribeClusterStatus --region <REGION> --ClusterIds '["<CLUSTER_ID>"]' \
  --filter "ClusterStatusSet[0].ClusterState" --output text
# expected: Running

# 2. 查当前 Master 节点（独立集群 Master 不在 DescribeClusterInstances 工作节点列表，
#    用 DescribeClusterStatus 看节点计数；Master 节点 CVM 用 cvm:DescribeInstances 按 集群标签查）
tccli tke DescribeClusterStatus --region <REGION> --ClusterIds '["<CLUSTER_ID>"]' \
  --filter "ClusterStatusSet[0].{nodes:ClusterRunningNodeNum,failed:ClusterFailedNodeNum}" --output text
# expected: 节点数 + 失败节点数
```

> ⚠️ **CAM 标签授权（实测）**：`ScaleOut/ScaleInClusterMaster` 要求目标集群带特定标签（实测要求 `billing` 标签，CAM 匹配 `qcs:resource_tag`）才放行。不带标签的独立集群被拒，错误码与修复见 [§故障恢复](#故障恢复)。此约束与 [维护窗口](maintenance-window.md) 同款。

## 关键字段

> 来源：`tccli tke ScaleOutClusterMaster/ScaleInClusterMaster --generate-cli-skeleton` + tccli `api.json` 枚举定义（实测 P7）。

### ScaleOutClusterMaster — 扩容

| 字段 | 类型 | 必填 | 约束 | 填错时的错误 |
|:------|------|:--------:|------------|---------------|
| ClusterId | string | 是 | 独立集群 ID | `InvalidParameter`（托管集群）/ `ResourceNotFound` |
| RunInstancesForNode | list | 是 | 节点角色 + CVM 透传参数 | `InvalidParameterValue` |
| RunInstancesForNode[].NodeRole | string | 是 | `MASTER_ETCD` / `WORKER`（扩 Master 用 `MASTER_ETCD`） | `InvalidParameterValue` |
| RunInstancesForNode[].RunInstancesPara | list of string | 是 | CVM 创建参数 JSON 字符串，详见 [CVM RunInstances](https://cloud.tencent.com/document/api/213/15730)，与 [创建集群](create.md) 同款透传 | CVM 侧校验错误 |

> `NodeRole` 枚举（api.json 权威）：`MASTER_ETCD`（Master+etcd 同机）/ `WORKER`。`MASTER_ETCD` 仅独立集群可用，数量 3～7 建议奇数，最小配置 4C8G。扩容 Master 一律用 `MASTER_ETCD`。

### ScaleInClusterMaster — 缩容

| 字段 | 类型 | 必填 | 约束 | 填错时的错误 |
|:------|------|:--------:|------------|---------------|
| ClusterId | string | 是 | 独立集群 ID | `InvalidParameter`（托管集群） |
| ScaleInMasters | list | 是 | 待缩容节点列表 | `InvalidParameterValue` |
| ScaleInMasters[].InstanceId | string | 是 | 待移除 Master 的 CVM ID（`ins-xxxxxxxx`） | `ResourceNotFound` |
| ScaleInMasters[].NodeRole | string | 是 | `MASTER` / `ETCD` / `MASTER_ETCD`（缩容时可区分三角色） | `InvalidParameterValue` |
| ScaleInMasters[].InstanceDeleteMode | string | 是 | `terminate`（销毁 CVM，仅按量计费）/ `retain`（仅移除，保留 CVM） | `InvalidParameterValue` |

> ⚠️ **缩容不可破坏 etcd 多数派**：`MASTER_ETCD` 节点缩容后剩余数量必须仍构成 etcd 多数派（≥3 且为奇数），否则集群控制面不可用。缩容前务必核对剩余 Master 数。

## 操作步骤

### 步骤 1：决策 — 扩还是缩，缩哪个

#### 扩容时机

- **Master 负载高**：API Server / etcd CPU 持续高位，现有 Master 数不足
- **提升 HA 冗余**：从 3 Master 扩到 5 Master，容忍更多节点故障
- **能缩回吗**：能，用 `ScaleInClusterMaster` 移除多余 Master，但不可破坏多数派

#### 缩容时机

- **Master 过剩**：集群规模收缩，3 Master 已足够
- **下线故障 Master**：某 Master CVM 硬件故障，先扩新节点再缩故障节点（保多数派）

### 步骤 2：扩容 Master — 最小化

```bash
tccli tke ScaleOutClusterMaster --region <REGION> \
  --ClusterId "<CLUSTER_ID>" \
  --RunInstancesForNode '[
    {
      "NodeRole": "MASTER_ETCD",
      "RunInstancesPara": [
        "{\"InstanceType\":\"S5.LARGE8\",\"Placement\":{\"Zone\":\"<ZONE>\"},\"InstanceChargeType\":\"POSTPAID_BY_HOUR\",\"ImageId\":\"<IMAGE_ID>\"}"
      ]
    }
  ]'
# expected: exit 0, 返回 RequestId（异步任务已提交，集群进入 MasterScaling）
```

| 占位符 | 含义 | 约束 | 如何获取 |
|:------------|:-----|:-----|:---------|
| `<CLUSTER_ID>` | 独立集群 ID | `cls-xxxxxxxx`，须 INDEPENDENT_CLUSTER | `tccli tke DescribeClusters` → `Clusters[?ClusterType==\`INDEPENDENT_CLUSTER\`]` |
| `<ZONE>` | 可用区 | 与集群同地域 | `tccli cvm DescribeZones --region <REGION>` |
| `<IMAGE_ID>` | CVM 镜像 ID | 与现有 Master 同镜像 | `tccli tke DescribeOSImages --region <REGION>` |

> `RunInstancesPara` 是 CVM `RunInstances` 参数的 JSON 字符串透传——与 [创建集群](create.md) 步骤 3 的节点创建参数同款机制。Master 最小 4C8G（如 `S5.LARGE8`），与现有 Master 同规格以保证一致性。

### 步骤 3：扩容 — 增强：一次扩多台

```bash
tccli tke ScaleOutClusterMaster --region <REGION> \
  --ClusterId "<CLUSTER_ID>" \
  --RunInstancesForNode '[
    {
      "NodeRole": "MASTER_ETCD",
      "RunInstancesPara": [
        "{\"InstanceType\":\"S5.LARGE8\",\"Placement\":{\"Zone\":\"<ZONE_A>\"},\"InstanceChargeType\":\"POSTPAID_BY_HOUR\",\"ImageId\":\"<IMAGE_ID>\"}",
        "{\"InstanceType\":\"S5.LARGE8\",\"Placement\":{\"Zone\":\"<ZONE_B>\"},\"InstanceChargeType\":\"POSTPAID_BY_HOUR\",\"ImageId\":\"<IMAGE_ID>\"}"
      ]
    }
  ]'
# expected: exit 0（一次扩 2 台，跨可用区提升 HA；3→5 Master）
```

> 一次扩多台时 `RunInstancesPara` 数组每个元素一台 CVM。建议跨可用区分布（`Zone_A`/`Zone_B`），避免单可用区故障导致 Master 全损。

### 步骤 4：缩容 Master

```bash
tccli tke ScaleInClusterMaster --region <REGION> \
  --ClusterId "<CLUSTER_ID>" \
  --ScaleInMasters '[
    {"InstanceId":"<INSTANCE_ID>","NodeRole":"MASTER_ETCD","InstanceDeleteMode":"terminate"}
  ]'
# expected: exit 0, 返回 RequestId（异步任务已提交）
```

| 占位符 | 含义 | 约束 | 如何获取 |
|:-------|:-----|:-----|:---------|
| `<INSTANCE_ID>` | 待缩容 Master CVM ID | `ins-xxxxxxxx`，须当前 Master | `tccli cvm DescribeInstances --region <REGION>` 按集群标签过滤，或控制台 Master 节点列表 |

> `InstanceDeleteMode=terminate` 销毁 CVM（仅按量计费）；`retain` 仅从集群移除保留 CVM（包年包月必须 retain）。缩容前确认剩余 Master ≥3 且为奇数。

### 步骤 5：验证

异步操作，检查 ≥4 个维度：

```bash
# 轮询集群状态（扩缩容中为 MasterScaling，完成后回 Running）
tccli tke DescribeClusterStatus --region <REGION> --ClusterIds '["<CLUSTER_ID>"]' \
  --filter "ClusterStatusSet[0].ClusterState"
# expected: "MasterScaling" → "Running"
```

> 下方 kubectl 验证需先获取 kubeconfig（注意 `--output text` 剥引号，无它则文件带引号 kubectl 无法解析）：
> ```bash
> tccli tke DescribeClusterKubeconfig --region <REGION> --ClusterId "<CLUSTER_ID>" \
>   --filter "Kubeconfig" --output text > kubeconfig.yaml
> # expected: kubeconfig 文件生成，可 KUBECONFIG=kubeconfig.yaml kubectl get nodes
> ```

| 维度 | 命令 | 预期 |
|:-----|:-----|:-----|
| 集群状态 | `DescribeClusterStatus` → `ClusterState` | `MasterScaling` → `Running` |
| Master 数量 | 控制台 Master 节点列表，或 `tccli cvm DescribeInstances` 按集群标签过滤 Master CVM（独立集群 Master 不在 `DescribeClusterInstances` 工作节点列表，`ClusterRunningNodeNum` 仅计工作节点） | 扩容后增加，缩容后减少，且 ≥3 奇数 |
| etcd 健康 | 集群 `Running` 后 `kubectl get nodes`（用 kubeconfig） | 所有 Master 节点 Ready |
| 集群可用性 | `kubectl get --raw='/healthz'` 或 `kubectl get pods -n kube-system`（`kubectl get cs` 在 K8s 1.19+ 已弃用，1.34 集群不返回） | 控制面健康 |

> `MasterScaling` 状态见 [状态机](../reference/states.md)。超 30 分钟未回 `Running` 属异常，见 [故障恢复](#故障恢复)。

## 清理

> **缩容即清理**：`ScaleInClusterMaster --InstanceDeleteMode terminate` 直接销毁 Master CVM，无独立清理步骤。`retain` 模式保留的 CVM 需到 CVM 侧手动销毁（`tccli cvm TerminateInstances`，仅按量计费）。
>
> **计费提示（P8）**：Master CVM 按 CVM 计费规则收费，扩容即新增 CVM 费用，缩容 `terminate` 后立即停止计费。独立集群无集群管理费（与托管集群的差异，见 [创建集群](create.md)）。

## 故障恢复

### 命令返回错误 (exit ≠ 0)

| 现象 | 诊断 | 根因 | 修复 |
|:--------|:----------|:------------|:-----|
| `InvalidParameter: only independent cluster allowed to scale master or etcd` | `tccli tke DescribeClusters --ClusterIds '["<ID>"]' --filter "Clusters[0].ClusterType"` | 目标是托管集群（MANAGED_CLUSTER） | 托管集群 Master 不可扩缩容；如确需，新建独立集群迁移 |
| CAM `has no permission`（含 `qcs:resource_tag` `billing` 条件） | `tccli tke DescribeClusters --ClusterIds '["<ID>"]' --filter "Clusters[0].TagSpecification[*].Tags[*]"` | 目标独立集群未带 CAM 要求的 `billing` 标签 | 给集群加 `billing` 标签，或申请 `tke:ScaleOutClusterMaster`/`tke:ScaleInClusterMaster` 权限。环境限制，非命令错误 |
| `ResourceNotFound` (InstanceId) | `tccli cvm DescribeInstances --InstanceIds '["<ID>"]'` | 缩容的 InstanceId 不存在或不在该集群 | 核对 CVM ID，确认属于该集群 Master |
| `UnsupportedOperation` | `tccli tke DescribeClusterStatus` 看状态 | 集群非 `Running`（MasterScaling 中或异常） | 等集群 `Running` 后重试 |
| CVM 创建失败（扩容时） | 看返回的 RequestId 对应 CVM 侧错误 | 可用区资源不足 / 机型售罄 / 镜像不存在 | 换可用区或机型，`tccli tke DescribeOSImages` 核对镜像 |

### 命令成功但状态不对 (exit = 0)

| 现象 | 诊断 | 根因 | 修复 |
|:--------|:----------|:------------|:-----|
| 长时间停在 `MasterScaling` | `tccli tke DescribeClusterStatus` + `kubectl get nodes` | 新 Master 加入 etcd 集群卡住 / CVM 初始化失败 | 查新节点 CVM 状态，必要时 `ScaleIn` 回滚新增节点 |
| 缩容后 etcd 不可用 | `kubectl get --raw='/healthz'` 或 `kubectl get nodes` 看节点 Ready | 缩容破坏了 etcd 多数派（剩余 Master <3 或偶数） | 立即 `ScaleOutClusterMaster` 扩回节点恢复多数派 |
| 扩容后 Master 数未增加 | `DescribeClusterStatus` 节点计数 | CVM 创建成功但加入集群失败 | 查 CVM 是否 running，`kubectl get nodes` 看是否 Ready |
| 缩容 `retain` 后 CVM 仍在计费 | `tccli cvm DescribeInstances` | `retain` 仅移出集群不销毁 CVM | 手动 `tccli cvm TerminateInstances --InstanceIds '["<ID>"]'`（按量计费） |

> etcd 多数派破坏是最严重故障——缩容必须保留 ≥3 且奇数个 `MASTER_ETCD` 节点。生产环境缩容前先扩后缩（扩新节点 → 确认加入 → 缩旧节点），全程保持多数派。

## 下一步

- [创建集群](create.md) — 独立集群与托管集群的类型决策
- [配置集群属性与运行时](configure.md) — Master 组件启停（`ModifyMasterComponent`）
- [集群状态机](../reference/states.md) — `MasterScaling` 等状态含义
- [故障排查](../troubleshooting.md) — Master 异常诊断路径

## 控制台替代方案

[容器服务控制台 - 集群详情 - Master 节点](https://console.cloud.tencent.com/tke2/cluster)

## Action 清单

| Action | 类型 | 版本 | 说明 |
|:-------|:-----|:-----|:-----|
| `ScaleOutClusterMaster` | 主操作 | 2018-05-25 | 扩容独立集群 Master/etcd（`RunInstancesForNode`，NodeRole=MASTER_ETCD） |
| `ScaleInClusterMaster` | 主操作 | 2018-05-25 | 缩容指定 Master/etcd（`ScaleInMasters`，NodeRole=MASTER/ETCD/MASTER_ETCD） |
| `DescribeClusters` | 验证 | 2018-05-25 | 确认集群类型为 INDEPENDENT_CLUSTER |
| `DescribeClusterStatus` | 验证 | 2018-05-25 | 轮询 MasterScaling→Running + 节点计数 |
| `DescribeOSImages` | 验证 | 2018-05-25 | 获取 Master CVM 镜像 ID |
| `DescribeRegions` | 验证 | 2018-05-25 | 凭证有效性检查 |
| `cvm:DescribeInstances` | 跨产品 | cvm | 查 Master CVM ID（缩容目标） |
| `cvm:TerminateInstances` | 跨产品 | cvm | 清理 retain 模式保留的 CVM |
| `cvm:DescribeZones` | 跨产品 | cvm | 获取可用区（扩容参数） |
