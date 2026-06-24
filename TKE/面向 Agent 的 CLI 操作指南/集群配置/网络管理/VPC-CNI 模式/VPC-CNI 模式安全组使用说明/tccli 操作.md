# VPC-CNI 模式安全组使用说明（tccli）

> 对照官方：[VPC-CNI 模式安全组使用说明](https://cloud.tencent.com/document/product/457/50360) · page_id `50360`

## 概述

VPC-CNI 模式下，Pod 的弹性网卡（ENI）可绑定独立的安全组，实现 Pod 级网络访问控制。默认未开启安全组特性，ENI 不绑定安全组。开启后通过 eniipamd 组件的 `vpcCni.securityGroups` 参数指定安全组，ENI 绑定安全组后立即生效。支持全局安全组（所有节点 ENI 统一）和节点级安全组（每个节点单独指定，通过 NEC CRD 管理）。目前仅支持**多 Pod 共享网卡**模式（routeEni）。

### 安全组配置方式

| 方式 | 适用版本 | 配置方法 | 生效范围 |
|------|---------|---------|---------|
| 组件管理（推荐） | eniipamd >= v3.5.0 | `tccli tke UpdateAddon` 修改 RawValues 或控制台勾选 | 所有新节点 ENI |
| ConfigMap 修改 | eniipamd >= v3.5.0 | `kubectl edit cm tke-eni-ipamd -n kube-system` | 所有节点 ENI |
| Deployment 修改 | eniipamd < v3.5.0 | `kubectl edit deploy tke-eni-ipamd -n kube-system` | 所有节点 ENI |

## 前置条件

- [环境准备](../../../../环境准备.md)
- 集群已开启 VPC-CNI（`Cni: true`）
- eniipamd 组件已安装，版本 >= v3.2.0（安全组特性最低要求）

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId、secretKey、region 均已配置

# 3. 检查 CAM 权限 — 需要以下 Action
#    tke:DescribeClusters, tke:DescribeAddon, tke:DescribeAddonValues, tke:UpdateAddon
#    vpc:CreateSecurityGroup, vpc:DescribeSecurityGroups
#    验证
tccli tke DescribeClusters --region <Region>
# expected: exit 0，返回集群列表
```

### 资源检查

```bash
# 4. 确认集群为 VPC-CNI 模式
tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]' \
    | jq '.Clusters[0].ClusterNetworkSettings | {Cni, VpcId, SubnetId}'
# expected: Cni=true

# 5. 确认 eniipamd 版本 >= v3.2.0
tccli tke DescribeAddon --region <Region> \
    --ClusterId <ClusterId> --AddonName eniipamd \
    | jq '.Addons[0] | {AddonName, AddonVersion, Phase}'
# expected: AddonVersion 如 "3.11.0"，Phase "Running" 或 "Succeeded"
```

**已验证输出**（<ClusterId>, <Region>）：

集群基本信息：
```json
{
  "ClusterId": "<ClusterId>",
  "ClusterVersion": "1.32.2",
  "ClusterStatus": "Running",
  "ClusterType": "MANAGED_CLUSTER",
  "ClusterNetworkSettings": {
    "Cni": true,
    "VpcId": "<VpcId>",
    "SubnetId": "<SubnetId>"
  },
  "ClusterNodeNum": 0
}
```

eniipamd 组件信息：
```json
{
  "Addons": [{
    "AddonName": "eniipamd",
    "AddonVersion": "3.11.0",
    "Phase": "Running"
  }]
}
```

### 安全组准备

```bash
# 6. 查询已有安全组（无则创建）
tccli vpc DescribeSecurityGroups --region <Region> --Limit 10 \
    | jq '.TotalCount'
# expected: TotalCount >= 1

# 如果没有可用安全组，创建一个
tccli vpc CreateSecurityGroup --region <Region> \
    --GroupName "<SecurityGroupName>" \
    --GroupDescription "<Description>"
# expected: exit 0，返回 SecurityGroupId
```

**已验证输出**（创建安全组）：
```json
{
  "SecurityGroup": {
    "SecurityGroupId": "sg-xxxxxxxx",
    "SecurityGroupName": "<SecurityGroupName>",
    "SecurityGroupDesc": "<Description>",
    "CreatedTime": "2026-06-23T08:19:50Z"
  }
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli/kubectl 命令 | 幂等 |
|-----------|-------------------|:--:|
| 查看 eniipamd 版本 | `tccli tke DescribeAddon --AddonName eniipamd` | 是 |
| 查看 eniipamd 默认配置 | `tccli tke DescribeAddonValues --AddonName eniipamd` | 是 |
| 查看可用安全组 | `tccli vpc DescribeSecurityGroups` | 是 |
| 创建安全组 | `tccli vpc CreateSecurityGroup` | 否 |
| 开启安全组特性（>= v3.5.0） | `tccli tke UpdateAddon --RawValues` 或控制台勾选 | 否 |
| 开启安全组特性（>= v3.5.0，ConfigMap 方式） | `kubectl edit cm tke-eni-ipamd -n kube-system` | 否 |
| 开启安全组特性（< v3.5.0） | `kubectl edit deploy tke-eni-ipamd -n kube-system` | 否 |
| 查看节点安全组 | `kubectl get nec <NodeName> -o yaml` | 是 |
| 修改节点安全组 | `kubectl edit nec <NodeName>` | 否 |
| 存量节点同步（>= v3.5.8） | `kubectl annotate node <NodeName> tke.cloud.tencent.com/need-sync-security-groups=true` | 否 |
| 存量节点同步（< v3.5.8） | 解绑-重绑注解 `disable-node-eni-security-groups` | 否 |

## 操作步骤

### 选择依据

- **eniipamd >= v3.5.0 推荐使用 UpdateAddon/控制台组件管理方式**：通过 RawValues 的 `vpcCni.securityGroups` 字段直接配置，无需 kubectl 访问，配置更标准化。
- **kubectl ConfigMap 方式**（>= v3.5.0）：需要集群端点可达（kubectl），适合已有 kubectl 访问的运维环境。
- **kubectl Deployment 方式**（< v3.5.0）：旧版本不支持组件管理界面，只能通过修改 Deployment 参数实现。

### 步骤 1：查看当前 eniipamd 安全组配置

```bash
tccli tke DescribeAddonValues --region <Region> \
    --ClusterId <ClusterId> --AddonName eniipamd \
    | jq '.Values | fromjson | .vpcCni.securityGroups'
# expected: 默认 {"enableSecurityGroups": false, "securityGroups": []}
```

**已验证输出**（默认配置，未开启安全组）：
```json
{
  "enableSecurityGroups": false,
  "securityGroups": []
}
```

### 步骤 2：创建或确认安全组

```bash
# 查询现有安全组
tccli vpc DescribeSecurityGroups --region <Region> --Limit 10 \
    | jq '.SecurityGroupSet[] | {SecurityGroupId, SecurityGroupName}'

# 如无可用安全组，创建新安全组
tccli vpc CreateSecurityGroup --region <Region> \
    --GroupName "my-pod-security-group" \
    --GroupDescription "VPC-CNI Pod ENI 安全组"
# expected: exit 0，返回 SecurityGroupId
```

### 步骤 3：开启安全组特性（>= v3.5.0，组件管理方式）

#### 构造 RawValues（JSON -> Base64）

修改 `vpcCni.securityGroups` 字段：
```json
{
  "vpcCni": {
    "securityGroups": {
      "enableSecurityGroups": true,
      "securityGroups": ["<SecurityGroupId>"]
    }
  }
}
```

> **注意**：RawValues 必须包含完整的 eniipamd 配置 JSON，不能只传 securityGroups 字段。建议先通过 `DescribeAddon` 获取当前 RawValues，解码后修改目标字段，再重新提交。

```bash
# 1. 获取当前 RawValues
CURRENT_CONFIG=$(tccli tke DescribeAddon --region <Region> \
    --ClusterId <ClusterId> --AddonName eniipamd \
    | jq -r '.Addons[0].RawValues')

# 2. 解码 -> 修改 securityGroups -> 编码
UPDATED_CONFIG=$(echo "$CURRENT_CONFIG" | base64 -d | \
    jq '.vpcCni.securityGroups = {"enableSecurityGroups": true, "securityGroups": ["<SecurityGroupId>"]}' | \
    base64)

# 3. 更新组件配置
tccli tke UpdateAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName eniipamd \
    --AddonVersion <AddonVersion> \
    --RawValues "$UPDATED_CONFIG"
# expected: exit 0，返回 RequestId
```

**已验证输出**：
```json
{
  "RequestId": "14db97fc-acb5-4355-929d-a337623f8c98"
}
```

### 步骤 4：安全组生效逻辑

> **注意**：UpdateAddon 成功返回 RequestId 表示配置已提交。但在空集群（0 节点）上，组件 Pod 无法调度运行，需添加节点后组件 Pod 才会启动并应用安全组配置。

| `securityGroups.enableSecurityGroups` | `securityGroups.securityGroups` | 行为 |
|--------------------------------------|-------------------------------|------|
| `false` | `[]`（默认） | ENI 继承节点主网卡的安全组 |
| `true` | `[]` | ENI 不绑定额外安全组 |
| `true` | `["<SecurityGroupId>"]` | 所有节点 ENI 绑定指定安全组集合 |

### 步骤 5：ConfigMap 方式（>= v3.5.0，kubectl）

> **需要集群端点可达**：公网端点或 VPN/IOA/专线连接内网端点。

```bash
# 方式 A：直接编辑 ConfigMap
kubectl edit cm tke-eni-ipamd -n kube-system
# 在 data 中修改 securityGroups 配置

# 方式 B：patch 方式修改
kubectl patch cm tke-eni-ipamd -n kube-system \
    --type merge -p '{"data":{"tke-network-conf.yaml":"..."}}'
# expected: configmap/tke-eni-ipamd patched (no restart needed)
```

> ConfigMap 方式修改后不需要重启组件，ipamd 自动监听配置变更并生效。

### 步骤 6：Deployment 方式（< v3.5.0，kubectl）

> **需要集群端点可达**。此方式用于不支持组件管理的旧版本。

```bash
kubectl edit deploy tke-eni-ipamd -n kube-system
```

在 `spec.template.spec.containers[0].args` 中添加：
```text
--enable-security-groups
--security-groups=<SecurityGroupId>,<SecurityGroupId>
```

保存后 ipamd 自动重启。ENI 未绑定安全组的将被绑定，已绑定的将被**强制同步**至新配置。

### 步骤 7：查看与修改节点级安全组（>= v3.2.0，kubectl）

> **需要集群端点可达 + 集群有节点**。

```bash
# 查看所有节点的安全组配置
kubectl get nec -o wide
# expected: 列出各节点及其安全组

# 查看指定节点
kubectl get nec <NodeName> -o yaml | grep -A 5 securityGroups
# expected: 列出 spec.securityGroups

# 修改节点安全组（立即生效）
kubectl edit nec <NodeName>
# 修改 spec.securityGroups 列表
```

### 步骤 8：存量节点同步（kubectl）

> **需要集群端点可达 + 集群有节点**。

#### >= v3.5.8（推荐方式）

```bash
kubectl annotate node <NodeName> --overwrite \
    tke.cloud.tencent.com/need-sync-security-groups="true"
# expected: node/<NodeName> annotated
# 注解在 SG 更新完成后自动移除
```

#### < v3.5.8（强制同步）

> **警告**：强制解绑期间该节点上 Pod 网络可能中断。

```bash
# 步骤 A：清除安全组
kubectl annotate node <NodeName> --overwrite \
    tke.cloud.tencent.com/disable-node-eni-security-groups="yes"

# 步骤 B（等待 2-5 秒后）：重新绑定
kubectl annotate node <NodeName> --overwrite \
    tke.cloud.tencent.com/disable-node-eni-security-groups="no"
```

## 验证

### 控制面（tccli）

```bash
# 1. 确认 eniipamd 版本和安全组配置
tccli tke DescribeAddon --region <Region> \
    --ClusterId <ClusterId> --AddonName eniipamd \
    | jq '.Addons[0] | {AddonVersion, Phase}'
# expected: AddonVersion >= "3.2.0", Phase "Running"

# 2. 确认安全组配置已更新
tccli tke DescribeAddon --region <Region> \
    --ClusterId <ClusterId> --AddonName eniipamd \
    | jq -r '.Addons[0].RawValues' | base64 -d | \
    jq '.vpcCni.securityGroups'
# expected: {"enableSecurityGroups": true, "securityGroups": ["<SecurityGroupId>"]}

# 3. 确认安全组存在
tccli vpc DescribeSecurityGroups --region <Region> \
    --SecurityGroupIds '["<SecurityGroupId>"]' \
    | jq '.SecurityGroupSet[0].SecurityGroupId'
# expected: 返回 SecurityGroupId，不为 null
```

### 数据面（需要集群端点可达 + 集群有节点）

```bash
# 1. 确认 ipamd 安全组参数已启用
kubectl describe deploy tke-eni-ipamd -n kube-system | grep -E "enable-security-groups|security-groups"
# expected: 返回 --enable-security-groups 参数（如已开启）

# 2. 确认节点 ENI 已绑定安全组
kubectl get nec <NodeName> -o yaml | grep -A 5 securityGroups
# expected: 列出安全组 ID

# 3. 确认安全组列表与配置一致
kubectl get nec <NodeName> -o json | jq '.spec.securityGroups'
# expected: ["<SecurityGroupId>"]
```

### 验证维度

| 维度 | 命令 | 预期 |
|------|------|------|
| eniipamd 版本 | `DescribeAddon` | >= v3.2.0 |
| 安全组特性开启 | `DescribeAddonValues` -> `vpcCni.securityGroups` | `enableSecurityGroups: true` |
| 配置安全组 | 同上 -> `securityGroups` | 包含目标安全组 ID |
| 节点安全组（kubectl） | `kubectl get nec <NodeName>` | `spec.securityGroups` 非空 |
| 存量同步状态（kubectl） | `kubectl describe node <NodeName>` | 无残留同步注解（>= v3.5.8） |

## 清理

> **注意**：移除安全组配置后 ENI 恢复为不绑定安全组状态，可能暴露 Pod 网络。**生产环境不建议关闭安全组特性。**

### 关闭安全组特性（>= v3.5.0，组件管理方式）

```bash
# 将 enableSecurityGroups 设回 false
tccli tke UpdateAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName eniipamd \
    --AddonVersion <AddonVersion> \
    --RawValues '<base64-json-with-enableSecurityGroups:false>'
# expected: exit 0，返回 RequestId
```

### 关闭安全组特性（kubectl）

> **需要集群端点可达**。

```bash
# ConfigMap 方式（>= v3.5.0）
kubectl edit cm tke-eni-ipamd -n kube-system
# 删除或清空 securityGroups 配置

# Deployment 方式（< v3.5.0）
kubectl edit deploy tke-eni-ipamd -n kube-system
# 移除 --enable-security-groups 和 --security-groups 参数
# 等待 ipamd 重启生效
```

### 移除节点级安全组（kubectl）

> **需要集群端点可达 + 集群有节点**。

```bash
kubectl edit nec <NodeName>
# 删除 spec.securityGroups 字段或置空
```

### 删除自建安全组

```bash
# 确认无 ENI 绑定该安全组后再删除
tccli vpc DeleteSecurityGroup --region <Region> \
    --SecurityGroupId <SecurityGroupId>
# expected: exit 0，返回 RequestId
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DescribeAddon eniipamd` 返回 `ResourceNotFound` | `tccli tke DescribeClusters --ClusterIds '["<ClusterId>"]' \| jq '.Clusters[0].ClusterNetworkSettings.Cni'` | VPC-CNI 未开启，eniipamd 未安装 | 先 `tccli tke EnableVpcCniNetworkType` 开启 VPC-CNI |
| `UpdateAddon` 安全组配置不生效 | `tccli tke DescribeAddon --AddonName eniipamd \| jq -r '.Addons[0].RawValues' \| base64 -d \| jq '.vpcCni.securityGroups'` | RawValues 中 securityGroups 路径错误 | 正确路径为 `vpcCni.securityGroups`，不是顶层 `securityGroups` |
| `CreateClusterEndpoint --IsExtranet true` 返回 `InvalidParameter.Param` | 查看错误信息的 condition 字段 | 组织级 CAM 策略 `strategyId:240463971` 以 `tke:clusterExtranetEndpoint=true` 条件拒绝公网端点 | 使用内网端点 (`--IsExtranet false`) 或联系 CAM 管理员 |
| kubectl 连接超时 `Client.Timeout` | `tccli tke DescribeClusterEndpoints --ClusterId <ClusterId>` | 内网端点 IP (`172.24.x.x`) 从本地网络不可达 | 通过 VPN/IOA/专线或同 VPC CVM 跳板机连接 |
| `kubectl get nec` 返回 `No resources found` | `kubectl get crd \| grep nec` | eniipamd < v3.2.0，NEC CRD 未注册 | 升级 eniipamd 到 >= v3.2.0 |

### 功能异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 开启后 ENI 未绑定安全组 | `kubectl describe deploy tke-eni-ipamd -n kube-system \| grep security` | 参数拼写错误（如 `--enable-security-group` 缺 s） | 确认参数为 `--enable-security-groups`（复数） |
| 主网卡 SG 变更未同步到 ENI | `kubectl get nec <NodeName> -o yaml \| grep securityGroups` | 官方设计：主网卡 SG 变更后辅助 ENI 不自动同步 | 手动触发同步（`need-sync-security-groups` 注解） |
| 全局安全组变更后存量节点未更新 | `kubectl describe node <NodeName> \| grep security` | 官方设计：存量节点 ENI SG 不变 | 逐个节点执行同步操作 |
| 同步时出现网络中断 | `kubectl get pods -o wide` 观察 Pod 网络 | 预期行为：强制解绑期间 Pod 网络可能中断（< v3.5.8） | 建议低峰操作；>= v3.5.8 使用 `need-sync-security-groups` |
| 0 节点集群上 Addon Phase 不稳定 | `tccli tke DescribeAddon --AddonName eniipamd` 反复查询 | eniipamd 组件 Pod 需节点才能调度。无节点时系统可能自动清理和重建组件 | 添加至少 1 个节点后组件将稳定运行 |

## 下一步

- [Pod 直接绑定弹性公网 IP 使用说明](../Pod 直接绑定弹性公网 IP 使用说明/tccli 操作.md) -- page_id `64886`
- [固定 IP 使用方法](../固定 IP 使用方法/tccli 操作.md) -- page_id `34994`
- [非固定 IP 模式使用说明](../非固定 IP 模式使用说明/tccli 操作.md) -- page_id `64940`
- [VPC 安全组概述](https://cloud.tencent.com/document/product/215/20089)

## 控制台替代

[容器服务控制台 → 集群](https://console.cloud.tencent.com/tke2/cluster) → 选择目标集群 → 组件管理 → **eniipamd** → 更新配置 → 勾选「安全组」并填入安全组 ID（或不填，继承主网卡安全组）。
