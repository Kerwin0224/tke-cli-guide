# VPC-CNI（eniipamd）组件介绍（tccli）

> 对照官方：[VPC-CNI（eniipamd）组件介绍](https://cloud.tencent.com/document/product/457/64919) · page_id `64919`

## 概述

eniipamd（ENI IP Address Management Daemon）是 VPC-CNI 模式的核心组件，以 DaemonSet 形式部署在 kube-system 命名空间。该组件负责 ENI IP 分配、Pod IP 分配、安全组绑定、EIP 绑定以及 Trunking ENI（tke-sub-eni）管理。eniipamd 由以下子组件构成：

- **eni-ipamd**：管理每个节点上的 ENI 生命周期，包括 ENI 的创建、绑定、回收
- **eni-agent**：处理 Pod 的 IP 分配和释放，与 eni-ipamd 协同工作
- **tke-eni-ip-scheduler**：自定义调度器，实现 ENI 资源感知调度

组件在通过 EnableVpcCniNetworkType 开通 VPC-CNI 时自动安装，无需手动部署。

## 前置条件

- 集群已开启 VPC-CNI 网络模式（eniipamd 组件随 VPC-CNI 启用自动安装）
- 组件依赖 VPC、子网等基础网络资源就绪

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 查询集群 VPC-CNI 状态 | `tccli tke DescribeIPAMD --ClusterId cls-xxxxxxxx --region ap-guangzhou` | 是 |
| 查询 eniipamd DaemonSet | `kubectl get ds -n kube-system eni-ipamd` | 是 |
| 查询组件 Pod 状态 | `kubectl get pods -n kube-system -l app=eni-ipamd` | 是 |
| 查看组件日志 | `kubectl logs -n kube-system <eni-ipamd-pod>` | 是 |

## 操作步骤

本页为概念介绍页，无实际操作步骤。eniipamd 组件随 EnableVpcCniNetworkType 自动安装，日常运维可通过以下命令查看组件状态：

```bash
# 查看 DaemonSet 状态（仅 VPC-CNI 已启用时可用）
kubectl get ds -n kube-system eni-ipamd

# 查看组件 Pod 运行状态
kubectl get pods -n kube-system -l app=eni-ipamd -o wide

# 查看组件日志
kubectl logs -n kube-system -l app=eni-ipamd --tail=50

# 排查调度器状态
kubectl get pods -n kube-system -l app=tke-eni-ip-scheduler
```

```text
NAME  STATUS  AGE
...
```

当前集群 cls-xxxxxxxx 为 GR 模式，EnableIPAMD 为 false，以上 DaemonSet 查询命令将返回 NotFound。VPC-CNI 相关查询需先通过 EnableVpcCniNetworkType 开通。

## 验证

- VPC-CNI 启用后，通过 `kubectl get ds -n kube-system eni-ipamd` 确认 DaemonSet 的 DESIRED 与 READY 数量一致
- 通过 `kubectl get pods -n kube-system -l app=eni-ipamd` 确认所有节点上的组件 Pod 均为 Running 状态
- 通过 `tccli tke DescribeIPAMD` 确认 EnableIPAMD 为 true，Phase 为空或表示正常

## 清理

eniipamd 组件为 VPC-CNI 模式基础设施组件，不建议单独删除。如需清理，可通过 DisableVpcCniNetworkType 关闭 VPC-CNI，组件将自动移除。

**注意**：DisableVpcCniNetworkType 会改变现有 Pod IP，需要重建 Pod，可能影响业务。

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| eniipamd Pod CrashLoopBackOff | 查看组件日志 `kubectl logs -n kube-system <eni-ipamd-pod>` | 节点 ENI 配额不足或 VPC 子网 IP 耗尽 | 检查节点 ENI 数量限制，通过 AddVpcCniSubnets 添加子网 |
| IP 分配失败 | 查看组件日志中的 allocate 相关错误 | 子网可用 IP 不足或 ENI 绑定数达到上限 | 添加更多子网或减少该节点 Pod 数量 |
| Pod 创建挂起 | 查看 tke-eni-ip-scheduler 日志 | 调度器无法分配 ENI IP 资源 | 确认 VPC-CNI 子网配置正确，子网 IP 充足 |
| eniipamd DaemonSet 未部署 | 确认 VPC-CNI 是否已启用 `tccli tke DescribeIPAMD` | VPC-CNI 未启用或启用失败 | 执行 EnableVpcCniNetworkType 开通 VPC-CNI |
| 安全组绑定失败 | 检查 eni-agent 日志 | eni-agent 与 VPC API 通信异常 | 检查节点到 VPC 服务的网络连通性 |

## 下一步

- [VPC-CNI模式介绍](https://cloud.tencent.com/document/product/457/50355) -- page_id `50355`
- [VPC-CNI模式安全组使用说明](https://cloud.tencent.com/document/product/457/50360) -- page_id `50360`

## 控制台替代

在 TKE 控制台进入集群详情，左侧导航栏选择"组件管理"，在组件列表中可查看 eniipamd 组件的安装状态、版本信息和运行情况。组件详情页提供 Pod 列表、日志查看和事件监控的图形化界面。
