# ip-masq-agent 说明（tccli）

> 对照官方：https://cloud.tencent.com/document/product/457/121346

## 概述

ip-masq-agent 以 DaemonSet 形式部署在 TKE 标准集群的每个节点上（超级节点除外），通过下发 iptables 规则实现对 Pod 出站流量的 IP Masquerade（IP 伪装）。IP 伪装是源网络地址转换（SNAT）的一种形式，Pod 访问外部网络时源 Pod IP 会被转换为节点 IP，使多个内部 IP 可共享一个地址进行出站访问。

使用场景：

1. **GlobalRouter 模式集群**：Pod 访问非 VPC 网络（如公网）时必须将源地址转为节点 IP。GR 模式下 Pod 无法绑定弹性公网 IP 和 NAT 网关，IP 伪装为强制行为。
2. **以节点 IP 身份进行验证或访问控制**：需要对 Pod 出站流量以节点 IP 身份进行访问控制或身份验证的场景。

### 部署对象

| 对象名称 | 类型 | 默认资源占用 | 命名空间 |
| --- | --- | --- | --- |
| ip-masq-agent-config | ConfigMap | — | kube-system |
| ip-masq-agent | DaemonSet | — | kube-system |

## 前置条件

环境准备请参考 [环境准备](../../../../环境准备.md)，连接集群请参考 [连接集群（tccli）](../../../../集群配置/集群管理/连接集群/tccli 操作.md)。

先确认集群状态正常：

```bash
tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'
# expected: ClusterStatus "Running"
```

```json
{
  "TotalCount": "<TotalCount>",
  "Clusters": "<Clusters>",
  "ClusterId": "<ClusterId>",
  "ClusterName": "<ClusterName>",
  "ClusterDescription": "<ClusterDescription>",
  "ClusterVersion": "<ClusterVersion>"
}
```

### 前置约束

- 超级节点上不部署 ip-masq-agent。
- GlobalRouter 模式集群中，Pod 无法绑定弹性公网 IP 或使用 NAT 网关，IP 伪装为**必选**组件。
- `NonMasqueradeSrcCIDRs` 参数仅支持 **v2.6.2 及以上版本**。
- 添加 VPC-CNI 容器子网时建议同步 `NonMasqueradeCIDRs`（控制台添加子网时 TKE 提供自动同步选项）。

### CAM 权限

| 操作 | API 权限 |
| --- | --- |
| 安装组件 | `tke:InstallAddon` |
| 查询组件信息 | `tke:DescribeAddon` |
| 查询组件参数 | `tke:DescribeAddonValues` |
| 更新组件 | `tke:UpdateAddon` |
| 删除组件 | `tke:DeleteAddon` |

## 控制台与 CLI 参数映射

控制台「组件管理」页面对应的 tccli 核心命令：

| 控制台操作 | tccli 命令 |
| --- | --- |
| 查询集群组件列表 | `tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'`（含 Addons 字段） |
| 查询组件详情 | `tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName ip-masq-agent` |
| 安装组件 | `tccli tke InstallAddon --region <Region> --ClusterId <ClusterId> --AddonName ip-masq-agent --AddonVersion <AddonVersion>` |
| 更新组件 | `tccli tke UpdateAddon --region <Region> --ClusterId <ClusterId> --AddonName ip-masq-agent --AddonVersion <AddonVersion>` |
| 删除组件 | `tccli tke DeleteAddon --region <Region> --ClusterId <ClusterId> --AddonName ip-masq-agent` |

### 配置参数

| 参数 | 类型 | 说明 |
| --- | --- | --- |
| NonMasqueradeCIDRs | String 列表（CIDR） | 目标 CIDR 列表。Pod 访问这些 CIDR 时**不进行** IP 伪装 |
| NonMasqueradeSrcCIDRs | String 列表（CIDR） | 源 CIDR 列表。源 IP 在这些 CIDR 范围内的 Pod **不进行** IP 伪装（**v2.6.2+**） |
| MasqLinkLocal | Boolean | Pod 访问链路本地 `169.254.0.0/16` 时是否伪装 |
| MasqLinkLocalIPv6 | Boolean | IPv6 链路本地 `fe80::/10` 访问时是否伪装 |
| ResyncInterval | Duration 字符串 | 热加载间隔，默认 `1m0s`（60 秒） |

## 操作步骤

### 1. 安装 ip-masq-agent 组件

通过 InstallAddon 安装组件：

```bash
tccli tke InstallAddon --region <Region> --ClusterId <ClusterId> --AddonName ip-masq-agent --AddonVersion <AddonVersion>
# expected: 返回安装任务，状态为 Running/Installing
```

可通过 DescribeAddon 查询安装进度：

```bash
tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName ip-masq-agent
# expected: Phase "Running"
```

### 2. 配置 ConfigMap

ip-masq-agent 的配置存储在 `kube-system` 命名空间的 ConfigMap `ip-masq-agent-config` 中，支持 YAML 或 JSON 格式，组件会定期热加载（间隔由 `ResyncInterval` 设置）。

> **注意**：kubectl 数据面当前因公网端点被 CAM 策略 strategyId:240463971 拒绝而不可达。以下 kubectl 命令为参考格式，需在内网/VPN 环境执行。

默认配置示例：

```yaml
apiVersion: v1
data:
  config: |
    NonMasqueradeCIDRs:
        - 192.168.0.0/16
    NonMasqueradeSrcCIDRs:
        - 10.0.0.0/12
    MasqLinkLocal: true
    MasqLinkLocalIPv6: false
    ResyncInterval: 1m0s
kind: ConfigMap
metadata:
  name: ip-masq-agent-config
  namespace: kube-system
```

编辑 ConfigMap：

```bash
kubectl -n kube-system edit cm ip-masq-agent-config
```

保存后等待约 1 分钟（取决于 `ResyncInterval`）生效，可查看组件日志确认。

### 最佳实践

#### 性能说明

IP 伪装使用 iptables 规则，延长网络路径，会带来性能开销。不建议在非必要场景下大规模使用。

#### GlobalRouter 模式集群

推荐的 `NonMasqueradeCIDRs`（仅私有地址段不做伪装，因为 GlobalRouter 子网不具备完整 VPC 能力如 NAT 网关）：

```yaml
NonMasqueradeCIDRs:
    - 10.0.0.0/8
    - 172.16.0.0/12
    - 192.168.0.0/16
```

此配置下：Pod 访问私有地址段保持 Pod IP 为源地址；访问非私有地址段时伪装为节点 IP。

#### VPC-CNI 模式集群

Pod 具有完整的 VPC 网络能力，应尽量减少伪装。新建的 VPC-CNI 集群可以**跳过安装此组件**。如已安装，建议设置：

```yaml
NonMasqueradeCIDRs:
    - 0.0.0.0/0
```

此配置禁用所有目标地址的伪装。

#### 混合模式集群（GlobalRouter + VPC-CNI）

假设 VPC CIDR 为 `172.16.0.0/16` 和 `10.16.0.0/16`：

```yaml
NonMasqueradeCIDRs:
    - 10.0.0.0/8
    - 172.16.0.0/12
    - 192.168.0.0/16
NonMasqueradeSrcCIDRs:
    - 172.16.0.0/16
    - 10.16.0.0/16
```

效果：GlobalRouter Pod（源 IP 不在 VPC CIDR）— 访问私有地址段不伪装，访问非私有地址段伪装为节点 IP；VPC-CNI Pod（源 IP 在 VPC CIDR 中）— 访问任何目标地址都不伪装。

#### 添加 VPC-CNI 容器子网时同步 NonMasqueradeCIDRs

控制台路径：**容器服务控制台 → 集群 ID → 节点和网络信息 → 添加子网**。添加 VPC-CNI 容器子网时，TKE 提供自动将 VPC 辅助 CIDR 同步到 `NonMasqueradeCIDRs` 的选项，建议在大多数不需要伪装的场景下勾选。如已配置 `0.0.0.0/0` 则可忽略。

## 验证

查询组件状态：

```bash
tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName ip-masq-agent
# expected: Phase "Running"
```

```json
{
  "Addons": "<Addons>",
  "AddonName": "<AddonName>",
  "AddonVersion": "<AddonVersion>",
  "RawValues": "<RawValues>",
  "Phase": "<Phase>",
  "Reason": "<Reason>"
}
```

> **注意**：kubectl 数据面当前因公网端点被 CAM 策略 strategyId:240463971 拒绝而不可达。以下 kubectl 命令为参考格式，需在内网/VPN 环境执行。

验证 DaemonSet 运行状态：

```bash
kubectl get ds ip-masq-agent -n kube-system
# expected: DESIRED 与 READY 数量一致
```

```text
NAME  STATUS  AGE
...
```

查看当前配置：

```bash
kubectl get cm ip-masq-agent-config -n kube-system -o yaml
# expected: 输出含 NonMasqueradeCIDRs、ResyncInterval 等字段
```

```text
NAME  STATUS  AGE
...
```

验证 iptables 规则（在任意节点上执行）：

```bash
iptables -t nat -L IP-MASQ-AGENT -n
# expected: 私有网段 RETURN，其余 MASQUERADE
```

## 清理

通过 DeleteAddon 卸载组件：

```bash
tccli tke DeleteAddon --region <Region> --ClusterId <ClusterId> --AddonName ip-masq-agent
# expected: 返回删除任务
```

> **注意**：对于 GlobalRouter 模式集群，删除 ip-masq-agent 后 Pod 将无法访问公网。请确认集群网络模式后再决定是否删除。

验证卸载结果：

```bash
tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName ip-masq-agent
# expected: Phase "NotFound" 或返回 NotFound 错误
```

```json
{
  "Addons": "<Addons>",
  "AddonName": "<AddonName>",
  "AddonVersion": "<AddonVersion>",
  "RawValues": "<RawValues>",
  "Phase": "<Phase>",
  "Reason": "<Reason>"
}
```

> **计费提醒**：ip-masq-agent 组件本身免费，不产生额外费用。

## 排障

| 现象 | 诊断 | 根因 | 修复 |
| --- | --- | --- | --- |
| VPC-CNI Pod 无法访问公网 | 检查 `NonMasqueradeCIDRs` 是否设为 `0.0.0.0/0` 且 VPC-CNI 子网未配置 NAT 网关/EIP | VPC-CNI 模式下公网访问依赖 NAT 网关或 EIP，与 IP 伪装无关 | 为 VPC 配置 NAT 网关或为 Pod 绑定 EIP |
| GlobalRouter Pod 无法访问公网 | 检查组件是否安装、`NonMasqueradeCIDRs` 是否过宽 | 组件未安装或 `NonMasqueradeCIDRs` 包含 `0.0.0.0/0` 导致公网不伪装 | 安装组件，`NonMasqueradeCIDRs` 仅保留私有地址段，不包含 `0.0.0.0/0` |
| ConfigMap 修改不生效 | 检查修改时间与生效时间间隔 | 热加载间隔未到 | 等待 `ResyncInterval` 时长（默认 60 秒），或检查组件日志确认配置已加载 |
| `NonMasqueradeSrcCIDRs` 不生效 | 查询组件版本 | 组件版本低于 v2.6.2 不支持该参数 | 通过 UpdateAddon 升级至 v2.6.2 及以上版本 |
| tccli/kubectl Unable to connect | 检查公网端点访问策略 | 公网端点被 CAM 策略 strategyId:240463971（`tke:clusterExtranetEndpoint=true`）拒绝 | 在内网/VPN 环境执行命令，或调整 CAM 策略放行公网端点 |

## 下一步

- [TKE VPC-CNI 网络模式](https://cloud.tencent.com/document/product/457/34993)
- [TKE GlobalRouter 网络模式](https://cloud.tencent.com/document/product/457/50354)
- [Network Policy 说明](https://cloud.tencent.com/document/product/457/50841)
- [NodeLocalDNSCache 说明](https://cloud.tencent.com/document/product/457/49423)
- [Kubernetes ip-masq-agent 社区文档](https://github.com/kubernetes-sigs/ip-masq-agent)

## 控制台替代

如需通过控制台操作，登录 [腾讯云容器服务控制台](https://console.cloud.tencent.com/tke2/cluster)，进入集群详情 →「组件管理」→ 找到 ip-masq-agent → 进行安装、更新或卸载。配置变更可在组件详情页的参数配置中修改。
