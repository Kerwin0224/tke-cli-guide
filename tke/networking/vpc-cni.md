---
doc_type: How-to
subtype: 6B
fused: false
---
# 配置 VPC-CNI 网络

> 控制台: [容器服务控制台 - 集群网络](https://console.cloud.tencent.com/tke2/cluster)
> 开启/关闭 VPC-CNI 网络模型。VPC-CNI 让 Pod 直接从 VPC 子网获取 IP，支持固定 IP 与安全组直通。配置型操作（改变行为，不创建/销毁资源）。

## 触发条件

- `DescribeClusters` → `NetworkType` 含 `GR`（Global Router）但需 Pod 固定 IP 或安全组直通，要开启 VPC-CNI
- `DescribeEnableVpcCniProgress` 返回 `Status` 非 `Succeed`（枚举为 `Running`/`Succeed`/`Failed`），或 Pod 卡在 `ContainerCreating` 且 `kubectl describe pod` <!-- tccli管VPC-CNI配置，kubectl describe查Pod详情诊断IP分配，非tccli边界 --> 显示 IP 分配失败
- `DescribeIPAMD` → `EnableIPAMD=false`，需开启 VPC-CNI 让 Pod 从 VPC 子网获 IP — 看 [故障恢复]段


## 概述

VPC-CNI 是 TKE 的三种 Pod 网络模型之一（另两种：Global Router / CiliumOverlay）。开启后，新建 Pod 可从指定 VPC 子网获取 IP，与 CVM 同级，支持安全组直通与固定 IP。

| 模型 | Pod IP 来源 | 固定 IP | 安全组直通 | IP 消耗 | 后期扩网段 |
|:-----|:-----------|:------:|:----------|:--------|:----------:|
| Global Router（默认） | 容器网段 | ❌ | ❌ | 少 | ✅ `AddClusterCIDR` |
| VPC-CNI | VPC 子网 | ✅ | ✅ | 多（每 Pod 一个 VPC IP） | ❌ |
| CiliumOverlay | Overlay 隧道 | ❌ | ❌ | 少（不占 VPC IP） | ❌ |

> VPC-CNI 可与 Global Router 共存：Global Router 为主，VPC-CNI 子网补充。开启 VPC-CNI 不影响已有 Global Router Pod。CiliumOverlay 与两者互斥（创建时定型，不可切换），见 [配置 CiliumOverlay](cilium-overlay.md)。

> 官方文档：[容器网络概述](https://cloud.tencent.com/document/product/457/50353) · [网络方案选型](https://cloud.tencent.com/document/product/457/106561) · [容器集群网络规划](https://cloud.tencent.com/document/product/457/106706)
> 配额：VPC 子网可用 IP 数决定 VPC-CNI Pod 上限；容器网段 CIDR 创建时自定义暂不支持变更。[配额限制](https://cloud.tencent.com/document/product/457/9087)
> ⚠️ **高危操作**：开启 VPC-CNI 后不可回退至 GlobalRouter（关闭前须先迁移已有 VPC-CNI Pod）；子网 IP 耗尽致新 Pod 无法调度。[常见高危操作](https://cloud.tencent.com/document/product/457/39539)

### IPAMD 服务角色

> **服务角色** `IPAMDofTKE_QCSRole` 与集群内 **eniipamd 组件** 是两层：前者是 CAM 授权 TKE IPAMD 访问 CVM/VPC/ENI；后者是集群里跑的 DaemonSet/Deploy。缺角色时控制台/API 侧无法正常完成 VPC-CNI 授权路径，与 `DescribeIPAMD` 的 `EnableIPAMD=false`（组件未开）不同。官方： [43416 — IPAMDofTKE_QCSRole](https://cloud.tencent.com/document/product/457/43416)。

```bash
# 探测
tccli cam DescribeRoleList --Page 1 --Rp 100 \
  --filter "List[?RoleName=='IPAMDofTKE_QCSRole'].RoleName" --output text
# expected: IPAMDofTKE_QCSRole；空 → 补齐

# 补齐（Principal = ccs.qcloud.com；策略名以 ListPolicies 为准）
tccli cam CreateRole \
  --RoleName IPAMDofTKE_QCSRole \
  --Description "TKE IPAMD service role for ENI and VPC resources" \
  --PolicyDocument '{"version":"2.0","statement":[{"effect":"allow","action":"sts:AssumeRole","principal":{"service":"ccs.qcloud.com"}}]}'
# expected: RoleId；已存在则跳过

tccli cam AttachRolePolicy \
  --AttachRoleName IPAMDofTKE_QCSRole \
  --PolicyName QcloudAccessForIPAMDofTKERole
# expected: RequestId
```

| 项 | 说明 |
|:---|:-----|
| 何时必查 | 创建时 `NetworkType=VPC-CNI`、或事后 `EnableVpcCniNetworkType` |
| 与 TKE_QCSRole | **另一条**角色，不能互相替代；见 [配置凭证 — 服务角色](../../getting-started/credentials.md#服务角色tke--ipamd--as--tcr--可观测) |
| 创建时 VPC-CNI | [创建集群](../clusters/create.md) 资源检查第 6 步；[quickstart](../../quickstart/tke-first-cluster.md) 准备工作第 6 行 |

>
> **eniipamd 组件**：VPC-CNI 依赖集群内 `tke-eni-agent` / `tke-eni-ipamd` / `tke-eni-ip-scheduler`（Addon 名常为 `eniipamd`）。三组件版本一般相同，`tke-eni-ip-scheduler` 可能略旧。排障或升级前用镜像 Tag 核对版本；变更记录见 [VPC-CNI（eniipamd）组件变更记录](https://cloud.tencent.com/document/product/457/64920)。安装/升级走 [插件管理](../addons/manage.md)。

> kubectl（K8s 原生命令，非 tccli；TCCLI 管 TKE 抽象层不提供 K8s 资源操作能力）
```bash
# 核对 eniipamd 相关组件镜像 Tag（版本）
<!-- tccli管VPC-CNI开启/关闭/子网配置，kubectl查K8s层组件镜像版本，非tccli边界 -->
kubectl -n kube-system get ds tke-eni-agent -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
kubectl -n kube-system get deploy tke-eni-ipamd -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
kubectl -n kube-system get deploy tke-eni-ip-scheduler -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
# expected: 镜像地址含版本 Tag；组件未装则报 NotFound
```

## 决策依据

#### 为什么用 VPC-CNI

- **VPC-CNI vs Global Router vs CiliumOverlay**: VPC-CNI 让 Pod 拿 VPC IP，支持固定 IP（StatefulSet 稳定地址）与安全组直通（Pod 级网络策略）；Global Router 的 Pod 用容器网段 IP，不暴露到 VPC；CiliumOverlay 用 Overlay 隧道，不占 VPC IP 但无固定 IP/安全组直通，适合要 Cilium 数据面的场景（见 [配置 CiliumOverlay](cilium-overlay.md)）
- **默认推荐**: 不需要固定 IP / 安全组直通时用 Global Router（契约默认 `NetworkType=GR`）。需要固定 IP 或安全组直通时开启 VPC-CNI；需要 Cilium 数据面且接受创建时定型时选 CiliumOverlay
- **可关闭**：VPC-CNI 可关闭，`DisableVpcCniNetworkType`（已有 VPC-CNI Pod 需先迁移）。CiliumOverlay 不可关闭——创建时定型，无独立开关 Action
- **与 IPVS 的关系**: IPVS / kube-proxy 模式在**创建集群**时选定，开启 IPVS 后不可关闭（见 [网络管理 — 转发模式半常量](index.md#转发模式半常量与-networktype-正交)）；开启 VPC-CNI **不改变**已选定的转发模式

## 配置项

| 字段 | 类型 | 必填 | 默认值 | 有效值 | 填错的影响 |
|:------|------|:--------:|:------:|-------|-----------|
| ClusterId | string | 是 | — | `cls-xxxxxxxx` | `ResourceNotFound` |
| VpcCniType | string | 是 | — | `tke-route-eni` / `tke-direct-eni` | IP 分配方式不对 |
| EnableStaticIp | boolean | 是 | — | `true`/`false` | 固定 IP 不生效 |
| Subnets | list | 是 | — | VPC 子网 ID 列表 | `ResourceNotFound.SubnetId` |
| ExpiredSeconds | int | 条件 | 0 | 固定 IP 回收秒数；`EnableStaticIp=true` 时必填且须 >300，不传默认 IP 永不销毁 | IP 回收时机不对 |
| SkipAddingNonMasqueradeCIDRs | boolean | 否 | false | `true`/`false` | 路由配置影响 |

> `VpcCniType`: `tke-route-eni`（策略路由/共享网卡多 IP，常用）/ `tke-direct-eni`（独立网卡）。**不是** `tke-direct-route-eni`。`EnableStaticIp=true` 开启固定 IP，配合 `ExpiredSeconds` 设回收时间（须 >300 秒）。
>
> ⚠️ **必填对齐 Action 入参契约**：`EnableStaticIp` 在 `EnableVpcCniNetworkType` 入参中为必填（非可选）——开启 VPC-CNI 时必须显式传 `true`/`false` 声明是否固定 IP；`ExpiredSeconds` 是条件必填（`EnableStaticIp=true` 时必填且 >300）。完整入参以 `tccli tke EnableVpcCniNetworkType help --detail` 为准。

## 应用

### 开启 VPC-CNI

```bash
tccli tke EnableVpcCniNetworkType --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" \
  --VpcCniType tke-route-eni \
  --EnableStaticIp true \
  --Subnets '["<SUBNET_ID>"]' \
  --ExpiredSeconds 300
# expected: exit 0, 返回 RequestId
```

| 占位符 | 含义 | 约束 | 如何获取 |
|:------------|:-----|:-----|:---------|
| `<CLUSTER_ID>` | 集群 ID | `cls-xxxxxxxx` | `tccli tke DescribeClusters` → `Clusters[].ClusterId` |
| `<SUBNET_ID>` | VPC 子网 ID | 须在集群 VPC 内，且有可用 IP | `tccli vpc DescribeSubnets` |

> 子网 IP 数量决定可运行的 VPC-CNI Pod 数。IP 不足时新 Pod 卡在 ContainerCreating。

### 查询 Pod 上限（按机型）

```bash
# 查某机型在该可用区的 VPC-CNI Pod 上限
tccli tke DescribeVpcCniPodLimits --region ap-guangzhou \
  --Zone "ap-guangzhou-3" --InstanceType "S5.MEDIUM4"
# expected: 该机型支持的 Pod 上限
```

> `DescribeVpcCniPodLimits` 入参是 `Zone`/`InstanceFamily`/`InstanceType`（CVM 机型维度），非 ClusterId。不同机型支持的 VPC-CNI Pod 数不同。

### 查询 IPAMD 状态

> `DescribeIPAMD` 诊断 VPC-CNI IPAMD 组件状态（IPAMD 负责 Pod IP 分配）。

```bash
tccli tke DescribeIPAMD --ClusterId "<CLUSTER_ID>" --region <REGION>
# expected: exit 0, EnableIPAMD/Phase/SubnetIds 反映 IPAMD 状态
```
```json
{
    "EnableIPAMD": false,
    "EnableCustomizedPodCidr": false,
    "DisableVpcCniMode": false,
    "Phase": "",
    "Reason": "",
    "SubnetIds": null,
    "ClaimExpiredDuration": "",
    "EnableTrunkingENI": false
}
```

> `EnableIPAMD=false` 表示未启用 IPAMD（集群非 VPC-CNI 或未开）。`Phase` 反映运行阶段，`Reason` 非空表示异常原因。`EnableTrunkingENI` 是中继弹性网卡模式。

### 集群路由表管理

> 集群路由表（`ClusterRouteTable`）是 Global Router 模式的底层路由，TKE 用它把 Pod 网段路由到节点。路由表用 `RouteTableName`（字符串名）标识，**不绑 ClusterId**。

```bash
# 查询所有集群路由表 (无入参)
tccli tke DescribeClusterRouteTables --region <REGION>
# expected: exit 0, RouteTableSet[] 含 RouteTableName/RouteTableCidrBlock/VpcId
```
```json
{
    "TotalCount": 2,
    "RouteTableSet": [
        {"RouteTableName": "rt-example", "RouteTableCidrBlock": "10.20.0.0/16", "VpcId": "vpc-example"}
    ]
}
```

```bash
# 查询路由表冲突 (建表前检查 CIDR 是否与 VPC 已有路由冲突)
tccli tke DescribeRouteTableConflicts --region <REGION> \
  --RouteTableCidrBlock "<CIDR>" --VpcId "<VPC_ID>"
# expected: exit 0, 返回冲突列表 (无冲突则空)

# 创建路由表
tccli tke CreateClusterRouteTable --region <REGION> \
  --RouteTableName "<RT_NAME>" --RouteTableCidrBlock "<CIDR>" --VpcId "<VPC_ID>" --IgnoreClusterCidrConflict 0
# expected: exit 0

# 查询路由表下的路由
tccli tke DescribeClusterRoutes --region <REGION> --RouteTableName "<RT_NAME>"
# expected: exit 0, TotalCount + 路由条目

# 添加路由
tccli tke CreateClusterRoute --region <REGION> \
  --RouteTableName "<RT_NAME>" --DestinationCidrBlock "<DEST_CIDR>" --GatewayIp "<GW_IP>"
# expected: exit 0

# 删除路由
tccli tke DeleteClusterRoute --region <REGION> \
  --RouteTableName "<RT_NAME>" --GatewayIp "<GW_IP>" --DestinationCidrBlock "<DEST_CIDR>"
# expected: exit 0

# 删除路由表
tccli tke DeleteClusterRouteTable --region <REGION> --RouteTableName "<RT_NAME>"
# expected: exit 0
```

> ⚠️ 路由操作用 `RouteTableName`（非 ClusterId，非路由表 ID）。`DeleteClusterRoute` 需同时传 `RouteTableName`+`GatewayIp`+`DestinationCidrBlock` 三者定位路由。`CreateClusterRouteTable` 的 `IgnoreClusterCidrConflict` 是 Integer（0/1）非 Boolean。建表前用 `DescribeRouteTableConflicts` 查冲突。

## 验证

```bash
# 查询开启进度（仅已是 / 正在开启 VPC-CNI 的集群可用）
# 非 VPC-CNI 集群会报 FailedOperation.EnableVPCCNIFailed: ... is not vpc-cni cluster
tccli tke DescribeEnableVpcCniProgress --region ap-guangzhou --ClusterId "<CLUSTER_ID>"
# expected: Status="Succeed"（进行中 Running / 失败 Failed）；出参仅 Status/ErrorMessage（无进度百分比字段）

# 未开启时先用 IPAMD 诊断（沙箱 Global Router 常见：EnableIPAMD=false）
tccli tke DescribeIPAMD --region ap-guangzhou --ClusterId "<CLUSTER_ID>" \
  --filter "{enable:EnableIPAMD,phase:Phase,subnets:SubnetIds}"
# expected: EnableIPAMD=true 且 Phase 就绪后，再查 Progress
```

| 维度 | 命令 | 预期 |
|:-----|:-----|:-----|
| IPAMD | `DescribeIPAMD` → `EnableIPAMD` | `true`（开启后） |
| 开启进度 | `DescribeEnableVpcCniProgress` → `Status` | `Succeed`（`Running`/`Succeed`/`Failed`；非 VPC-CNI 集群勿调，会 FailedOperation） |
| 网络类型 | `DescribeClusters` → `NetworkType` | 含 `VPC-CNI` |
| Pod 获 IP | `kubectl get pod -o wide` <!-- tccli管VPC-CNI网络配置，kubectl查Pod IP状态验证IP分配，非tccli边界 --> | Pod IP 在 VPC 子网段内 |

## 回滚

```bash
# 关闭 VPC-CNI（已有 VPC-CNI Pod 需先迁移）
tccli tke DisableVpcCniNetworkType --region ap-guangzhou --ClusterId "<CLUSTER_ID>"
# expected: exit 0
```

> 关闭前确认无 VPC-CNI Pod 运行，否则这些 Pod 会失联。建议先迁移到 Global Router 节点。

## 故障恢复

### 命令返回错误 (exit ≠ 0)

| 现象 | 诊断 | 根因 | 修复 |
|:--------|:----------|:------------|:-----|
| `ResourceNotFound.SubnetId` | `tccli vpc DescribeSubnets` | 子网不在集群 VPC | 用集群 VPC 内子网 |
| `InvalidParameterValue.VpcCniType` | 查枚举 | VpcCniType 拼错 | 用 `tke-route-eni` 或 `tke-direct-eni` |
| `UnsupportedOperation` | `DescribeClusters` → `NetworkType` | 已开启 VPC-CNI 或集群非 Running | 先 Disable 或等 Running |
| 创建/开启 VPC-CNI 失败且消息含授权/IPAMD/ENI | `tccli cam DescribeRoleList --Page 1 --Rp 100 --filter "List[?RoleName=='IPAMDofTKE_QCSRole'].RoleName" --output text` | 缺 `IPAMDofTKE_QCSRole` 或未挂 `QcloudAccessForIPAMDofTKERole` | 见 [IPAMD 服务角色](#ipamd-服务角色)；总表 [配置凭证](../../getting-started/credentials.md#补-ipamdoftke_qcsrolevpc-cni-前置) |
| `UnauthorizedOperation` / CAM 拒绝（用户侧） | 查子账号策略 | 用户无 `tke:EnableVpcCniNetworkType` | 给用户挂 TKE 策略，与服务角色无关 |

### 命令成功但状态不对 (exit = 0)

| 现象 | 诊断 | 根因 | 修复 |
|:--------|:----------|:------------|:-----|
| 开启卡住进度不动 | `DescribeEnableVpcCniProgress` | 子网 IP 不足或路由冲突 | 加子网（`AddVpcCniSubnets`）或换子网 |
| Pod 卡在 ContainerCreating | `kubectl describe pod` <!-- tccli管VPC-CNI子网配置，kubectl describe查Pod详情诊断IP耗尽，非tccli边界 --> | 子网 IP 耗尽 | `AddVpcCniSubnets` 增加子网 |
| 固定 IP 不生效 | `EnableStaticIp` 值 | 未设 true 或 StatefulSet 未配 | `EnableStaticIp=true`，StatefulSet 注解 `tke.cloud.tencent.com/vpc-ips` |

### 子网 IP 不足时增加子网

> VPC-CNI Pod 卡在 ContainerCreating 多因子网 IP 耗尽。`AddVpcCniSubnets` 给已开启的 VPC-CNI 追加子网，参数以 `tccli tke AddVpcCniSubnets help --detail` 为准（`SubnetIds[]` 复数 + `VpcId`）。

```bash
# 给 VPC-CNI 追加子网（ClusterId + VpcId + SubnetIds[]）
tccli tke AddVpcCniSubnets --region ap-guangzhou \
  --ClusterId "<CLUSTER_ID>" \
  --VpcId "<VPC_ID>" \
  --SubnetIds '["<SUBNET_ID>"]'
# expected: exit 0 返回 RequestId; 子网无效报 InvalidParameter.Param: subnet <ID> is invalid
```

| 占位符 | 含义 | 如何获取 |
|:-------|:-----|:---------|
| `<VPC_ID>` | 集群所在 VPC | `tccli tke DescribeClusters` → `Clusters[].ClusterNetworkSettings.VpcId` |
| `<SUBNET_ID>` | 追加的子网 ID | `tccli vpc DescribeSubnets --Filters '[{"Name":"vpc-id","Values":["<VPC_ID>"]}]'` |

> `AddVpcCniSubnets` 用复数 `SubnetIds[]`（可一次追加多个子网），需带 `VpcId` 标识子网所属 VPC。追加后用 `DescribeEnableVpcCniProgress` 确认子网已生效，新 Pod 将从新子网获 IP。

## 收尾确认

> kubectl（K8s 原生命令，非 tccli；TCCLI 管 TKE 抽象层不提供 K8s 资源操作能力）
```bash
# VPC-CNI 开启进度完成（Verify 查进度，此处端到端核 Pod 真从子网获 IP）
tccli tke DescribeEnableVpcCniProgress --region ap-guangzhou --ClusterId "<CLUSTER_ID>" \
  --filter "{status:Status}"
# expected: status=Succeed

# 业务可用性端到端：部署测试 Pod，核 Pod IP 在指定子网段内（Verify 仅列维度未端到端验证）
<!-- tccli管VPC-CNI网络能力配置，kubectl管Pod生命周期验证IP分配，非tccli边界 -->
kubectl run vpc-cni-test --image=nginx --restart=Never
kubectl get pod vpc-cni-test -o wide --no-headers | awk '{print $6}'
# expected: Pod IP 在 <SUBNET_ID> 子网 CidrBlock 段内 → VPC-CNI 配置闭环完成
kubectl delete pod vpc-cni-test
```

> 开启进度 `Status=Succeed` + Pod IP 落在 VPC 子网段 = 端到端闭环。Verify 段查进度与开关状态，此处用真实 Pod 验证 IP 分配行为符合 VPC-CNI 契约（Pod 与 CVM 同级从子网拿 IP），是固定 IP / 安全组直通功能的前置。

---

## 下一步

- [管理访问端点](endpoints.md) — API Server 访问入口
- [配置 CiliumOverlay](cilium-overlay.md) — 第三种网络模型，Cilium 数据面
- [创建集群](../clusters/create.md) — 建集群时选网络模型
- [故障排查](../troubleshooting.md) — Pod IP 不足诊断
