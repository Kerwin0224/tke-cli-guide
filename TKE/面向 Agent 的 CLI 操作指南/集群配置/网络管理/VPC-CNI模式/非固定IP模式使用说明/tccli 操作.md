# 非固定IP模式使用说明（tccli）

> 对照官方：[非固定IP模式使用说明](https://cloud.tencent.com/document/product/457/64940) · page_id `64940`

## 概述

VPC-CNI 非固定 IP 模式是 VPC-CNI 的默认工作模式。Pod 从 VPC 子网中动态获取 IP 地址，Pod 删除后 IP 立即释放回子网 IP 池。该模式适用于无状态工作负载（Deployment、Job 等），Pod IP 不固定，每次重建都会分配新 IP。

以 S5.MEDIUM2 实例为例，tke-route-eni 模式下单节点最多支持 27 个 Pod。当前集群 `cls-xxxxxxxx`（kerwinwjyan-rewrite-s1）尚未开启 VPC-CNI（EnableIPAMD: false），需要先开启。

## 前置条件

- 集群状态为 Running
- VPC（vpc-of73262z）下有可用子网，子网 IP 充足，需满足 Pod 数量需求
- 已安装 kubectl 并配置好集群访问凭证
- 具备 tccli 和 kubectl 的执行权限

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 查询 VPC-CNI 状态 | tccli tke DescribeIPAMD --region ap-guangzhou --ClusterId cls-xxxxxxxx | 是 |
| 开启 VPC-CNI（默认非固定 IP） | tccli tke EnableVpcCniNetworkType --region ap-guangzhou --ClusterId cls-xxxxxxxx --VpcCniType "tke-route-eni" | 否 |
| 添加子网 | tccli tke AddVpcCniSubnets --region ap-guangzhou --ClusterId cls-xxxxxxxx --SubnetIds '["subnet-xxxxxxxx"]' | 是 |
| 查询 Pod 数量限制 | tccli tke DescribeVpcCniPodLimits --region ap-guangzhou --Zone ap-guangzhou-3 --InstanceFamily S5 --InstanceType S5.MEDIUM2 | 是 |
| 验证 Pod IP 来自 VPC 子网 | kubectl get pod -o wide | 是 |

## 操作步骤

### 1. 查询 VPC-CNI 状态

```bash
tccli tke DescribeIPAMD --region ap-guangzhou --ClusterId cls-xxxxxxxx
```

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
```

当前状态：`EnableIPAMD: false`，尚未开启 VPC-CNI。

### 2. 开启 VPC-CNI（默认非固定 IP 模式）

```bash
tccli tke EnableVpcCniNetworkType \
  --region ap-guangzhou \
  --ClusterId cls-xxxxxxxx \
  --VpcCniType "tke-route-eni"
```

`VpcCniType` 可选值：
- `tke-route-eni`：共享网卡模式（共享 ENI），推荐，S5.MEDIUM2 最大支持 27 Pod
- `tke-direct-eni`：独立网卡模式（独占 ENI），S5.MEDIUM2 最大支持 9 Pod

开启后，可通过 `DescribeIPAMD` 轮询确认 `Phase` 变为 `Running`。

### 3. 添加子网

```bash
tccli tke AddVpcCniSubnets \
  --region ap-guangzhou \
  --ClusterId cls-xxxxxxxx \
  --SubnetIds '["subnet-xxxxxxxx"]'
```

若子网 IP 不足会触发 `LimitExceeded` 错误，需添加更多子网或扩展现有子网 CIDR。

### 4. 创建 Deployment 并验证

```bash
kubectl create deployment nginx-demo --image=nginx --replicas=3
kubectl get pod -o wide
```

```text
NAME  STATUS  AGE
...
```

新创建的 Pod 应分配 VPC 子网 IP（非集群容器网段 IP）。

### 5. 验证非固定 IP 行为

```bash
# 记录 Pod IP
kubectl get pod -o wide
# 删除 Pod
kubectl delete pod <POD_NAME>
# 观察重建 Pod 的 IP
kubectl get pod -o wide
```

```text
NAME  STATUS  AGE
...
```

重建后的 Pod IP 通常会发生变化（与删除前不同），这是非固定 IP 模式的预期行为。

## 验证

- `DescribeIPAMD` 返回 `EnableIPAMD: true` 且 `Phase: Running`
- Deployment Pod 成功调度并分配 VPC 子网内的 IP 地址
- `DescribeVpcCniPodLimits` 查询结果：S5.MEDIUM2 在 ap-guangzhou-3 的 tke-route-eni 模式下上限为 27（TKERouteENINonStaticIP: 27）
- 删除 Pod 后重建的 Pod 获得不同的 IP 地址（非固定 IP 预期行为）
- Pod IP 段属于 VPC 子网 CIDR 范围，而非集群容器 CIDR

## 清理

- 删除 Deployment：`kubectl delete deployment nginx-demo`
- 关闭 VPC-CNI：`DisableVpcCniNetworkType`（需谨慎，会改变所有 Pod IP 并要求 Pod 重建）
- 删除添加的子网：通过控制台或 `DisableVpcCniNetworkType` 操作

**警告**：关闭 VPC-CNI 网络模式会改变所有 Pod 的 IP 地址，可能需要重建 Pod，对业务有影响。请在变更窗口内操作。

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Pod 创建后一直 Pending | kubectl describe pod 查看 Events | VPC 子网 IP 不足或子网未添加 | 调用 AddVpcCniSubnets 添加子网，或检查 VPC 子网 IP 使用情况 |
| 开启 VPC-CNI 失败 | 检查 API 返回错误码 | VPC 下无可用子网或集群状态异常 | 确认 VPC 下有可用子网，集群状态为 Running |
| DescribeIPAMD 报错 | 查看错误信息 FailedOperation.EnableVPCCNIFailed | GR 集群未开启 VPC-CNI，无法查询 VPC-CNI 相关状态 | 先调用 EnableVpcCniNetworkType 开启 VPC-CNI |
| Pod 数量超限 | 节点上 Pod 数达到上限后新 Pod 无法调度 | 实例规格限制了单节点最大 Pod 数（S5.MEDIUM2: 27） | 扩容节点或使用更高规格实例类型 |
| Pod 重启后 IP 变化 | kubectl get pod -o wide 对比重启前后 IP | 非固定 IP 模式下 Pod IP 不保留，属正常行为 | 若需保留 IP，请开启固定 IP 模式（参见固定IP使用方法） |

## 下一步

- [固定 IP 使用方法](https://cloud.tencent.com/document/product/457/34994) -- page_id `34994`
- [多 Pod 共享网卡](https://cloud.tencent.com/document/product/457/50356) -- page_id `50356`

## 控制台替代

登录 [容器服务控制台](https://console.cloud.tencent.com/tke2/cluster/sub/detail/resource/deployment?rid=1&clusterId=cls-xxxxxxxx)，进入集群详情页，在「基本信息」→「网络信息」中可查看和修改 VPC-CNI 配置。创建 Deployment 时选择 VPC-CNI 网络模式即可使用非固定 IP。
