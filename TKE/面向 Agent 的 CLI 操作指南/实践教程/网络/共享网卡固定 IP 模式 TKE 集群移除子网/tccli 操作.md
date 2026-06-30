# 共享网卡固定 IP 模式 TKE 集群移除子网（tccli）

> 对照官方：[共享网卡固定 IP 模式 TKE 集群移除子网](https://cloud.tencent.com/document/product/457/126847) · page_id `126847`

## 概述

VPC-CNI 共享网卡固定 IP 模式下，Pod 从指定子网分配固定 IP 并随 Pod 生命周期保留。移除子网是不可逆高风险操作：若仍有 Pod 使用该子网的 IP，移除后这些 Pod 将失联且无法重建。本文通过 `DescribeClusters` 校验当前 VPC-CNI 配置、核查 Pod IP 归属，再用 `ModifyClusterVpcCni` 更新 `EniSubnetIds` 完成移除。操作以控制面 tccli 为主，仅 IP 归属核查需 kubectl 数据面。

> **kubectl 数据面不可达**：CAM拒绝公网端点(strategyId:240463971)，内网需VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

## 前置条件

- [环境准备](../../../环境准备.md)：`tccli` 已配置
- 集群为 VPC-CNI 共享网卡固定 IP 模式，MANAGED_CLUSTER，K8s 1.30.0，ap-guangzhou
- 已确认保留子网 IP 容量充足（容纳待迁移 Pod 的固定 IP）
- 操作前已对集群打快照或确认可接受短暂影响（移除不可逆）

### 环境检查

```bash
tccli --version
# expected: tccli version >= 1.0.0

tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterStatus "Running"，ClusterNetworkSettings 中 VpcCniType 为共享网卡固定 IP 模式
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
| 查看 Pod IP 归属（数据面） | `kubectl get pods -A -o wide`（需 VPN/IOA） | 是 |

## 操作步骤

### 步骤 1：确认当前 VPC-CNI 子网配置（tccli）

#### 选择依据

- `DescribeClusters` 返回的 `ClusterNetworkSettings.EniSubnetIds` 即 ModifyClusterVpcCni 要更新的目标字段
- 固定 IP 模式下 `VpcCniType` 应为 `SharedIPWithStatic`（或集群创建时指定的固定 IP 标识），确认后再操作

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
  "VpcCniType": "SharedIPWithStatic"
}
```

记录 `EniSubnetIds` 当前完整列表，后续移除时需传入剔除目标子网后的新列表。

### 步骤 2：核查目标子网无活跃固定 IP Pod（需 VPN/IOA）

#### 选择依据

- 固定 IP 模式下 Pod IP 绑定到子网。移除前必须确认目标子网 CIDR 范围内无 Pod IP，否则移除后这些 Pod 失联
- 先用 `DescribeSubnets` 查子网 CIDR，再用 kubectl 按网段过滤 Pod IP

```bash
# 控制面：查询目标子网 CIDR 和可用 IP
tccli vpc DescribeSubnets --region <Region> --SubnetIds '["SUBNET_ID"]'
# expected: 返回 CidrBlock 和 AvailableIpAddressCount
```

```output
{
  "SubnetSet": [
    {
      "SubnetId": "subnet-remove",
      "CidrBlock": "10.0.2.0/24",
      "AvailableIpAddressCount": 250
    }
  ]
}
```

```bash
# 数据面：按目标子网 CIDR 前缀过滤 Pod IP（需 VPN/IOA）
kubectl get pods -A -o wide | grep "10.0.2."
# expected: 无结果（所有固定 IP Pod 已迁移出该子网）
```

```text
NAME  STATUS  AGE
...
```

> 如有结果：必须先迁移 Pod。固定 IP Pod 需删除 Pod 并清除固定 IP 注解（如 `tke.cloud.tencent.com/network-static-ip`）后重建到保留子网，再重新执行步骤 2。

### 步骤 3：核查保留子网 IP 容量（tccli）

#### 选择依据

- 移除后所有新固定 IP Pod 从保留子网分配。保留子网 `AvailableIpAddressCount` 必须满足未来 Pod 规模
- 固定 IP 在 Pod 删除后释放，但 StatefulSet 可能长期占用，需预留余量

```bash
tccli vpc DescribeSubnets --region <Region> --SubnetIds '["KEEP_SUBNET_ID_1"]'
# expected: AvailableIpAddressCount 远大于当前 Pod 数（建议 >= 2 倍 Pod 数）
```

### 步骤 4：移除子网（tccli）

#### 选择依据

- `ModifyClusterVpcCni` 是修改 VPC-CNI 子网配置的正确 API，`EniSubnetIds` 传入剔除目标子网后的新完整列表（非增量）
- 参数 >=4 个，使用 `--cli-input-json` 格式避免命令行过长和转义错误
- 此操作不可逆，执行前再次确认步骤 1-3 的前置条件

```bash
cat > modify-vpccni.json <<'EOF'
{
    "ClusterId": "CLUSTER_ID",
    "EniSubnetIds": ["KEEP_SUBNET_ID_1"],
    "VpcCniType": "SharedIPWithStatic"
}
EOF
tccli tke ModifyClusterVpcCni --region <Region> --cli-input-json file://modify-vpccni.json
# expected: RequestId 返回，无 Error
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

> 此步骤为不可逆操作的最终确认。务必核对 `EniSubnetIds` 已不含目标子网且集群仍 Running。

## 清理

移除子网操作本身即是清理。子网从集群 VPC-CNI 配置解绑后，VPC 子网资源本身不受影响（仍可在 VPC 控制台查看/用于其他用途）。如需彻底删除子网：

```bash
# 确认子网无任何资源占用后再删除
tccli vpc DeleteSubnet --region <Region> --SubnetId SUBNET_ID
# expected: RequestId 返回，无 Error
```

> **计费警告**：子网本身免费，但删除前必须确认无 CVM、CLB、NAT 等资源占用，否则 API 报错。删除子网不可恢复。

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `ModifyClusterVpcCni` 报 "subnet in use" | `kubectl get pods -A -o wide \| grep <子网CIDR>` | 目标子网仍有固定 IP Pod 未迁移 | 迁移所有 Pod 到保留子网后重试；固定 IP Pod 需清除注解 `tke.cloud.tencent.com/network-static-ip` 后重建 |
| 移除后新 Pod 无法分配 IP | `kubectl describe pod POD_NAME -n NAMESPACE` 查看 Events | 保留子网 IP 耗尽或 ENI 配额不足 | `tccli vpc DescribeSubnets --SubnetIds '["KEEP_SUBNET_ID"]'` 确认可用 IP；增加保留子网或扩容子网 CIDR |
| 移除后已有 Pod 网络中断 | `kubectl get pod POD_NAME -n NAMESPACE -o wide` 查看 IP 段 | 步骤 2 未完全迁移，仍有 Pod 使用已移除子网 IP | 立即删除受影响 Pod 重建到保留子网；恢复子网不可行（操作不可逆） |
| StatefulSet Pod 重建后固定 IP 失败 | `kubectl get pod POD_NAME -n NAMESPACE -o yaml \| grep network-static-ip` | StatefulSet 固定 IP 注解引用已移除子网 | 删除 Pod 并清除固定 IP 注解，重建到保留子网；或更新 StatefulSet 模板注解指向新子网 |
| `ModifyClusterVpcCni` 报权限错误 | 检查 CAM 策略 | 子账号缺少 `tke:ModifyClusterVpcCni` 权限 | 主账号或管理员授予 QcloudTKEFullAccess 或 `tke:ModifyClusterVpcCni` 操作权限 |

## 下一步

- [共享网卡非固定 IP 模式 TKE 集群移除子网](../共享网卡非固定%20IP%20模式%20TKE%20集群移除子网/tccli%20操作.md) -- page_id `126848`
- [TKE VPC-CNI 集群 Node Taint 与 Condition 配置指南](../TKE%20VPC-CNI%20集群%20Node%20Taint%20与%20Condition%20配置指南/tccli%20操作.md) -- page_id `129804`
- [在 TKE 上对 Pod 进行带宽限速](../在%20TKE%20上对%20Pod%20进行带宽限速/tccli%20操作.md) -- page_id `48766`

## 控制台替代

[容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) → 集群 → 基本信息 → 网络配置 → VPC-CNI 子网管理 → 移除子网。
