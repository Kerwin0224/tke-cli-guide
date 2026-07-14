---
doc_type: How-to
subtype: 6A
fused: true
---
# 管理集群访问端点

> 控制台: [容器服务控制台 - 集群访问地址](https://console.cloud.tencent.com/tke2/cluster)
> 开启/关闭集群的公网或内网访问端点。端点是 kubectl/API 访问 API Server 的入口。异步操作。

## 触发条件

- `DescribeClusterEndpointStatus` 返回 `Status=NotFound`，需从公网或 VPC 内网访问 API Server
- `kubectl get nodes` <!-- kubectl验证端点连通性，非tccli边界 --> 报 `Unable to connect to the server` / `context deadline exceeded` / `connection refused`，集群端点未开启，或本机不在端点可达网络（内网 VIP 从公网不可达）
- 公网端点已 `Created` 但 `ModifyClusterEndpointSP` 未放行你的出口 IP，kubectl 被 ACL 拒绝 — 看 [故障恢复]段

## 概述

集群创建后默认无访问端点（`DescribeClusterEndpointStatus` 返回 `NotFound`）。需显式开启后，kubectl 才能连 API Server。

| 端点 | 接口 | 本机/公网 CI 能否直连 | 安全 |
|:-----|:-----|:---------------------|:-----|
| 公网端点 | `CreateClusterEndpoint --IsExtranet true` | **能**（再配 ACL 白名单） | `ModifyClusterEndpointSP` 放行出口 IP `/32`；未配默认拒绝 |
| 内网端点 | `CreateClusterEndpoint --IsExtranet false` | **不能**（仅同 VPC / 专线 / VPN） | VPC 隔离 |

操作是**异步**的：`CreateClusterEndpoint` 返回即提交；就绪时 `DescribeClusterEndpointStatus` 的 `Status` 为 **`Created`**（不是 `Running`）。关闭后回到 `NotFound`。

> **本机 kubectl**：必须走公网端点（或本机已接入集群 VPC）。只开内网端点时，`DescribeClusterKubeconfig` 里的 `server` 是内网 VIP，本机访问会超时。

> 官方文档：[容器网络概述](https://cloud.tencent.com/document/product/457/50353) · [网络方案选型](https://cloud.tencent.com/document/product/457/106561) · [容器服务安全组设置](https://cloud.tencent.com/document/product/457/9084)
> 配额：单地域集群数 20、安全组数限制等。[配额限制](https://cloud.tencent.com/document/product/457/9087)
> ⚠️ **高危操作**：公网端点 ACL 白名单配置错误致 API Server 暴露；安全组 0.0.0.0/0 开放致公网可发现。[常见高危操作](https://cloud.tencent.com/document/product/457/39539)

## 准备工作

### 环境检查

```bash
tccli --version
# expected: tccli 版本号

tccli tke DescribeClusterStatus --region ap-guangzhou --filter "ClusterStatusSet[?ClusterId=='<CLUSTER_ID>'] | [0].ClusterState"
# expected: "Running"
```

### 资源检查

```bash
# 确认当前无端点或端点状态（建议显式传 IsExtranet；省略时亦返回 Status）
tccli tke DescribeClusterEndpointStatus --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" --IsExtranet true
# expected: Status="NotFound"（未开）或 "Created"（已开）；内网用 --IsExtranet false

# 内网端点需子网
tccli vpc DescribeSubnets --region <REGION> --Filters '[{"Name":"vpc-id","Values":["<VPC_ID>"]}]' \
  --filter "SubnetSet[].{id:SubnetId,name:SubnetName,cidr:CidrBlock}" --output text
# expected: 子网列表（内网端点需指定 SubnetId）
```

## 关键字段

> 完整入参以 `tccli tke CreateClusterEndpoint help --detail` 为准。

| 字段 | 类型 | 必填 | 约束 | 填错时的错误 |
|:------|------|:--------:|------------|---------------|
| ClusterId | string | 是 | `cls-xxxxxxxx` | `ResourceNotFound` |
| IsExtranet | boolean | 是 | `true`（公网）/ `false`（内网） | `InvalidParameterValue` |
| SubnetId | string | 内网必填 | VPC 内子网 ID | `ResourceNotFound.SubnetId` |
| SecurityGroup | string | 公网常见必填 | 安全组 ID；缺省可能 `InvalidParameter`：`SecurityGroup can not be empty` | `ResourceNotFound.SecurityGroup` / `InvalidParameter` |
| ExtensiveParameters | string | 公网建议 | JSON 字符串，如 `{"InternetAccessible":{"InternetChargeType":"TRAFFIC_POSTPAID_BY_HOUR","InternetMaxBandwidthOut":1}}` | `InvalidParameter*` |
| Domain | string | 否 | 自定义域名 | `InvalidParameterValue` |
| ExistedLoadBalancerId | string | 否 | 复用已有 CLB | `ResourceNotFound` |

> 公网端点用公网 CLB；内网端点用子网内 VIP。源 IP 白名单用 `ModifyClusterEndpointSP` 的 `SecurityPolicies`（CIDR 字符串数组），**不是** `SecurityGroup` 的替代。

## 操作步骤

### 步骤 1：决策 — 公网 vs 内网

#### 为什么选内网端点

- **公网端点**: 本地开发、CI/CD 从公网访问。需 ACL 白名单限定源 IP，否则暴露 API Server 到公网
- **内网端点（推荐）**: 同 VPC 的 CVM/kubectl 访问。VPC 隔离，不暴露公网，低延迟
- **默认推荐**: 生产用内网端点；本地开发临时用公网端点 + 白名单
- **能切换吗?**: 能。`SwitchClusterEndpoint` 切换公网/内网，或先 `DeleteClusterEndpoint` 再建

### 步骤 2：开启公网端点（本机 / 公网 CI）

```bash
tccli tke CreateClusterEndpoint --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" \
  --IsExtranet true \
  --SecurityGroup "<SECURITY_GROUP_ID>" \
  --ExtensiveParameters '{"InternetAccessible":{"InternetChargeType":"TRAFFIC_POSTPAID_BY_HOUR","InternetMaxBandwidthOut":1}}'
# expected: exit 0，返回 RequestId；再 waiter 到 Status=Created
```

| 占位符 | 含义 | 约束 | 如何获取 |
|:------------|:-----|:-----|:---------|
| `<CLUSTER_ID>` | 集群 ID | `cls-xxxxxxxx` | `tccli tke DescribeClusters` → `Clusters[].ClusterId` |
| `<SECURITY_GROUP_ID>` | 安全组 | 公网创建时常必填 | `tccli vpc DescribeSecurityGroups` |

```bash
tccli tke DescribeClusterEndpointStatus --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" --IsExtranet true \
  --waiter '{"expr":"Status","to":"Created","timeout":300,"interval":10}'
# expected: Status="Created"
```

> 部分账号对公网端点有 CAM 条件拒绝（消息含 `tke:clusterExtranetEndpoint` = `true` 的 deny）→ `InvalidParameter.Param` / `ACTION_NO_AUTH`，须改 CAM，非参数拼写问题。`CreateClusterEndpointVip` 另要求集群已有 worker 节点，否则 `ResourceUnavailable.ClusterState`（`cluster without worker Node is not allowed to enable extranet access`）。

### 步骤 3：开启内网端点（同 VPC）

```bash
tccli tke CreateClusterEndpoint --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" --IsExtranet false \
  --SubnetId "<SUBNET_ID>" --SecurityGroup "<SECURITY_GROUP_ID>"
# expected: exit 0；waiter Status=Created；DescribeClusterEndpoints → ClusterIntranetEndpoint 非空
```

> 内网端点必须指定 `SubnetId`。本机不在该 VPC 时，即使端点 `Created`，kubectl 仍会超时——这是网络路径问题，不是 kubeconfig 无效。

### 步骤 4：配置 ACL 白名单（公网端点）

`SecurityPolicies` 是 **CIDR 字符串数组**（如 `"x.x.x.x/32"`），不是 `{"Action","CidrBlock"}` 对象。

```bash
# 修改公网端点安全策略（放行本机/CI 出口 IP）
tccli tke ModifyClusterEndpointSP --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" \
  --SecurityPolicies '["<YOUR_EGRESS_IP>/32"]'
# expected: exit 0
```

> 公网端点未配白名单时默认拒绝所有。先查出口 IP（如 `curl -s https://api.ipify.org`），再写入 `/32`。

### 步骤 5：验证

```bash
tccli tke DescribeClusterEndpointStatus --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" --IsExtranet true
# expected: Status="Created"
```

| 维度 | 命令 | 预期 |
|:-----|:-----|:-----|
| 端点状态 | `DescribeClusterEndpointStatus` → `Status` | `Created`（未开为 `NotFound`） |
| 端点地址 | `DescribeClusterEndpoints` → `ClusterExternalEndpoint` / `ClusterIntranetEndpoint` | 非空 IP |
| ACL | `DescribeClusterEndpoints` → `ClusterExternalACL` | 含你放行的 CIDR |
| kubectl | `kubectl --kubeconfig kubeconfig.yaml get nodes` <!-- kubectl验证端点连通后核节点列表，非tccli边界 --> | 公网端点 + ACL 后返回节点列表；仅内网端点时本机超时 |

### 独立 VIP 端点

> 除普通端点外，可创建独立 VIP 端点（`CreateClusterEndpointVip`），用独立 CLB VIP 暴露 API Server，配 `SecurityPolicies` 白名单。区别于普通端点的 `IsExtranet` 模式。

```bash
# 创建 VIP 端点 (配白名单源 IP)
tccli tke CreateClusterEndpointVip --region <REGION> \
  --ClusterId "<CLUSTER_ID>" --SecurityPolicies '["<YOUR_IP>/32"]'
# expected: exit 0

# 查询 VIP 端点状态
tccli tke DescribeClusterEndpointVipStatus --ClusterId "<CLUSTER_ID>" --region <REGION>
# expected: exit 0；无 worker 节点时 ResourceUnavailable.ClusterState
```

```bash
# 删除 VIP 端点
tccli tke DeleteClusterEndpointVip --ClusterId "<CLUSTER_ID>" --region <REGION>
# expected: exit 0
```

> VIP 端点与 `IsExtranet` 普通公网端点不同；空集群（无 worker）不可开 VIP 外网访问。`DeleteClusterEndpointVip` 仅需 `ClusterId`。

### 切换公网/内网端点

> 已开启的端点可在公网/内网间切换，避免先删再建。`SwitchClusterEndpoint` 的 `IsExtranet` 指定目标类型，`Rollback` 控制切换失败是否回滚。参数以 `tccli tke SwitchClusterEndpoint help --detail` 为准。

```bash
# 切换端点类型（IsExtranet=true 切公网，false 切内网；Rollback=true 失败自动回滚）
tccli tke SwitchClusterEndpoint --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" --IsExtranet false --Rollback true
# expected: exit 0 返回 RequestId; 集群类型/端点类型不支持报 FailedOperation.SwitchClusterEndpoint: SWITCH_CLUSTER_ENDPOINT_ERROR
```

| 占位符 | 含义 | 约束 |
|:-------|:-----|:-----|
| `--IsExtranet` | 目标端点类型 | `true`=公网，`false`=内网 |
| `--Rollback` | 切换失败是否回滚到原端点 | `true`（推荐，避免切换失败导致无端点可用）/ `false` |

> `SwitchClusterEndpoint` 切换是异步操作，提交后用 `DescribeClusterEndpointStatus` 轮询到 `Created`。切换到内网端点需集群 VPC 内有可用子网（同 `CreateClusterEndpoint --IsExtranet false` 的子网要求）。

## 清理

> **副作用警告**：关闭端点会断开所有通过该端点的 kubectl/API 访问。生产环境关闭内网端点前确认无业务依赖。

```bash
# 1. 关闭端点
tccli tke DeleteClusterEndpoint --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" --IsExtranet true
# expected: exit 0

# 2. 验证已关闭（显式传 IsExtranet）
tccli tke DescribeClusterEndpointStatus --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" --IsExtranet true
# expected: Status="NotFound"
```

## 故障恢复

### 命令返回错误 (exit ≠ 0)

| 现象 | 诊断 | 根因 | 修复 |
|:--------|:----------|:------------|:-----|
| `InvalidParameter`：`SecurityGroup can not be empty` | 查入参 | 公网创建未传 `SecurityGroup` | 补 `--SecurityGroup` |
| `InvalidParameter.Param` / `ACTION_NO_AUTH`（含 `tke:clusterExtranetEndpoint`） | 读 Error 消息中的 condition | CAM 策略拒绝公网端点 | 调整 CAM；非参数名错误 |
| `ResourceUnavailable.ClusterState`（VIP：无 worker） | `DescribeClusterInstances` | 空集群不允许开 VIP 外网 | 先加节点，或改用 `CreateClusterEndpoint --IsExtranet true`（仍受 CAM 约束） |
| `ResourceNotFound.SubnetId` | `tccli vpc DescribeSubnets` | 内网端点未指定子网或子网不在集群 VPC | 用集群 VPC 内的子网 |
| `ResourceNotFound.SecurityGroup` | `tccli vpc DescribeSecurityGroups` | 安全组不存在 | 重建安全组或换一个 |
| `FailedOperation` | `DescribeClusterEndpointStatus` → `ErrorMsg` | 端点创建中或 CLB 资源不足 | 等待；超时查 ErrorMsg |
| `UnsupportedOperation` | `DescribeClusterStatus` 查看状态 | 集群非 Running | 等集群 Running 后重试 |
| `ResourceInUse` | `DescribeClusterEndpointStatus` | 端点已存在 | 先 `DeleteClusterEndpoint` 再建 |

### 命令成功但状态不对 (exit = 0)

| 现象 | 诊断 | 根因 | 修复 |
|:--------|:----------|:------------|:-----|
| 长时间停在 `Creating` | `DescribeClusterEndpointStatus` → `ErrorMsg` | CLB 创建失败或子网 IP 不足 | 查 ErrorMsg，换子网或提工单 |
| 端点 `Created` 但本机 kubectl 超时 | 看 kubeconfig `server` 是内网还是公网 IP | 只开了内网端点，本机不在 VPC | 开公网端点 + `ModifyClusterEndpointSP`，或从 VPC 内访问 |
| 公网端点 `Created` 但 kubectl 被拒 | `kubectl get nodes --v=6` <!-- kubectl诊断端点ACL连通性，非tccli边界 --> | ACL 未放行出口 IP | `ModifyClusterEndpointSP` 加 `"<IP>/32"` |
| 公网端点 `Created` 但 `ClusterExternalEndpoint` 为空 | `DescribeClusterEndpoints` | CLB VIP 分配延迟 | 等 1-2 分钟重查 |

## 收尾确认

> kubectl（K8s 原生命令，非 tccli；TCCLI 管 TKE 抽象层不提供 K8s 资源操作能力）
```bash
tccli tke DescribeClusterEndpointStatus --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" --IsExtranet true \
  --filter "{status:Status}"
# expected: status=Created（公网）

tccli tke DescribeClusterEndpoints --region ap-guangzhou --ClusterId "<CLUSTER_ID>" \
  --filter "ClusterExternalEndpoint" --output text
# expected: 非空公网 IP

tccli tke DescribeClusterKubeconfig --region ap-guangzhou --ClusterId "<CLUSTER_ID>" \
  --filter "Kubeconfig" --output text > kubeconfig.yaml
<!-- kubectl验证tccli开启端点后API Server可达，非tccli边界 -->
kubectl --kubeconfig kubeconfig.yaml get nodes
# expected: 节点列表（本机须走公网端点 + ACL；仅内网 VIP 时本机超时）
```

> 端点 `Status=Created` + 地址非空 +（公网场景）ACL 含本机 IP + kubectl 可连通 = 闭环。

---

## 下一步

- [查询集群](../clusters/query.md) — `DescribeClusterEndpoints` 看端点地址
- [认证配置](../security/auth.md) — 用端点 + kubeconfig 配置 kubectl
- [配置 VPC-CNI](vpc-cni.md) — Pod 网络模型
- [故障排查](../troubleshooting.md) — 端点不通的诊断
