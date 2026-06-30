# eniipamd组件介绍

> 对照官方：[eniipamd组件介绍](https://cloud.tencent.com/document/product/457/64919) · page_id `64919`

## 概述

VPC-CNI 模式的核心组件在 TKE 组件管理中名为 **eniipamd**，它不是一个单一进程，而是由三个 Kubernetes 集群组件协同工作，共同实现 Pod 直通 VPC 网络的能力：

| 组件 | 部署形式 | 核心职责 | 是否必须部署 |
|------|----------|----------|-------------|
| `tke-eni-agent` | DaemonSet（每节点一个 Pod） | CNI 插件部署、节点网络配置、Pod IP 分配与回收服务（gRPC Server）、Device Plugin 上报网卡/IP 扩展资源 | 是 |
| `tke-eni-ipamd` | Deployment（多副本 LeaderElection） | CRD 管理（nec/vipc/vip/veni）、弹性网卡生命周期管理、IP 分配/释放、安全组管理、EIP 管理 | 是 |
| `tke-eni-ip-scheduler` | Deployment（多副本 LeaderElection） | 固定 IP 模式下的调度扩展：多子网调度、节点子网 IP 充足性判断 | 仅固定 IP 模式 |

三个组件共同覆盖了 **非固定 IP** 和 **固定 IP** 两种 VPC-CNI 子模式，涉及的关键 CRD 包括：`NodeENIConfig`（nec）、`VpcIPClaim`（vipc）、`VpcIP`（vip）、`VpcENI`（veni）。

## 前置条件

- `tccli` 已安装并配置凭证（`tccli configure` 已设置 `region` 与密钥对）
  ```bash
  tccli tke DescribeClusters --region <Region> --output json
  ```

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
```
  # expected: 返回 Clusters 列表，exit 0

- `kubectl` 已配置对应集群的 kubeconfig（用于数据面检查组件运行状态）
  ```bash
  kubectl cluster-info
  ```
  # expected: 返回 Kubernetes control plane 地址，exit 0

- 目标集群已启用 VPC-CNI 网络模式（在创建集群时选择，或后续通过 `EnableVpcCniNetworkType` 开启）

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 查看 eniipamd 组件信息 | `tccli tke DescribeIPAMD --ClusterId <ClusterId> --region <Region>` | 是 |
| 查看组件列表及版本 | `tccli tke DescribeAddon --ClusterId <ClusterId> --region <Region> --AddonName eniipamd` | 是 |
| 查看 eniipamd 配置参数 | `tccli tke DescribeAddonValues --ClusterId <ClusterId> --region <Region> --AddonName eniipamd` | 是 |
| 查看节点上 tke-eni-agent Pod | `kubectl get pods -n kube-system -l app=tke-eni-agent` | 是 |
| 查看 ipamd/scheduler Pod | `kubectl get pods -n kube-system -l app=tke-eni-ipamd` 及 `kubectl get pods -n kube-system -l app=tke-eni-ip-scheduler` | 是 |
| 查看节点弹性网卡/Pod 数量上限 | `tccli tke DescribeVpcCniPodLimits --region <Region> --Zone <Zone> --InstanceFamily <InstanceFamily>` | 是 |

## 操作步骤

---

### tke-eni-agent（节点代理）

**部署形式**：DaemonSet，每个节点自动运行一个 Pod，位于 `kube-system` 命名空间。

**核心职责**：

1. **CNI 插件部署**：将 `tke-route-eni` 和 `tke-eni-ipamc` 等 CNI 二进制拷贝到节点 CNI 执行文件目录（默认 `/opt/cni/bin`），并在 CNI 配置目录（默认 `/etc/cni/net.d/`）生成配置文件。
2. **节点网络配置**：为节点设置策略路由和弹性网卡相关内核参数（如 `net.ipv4.ip_forward`、`net.ipv4.rp_filter`），因此需要特权容器运行。
3. **IP 分配服务**：以 gRPC Server 形式提供 Pod 的 IP 分配/释放接口，供 `tke-eni-ipamc`（CNI 插件）调用。
4. **IP 垃圾回收**：定期检查并回收"Pod 已不在本节点、但 IP 仍未释放"的 IP 资源。
5. **扩展资源管理**：通过 Kubernetes Device Plugin 机制，将节点可用网卡数和 IP 数上报为扩展资源（如 `tke.cloud.tencent.com/eni-ip`、`tke.cloud.tencent.com/direct-eni`），供调度器决策。

**查询命令**：

```bash
kubectl get daemonset tke-eni-agent -n kube-system
```

```text
NAME             DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
tke-eni-agent    3         3         3       3            3           <none>          7d
```

```bash
kubectl get pods -n kube-system -l app=tke-eni-agent -o wide
```

```text
NAME                   READY   STATUS    RESTARTS   AGE   NODE
tke-eni-agent-abcde    1/1     Running   0          7d    <node-ip-1>
tke-eni-agent-fghij    1/1     Running   0          7d    <node-ip-2>
tke-eni-agent-klmno    1/1     Running   0          7d    <node-ip-3>
```

---

### tke-eni-ipamd（IP 地址管理控制器）

**部署形式**：Deployment，多副本通过 LeaderElection 选主，部署在集群特定节点或 Master 上。

**核心职责**：

1. **CRD 管理**：创建并管理四类 CRD 资源——`NodeENIConfig`（节点弹性网卡配置）、`VpcIPClaim`（IP 申领）、`VpcIP`（IP 记录）、`VpcENI`（弹性网卡记录）。
2. **非固定 IP 模式**：依据节点维度的需求和状态，创建/绑定/解绑/删除弹性

```json
{
  "EnableIPAMD": true,
  "EnableCustomizedPodCidr": true,
  "DisableVpcCniMode": true,
  "Phase": "<Phase>",
  "Reason": "<Reason>",
  "SubnetIds": [],
  "ClaimExpiredDuration": "<ClaimExpiredDuration>",
  "EnableTrunkingENI": true
}
```网卡，分配/释放弹性网卡 IP。IP 池以节点为粒度管理。
3. **固定 IP 模式**：依据 Pod 维度的需求和状态，创建/绑定/解绑/删除弹性网卡，分配/释放弹性网卡 IP。Pod 重建后 IP 保持不变。
4. **安全组管理**：管理节点弹性网卡的安全组绑定。
5. **EIP 管理**：依据 Pod 需求创建/绑定/解绑/删除弹性公网 IP。

**查询命令**：

```bash
tccli tke DescribeIPAMD --ClusterId <ClusterId> --region <Region>
```

```json
{
  "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "EnableEni": true,
  "SubnetIds": [
    "subnet-example"
  ],
  "EniSubnetIds": [
    "subnet-example"
  ]
}
```

```bash
tccli tke DescribeAddon --ClusterId <ClusterId> --region <Region> --AddonName eniipamd --output json
```

```json
{
  "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "Addons": [
    {
      "AddonName": "eniipamd",
      "AddonVersion": "3.8.7",
      "RawValues": "...",
      "Phase": "Succeeded",
      "Reason": ""
    }
  ]
}
```

检查 ipamd Pod 运行状态：

```bash
kubectl get deployment tke-eni-ipamd -n kube-system
```

```text
NAME             READY   UP-TO-DATE   AVAILABLE   AGE
tke-eni-ipamd    2/2     2            2           7d
```

---

### tke-eni-ip-scheduler（固定 IP 调度扩展）

**部署形式**：Deployment，多副本通过 LeaderElection 选主。**仅在集群开启固定 IP 模式时部署**，非固定 IP 模式的集群中不会出现此组件。

**核心职责**：

1. **多子网调度**：当集群配置了多个容器子网时，确保已分配固定 IP 的 Pod 被调度到指定子网对应的节点上。
2. **IP 充足性判断**：在固定 IP 模式下，调度 Pod 前判断目标节点对应子网的可用 IP 是否充足，不足时拒绝调度到该节点。

**调度扩展原理**：`tke-eni-ip-scheduler` 通过扩展 Kubernetes 调度器的 `bind` 动词，在 Pod 绑定到节点的环节介入，解决并发绑定时的 IP 分配冲突问题。调度器需要挂载主机 `/var/lib/kubelet` 目录，并以特权容器运行。

**查询命令**：

```bash
kubectl get deployment tke-eni-ip-scheduler -n kube-system
```

```text
NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
tke-eni-ip-scheduler    2/2     2            2           7d
```

> **注意**：若集群未启用固定 IP 模式，上述命令将返回 `NotFound`。此为正常现象，无需处理。

---

### 固定 IP vs 非固定 IP 模式对比

| 维度 | 非固定 IP 模式 | 固定 IP 模式 |
|------|---------------|-------------|
| IP 分配粒度 | 节点维度（节点维护 IP 池） | Pod 维度（每个 Po

```json
{
  "Addons": [],
  "AddonName": "<AddonName>",
  "AddonVersion": "<AddonVersion>",
  "RawValues": "<RawValues>",
  "Phase": "<Phase>",
  "Reason": "<Reason>",
  "CreateTime": "<CreateTime>",
  "RequestId": "<RequestId>"
}
```d 独立分配） |
| Pod 重建后 IP | 可能变化 | 保持不变 |
| 部署组件 | tke-eni-agent + tke-eni-ipamd | tke-eni-agent + tke-eni-ipamd + tke-eni-ip-scheduler |
| 适用场景 | 无状态、多副本、IP 不敏感业务 | 需要固定 IP 的有状态业务（如数据库、MQ） |
| 创建配置项 | 创建集群时不勾选"固定 Pod IP" | 创建集群时勾选"固定 Pod IP" |

## 验证

### 控制面（tccli）

```bash
# 检查 eniipamd 组件信息
tccli tke DescribeIPAMD --ClusterId <ClusterId> --region <Region>
```

```json
{
  "RequestId": "..."
}
```

# expected: 返回 `EnableEni` 相关字段，无鉴权错误

```bash
# 检查 addon 版本与状态
tccli tke DescribeAddon --ClusterId <ClusterId> --region <Region> --AddonName eniipamd
```

```json
{
  "RequestId": "..."
}
```

# expected: `Phase` 为 `Succeeded`，无 `Reason` 报错

### 数据面（kubectl）

```bash
# 检查三个组件的 Pod 运行状态
kubectl get pods -n kube-system -l 'app in (tke-eni-agent,tke-eni-ipamd,tke-eni-ip-scheduler)' --no-headers
```

```text
NAME  STATUS  AGE
...
```

# expected: 所有 Pod 状态为 Running，READY 列 1/1 或 Ready 数等于 Desired

```bash
# 确认 DaemonSet 覆盖全部节点
kubectl get ds tke-eni-agent -n kube-system -o jsonpath='{.status.numberReady}'
```

```text
NAME  STATUS  AGE
...
```

# expected: 返回值等于集群节点数

## 清理

本页为概念说明与只读查询，不涉及资源创建，无需清理。若需卸载 eniipamd 组件（将导致集群 VPC-CNI 网络功能不可用），应通过控制台「组件管理」页面操作，**请勿直接使用 `kubectl delete` 删除组件 Pod 或 Deployment**，组件控制器会将其自动重建。

## 排障

### 命令返回错误

| 现象 | 原因 | 处理 |
|------|------|------|
| `DescribeIPAMD` 返回 `ClusterNotFound` | ClusterId 无效或 region 不匹配 | 检查 `--ClusterId` 与 `--region` 是否正确，确认集群未被删除 |
| `DescribeAddon` 返回空列表 | 集群未安装 eniipamd，或 AddonName 拼写错误 | 确认集群网络模式为 VPC-CNI；检查 AddonName 拼写为 `eniipamd`（非 `eni-ipamd`） |
| `kubectl get pods` 返回 `tke-eni-ip-scheduler` 的 `NotFound` | 集群未启用固定 IP 模式，scheduler 未部署 | 正常现象。如需要使用固定 IP，参考 [固定 IP 模式使用说明](../固定%20IP%20模式使用说明/固定%20IP%20使用方法/tccli%20操作.md) |

### 组件状态异常（命令执行成功但状态不符合预期）

| 现象 | 原因 | 处理 |
|------|------|------|
| tke-eni-agent 个别节点 `Running 0/1` | 节点上 CNI 插件部署失败或内核参数设置失败 | `kubectl describe pod -n kube-system <pod-name>` 查看 Events；检查节点 `/opt/cni/bin` 和 `/etc/cni/net.d/` 目录权限 |
| tke-eni-ipamd `CrashLoopBackOff` | LeaderElection 异常或 VPC API 调用鉴权失败 | `kubectl logs -n kube-system deployment/tke-eni-ipamd` 查看日志；检查节点绑定的 CAM 角色是否有 VPC 操作权限 |
| `DescribeAddon` 的 `Phase` 不为 `Succeeded` | 组件安装/升级失败 | 查看 `Reason` 字段的详细报错信息 |
| 固定 IP 模式下 Pod 调度失败 | tke-eni-ip-scheduler 未运行或子网 IP 耗尽 | 确认 scheduler Deployment 存在且 Ready；检查目标子网可用 IP 数是否充足 |

## 下一步

- [非固定 IP 模式使用说明](../非固定%20IP%20模式使用说明/tccli%20操作.md)
- [固定 IP 模式使用说明](../固定%20IP%20模式使用说明)
- [VPC-CNI 模式 Pod 数量限制](../VPC-CNI%20模式%20Pod%20数量限制/tccli%20操作.md)
- [多 Pod 共享网卡模式](../多%20Pod%20共享网卡模式/tccli%20操作.md)
- [Pod 间独占网卡模式](../Pod%20间独占网卡模式/tccli%20操作.md)

## 控制台替代

控制台 → 集群 → 基本信息 → 组件管理 → 找到 `eniipamd`：可查看组件版本、状态、配置参数，以及进行升级或配置变更操作。
