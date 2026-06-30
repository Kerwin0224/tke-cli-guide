# VPC-CNI（eniipamd）组件介绍（tccli）

> 对照官方：[VPC-CNI（eniipamd）组件介绍](https://cloud.tencent.com/document/product/457/64919) · page_id `64919`

## 概述

对照[官方 VPC-CNI（eniipamd）组件介绍](https://cloud.tencent.com/document/product/457/64919)，保留组件架构、功能说明与权限范围，并用 **tccli** 完成组件版本与状态审计。

## 前置条件

- [环境准备](../../../../环境准备.md)：`tccli`、`kubectl`、地域 `ap-guangzhou` 与凭证已配置。
- 集群已开启 VPC-CNI 模式（见 [非固定 IP 模式使用说明](../%E9%9D%9E%E5%9B%BA%E5%AE%9A%20IP%20%E6%A8%A1%E5%BC%8F%E4%BD%BF%E7%94%A8%E8%AF%B4%E6%98%8E/tccli%20%E6%93%8D%E4%BD%9C.md) 或 [固定 IP 使用方法](../%E5%9B%BA%E5%AE%9A%20IP%20%E6%A8%A1%E5%BC%8F%E4%BD%BF%E7%94%A8%E8%AF%B4%E6%98%8E/%E5%9B%BA%E5%AE%9A%20IP%20%E4%BD%BF%E7%94%A8%E6%96%B9%E6%B3%95/tccli%20%E6%93%8D%E4%BD%9C.md)）。
- 下载 kubeconfig 见 [连接集群](../../../集群管理/连接集群/tccli%20操作.md)。

## 控制台与 CLI 参数映射

| 官方 / 控制台 | 含义 | tccli / 说明 | 幂等 |
|---------------|------|-------------|------|
| 查看 eniipamd 组件版本/状态 | 组件管理 | `tccli tke DescribeAddon` | 是 |
| 查看 tke-eni-agent DaemonSet | 数据面 agent | `kubectl get ds tke-eni-agent -n kube-system` | 是 |
| 查看 tke-eni-ipamd Deployment | 控制面 ipamd | `kubectl get deploy tke-eni-ipamd -n kube-system` | 是 |
| 查看 tke-eni-ip-scheduler | 调度扩展（固定 IP 模式） | `kubectl get deploy tke-eni-ip-scheduler -n kube-system` | 是 |

## 操作步骤

### 组件概览

VPC-CNI 组件（组件管理中名为 **eniipamd**）由三个 Kubernetes 工作负载组成：

| 组件 | 类型 | 作用 |
|------|------|------|
| **tke-eni-agent** | DaemonSet（每节点） | 部署 CNI 插件、配置策略路由与 ENI、GRPC Pod IP 分配/释放、IP 垃圾回收、设备插件资源上报 |
| **tke-eni-ipamd** | Deployment | CRD 管理（nec / vipc / vip / veni）、ENI 创建/绑定/解绑、非固定 IP 模式节点级 ENI/IP 池管理、固定 IP 模式 Pod 级 IP 管理、安全组管理、EIP 管理 |
| **tke-eni-ip-scheduler** | Deployment（**仅固定 IP 模式**） | 调度扩展插件：多子网场景确保固定 IP Pod 调度到对应子网节点，检查子网 IP 是否充足 |

#### tccli：查看 eniipamd 组件信息

```bash
tccli tke DescribeAddon --ClusterId <ClusterId> --region ap-guangzhou --output json \
  --filter "Addons[?AddonName=='eniipamd'] | [0].{AddonName:AddonName,AddonVersion:AddonVersion,Phase:Phase}"
```

```json
{
  "AddonName": "eniipamd",
  "AddonVersion": "v3.5.0",
  "Phase": "Running"
}
```

# expected: exit 0, contains "AddonVersion"

#### kubectl：查看组件运行状态

```bash
kubectl get deploy,ds -n kube-system | grep tke-eni
```

```text
deployment.apps/tke-eni-ipamd         1/1   1     1      30d
daemonset.apps/tke-eni-agent          3/3   3     3      30d
deployment.apps/tke-eni-ip-scheduler  1/1   1     1      30d
```

# expected: exit 0, contains "tke-eni"

---

### tke-eni-agent

DaemonSet 部署于每节点，职责：

- 复制 CNI 插件（`tke-route-eni`、`tke-eni-ipamc`）至 `/opt/cni/bin`
- 在 `/etc/cni/net.d/` 生成 CNI 配置
- 配置节点策略路由与 ENI
- 运行 GRPC Server 处理 Pod IP 分配/释放
- 定期 IP 垃圾回收
- 通过 device-plugin 机制上报 ENI 和 IP 扩展资源

权限要点（最小化）：需 `privileged` 容器（修改 `net.ipv4.ip_forward`、`rp_filter` 等内核参数）；需读写 pods / nodes / CRD（underlayips, nodeeniconfigs, vpcipclaims, vpcips, vpcenis）。

---

### tke-eni-ipamd

Deployment 部署，职责：

- 创建并管理 CRD（nec、vipc、vip、veni、eipclaim）
- **非固定 IP 模式**：按节点需求管理 ENI 创建/绑定/解绑
- **固定 IP 模式**：按 Pod 需求管理 ENI 与 IP
- ENI 安全组管理
- EIP 创建/绑定/解绑

权限要点：需完整读写 CRD（networking.tke.cloud.tencent.com）、读写 pods / nodes（包括状态更新）、LeaderElection（configmaps / endpoints）。

---

### tke-eni-ip-scheduler

仅固定 IP 模式部署，职责：

- 多子网场景：确保固定 IP Pod 调度到对应子网节点
- 固定 IP 模式：检查目标节点子网 IP 是否充足

权限要点：需 `privileged` 容器（挂载 `/var/lib/kubelet`）；扩展 bindVerb 解决并发 Pod 绑定 IP 冲突；读写 CRD（nodeeniconfigs, vpcipclaims, vpcips）。

## 验证

### Control plane (tccli)

- `DescribeAddon` → eniipamd `Phase: Running`。
- `DescribeClusters` → `Cni: true` 或 `EnableStaticIp` 与启用模式一致。

### Data plane (kubectl)

- `kubectl get ds tke-eni-agent -n kube-system` 确认 DaemonSet 就绪数与节点数一致。
- `kubectl get deploy tke-eni-ipamd -n kube-system` 确认 Deployment 就绪。

## 清理

本页为组件说明与只读查询，无额外资源。

## 排障

| 现象 | 处理 |
|------|------|
| `DescribeAddon` 无 eniipamd | VPC-CNI 未开启；`EnableVpcCniNetworkType` 或创建 VPC-CNI 集群 |
| tke-eni-agent 部分节点未 Ready | 检查该节点 ENI 配额、子网 IP 是否耗尽；查看 Pod 日志 |
| tke-eni-ip-scheduler 未部署 | 正常：仅固定 IP 模式部署；非固定 IP 模式无此组件 |
| 组件版本过低 | 升级至 ≥ v3.5.0 以使用组件管理配置能力 |

## 下一步

- [VPC-CNI 模式 Pod 数量限制](../VPC-CNI%20%E6%A8%A1%E5%BC%8F%20Pod%20%E6%95%B0%E9%87%8F%E9%99%90%E5%88%B6/tccli%20%E6%93%8D%E4%BD%9C.md) · [固定 IP 使用方法](../%E5%9B%BA%E5%AE%9A%20IP%20%E6%A8%A1%E5%BC%8F%E4%BD%BF%E7%94%A8%E8%AF%B4%E6%98%8E/%E5%9B%BA%E5%AE%9A%20IP%20%E4%BD%BF%E7%94%A8%E6%96%B9%E6%B3%95/tccli%20%E6%93%8D%E4%BD%9C.md) · [非固定 IP 模式使用说明](../%E9%9D%9E%E5%9B%BA%E5%AE%9A%20IP%20%E6%A8%A1%E5%BC%8F%E4%BD%BF%E7%94%A8%E8%AF%B4%E6%98%8E/tccli%20%E6%93%8D%E4%BD%9C.md)

## 控制台替代

[控制台 → 集群](https://console.cloud.tencent.com/tke2/cluster) → 组件管理 → eniipamd 查看组件版本、状态与配置。
