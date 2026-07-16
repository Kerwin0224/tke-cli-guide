---
doc_type: How-to
subtype: 6A
fused: true
---
# 管理集群访问端点

> 控制台: [容器服务控制台 - 集群访问地址](https://console.cloud.tencent.com/tke2/cluster)
> 开启/关闭集群的公网或内网访问端点。端点是 kubectl/API 访问 API Server 的入口。异步操作。

## 本机 / 公网 CI 唯一路径（优先照抄）

> **目标**：本机或公网 CI 上 `kubectl` 能连 API Server。  
> **现行唯一推荐 API 链**：`CreateClusterEndpoint --IsExtranet true` → `ModifyClusterEndpointSP` → 用 `ClusterExternalEndpoint` 改写 kubeconfig `server`。  
> **不要**用已废弃的 `CreateClusterEndpointVip` 绕路；**不要**只开内网端点（本机必超时）。

| 步 | 动作 | 成功判据 |
|:--:|:-----|:---------|
| 0 | 集群 `Running` 且 **≥1 worker** | `DescribeClusterInstances` / 节点数 ≥1；无 worker 不能开公网端点 |
| 1 | `CreateClusterEndpoint --IsExtranet true`（常需 `--SecurityGroup` + `ExtensiveParameters`） | 返回 RequestId |
| 2 | waiter `DescribeClusterEndpointStatus --IsExtranet true` | **`Status=Created`**（不是 `Running`） |
| 3 | 查出口 IP → `ModifyClusterEndpointSP --SecurityPolicies '["<IP>/32"]'` | exit 0；若 `FailedOperation.LbCommon`：等 15–60s 重试 |
| 4 | `DescribeClusterEndpoints` | `ClusterExternalEndpoint` **非空**；`ClusterExternalACL` **含**你的 CIDR |
| 5 | `DescribeClusterKubeconfig` 写盘，**把 `server` 改成** `https://` + `ClusterExternalEndpoint` | 不依赖可能 NXDOMAIN 的 `cls-*.ccs.tencent-cloud.com` |
| 6 | `kubectl --kubeconfig … get --raw=/healthz` 与 `get nodes` | `/healthz` → `ok`；`get nodes` 成功 |

**反模式（禁止）**

| 禁止 | 原因 |
|:-----|:-----|
| 用 `CreateClusterEndpointVip` 代替公网端点 / 绕 `LbCommon` | 官方 **不再维护、准备下线**，请用 `CreateClusterEndpoint`（[39413](https://cloud.tencent.com/document/product/457/39413)） |
| `Status` 未到 `Created` 或 `ClusterExternalEndpoint` 仍空就 `ModifyClusterEndpointSP` | 易 `FailedOperation.LbCommon`；ACL 未生效 |
| `ClusterExternalACL` 为 `[]` 或未含本机出口 IP 就跑 kubectl | 公网 ACL **默认拒绝所有** |
| 把 `SecurityGroup` / `SecurityGroupId` 当成 `SecurityPolicies` | ACL 只认 CIDR **字符串数组** |
| 本机场景只开 `IsExtranet false` 内网端点 | 本机不在 VPC 时必超时 |
| 端点 `Created` 后不改写 kubeconfig `server` | 域名 NXDOMAIN / 指到内网 VIP → `no such host` 或超时 |

可复制完整命令见下文 [步骤 2–5](#步骤-2开启公网端点本机--公网-ci)；Quickstart 摘要：[tke-first-cluster — 本机 kubectl 可达](../../quickstart/tke-first-cluster.md#本机-kubectl-可达必做若目标是本机公网-ci-操作集群)。

## 触发条件

- `DescribeClusterEndpointStatus` 返回 `Status=NotFound`，需从公网或 VPC 内网访问 API Server
- `kubectl get nodes` <!-- kubectl验证端点连通性，非tccli边界 --> 报 `Unable to connect to the server` / `context deadline exceeded` / `connection refused`，集群端点未开启，或本机不在端点可达网络（内网 VIP 从公网不可达）
- 公网端点已 `Created` 但 `ModifyClusterEndpointSP` 未放行你的出口 IP，kubectl 被 ACL 拒绝 — 看 [故障恢复]段
- 本机/公网 CI 需要 kubectl：走文首 **唯一路径**（公网端点 + ACL + 改写 `server`）

## 概述

集群创建后默认无访问端点（`DescribeClusterEndpointStatus` 返回 `NotFound`）。需显式开启后，kubectl 才能连 API Server。

| 端点 | 接口 | 本机/公网 CI 能否直连 | 安全 |
|:-----|:-----|:---------------------|:-----|
| **公网端点（本机/公网 CI 必选）** | `CreateClusterEndpoint --IsExtranet true` | **能**（再配 ACL 白名单） | `ModifyClusterEndpointSP` 放行出口 IP `/32`；未配默认拒绝 |
| 内网端点（仅同 VPC / 专线 / VPN） | `CreateClusterEndpoint --IsExtranet false` | **不能**（本机不在 VPC 时） | VPC 隔离 |
| ~~VIP 外网端口~~ | ~~`CreateClusterEndpointVip`~~ | **禁止新用** | 官方废弃，见 [独立 VIP 端点](#独立-vip-端点已废弃禁止新用) |

操作是**异步**的：`CreateClusterEndpoint` 返回即提交；就绪时 `DescribeClusterEndpointStatus` 的 `Status` 为 **`Created`**（不是 `Running`）。完整枚举：`NotFound`（未开）/ `Creating`（开启中）/ `Created`（成功）/ `CreateFailed`（失败，看 `ErrorMsg`）。关闭后回到 `NotFound`。

> **本机 kubectl**：必须走公网端点（或本机已接入集群 VPC）。只开内网端点时，`DescribeClusterKubeconfig` 里的 `server` 是内网 VIP，本机访问会超时。

> 官方文档：[创建集群访问端口 CreateClusterEndpoint](https://cloud.tencent.com/document/api/457/39414) · [修改外网端口安全策略 ModifyClusterEndpointSP](https://cloud.tencent.com/document/api/457/39408) · [容器网络概述](https://cloud.tencent.com/document/product/457/50353) · [网络方案选型](https://cloud.tencent.com/document/product/457/106561) · [容器服务安全组设置](https://cloud.tencent.com/document/product/457/9084)
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

| 你的位置 | 必选端点 | 说明 |
|:---------|:---------|:-----|
| **本机 / 公网 CI**（评测、本地开发、公网流水线） | **公网** `IsExtranet true` | 走文首 [唯一路径](#本机--公网-ci-唯一路径优先照抄)；**不要**选内网 |
| **生产 / 同 VPC 内**（CVM、同 VPC Pod、专线/VPN 已接入） | **内网** `IsExtranet false` | VPC 隔离、不暴露公网；本机不在 VPC 时**不可**用内网冒充「推荐」 |

- **能切换吗?**：能。`SwitchClusterEndpoint` 切换公网/内网，或先 `DeleteClusterEndpoint` 再建。
- **禁止**：把「生产用内网」当成默认，导致本机场景只开内网 → kubectl 超时。

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

> 部分账号对公网端点有 CAM 条件拒绝（消息含 `tke:clusterExtranetEndpoint` = `true` 的 deny）→ `InvalidParameter.Param` / `ACTION_NO_AUTH`，须改 CAM，非参数拼写问题。**公网端点**（`CreateClusterEndpoint --IsExtranet true`）要求集群已有 worker 节点，否则 `ResourceUnavailable.ClusterState`（`cluster without worker Node is not allowed to enable extranet access`）——先加节点再开端点。已废弃的 VIP 接口同样有 worker 约束，但**禁止新用 VIP**。

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

> **时序（agent 易错）**：
> 1. 先 `DescribeClusterEndpointStatus --IsExtranet true` 等到 **`Created`**，再 `ModifyClusterEndpointSP`。  
> 2. 若返回 `FailedOperation.LbCommon`：等待 15–60s 后重试；可用 `DescribeClusterEndpoints` 看 `ClusterExternalEndpoint` 是否已非空。  
> 3. 改完 ACL 后查 `ClusterExternalACL`：若仍为 `[]` 或未含你的 CIDR，**不要**假定 kubectl 会通。  
> 4. **禁止**把 `SecurityGroupId` 当作 `SecurityPolicies` 的替代字段传入 `ModifyClusterEndpointSP`。

```bash
# 核对 ACL 是否生效（公网端点）
tccli tke DescribeClusterEndpoints --region ap-guangzhou --ClusterId "<CLUSTER_ID>"   --filter "{ext:ClusterExternalEndpoint,acl:ClusterExternalACL}" --output json
# expected: ext 非空；acl 含 "<YOUR_EGRESS_IP>/32"（或你放行的 CIDR）
```

### 步骤 5：取 kubeconfig、改写 server、验证连通

> **本机默认动作（必做）**：写出 kubeconfig 后，**必须**用 `ClusterExternalEndpoint` 作为 `server`（`https://` + 该字段）。不要假设 `ClusterDomain`（`cls-*.ccs.tencent-cloud.com`）一定能解析。

```bash
REGION=ap-guangzhou
CLUSTER_ID="<CLUSTER_ID>"
KUBECONFIG_PATH=kubeconfig.yaml

# 1) 取 kubeconfig
tccli tke DescribeClusterKubeconfig --region "$REGION" --ClusterId "$CLUSTER_ID" \
  --filter "Kubeconfig" --output text > "$KUBECONFIG_PATH"
# expected: 文件非空 YAML

# 2) 取公网入口与 ACL（ACL 仍空则先回到步骤 4，禁止直接 kubectl）
tccli tke DescribeClusterEndpoints --region "$REGION" --ClusterId "$CLUSTER_ID" \
  --filter "{ext:ClusterExternalEndpoint,domain:ClusterDomain,acl:ClusterExternalACL}" --output json
# expected: ext 非空（形如 lb-xxxx.clb.*.tencentclb.com:443）；acl 含你的 CIDR

# 3) 用 ClusterExternalEndpoint 改写 server（可执行；macOS/Linux 通用 python 改写）
EXT="$(tccli tke DescribeClusterEndpoints --region "$REGION" --ClusterId "$CLUSTER_ID" \
  --filter "ClusterExternalEndpoint" --output text | tr -d '[:space:]')"
# expected: EXT 非空
test -n "$EXT"
case "$EXT" in
  https://*) SERVER="$EXT" ;;
  *) SERVER="https://${EXT}" ;;
esac
python3 - "$KUBECONFIG_PATH" "$SERVER" <<'PY'
import re, sys
path, server = sys.argv[1], sys.argv[2]
text = open(path, encoding="utf-8").read()
new, n = re.subn(r"(?m)^(\s*server:\s*).*$", r"\1" + server, text, count=1)
if n != 1:
    raise SystemExit(f"rewrite server failed: replacements={n}")
open(path, "w", encoding="utf-8").write(new)
print("server ->", server)
PY
# expected: 打印 server -> https://...

# 4) kubectl 验证（K8s 原生命令，非 tccli）
kubectl --kubeconfig "$KUBECONFIG_PATH" get --raw=/healthz --request-timeout=20s
# expected: ok

kubectl --kubeconfig "$KUBECONFIG_PATH" get nodes --request-timeout=20s
# expected: 节点列表；命令成功即 reach 成功
```

> **域名 NXDOMAIN**：部分环境对 `cls-*.ccs.tencent-cloud.com` **无法解析**，未改写 `server` 会 `no such host`。以 `ClusterExternalEndpoint` 为准（值已含端口则勿再拼 `:443`）。

### 步骤 6：验证端点状态

```bash
tccli tke DescribeClusterEndpointStatus --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" --IsExtranet true
# expected: Status="Created"
```

| 维度 | 命令 | 预期 |
|:-----|:-----|:-----|
| 端点状态 | `DescribeClusterEndpointStatus` → `Status` | `Created`（未开 `NotFound`；开启中 `Creating`；失败 `CreateFailed`） |
| 端点地址 | `DescribeClusterEndpoints` → `ClusterExternalEndpoint` / `ClusterIntranetEndpoint` | 非空（CLB host 或 IP） |
| ACL | `DescribeClusterEndpoints` → `ClusterExternalACL` | 含你放行的 CIDR（若已 Modify） |
| DNS | `dig +short` 对 `ClusterDomain` | 可解析 **或** 不可解析时已改用 `ClusterExternalEndpoint` |
| kubectl | `kubectl --kubeconfig kubeconfig.yaml get --raw=/healthz` | `ok` |

### 独立 VIP 端点（已废弃，禁止新用）

> ⚠️ **官方状态**：`CreateClusterEndpointVip` / `DeleteClusterEndpointVip` / `DescribeClusterEndpointVipStatus` **不再维护，准备下线**。请使用新接口 **`CreateClusterEndpoint`**（[官方 39413](https://cloud.tencent.com/document/product/457/39413) · [CreateClusterEndpoint 39414](https://cloud.tencent.com/document/api/457/39414)）。
>
> **禁止**：在 `ModifyClusterEndpointSP` 报 `FailedOperation.LbCommon` 时改走 VIP「绕路」；正确做法是等公网端点 `Created` + `ClusterExternalEndpoint` 非空后，间隔 15–60s **重试** `ModifyClusterEndpointSP`（见文首反模式与 [故障恢复](#故障恢复)）。
>
> 以下命令仅供**存量 VIP 清理/查询**历史兼容，**新集群不要创建 VIP**。

```bash
# （历史兼容）查询存量 VIP 状态 — 新开通请用 CreateClusterEndpoint
tccli tke DescribeClusterEndpointVipStatus --ClusterId "<CLUSTER_ID>" --region <REGION>
# expected: 仅用于存量 VIP 查询；无 VIP 时按实际响应；禁止作为新开通路径

# （历史兼容）删除存量 VIP
tccli tke DeleteClusterEndpointVip --ClusterId "<CLUSTER_ID>" --region <REGION>
# expected: 仅清理存量 VIP；新集群无 VIP 可删；禁止代替 DeleteClusterEndpoint
```

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
| `ResourceUnavailable.ClusterState`（消息含 `without worker Node` / 无 worker） | `DescribeClusterInstances` / `DescribeClusterStatus` → `ClusterRunningNodeNum` | **空集群不允许开公网/VIP 外网端点** | 先加 ≥1 worker（[创建节点池](../nodes/nodepool-create.md) 或 [CreateClusterInstances](../nodes/instance-ops.md#新建-cvm-作节点createclusterinstances)），再 `CreateClusterEndpoint` |
| `ResourceNotFound.SubnetId` | `tccli vpc DescribeSubnets` | 内网端点未指定子网或子网不在集群 VPC | 用集群 VPC 内的子网 |
| `ResourceNotFound.SecurityGroup` | `tccli vpc DescribeSecurityGroups` | 安全组不存在 | 重建安全组或换一个 |
| `FailedOperation` | `DescribeClusterEndpointStatus` → `ErrorMsg` | 端点创建中或 CLB 资源不足 | 等待；超时查 ErrorMsg |
| `FailedOperation.LbCommon`（`ModifyClusterEndpointSP`） | `DescribeClusterEndpointStatus` 是否已 `Created`；`DescribeClusterEndpoints` → ext/acl | CLB 未绑定完成就改 ACL，或参数非 CIDR 字符串数组 | **等 `Created` 且 ext 非空后再改**；间隔 15–60s 重试；确认 `SecurityPolicies` 为 `'["x.x.x.x/32"]'`；成功后核对 `ClusterExternalACL` 非空。**禁止**改走 `CreateClusterEndpointVip`（已废弃） |
| 想用 VIP 绕过 LbCommon / 公网开通失败 | 查官方 39413 | VIP 接口不再维护 | **只用** `CreateClusterEndpoint` + 重试 SP；见文首反模式 |
| `UnsupportedOperation` | `DescribeClusterStatus` 查看状态 | 集群非 Running | 等集群 Running 后重试 |
| `ResourceInUse` | `DescribeClusterEndpointStatus` | 端点已存在 | 先 `DeleteClusterEndpoint` 再建 |

### 命令成功但状态不对 (exit = 0)

| 现象 | 诊断 | 根因 | 修复 |
|:--------|:----------|:------------|:-----|
| 长时间停在 `Creating` | `DescribeClusterEndpointStatus` → `ErrorMsg` | CLB 创建失败或子网 IP 不足 | 查 ErrorMsg，换子网或提工单 |
| 端点 `Created` 但本机 kubectl 超时 | 看 kubeconfig `server` 是内网还是公网 | 只开了内网端点，本机不在 VPC | 开公网端点 + ACL，或从 VPC 内访问 |
| 端点 `Created` 但 `lookup cls-*.ccs.tencent-cloud.com: no such host` / NXDOMAIN | `dig +short <ClusterDomain>`；`DescribeClusterEndpoints` → `ClusterExternalEndpoint` | kubeconfig 默认域名不可解析 | **把 kubeconfig `server` 改为 `https://` + `ClusterExternalEndpoint`**（见 [步骤 5](#步骤-5取-kubeconfig-并验证连通)） |
| 公网端点 `Created` 但 kubectl 被拒 | `kubectl get nodes --v=6` <!-- kubectl诊断端点ACL连通性，非tccli边界 --> | ACL 未放行出口 IP | `ModifyClusterEndpointSP` 加 `"<IP>/32"` |
| 公网端点 `Created` 但 `ClusterExternalEndpoint` 为空 | `DescribeClusterEndpoints` | CLB VIP 分配延迟 | 等 1-2 分钟重查 |

## 收尾确认

> 公网/本机场景：**四条件全部满足**才算任务完成（缺一不可）。

| # | 条件 | 命令 / 字段 | 预期 |
|:--|:-----|:------------|:-----|
| 1 | 公网端点就绪 | `DescribeClusterEndpointStatus --IsExtranet true` → `Status` | `Created` |
| 2 | 公网地址非空 | `DescribeClusterEndpoints` → `ClusterExternalEndpoint` | 非空 CLB host:port |
| 3 | **ACL 已放行** | `DescribeClusterEndpoints` → `ClusterExternalACL` | **非空**且含本机/CI 出口 CIDR（如 `x.x.x.x/32`） |
| 4 | **kubectl 可达** | `kubectl --kubeconfig … get --raw=/healthz` | **`ok`**（另建议 `get nodes` 成功） |

```bash
REGION=ap-guangzhou
CLUSTER_ID="<CLUSTER_ID>"
KUBECONFIG_PATH=kubeconfig.yaml

tccli tke DescribeClusterEndpointStatus --region "$REGION" \
  --ClusterId "$CLUSTER_ID" --IsExtranet true \
  --filter "{status:Status}"
# expected: status=Created

tccli tke DescribeClusterEndpoints --region "$REGION" --ClusterId "$CLUSTER_ID" \
  --filter "{ext:ClusterExternalEndpoint,acl:ClusterExternalACL,domain:ClusterDomain}" --output json
# expected: ext 非空；acl 为非空数组且含你的 CIDR；若 domain 不可 dig 通则 kubeconfig server 须已用 ext

# 若尚未改写 server，先执行步骤 5 的 python 改写，再：
kubectl --kubeconfig "$KUBECONFIG_PATH" get --raw=/healthz --request-timeout=20s
# expected: ok
kubectl --kubeconfig "$KUBECONFIG_PATH" get nodes --request-timeout=20s
# expected: 节点列表或空列表但命令成功（本机须走公网端点；仅内网 VIP 时本机超时）
```

> **闭环公式**：`Status=Created` + `ClusterExternalEndpoint` 非空 + **`ClusterExternalACL` 含出口 IP** + **`/healthz=ok`**。  
> ACL 仍为 `[]` 或未改写 `server` 时，**不得**宣称端点任务完成。

---

## 下一步

- [查询集群](../clusters/query.md) — `DescribeClusterEndpoints` 看端点地址
- [认证配置](../security/auth.md) — 用端点 + kubeconfig 配置 kubectl
- [配置 VPC-CNI](vpc-cni.md) — Pod 网络模型
- [故障排查](../troubleshooting.md) — 端点不通的诊断
