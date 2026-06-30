# 固定IP相关特性（tccli）

> 对照官方：[固定IP相关特性](https://cloud.tencent.com/document/product/457/50358) · page_id `50358`

## 概述

VPC-CNI 固定 IP 模式下，Pod 的 IP 地址由 VPC 子网分配，并支持以下核心特性：

- **IP 保留**：Pod 在同一 namespace 和 name 下重建时，自动获取相同的 VPC 子网 IP。该绑定关系由 `tke.cloud.tencent.com/vpc-ip` 注解维护。
- **IP 回收策略**：通过 `tke.cloud.tencent.com/vpc-ip-claim-delete-policy` 注解控制。`"Never"` 表示仅在 StatefulSet 删除时释放 IP（Pod 删除不影响），`"Immediate"` 表示 Pod 删除时立即释放 IP。
- **IP 预分配**：StatefulSet 扩容时，系统会预先从子网中预留 IP 地址，确保扩容后的 Pod 有可用 IP。
- **IP 池管理**：通过子网 CIDR 管理可用 IP 范围，`AddVpcCniSubnets` 可扩展 IP 池，`DescribeIPAMD` 可查询当前状态。

当前集群 `cls-xxxxxxxx`（kerwinwjyan-rewrite-s1）VPC-CNI 状态：`EnableIPAMD: false`，固定 IP 模式未启用。

## 前置条件

- 集群已开启 VPC-CNI 网络模式（EnableVpcCniNetworkType）
- 已启用固定 IP 模式（`--EnableStaticIp true`）
- 工作负载为 StatefulSet 类型
- VPC 子网 IP 充足

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 查询 VPC-CNI 状态（含固定 IP） | tccli tke DescribeIPAMD --region ap-guangzhou --ClusterId cls-xxxxxxxx | 是 |
| 查询 Pod 数量限制 | tccli tke DescribeVpcCniPodLimits --region ap-guangzhou --Zone ap-guangzhou-3 --InstanceFamily S5 --InstanceType S5.MEDIUM2 | 是 |
| 查看 StatefulSet IP 注解 | kubectl get statefulset <NAME> -o yaml | 是 |

## 操作步骤

本页为概念介绍页，无实际操作步骤。

## 验证

- `DescribeIPAMD` 输出中 `EnableIPAMD` 为 `true`，确认集群已开启 VPC-CNI
- StatefulSet Pod 的 annotations 中包含 `tke.cloud.tencent.com/vpc-ip` 和 `tke.cloud.tencent.com/vpc-ip-claim-delete-policy`
- 通过 `DescribeVpcCniPodLimits` 确认目标实例类型（如 S5.MEDIUM2）的 Pod 上限为 27（tke-route-eni 模式）
- 删除 Pod 后，新 Pod 的 `tke.cloud.tencent.com/vpc-ip` 注解值与删除前一致

## 清理

固定 IP 相关功能无需单独清理。若需整体关闭 VPC-CNI 或固定 IP 模式，请参阅「固定IP使用方法」文档中的清理章节。

**注意**：`DisableVpcCniNetworkType` 会改变所有 Pod IP，需重建 Pod，可能影响业务。

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Pod 重建后 IP 未保留 | `kubectl get pod <NAME> -o jsonpath='{.metadata.annotations}'` 检查注解 | 未设置 `tke.cloud.tencent.com/vpc-ip-claim-delete-policy` 或 StatefulSet 名称/namespace 变化 | 确认注解设置为 `"Never"` 或 `"Immediate"`，且 Pod 名称与 namespace 未变 |
| IP 回收行为不符合预期 | 检查 `vpc-ip-claim-delete-policy` 注解值 | 回收策略理解错误：`"Never"` 仅在 StatefulSet 删除时释放，`"Immediate"` 在 Pod 删除时释放 | 根据业务需求设置正确的回收策略值 |
| StatefulSet 扩容时 Pod Pending | `kubectl describe pod` 查看 Event，是否提示 IP 不足 | VPC 子网 IP 池耗尽，无法分配新 IP | 调用 `AddVpcCniSubnets` 扩展子网 |
| 固定 IP 功能未生效 | `DescribeIPAMD` 查看 `EnableStaticIp` 状态 | 开启 VPC-CNI 时未同时设置 `--EnableStaticIp true` | 重新调用 `EnableVpcCniNetworkType --EnableStaticIp true` |
| 查询 IPAMD 报错 | API 错误提示 `cluster is not vpc-cni cluster` | GR 集群未开启 VPC-CNI | 先调用 `EnableVpcCniNetworkType` 开启 VPC-CNI |

## 下一步

- [固定 IP 使用方法](https://cloud.tencent.com/document/product/457/34994) -- page_id `34994`
- [多 Pod 共享网卡](https://cloud.tencent.com/document/product/457/50356) -- page_id `50356`

## 控制台替代

在 [容器服务控制台](https://console.cloud.tencent.com/tke2/cluster/sub/detail/resource/statefulset?rid=1&clusterId=cls-xxxxxxxx) 中创建 StatefulSet 时，在「高级设置」中勾选「VPC-CNI 固定 IP」并选择回收策略。已创建的 StatefulSet 可在 YAML 编辑器中查看和修改相关注解。
