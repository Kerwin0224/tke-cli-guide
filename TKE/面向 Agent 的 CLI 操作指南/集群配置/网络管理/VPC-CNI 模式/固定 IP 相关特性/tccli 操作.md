# 固定 IP 相关特性（tccli）

> 对照官方：[固定 IP 相关特性](https://cloud.tencent.com/document/product/457/50358) · page_id `50358`

## 概述

VPC-CNI 固定 IP 模式下，Pod IP 具备保留、迁移、回收三大特性。IP 保留确保 Pod 重建后 IP 不变；IP 迁移允许 Pod 从故障节点迁移到同子网其他节点时保持 IP；IP 回收在 Pod 删除/缩容时释放 IP 回子网池。这些特性依赖 eniipamd 组件管理 ENI 和 IP 的绑定状态。

### 固定 IP 与 CRD 资源

| CRD | 用途 | 生命周期 |
|-----|------|---------|
| `EIPClaim`（`eipc`） | 记录 Pod 与 VPC IP 的绑定关系 | Pod 创建时自动创建，Pod 删除后根据策略回收 |
| `NEC`（`NodeEniConfig`） | 记录节点网卡和 IP 配置 | 节点加入集群时创建，与节点同生命周期 |

## 前置条件

- [环境准备](../../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId、secretKey、region 均已配置

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:DescribeClusters, tke:DescribeAddon, tke:EnableVpcCniNetworkType
#    验证
tccli tke DescribeClusters --region <Region>
# expected: exit 0，返回集群列表
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

### 资源检查

```bash
# 4. 确认集群已开启 VPC-CNI 固定 IP
tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]' \
    | jq '.Clusters[0].ClusterNetworkSettings | {Cni, VpcCniType, EnableStaticIp}'
# expected: Cni=true, VpcCniType="tke-route-eni", EnableStaticIp=true

# 5. 确认 eniipamd 组件运行
tccli tke DescribeAddon --region <Region> \
    --ClusterId CLUSTER_ID --AddonName eniipamd
# expected: Phase: "Running"
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

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看 EIPClaim（IP 绑定） | 数据面 `kubectl get eipc` | 是 |
| 查看节点 ENI 配置 | 数据面 `kubectl get nec` | 是 |
| 查看固定 IP 状态 | `DescribeClusters`（`EnableStaticIp`） | 是 |
| 手动释放 IP | 数据面 `kubectl delete eipc` | 是 |

## 关键字段说明

固定 IP 相关的核心参数和注解：

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `EnableStaticIp` | Boolean | 是（开启时） | `true`，且 `VpcCniType` 须为 `tke-route-eni` | `false` 时 IP 不保留，重建后变化 |
| `IsNonStaticIpMode` | Boolean | 否 | `true` 时固定 IP 与非固定 IP 共存 | 混部模式下需注意 Pod 注解区分 |
| `tke.cloud.tencent.com/networks` | Pod 注解 | 是 | `"tke-route-eni"` | 缺少注解 → Pod 使用容器 CIDR IP（非 VPC IP） |
| `tke.cloud.tencent.com/eni-ip` | 资源请求 | 是 | `"1"`（每 Pod 固定申请 1 个 ENI IP） | 缺少声明 → Pod 无法获取 VPC IP |
| `tke.cloud.tencent.com/eip-claim-delete-policy` | Pod 注解 | 否 | `"Never"`（保留 IP）/ 省略（默认回收） | `"Never"` + 忘清理 → IP 泄露 |

## 操作步骤

### 特性 1：IP 保留 — 查看 IP 绑定

Pod 创建后，IP 绑定信息记录在 `EIPClaim` CRD 中：

```bash
# 查看所有 IP 绑定
kubectl get eipc -A
# expected: 列出 Pod 名称、绑定的 VPC IP、状态
```

**预期输出**：

```text
NAMESPACE   NAME             EIP           STATUS
default     fixed-ip-app-0   10.0.1.15     Bound
default     fixed-ip-app-1   10.0.1.16     Bound
```

```bash
# 查看单个绑定的详细信息
kubectl get eipc fixed-ip-app-0 -n default -o yaml
# expected: spec 中包含 Pod 名称、子网、IP 等信息
```

```text
NAME  STATUS  AGE
...
```

```bash
# 确认 Pod 的 IP 注解
kubectl describe pod fixed-ip-app-0 -n default | grep "tke.cloud.tencent.com"
# expected: 包含 eip-claim-name、vpc-ip-address 等注解
```

```text
NAME  STATUS  AGE
...
```

### 特性 2：IP 迁移 — Pod 重建验证

当 Pod 所在节点故障，Pod 被调度到同子网其他节点时，IP 保持不变：

```bash
# 1. 记录当前 Pod IP 和节点
kubectl get pods -l app=fixed-ip-app -o wide
# expected: 记录 IP 和 NODE

# 2. 删除 Pod 触发重建
kubectl delete pod fixed-ip-app-0

# 3. 确认新 Pod 的 IP 不变
kubectl get pods -l app=fixed-ip-app -o wide
# expected: 新 Pod 的 IP 与第 1 步记录的 IP 相同，NODE 可能不同
```

```text
NAME  STATUS  AGE
...
```

> **迁移限制**：Pod 只能迁移到同一子网的节点。如果目标子网中没有可用节点，Pod 将 Pending，需在目标子网中添加节点。

### 特性 3：IP 回收 — 手动与自动

#### 手动回收

```bash
# 1. 先删除 Pod
kubectl delete pod fixed-ip-app-0

# 2. 再删除 EIPClaim 释放 IP
kubectl delete eipc fixed-ip-app-0 -n default
# expected: eipclaim.tke.cloud.tencent.com "fixed-ip-app-0" deleted

# 3. 验证 IP 已释放
kubectl get eipc fixed-ip-app-0 -n default
# expected: Error from server (NotFound): eipclaims... "fixed-ip-app-0" not found
```

```text
NAME  STATUS  AGE
...
```

#### 自动过期回收（eniipamd 配置）

eniipamd 组件支持自动回收机制。查看当前配置：

```bash
kubectl get deploy tke-eni-ipamd -n kube-system -o yaml \
    | grep -E "claim-expired-duration|enable-ownerref"
# expected: 显示过期时长和 ownerRef 开关
```

```text
NAME  STATUS  AGE
...
```

| 参数 | 默认值 | 作用 |
|------|--------|------|
| `--claim-expired-duration` | 未设置（不回收） | 设置后自动回收超过持续时间的孤立 IP 绑定 |
| `--enable-ownerref` | `false` | `true` 时，Pod 删除后级联删除 EIPClaim（仅增量 Pod） |

### 特性 4：IP 保留策略（`eip-claim-delete-policy`）

```bash
# 查看 Pod 的保留策略
kubectl get pod fixed-ip-app-0 -n default -o json \
    | jq '.metadata.annotations["tke.cloud.tencent.com/eip-claim-delete-policy"]'
# expected: "Never" 或 null
```

```text
NAME  STATUS  AGE
...
```

| 策略值 | 行为 | 适用场景 |
|--------|------|---------|
| 未设置（默认） | Pod 删除后 IP 自动释放 | 测试环境，IP 不关键 |
| `"Never"` | Pod 删除后 IP 保留，需手动 `kubectl delete eipc` | 生产有状态服务，IP 不可丢失 |

## 验证

### 控制面（tccli）

```bash
# 确认集群固定 IP 状态
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]' \
    | jq '.Clusters[0].ClusterNetworkSettings | {EnableStaticIp, IsNonStaticIpMode, Subnets}'
# expected: EnableStaticIp=true
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

### 数据面（须 VPN/IOA）

```bash
# 1. 查看所有 IP 绑定状态
kubectl get eipc -A -o wide
# expected: 列出所有绑定，状态 Bound

# 2. 查看 eniipamd 组件运行状态
kubectl get pods -n kube-system -l app=tke-eni-ipamd
# expected: 至少 1 个 Pod Running

# 3. 验证 NEC 配置
kubectl get nec
# expected: 每个 VPC-CNI 节点有 1 条记录
```

```text
NAME  STATUS  AGE
...
```

### 验证维度

| 维度 | 命令 | 预期 |
|------|------|------|
| EIPClaim 数量 | `kubectl get eipc -A \| wc -l` | 匹配固定 IP Pod 数 |
| 重建后 IP 保持 | 删除再查看 Pod IP | IP 不变 |
| eniipamd 运行 | `kubectl get pods -n kube-system -l app=tke-eni-ipamd` | Running |
| 保留策略 | `kubectl get pod -o json \| jq '.annotations'` | 策略符合预期 |

## 清理

> **警告**：`kubectl delete sts` 不会自动清理 EIPClaim。遗忘清理将导致 VPC 子网 IP 泄露，IP 耗尽后新 Pod 无法获取 IP。

```bash
# 1. 清理前状态检查
kubectl get eipc -A
# 确认是待清理的 IP 绑定

# 2. 删除工作负载
kubectl delete sts fixed-ip-app -n default

# 3. 释放所有关联 IP 绑定
kubectl get eipc -n default -o name | xargs -r kubectl delete -n default

# 4. 验证所有绑定已释放
kubectl get eipc -A
# expected: 空（或只有非测试 Pod 的绑定）
```

```text
NAME  STATUS  AGE
...
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl get eipc` 返回 `error: the server doesn't have a resource type "eipclaims"` | `kubectl api-resources \| grep eipc` | eniipamd 组件未安装或未完成初始化 | 确认 `tccli tke DescribeAddon --region <Region> --ClusterId CLUSTER_ID --AddonName eniipamd` 返回 Running |
| `kubectl delete eipc` 一直 pending | `kubectl describe eipc EIP_NAME -n NS` 查看状态 | Pod 尚未删除，CRD 有保护机制 | 先 `kubectl delete pod POD_NAME`，再 `kubectl delete eipc` |
| `DescribeClusters` 返回 `EnableStaticIp=false` | 检查开启时参数 | VPC-CNI 开启了但未开启固定 IP | 需重建集群或确认当前配置（开启后无法动态切换固定 IP 模式） |

### 功能异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Pod 重建后 IP 变化 | `kubectl describe pod POD_NAME \| grep "tke.cloud.tencent.com"` 检查注解 | Pod 未声明 `tke.cloud.tencent.com/networks: "tke-route-eni"` 或 `eni-ip` 资源 | 在 Pod spec 中添加注解和资源声明后重建 |
| 误删 Pod 后 IP 丢失 | `kubectl get eipc -n NS` 查绑定 | 未设置 `eip-claim-delete-policy: "Never"` | 下次部署时在 Pod 注解中添加 `tke.cloud.tencent.com/eip-claim-delete-policy: "Never"` |
| IP 泄露（子网 IP 耗尽） | `kubectl get eipc -A \| wc -l` 统计残留数 | 已删除 Pod 的 EIPClaim 未清理 | 手动 `kubectl delete eipc` 清理孤立绑定；开启 `--claim-expired-duration` 自动回收 |
| 故障节点上的 Pod 无法迁移 | `kubectl get nodes -o wide \| grep SUBNET_ID` | 目标子网无健康节点 | 在目标子网中添加节点或扩展子网 |
| Pod Pending，Events 显示 IP 不足 | `tccli vpc DescribeSubnets --region <Region> --SubnetIds '["SUBNET_ID"]' \| jq '.SubnetSet[0].AvailableIpAddressCount'` | 子网可用 IP 不足 | 添加更多子网到 VPC-CNI 配置，或手动释放泄露的 IP |

## 下一步

- [固定 IP 使用方法](../固定 IP 使用方法/tccli 操作.md) -- page_id `34994`
- [非固定 IP 模式使用说明](../非固定 IP 模式使用说明/tccli 操作.md) -- page_id `64940`
- [VPC-CNI 模式介绍](../VPC-CNI 模式介绍/tccli 操作.md) -- page_id `45709`
- [VPC-CNI（eniipamd）组件介绍](../VPC-CNI（eniipamd）组件介绍/tccli 操作.md)

## 控制台替代

[容器服务控制台 → 集群 → 工作负载](https://console.cloud.tencent.com/tke2/cluster) → 选择 StatefulSet → 查看 Pod 列表，IP 列显示 VPC 子网 IP。固定 IP 开关在「集群 → 基本信息」查看。
