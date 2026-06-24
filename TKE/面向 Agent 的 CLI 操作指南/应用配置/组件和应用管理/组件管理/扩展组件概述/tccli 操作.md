# 扩展组件概述（tccli）

> 对照官方：[扩展组件概述](https://cloud.tencent.com/document/product/457/39048) · page_id `39048`

## 概述

TKE 提供丰富的扩展组件生态，涵盖存储、网络、调度、监控、安全等类别。通过组件管理页面或 `tccli` 一键安装和配置。所有扩展组件均通过 `tccli tke` 的 Addon 系列接口完成安装、升级、卸载。

## 前置条件

- [环境准备](../../../../环境准备.md)：`tccli` 已配置
- 已创建集群且状态 `Running`

```bash
# 检查集群状态
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

- 需要 CAM 权限：`tke:DescribeClusters`、`tke:DescribeAddon`、`tke:InstallAddon`、`tke:DeleteAddon`

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 查看集群信息 | `tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'` | 是 |
| 查看组件列表 | `tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId>` | 是 |
| 查看指定组件详情 | `tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName <AddonName>` | 是 |
| 安装组件 | `tccli tke InstallAddon --region <Region> --ClusterId <ClusterId> --AddonName <AddonName>` | 否 |
| 升级组件 | `tccli tke UpdateAddon --region <Region> --ClusterId <ClusterId> --AddonName <AddonName> --AddonVersion <AddonVersion>` | 否 |
| 卸载组件 | `tccli tke DeleteAddon --region <Region> --ClusterId <ClusterId> --AddonName <AddonName>` | 是 |

## 操作步骤

本页面为概念性说明。组件分类与具体操作指引请参考子页面列表。

### 组件分类

| 类别 | 组件 | 功能 |
|------|------|------|
| 存储 | CBS-CSI、CFS-CSI、COS-CSI、CFSTURBO-CSI、GooseFS-CSI | 云硬盘 / 文件存储 / 对象存储 / Turbo 文件存储 / 数据加速 |
| 网络 | NginxIngress、Network Policy、eNP、ip-masq-agent、NodeLocalDNSCache、DNSAutoscaler | Ingress / 网络策略 / ebpf 网络 / masquerade / DNS 缓存与扩缩 |
| 调度 | DynamicScheduler、DeScheduler、tke-karpenter、Cluster Autoscaler、HPC | 动态调度 / 重调度 / 弹性节点池 / 集群弹性 / 水平伸缩 |
| 监控运维 | tke-monitor-agent、tke-log-agent、tke-event-collector、NodeProblemDetectorPlus、Node-Healing、OOMGuard | 监控 / 日志 / 事件采集 / 节点异常检测 / 节点自愈 / OOM 防护 |
| 安全 | ExternalSecrets、UserGroupAccessControl、Cerberus | 外部密钥 / 用户组权限 / 集群巡检 |
| DNS | CoreDNS、NodeLocalDNSCache、DNSAutoscaler | 集群 DNS / 本地缓存 / 自动扩缩 |
| GPU | nvidia-gpu、rdma-agent | GPU 驱动 / RDMA 网络 |
| 镜像 | TCR | 内网免密拉取 TCR 企业版镜像 |
| 其他 | DCU、tke-eni-ip-webhook | 数据中心单元 / ENI IP 管理 |

### 组件安装方式

所有扩展组件均通过 `tccli` 或控制台一键安装：

```bash
# 查看集群已安装组件
tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId>
# expected: 返回 Addons 列表，含 AddonName、AddonVersion、Status

# 安装组件
tccli tke InstallAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName <AddonName>
# expected: RequestId 返回，组件安装中
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

## 验证

不适用（概述页面）。

## 清理

不适用（概述页面）。

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `InstallAddon` 返回 `FailedOperation.AddonAlreadyInstalled` | `tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId>` 检查已安装组件 | 组件已安装 | 无需重复安装；如需重装请先 `DeleteAddon` |
| `InstallAddon` 返回版本不兼容错误 | `tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName <AddonName>` 查看可用版本 | 组件版本与集群 Kubernetes 版本不兼容 | 使用 `DescribeAddon` 查询可用版本后重试 |
| 组件安装后状态为 `Failed` | `tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName <AddonName>` 查看 Status 与 reason | 安装过程中出错或资源不足 | 检查组件版本兼容性，扩容节点后重试 |
| `Unable to connect to the server` | `kubectl cluster-info` 检查连通性 | 公网端点被 CAM 策略 strategyId:240463971 以 `tke:clusterExtranetEndpoint=true` 条件拒绝 | 通过 IOA/VPN 连接内网，或使用同 VPC CVM 执行 kubectl 命令 |

## 下一步

- [组件的生命周期管理](../组件的生命周期管理/tccli 操作.md) — 组件安装、升级与卸载
- [组件高可用](../组件高可用/tccli 操作.md) — 跨可用区高可用配置
- 各具体组件操作页面（见上文组件分类表）

## 控制台替代

[TKE 控制台 - 组件管理](https://console.cloud.tencent.com/tke2/cluster) → 集群 → 组件管理。
