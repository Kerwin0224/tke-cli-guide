---
doc_type: How-to
subtype: 6A
fused: true
---
# 管理集群访问端点

> 控制台: [容器服务控制台 - 集群访问地址](https://console.cloud.tencent.com/tke2/cluster)
> 开启或关闭集群的公网、内网访问端点。端点是 kubectl/API 访问 API Server 的入口，创建操作为异步操作。

## 先识别集群模式

`CreateClusterEndpoint` 的公网、内网能力取决于集群模式，不能把同一条公网命令用于所有集群：

| 集群模式 | `CreateClusterEndpoint` 支持范围 | 本机 / 公网 CI 的入口 |
|:---------|:---------------------------------|:----------------------|
| 独立集群 | 内网和公网 | 可创建公网端点，或先将客户端接入集群 VPC 后使用内网端点 |
| 托管集群 | 仅内网 | 新接口只创建内网端点；托管集群外网端口属于老方式，使用 `ModifyClusterEndpointSP` 管理其安全策略 |

> `ModifyClusterEndpointSP` 是**老的托管集群外网端口方式**，不要把它接到独立集群的 `CreateClusterEndpoint --IsExtranet true` 后组成统一流程。

## 触发条件

- 需要从公网或集群 VPC 内访问 API Server
- `DescribeClusterEndpointStatus` 返回 `Status=NotFound`，需要开启相应类型的端点
- `kubectl get nodes` <!-- kubectl验证端点连通性，非tccli边界 --> 报连接错误，需要核对端点类型、客户端网络位置及访问控制

## 概述

| 端点 | 接口与范围 | 客户端网络 | 访问控制 |
|:-----|:-----------|:-----------|:---------|
| 独立集群公网端点 | `CreateClusterEndpoint --IsExtranet true` | 公网可达 | 外网 CLB 绑定安全组，安全组需对客户端来源 IP 放通 443 |
| 独立或托管集群内网端点 | `CreateClusterEndpoint --IsExtranet false` | 集群 VPC、专线或 VPN | VPC 网络边界；创建时选择集群 VPC 内子网 |
| 托管集群老外网端口 | 老外网方式；`ModifyClusterEndpointSP` 仅管理该方式 | 公网可达 | `SecurityPolicies` CIDR 列表和可选 `SecurityGroup` |

创建端点后，用 `DescribeClusterEndpointStatus` 查询状态。完整枚举为 `NotFound`（未开启）、`Creating`（创建中）、`Created`（创建成功）、`CreateFailed`（创建失败，查看 `ErrorMsg`）。

> 官方文档：[连接集群](https://cloud.tencent.com/document/product/457/32191) · [创建集群访问端口](https://cloud.tencent.com/document/api/457/39414) · [查询集群访问端口状态](https://cloud.tencent.com/document/api/457/39410) · [修改托管集群外网端口安全策略](https://cloud.tencent.com/document/api/457/39408)
>
> ⚠️ **安全提示**：外网 CLB 绑定的安全组应只向实际客户端来源 IP 开放 443，不要向 `0.0.0.0/0` 无限制开放 API Server。

## 准备工作

### 检查集群状态和模式

```bash
tccli --version
# expected: tccli 版本号

tccli tke DescribeClusters --region <REGION> --ClusterIds '["<CLUSTER_ID>"]' \
  --filter "Clusters[0].{state:ClusterStatus,type:ClusterType,vpc:ClusterNetworkSettings.VpcId}" --output json
# expected: 集群状态可用，并取得集群模式与 VPC 信息
```

### 检查端点状态

```bash
# 查询公网端点状态；内网端点将 IsExtranet 改为 false
tccli tke DescribeClusterEndpointStatus --region <REGION> \
  --ClusterId "<CLUSTER_ID>" --IsExtranet true
# expected: Status 为 NotFound、Creating、Created 或 CreateFailed
```

公网访问要求集群内已有节点资源。开启前可用 `DescribeClusterInstances` 确认节点数量。

## 关键字段

> 完整入参以 `tccli tke CreateClusterEndpoint help --detail` 为准。

| 字段 | 类型 | 必填 | 约束 |
|:-----|:-----|:----:|:-----|
| `ClusterId` | string | 是 | `cls-xxxxxxxx` |
| `IsExtranet` | boolean | 是 | `true` 为公网，`false` 为内网；须遵守集群模式边界 |
| `SubnetId` | string | 内网必填 | 必须属于集群 VPC |
| `SecurityGroup` | string | 否 | 不复用已有 CLB 时，可为内网或公网端点指定安全组 |
| `ExtensiveParameters` | string | 否 | 仅用于公网设置的 JSON 字符串 |
| `Domain` | string | 否 | 自定义域名 |
| `ExistedLoadBalancerId` | string | 否 | 复用已有 CLB |

## 跨字段约束

| `IsExtranet` | `SubnetId` | `SecurityGroup` | `ExtensiveParameters` | 关系 |
|:--------------|:-----------|:----------------|:----------------------|:-----|
| `false` | 必填，且属于集群 VPC | 可选 | 不传 | 创建内网端点 |
| `true` | 不传 | 不使用已有 CLB 时可传 | 需要自定义公网 CLB 计费/带宽时传 JSON 字符串 | 创建公网端点 |

`SecurityGroup` 并非与公网或内网固定互斥；它受访问方式及是否复用已有 CLB 共同约束。

## 操作步骤

### 步骤 1：选择入口

1. 先确认集群是独立集群还是托管集群。
2. 客户端已接入集群 VPC、专线或 VPN 时，使用内网端点。
3. 独立集群的客户端需要公网访问时，可使用新接口创建公网端点。
4. 托管集群使用新接口时仅创建内网端点；其老外网端口安全策略见[管理托管集群老外网端口](#管理托管集群老外网端口)。

### 步骤 2：开启公网端点（本机 / 公网 CI）

```bash
tccli tke CreateClusterEndpoint --region <REGION> \
  --ClusterId "<CLUSTER_ID>" \
  --IsExtranet true \
  --SecurityGroup "<SECURITY_GROUP_ID>" \
  --ExtensiveParameters '{"InternetAccessible":{"InternetChargeType":"TRAFFIC_POSTPAID_BY_HOUR","InternetMaxBandwidthOut":1}}'
# expected: 返回 RequestId

tccli tke DescribeClusterEndpointStatus --region <REGION> \
  --ClusterId "<CLUSTER_ID>" --IsExtranet true \
  --waiter '{"expr":"Status","to":"Created","timeout":300,"interval":10}'
# expected: Status="Created"
```

公网端点使用外网 CLB。检查绑定到外网 CLB 的安全组，确保其入站规则仅对客户端来源 IP 放通 TCP 443。

### 步骤 3：开启内网端点（同 VPC）

独立集群和托管集群均可使用新接口创建内网端点：

```bash
tccli tke CreateClusterEndpoint --region <REGION> \
  --ClusterId "<CLUSTER_ID>" --IsExtranet false \
  --SubnetId "<SUBNET_ID>" --SecurityGroup "<SECURITY_GROUP_ID>"
# expected: 返回 RequestId；随后查询到 Status="Created"
```

内网端点可自动创建或复用已有 CLB。客户端必须位于集群 VPC 可达的网络环境中；公网客户端应先通过专线、VPN 等方式接入 VPC，或在受支持的集群模式下使用公网端点。

### 管理托管集群老外网端口

`ModifyClusterEndpointSP` 仅适用于**托管集群老外网端口**。`SecurityPolicies` 接受 CIDR 字符串数组；该接口还可通过 `SecurityGroup` 修改外网访问安全组。两者字段不同，但不能据此否定安全组对外网访问的控制能力。

```bash
tccli tke ModifyClusterEndpointSP --region <REGION> \
  --ClusterId "<CLUSTER_ID>" \
  --SecurityPolicies '["<YOUR_EGRESS_IP>/32"]' \
  --SecurityGroup "<SECURITY_GROUP_ID>"
# expected: 返回 RequestId
```

未设置 `SecurityPolicies` 时，该老方式默认拒绝所有 CIDR。无论采用 CIDR 策略还是安全组，都应遵循最小暴露原则。

### 步骤 4：将访问链路切换为直连

`SwitchClusterEndpoint` 的官方当前页（最近更新于 2025-11-26）与 `tccli tke SwitchClusterEndpoint help --detail` 均将其标为在线 Action，未标注废弃或下线。它切换的是指定公网或内网访问链路的**直连模式**，不是在公网端点与内网端点之间互换。官方页未给出集群类型或额外产品前置条件；因此只应在控制台或腾讯云支持已确认当前集群和目标端点支持直连，并且目标端点已经开启时使用，不要把下面命令泛化到所有集群。

以下为最小切换路径；`Rollback` 默认是 `false`，不要把 `true` 写成推荐值：

```bash
# 将已开启的内网端点切换为直连；公网端点将 IsExtranet 改为 true
tccli tke SwitchClusterEndpoint --region <REGION> \
  --ClusterId "<CLUSTER_ID>" --IsExtranet false --Rollback false
# expected: 返回 RequestId

# 验证目标端点仍处于已创建状态；该 Action 没有公开专用状态查询
tccli tke DescribeClusterEndpointStatus --region <REGION> \
  --ClusterId "<CLUSTER_ID>" --IsExtranet false
# expected: Status="Created"；随后按步骤 6 验证实际连通性
```

需要回退至非直连时，在相同 `IsExtranet` 目标上显式传 `--Rollback true`。该参数只表示“回滚至非直连”，不表示失败自动回滚；接口返回 `RequestId` 也不等于链路已经可用。

### 步骤 5：查询端点地址

```bash
tccli tke DescribeClusterEndpoints --region <REGION> --ClusterId "<CLUSTER_ID>" \
  --filter "{external:ClusterExternalEndpoint,intranet:ClusterIntranetEndpoint,domain:ClusterDomain,acl:ClusterExternalACL}" --output json
# expected: 已开启的端点返回相应地址；老外网端口可返回外网 ACL
```

### 步骤 6：获取 kubeconfig 并验证

公网和内网访问各有对应的 kubeconfig。优先下载与所选访问方式匹配的 kubeconfig；只有在使用自定义域名时，才按产品配置自行完成 DNS 解析，不要把手工改写 `server` 作为固定步骤。

```bash
tccli tke DescribeClusterKubeconfig --region <REGION> --ClusterId "<CLUSTER_ID>" \
  --filter "Kubeconfig" --output text > kubeconfig.yaml
# expected: 文件为非空 YAML

kubectl --kubeconfig kubeconfig.yaml get --raw=/healthz --request-timeout=20s
# expected: ok

kubectl --kubeconfig kubeconfig.yaml get nodes --request-timeout=20s
# expected: 命令成功
```

## `CreateClusterEndpointVip` 兼容边界

> ⚠️ **官方状态（禁止新建）**：`CreateClusterEndpointVip` 已明确标为不再维护、准备下线，禁止再用它创建新的托管集群外网访问端口。新开通应改用受前文集群模式约束的 `CreateClusterEndpoint`；不得为兼容旧脚本恢复 VIP 创建命令。

`DescribeClusterEndpointVipStatus` 和 `DeleteClusterEndpointVip` 仍可用于查询、清理已有 VIP；不要把创建接口的下线状态扩展为这两个 Action 也已下线。

```bash
# 查询已有 VIP
tccli tke DescribeClusterEndpointVipStatus --region <REGION> --ClusterId "<CLUSTER_ID>"
# expected: 返回已有 VIP 的实际状态

# 删除已有 VIP
tccli tke DeleteClusterEndpointVip --region <REGION> --ClusterId "<CLUSTER_ID>"
# expected: 返回 RequestId
```

## 清理

> **副作用警告**：关闭端点会断开依赖该入口的 kubectl/API 访问。操作前确认没有仍在使用该入口的客户端。

```bash
tccli tke DeleteClusterEndpoint --region <REGION> \
  --ClusterId "<CLUSTER_ID>" --IsExtranet true
# expected: 返回 RequestId；内网端点将 IsExtranet 改为 false
```

删除后继续查询 `DescribeClusterEndpointStatus`，根据实际返回判断关闭状态，不假设固定的收敛时长。

## 故障恢复

| 现象 | 诊断 | 修复 |
|:-----|:-----|:-----|
| `ResourceUnavailable.ClusterState` 或空集群错误 | `DescribeClusterInstances` 检查节点 | 先添加节点，再开启公网访问 |
| `ResourceNotFound.SubnetId` | 检查子网及集群 VPC | 改用集群 VPC 内子网 |
| `ResourceNotFound.SecurityGroup` | `DescribeSecurityGroups` 检查安全组 | 使用存在且规则正确的安全组 |
| 长时间停在 `Creating` | 查看 `DescribeClusterEndpointStatus.ErrorMsg` | 按错误信息检查 CLB、子网资源或提交工单 |
| 本机访问内网端点失败 | 核对客户端到 VPC 的网络路径 | 接入 VPC，或为独立集群使用公网端点 |
| 公网端点不可达 | 检查外网 CLB 安全组入站规则 | 仅对客户端来源 IP 放通 TCP 443 |
| 托管集群老外网端口拒绝访问 | 查询外网 ACL 和安全组 | 修正 `SecurityPolicies` 或 `SecurityGroup`，不要套用到独立集群新公网端点 |

## 收尾确认

| 条件 | 命令 / 字段 | 预期 |
|:-----|:------------|:-----|
| 集群模式与端点类型匹配 | `DescribeClusters` + 本文模式表 | 未跨模式调用公网能力 |
| 端点就绪 | `DescribeClusterEndpointStatus.Status` | `Created` |
| 地址已分配 | `DescribeClusterEndpoints` | 对应公网或内网地址非空 |
| 公网访问最小暴露 | 外网 CLB 安全组 | 仅向所需来源 IP 开放 TCP 443 |
| kubectl 可达 | `kubectl ... get --raw=/healthz` | `ok` |

---

## 下一步

- [查询集群](../clusters/query.md) — `DescribeClusterEndpoints` 查看端点地址
- [认证配置](../security/auth.md) — 使用端点和 kubeconfig 配置 kubectl
- [配置 VPC-CNI](vpc-cni.md) — Pod 网络模型
- [故障排查](../troubleshooting.md) — 端点不通的诊断
