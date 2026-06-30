# 多Pod共享网卡模式（tccli）

> 对照官方：[多Pod共享网卡模式](https://cloud.tencent.com/document/product/457/50356) · page_id `50356`

## 概述

多Pod共享网卡模式（`tke-route-eni`）是 VPC-CNI 的子类型之一。在此模式下，节点的弹性网卡（ENI）绑定多个辅助 IP 地址，多个 Pod 共享同一张 ENI。这种方式在单节点上可以承载更多 Pod，最大化 Pod 密度。例如 S5.MEDIUM2 机型在 `ap-guangzhou-3` 可用区支持 27 个非固定 IP Pod（`TKERouteENINonStaticIP=27`）。

与 `tke-direct-eni`（Pod 独占网卡）相比，`tke-route-eni` 以共享网卡换取更高的 Pod 密度，适合大多数通用场景。

关键 API：`EnableVpcCniNetworkType --VpcCniType "tke-route-eni"`

## 前置条件

- 集群处于 **Running** 状态
- VPC 下有可用子网，且子网内有足够空闲 IP（每个 Pod 将消耗一个子网 IP）
- 已通过 `DescribeVpcCniPodLimits` 确认目标机型的 Pod 上限满足需求
- 当前集群 `cls-xxxxxxxx` 为 GR 模式（`EnableIPAMD: false`），需通过本节操作切换至 VPC-CNI

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 查询集群VPC-CNI状态 | `tccli tke DescribeIPAMD --region ap-guangzhou --ClusterId cls-xxxxxxxx` | 是 |
| 开启tke-route-eni模式 | `tccli tke EnableVpcCniNetworkType --region ap-guangzhou --ClusterId cls-xxxxxxxx --SubnetIds '["subnet-xxxxxxxx"]' --VpcCniType "tke-route-eni"` | 否 |
| 查询开启进度 | `tccli tke DescribeEnableVpcCniProgress --region ap-guangzhou --ClusterId cls-xxxxxxxx` | 是 |
| 添加子网 | `tccli tke AddVpcCniSubnets --region ap-guangzhou --ClusterId cls-xxxxxxxx --SubnetIds '["subnet-xxxxxxxx"]'` | 是 |
| 查询Pod数量限制 | `tccli tke DescribeVpcCniPodLimits --region ap-guangzhou --Zone ap-guangzhou-3 --InstanceFamily S5 --InstanceType S5.MEDIUM2` | 是 |

## 操作步骤

### 1. 查询当前VPC-CNI状态

确认集群当前是否已启用 VPC-CNI。

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

在启用前确认目标机型的 Pod 上限。

```bash
tccli tke DescribeVpcCniPodLimits --region ap-guangzhou --Zone ap-guangzhou-3 --InstanceFamily S5 --InstanceType S5.MEDIUM2
# expected: "TKERouteENINonStaticIP": 27, "TKEDirectENI": 9
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

### 3. 启用tke-route-eni模式

> **注意**：此操作为破坏性操作，将改变集群 Pod 网络模型，已有 Pod 可能需重建。

```bash
tccli tke EnableVpcCniNetworkType \
  --region ap-guangzhou \
  --ClusterId cls-xxxxxxxx \
  --SubnetIds '["subnet-xxxxxxxx"]' \
  --VpcCniType "tke-route-eni"
# expected: RequestId 返回，异步任务提交成功
```

### 4. 查询启用进度

监控 VPC-CNI 开启状态。

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

### 5. 添加额外子网

当现有子网 IP 不足时，可动态添加新子网扩容。

```bash
tccli tke AddVpcCniSubnets \
  --region ap-guangzhou \
  --ClusterId cls-xxxxxxxx \
  --SubnetIds '["subnet-yyyyyyyy"]'
# expected: RequestId 返回，子网添加成功
```

## 验证

> 确认 tke-route-eni 模式已成功开启。

```bash
tccli tke DescribeIPAMD --region ap-guangzhou --ClusterId cls-xxxxxxxx
# expected: "EnableIPAMD": true, "VpcCniType": "tke-route-eni"

tccli tke DescribeEnableVpcCniProgress --region ap-guangzhou --ClusterId cls-xxxxxxxx
# expected: "Status": "Succeed"

# 创建一个测试 Pod 并检查其 IP 是否属于 VPC 子网
kubectl run test-eni-pod --image=nginx --restart=Never
kubectl get pod test-eni-pod -o wide
# expected: IP 地址属于 VPC 子网 CIDR 范围（非容器 CIDR）
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

> **警告**：关闭 VPC-CNI 模式会改变所有 Pod 的 IP 地址，需重建 Pod，可能影响运行业务。

```bash
tccli tke DisableVpcCniNetworkType --region ap-guangzhou --ClusterId cls-xxxxxxxx
# expected: RequestId 返回，异步任务提交。完成后集群恢复 GR 模式，Pod 需重建

# 清理测试 Pod
kubectl delete pod test-eni-pod
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| GR 集群查询 VPC-CNI 进度报错 `FailedOperation.EnableVPCCNIFailed: cluster is not vpc-cni cluster` | 执行 `DescribeIPAMD` 确认 `EnableIPAMD` 为 `false` | 集群未启用 VPC-CNI，无法查询相关进度 | 先调用 `EnableVpcCniNetworkType` 启用 tke-route-eni 模式 |
| 启用时报错 `LimitExceeded` | 查询 VPC 子网信息，检查可用 IP 数量 | VPC 子网中可用 IP 不足，无法为 Pod 分配地址 | 调用 `AddVpcCniSubnets` 添加新子网扩大 IP 池 |
| 开启后 Pod 无法获取 IP | `kubectl describe pod` 查看事件，检查 CNI 插件日志 | 节点 CNI 插件异常或子网 IP 耗尽 | 先确认子网 IP 充足，再重启 kubelet 或重建节点 |
| 开启操作超时无响应 | 检查 `DescribeEnableVpcCniProgress` 状态卡在 `Running` | 子网配置问题或后台任务阻塞 | 等待超时后重试，或提工单排查后台任务 |

## 下一步

- [固定IP使用方法](https://cloud.tencent.com/document/product/457/34994) -- page_id `34994`
- [非固定IP模式使用说明](https://cloud.tencent.com/document/product/457/64940) -- page_id `64940`
- [Pod间独占网卡模式](https://cloud.tencent.com/document/product/457/50357) -- page_id `50357`
- [VPC-CNI模式介绍](https://cloud.tencent.com/document/product/457/50355) -- page_id `50355`

## 控制台替代

在 TKE 控制台进入集群详情页，点击"基本信息" > "网络信息" > "VPC-CNI 模式"操作区，选择"多Pod共享网卡（tke-route-eni）"，指定子网后即可开启。控制台提供进度条展示开启状态，无需手动轮询 API。CLI 方式适用于自动化脚本和 IaC 部署场景。
