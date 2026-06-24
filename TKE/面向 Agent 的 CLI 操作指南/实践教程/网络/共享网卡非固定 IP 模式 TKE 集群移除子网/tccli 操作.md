# 共享网卡非固定 IP 模式 TKE 集群移除子网（tccli）

> 对照官方：[共享网卡非固定 IP 模式 TKE 集群移除子网](https://cloud.tencent.com/document/product/457/126848) · page_id `126848`

## 概述

VPC-CNI 共享网卡非固定 IP 模式下，Pod IP 动态分配，Pod 删除后 IP 立即释放。移除子网风险远低于固定 IP 模式（对照 [固定 IP 模式移除子网](../共享网卡固定%20IP%20模式%20TKE%20集群移除子网/tccli%20操作.md) page_id `126847`）：移除后 Pod 会被重新调度到保留子网，无需逐个迁移固定 IP。本文通过 `DescribeClusters` 核查配置、校验保留子网 IP 容量，再用 `ModifyClusterVpcCni` 更新 `EniSubnetIds` 完成移除，操作以控制面 tccli 为主。

> **kubectl 数据面不可达**：CAM拒绝公网端点(strategyId:240463971)，内网需VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

## 前置条件

- [环境准备](../../../环境准备.md)：`tccli` 已配置
- 集群为 VPC-CNI 共享网卡非固定 IP 模式，MANAGED_CLUSTER，K8s 1.30.0，ap-guangzhou
- 保留子网 IP 容量充足，可容纳移除子网上现有 Pod 重新调度后的 IP 需求
- 建议在业务低峰期操作，避免移除瞬间 Pod 重调度影响流量

### 环境检查

```bash
tccli --version
# expected: tccli version >= 1.0.0

tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterStatus "Running"，ClusterNetworkSettings 中 VpcCniType 为共享网卡非固定 IP 模式
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

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 查看集群 VPC-CNI 配置 | `tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'` | 是 |
| 查看子网 IP 余量 | `tccli vpc DescribeSubnets --region <Region> --SubnetIds '["SUBNET_ID"]'` | 是 |
| 修改集群 VPC-CNI 子网 | `tccli tke ModifyClusterVpcCni --region <Region> --cli-input-json file://modify-vpccni.json` | 否 |
| 查看节点池子网 | `tccli tke DescribeClusterNodePools --region <Region> --ClusterId CLUSTER_ID` | 是 |
| 查看 Pod IP（数据面） | `kubectl get pods -A -o wide`（需 VPN/IOA） | 是 |

## 操作步骤

### 步骤 1：确认当前子网配置和 IP 可用性（tccli）

#### 选择依据

- `DescribeClusters` 返回的 `EniSubnetIds` 为待修改的完整列表。非固定 IP 模式下移除子网时，列表中剔除目标子网即可
- `VpcCniType` 确认为非固定 IP 模式（非 `SharedIPWithStatic`），避免误操作固定 IP 集群

```bash
tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]' \
    --filter 'Clusters[0].ClusterNetworkSettings'
# expected: 返回 VpcId、Subnets、EniSubnetIds 列表
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

```output
{
  "VpcId": "vpc-example",
  "Subnets": ["subnet-keep1", "subnet-remove"],
  "EniSubnetIds": ["subnet-keep1", "subnet-remove"],
  "VpcCniType": "SharedIP"
}
```

```bash
# 查询所有子网 IP 可用性
tccli vpc DescribeSubnets --region <Region> --SubnetIds '["KEEP_SUBNET_ID_1","SUBNET_ID"]'
# expected: 返回各子网 CidrBlock 和 AvailableIpAddressCount
```

```output
{
  "SubnetSet": [
    {"SubnetId": "subnet-keep1", "CidrBlock": "10.0.1.0/24", "AvailableIpAddressCount": 200},
    {"SubnetId": "subnet-remove", "CidrBlock": "10.0.2.0/24", "AvailableIpAddressCount": 50}
  ]
}
```

#### 选择依据：保留子网容量判断

- 目标子网（subnet-remove）有 50 个 Pod IP 在用，保留子网（subnet-keep1）有 200 可用 IP
- 200 > 50，保留子网容量充足，可安全移除
- 若保留子网可用 IP < 目标子网在用 Pod 数，需先扩容保留子网或新增子网

### 步骤 2：核查目标子网 Pod 数（需 VPN/IOA）

#### 选择依据

- 非固定 IP 模式下 Pod 可重调度，但仍需了解影响范围（多少 Pod 将被重调度）
- 按目标子网 CIDR 前缀过滤 Pod IP，估算重调度规模

```bash
# 数据面：统计目标子网上的 Pod 数（需 VPN/IOA）
kubectl get pods -A -o wide | grep "10.0.2." | wc -l
# expected: 输出数字，即待重调度的 Pod 数（示例 50）
```

```text
NAME  STATUS  AGE
...
```

> 非固定 IP 模式下无需逐个迁移 Pod。移除子网后，ipamd 会自动从保留子网重新分配 IP 给受影响 Pod。建议在低峰期操作以减少重调度抖动。

### 步骤 3：移除子网（tccli）

#### 选择依据

- `ModifyClusterVpcCni` 是修改 VPC-CNI 子网配置的正确 API，`EniSubnetIds` 传入剔除目标子网后的新完整列表
- 参数 >=4 个，使用 `--cli-input-json` 格式
- `VpcCniType` 传 `SharedIP` 保持非固定 IP 模式不变

```bash
cat > modify-vpccni.json <<'EOF'
{
    "ClusterId": "CLUSTER_ID",
    "EniSubnetIds": ["KEEP_SUBNET_ID_1"],
    "VpcCniType": "SharedIP"
}
EOF
tccli tke ModifyClusterVpcCni --region <Region> --cli-input-json file://modify-vpccni.json
# expected: RequestId 返回，无 Error
```

### 步骤 4：触发 Pod 重调度到保留子网（需 VPN/IOA）

#### 选择依据

- 非固定 IP 模式下，移除子网后 ipamd 不再从该子网分配新 IP，但已分配的旧 IP Pod 仍使用旧子网 IP
- 需重启受影响 Pod（滚动重启 Deployment/重建 Pod）让其从保留子网重新分配 IP

```bash
# 滚动重启使用目标子网的 Deployment（需 VPN/IOA）
kubectl rollout restart deployment -n NAMESPACE APP_NAME
# expected: deployment restarted

# 或按 label 重启所有受影响 Pod
kubectl delete pods -n NAMESPACE -l app=APP_NAME --grace-period=30
# expected: pods deleted，ipamd 从保留子网重新分配 IP
```

## 验证

```bash
# 确认子网已从集群配置移除
tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]' \
    --filter 'Clusters[0].ClusterNetworkSettings.EniSubnetIds'
# expected: 列表中不含已移除的 SUBNET_ID
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

```output
{
  "EniSubnetIds": ["subnet-keep1"]
}
```

```bash
# 确认集群状态正常
tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]' \
    --filter 'Clusters[0].ClusterStatus'
# expected: "Running"

# 确认保留子网 IP 余量充足
tccli vpc DescribeSubnets --region <Region> --SubnetIds '["KEEP_SUBNET_ID_1"]'
# expected: AvailableIpAddressCount 仍大于 0
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

```bash
# 数据面：确认 Pod IP 已全部从保留子网分配（需 VPN/IOA）
kubectl get pods -A -o wide | grep "10.0.2."
# expected: 无结果（所有 Pod 已重调度到 10.0.1.x 等保留子网）

kubectl get pods -A -o wide | grep "10.0.1."
# expected: Pod IP 均来自保留子网 CIDR
```

```text
NAME  STATUS  AGE
...
```

## 清理

移除子网操作本身即是清理。子网从集群 VPC-CNI 配置解绑后，VPC 子网资源不受影响。如需彻底删除子网：

```bash
# 确认子网无任何资源占用后再删除
tccli vpc DeleteSubnet --region <Region> --SubnetId SUBNET_ID
# expected: RequestId 返回，无 Error
```

> **计费警告**：子网本身免费，但删除前必须确认无 CVM、CLB、NAT、ENI 等资源占用，否则 API 报错。删除子网不可恢复。

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `ModifyClusterVpcCni` 报 "subnet in use" | `tccli vpc DescribeSubnets --SubnetIds '["SUBNET_ID"]'` 查看 IP 占用 | 节点 ENI 仍绑定目标子网 IP | 先 `kubectl drain` 目标子网节点，待 Pod 迁移后再移除子网 |
| 移除后新 Pod 调度失败（无 IP） | `kubectl describe pod POD_NAME -n NAMESPACE` 查看 Events | 保留子网 IP 耗尽或 ENI 配额不足 | `tccli vpc DescribeSubnets --SubnetIds '["KEEP_SUBNET_ID"]'` 确认可用 IP；扩容子网 CIDR 或新增子网到 `EniSubnetIds` |
| 移除后部分 Pod 仍使用旧子网 IP | `kubectl get pods -A -o wide \| grep <旧CIDR>` | 非固定 IP 模式下旧 IP Pod 未重启，ipamd 未重分配 | `kubectl rollout restart` 或 `kubectl delete pod` 触发重调度，Pod 从保留子网重新分配 IP |
| ENI 清理卡住 | `tccli vpc DescribeNetworkInterfaces --region <Region> --NetworkInterfaceIds '[]'`（按子网过滤） | ipamd 未及时释放目标子网上的 ENI | 等待 1-2 分钟 ipamd 自动清理；若仍卡住，重启 ipamd：`kubectl rollout restart deployment tke-eni-ipamd -n kube-system` |
| `ModifyClusterVpcCni` 报权限错误 | 检查 CAM 策略 | 子账号缺少 `tke:ModifyClusterVpcCni` 权限 | 主账号或管理员授予 QcloudTKEFullAccess 或 `tke:ModifyClusterVpcCni` 操作权限 |
| Pod 重调度后仍无法获取 IP | `kubectl get pods -n NAMESPACE -o wide`；`tccli vpc DescribeSubnets` | 保留子网 ENI 配额满（每节点 ENI 数有上限） | 增加保留子网节点数，或新增子网到 `EniSubnetIds` 扩容 IP 池 |

## 下一步

- [共享网卡固定 IP 模式 TKE 集群移除子网](../共享网卡固定%20IP%20模式%20TKE%20集群移除子网/tccli%20操作.md) -- page_id `126847`
- [TKE VPC-CNI 集群 Node Taint 与 Condition 配置指南](../TKE%20VPC-CNI%20集群%20Node%20Taint%20与%20Condition%20配置指南/tccli%20操作.md) -- page_id `129804`
- [在 TKE 上对 Pod 进行带宽限速](../在%20TKE%20上对%20Pod%20进行带宽限速/tccli%20操作.md) -- page_id `48766`

## 控制台替代

[容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) → 集群 → 基本信息 → 网络配置 → VPC-CNI 子网管理 → 移除子网。
