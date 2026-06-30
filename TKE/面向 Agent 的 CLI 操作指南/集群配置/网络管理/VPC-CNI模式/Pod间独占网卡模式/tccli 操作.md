# Pod间独占网卡模式（tccli）

> 对照官方：[Pod间独占网卡模式](https://cloud.tencent.com/document/product/457/50357) · page_id `50357`

## 概述

Pod间独占网卡模式（`tke-direct-eni`）是 VPC-CNI 的子类型之一。在此模式下，每个 Pod 独占一张独立的弹性网卡（ENI），享有独享网卡的完整网络带宽和性能，网络延迟更低、吞吐更稳定。由于每个 Pod 独占一张 ENI，单节点的 Pod 密度受限于节点可绑定的 ENI 数量，因此 Pod 密度低于 `tke-route-eni` 模式。例如 S5.MEDIUM2 机型在 `ap-guangzhou-3` 可用区仅支持 9 个直接 ENI Pod（`TKEDirectENI=9`）。

适用场景：对网络性能有较高要求的应用（如高吞吐量服务、延迟敏感型应用），以及对网络隔离有严格要求的场景。

> 注意：当前集群 `cls-xxxxxxxx` 为 GR 模式，需通过 `EnableVpcCniNetworkType` 启用 `tke-direct-eni`。

## 前置条件

- 集群处于 **Running** 状态
- VPC 下有可用子网，子网 IP 数量需覆盖预期 Pod 总量
- 已通过 `DescribeVpcCniPodLimits` 确认目标机型的独占网卡 Pod 上限满足需求
- 了解 `tke-direct-eni` 模式下 Pod 密度较低（约 `tke-route-eni` 的 1/3），对节点数量有更高要求
- 当前集群 `cls-xxxxxxxx` 为 GR 模式（`EnableIPAMD: false`）

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 查询集群VPC-CNI状态 | `tccli tke DescribeIPAMD --region ap-guangzhou --ClusterId cls-xxxxxxxx` | 是 |
| 开启tke-direct-eni模式 | `tccli tke EnableVpcCniNetworkType --region ap-guangzhou --ClusterId cls-xxxxxxxx --SubnetIds '["subnet-xxxxxxxx"]' --VpcCniType "tke-direct-eni"` | 否 |
| 查询开启进度 | `tccli tke DescribeEnableVpcCniProgress --region ap-guangzhou --ClusterId cls-xxxxxxxx` | 是 |
| 查询Pod数量限制 | `tccli tke DescribeVpcCniPodLimits --region ap-guangzhou --Zone ap-guangzhou-3 --InstanceFamily S5 --InstanceType S5.MEDIUM2` | 是 |

## 操作步骤

### 1. 查询当前VPC-CNI状态

```bash
tccli tke DescribeIPAMD --region ap-guangzhou --ClusterId cls-xxxxxxxx
# expected: "EnableIPAMD": false （当前为 GR 模式，未启用 VPC-CNI）
```

```json
{
  "EnableIPAMD": "<EnableIPAMD>",
  "EnableCustomizedPodCidr": "<EnableCustomizedPodCidr>",
  "DisableVpcCniMode": "<DisableVpcCniMode>",
  "Phase": "<Phase>",
  "Reason": "<Reason>",
  "SubnetIds": "<SubnetIds>"
}
```

### 2. 查询Pod数量限制

确认独占网卡模式下的 Pod 上限。

```bash
tccli tke DescribeVpcCniPodLimits --region ap-guangzhou --Zone ap-guangzhou-3 --InstanceFamily S5 --InstanceType S5.MEDIUM2
# expected: "TKEDirectENI": 9 （独占网卡模式下仅支持 9 个 Pod/节点）
```

```json
{
  "TotalCount": "<TotalCount>",
  "PodLimitsInstanceSet": "<PodLimitsInstanceSet>",
  "Zone": "<Zone>",
  "InstanceFamily": "<InstanceFamily>",
  "InstanceType": "<InstanceType>",
  "PodLimits": "<PodLimits>"
}
```

### 3. 启用tke-direct-eni模式

> **注意**：此操作为破坏性操作，将改变集群 Pod 网络模型，已有 Pod 可能需重建。

```bash
tccli tke EnableVpcCniNetworkType \
  --region ap-guangzhou \
  --ClusterId cls-xxxxxxxx \
  --SubnetIds '["subnet-xxxxxxxx"]' \
  --VpcCniType "tke-direct-eni"
# expected: RequestId 返回，异步任务提交成功
```

### 4. 查询启用进度

```bash
tccli tke DescribeEnableVpcCniProgress --region ap-guangzhou --ClusterId cls-xxxxxxxx
# expected: "Status": "Running" 表示正在开启中，"Status": "Succeed" 表示开启完成
```

```json
{
  "Status": "<Status>",
  "ErrorMessage": "<ErrorMessage>",
  "RequestId": "<RequestId>"
}
```

## 验证

> 确认 tke-direct-eni 模式已成功开启。

```bash
tccli tke DescribeIPAMD --region ap-guangzhou --ClusterId cls-xxxxxxxx
# expected: "EnableIPAMD": true, "VpcCniType": "tke-direct-eni"

tccli tke DescribeEnableVpcCniProgress --region ap-guangzhou --ClusterId cls-xxxxxxxx
# expected: "Status": "Succeed"

# 创建一个测试 Pod，验证其独占网卡的网络性能
kubectl run test-direct-eni-pod --image=nginx --restart=Never
kubectl get pod test-direct-eni-pod -o wide
# expected: IP 地址属于 VPC 子网 CIDR，Pod 独占一张 ENI
```

```json
{
  "EnableIPAMD": "<EnableIPAMD>",
  "EnableCustomizedPodCidr": "<EnableCustomizedPodCidr>",
  "DisableVpcCniMode": "<DisableVpcCniMode>",
  "Phase": "<Phase>",
  "Reason": "<Reason>",
  "SubnetIds": "<SubnetIds>"
}
```

## 清理

> **警告**：关闭 VPC-CNI 模式会改变所有 Pod 的 IP 地址，需重建 Pod。`tke-direct-eni` 模式下每个 Pod 独占的 ENI 将被释放。

```bash
tccli tke DisableVpcCniNetworkType --region ap-guangzhou --ClusterId cls-xxxxxxxx
# expected: RequestId 返回，异步任务提交。完成后集群恢复 GR 模式，Pod 需重建

# 清理测试 Pod
kubectl delete pod test-direct-eni-pod
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| GR 集群查询 VPC-CNI 进度报错 `FailedOperation.EnableVPCCNIFailed: cluster is not vpc-cni cluster` | 执行 `DescribeIPAMD` 确认 `EnableIPAMD` 为 `false` | 集群未启用 VPC-CNI，无法查询相关进度 | 先调用 `EnableVpcCniNetworkType --VpcCniType "tke-direct-eni"` 启用 |
| 启用时报错 `LimitExceeded` | 查询 VPC 子网信息，检查可用 IP 及 ENI 配额 | 子网 IP 不足或账号 ENI 配额已达上限（独占模式消耗更多 ENI） | 添加新子网或提交工单提升 ENI 配额 |
| 开启后 Pod 调度失败 | `kubectl describe pod` 查看事件，`kubectl get nodes` 看节点状态 | 节点 ENI 数量已达上限，无法为更多 Pod 分配独占 ENI | 扩容节点，或改用 `tke-route-eni` 模式提升 Pod 密度 |
| `tke-direct-eni` 模式下 Pod 数量远超预期上限 | 检查 `DescribeVpcCniPodLimits` 确认该机型 `TKEDirectENI` 值 | 独占网卡模式下单节点 Pod 上限取决于机型 ENI 配额 | 合理规划节点数量，或评估使用 `tke-route-eni` 模式 |

## 下一步

- [多Pod共享网卡模式](https://cloud.tencent.com/document/product/457/50356) -- page_id `50356`
- [固定IP使用方法](https://cloud.tencent.com/document/product/457/34994) -- page_id `34994`
- [VPC-CNI模式介绍](https://cloud.tencent.com/document/product/457/50355) -- page_id `50355`

## 控制台替代

在 TKE 控制台进入集群详情页，点击"基本信息" > "网络信息" > "VPC-CNI 模式"操作区，选择"Pod间独占网卡（tke-direct-eni）"，指定子网后开启。控制台提供进度条展示。CLI 方式适用于自动化脚本和 IaC 部署场景，尤其在需要对多集群批量切换网络模式的场景下更高效。
