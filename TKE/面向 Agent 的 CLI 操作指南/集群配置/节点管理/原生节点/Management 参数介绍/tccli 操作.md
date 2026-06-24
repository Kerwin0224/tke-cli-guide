# Management 参数介绍

> 对照官方：[Management 参数介绍](https://cloud.tencent.com/document/product/457/79698) · page_id `79698`

## 概述

`Management` 是原生节点池（`Native` 类型节点池）的核心配置参数，用于控制节点的运维管理行为和故障自愈策略。它出现在 `CreateClusterNodePool` 和 `ModifyClusterNodePool` 请求中，是一个嵌套的复合结构。

`Management` 的设计目标是将**声明式管理**落地到节点运维层面：你不需要逐个 SSH 登录节点修改配置，而是通过 API 声明期望的 DNS、hosts、内核参数、kubelet 参数和自愈策略，由节点管家（Node Operator）自动下发和维护。

| 维度 | 包含 Management | 不包含 Management |
|------|:--:|:--:|
| 节点池类型 | 原生节点池（`Native`） | 普通节点池（`Regular`） |
| DNS/Hosts 管理 | 声明式，API 下发 | 需手动 SSH 修改 |
| 内核参数 | `KernelArgs` 声明，节点管家自动生效 | 需手动修改 grub/启动参数 |
| 故障自愈 | `AutoRepair` + `HealthCheckPolicyName` 自动触发 | 需自行配置监控和修复脚本 |
| Kubelet 参数 | `KubeletArgs` 声明，节点管家自动下发 | 需手动修改 `/etc/kubernetes/kubelet` |

**核心结论**：只要是原生节点池，必须配置 `Management`；普通节点池不支持此参数，传入会被忽略。

## 前置条件

- [环境准备](../../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:DescribeClusterNodePools, tke:DescribeClusterNodePoolDetail
#    tke:CreateClusterNodePool, tke:ModifyClusterNodePool
# 验证：执行 DescribeClusterNodePools 确认权限
tccli tke DescribeClusterNodePools --region <Region> --ClusterId CLUSTER_ID
# expected: exit 0，返回节点池列表（可为空）
```

```json
{
  "NodePoolSet": "<NodePoolSet>",
  "NodePoolId": "<NodePoolId>",
  "Name": "<Name>",
  "ClusterInstanceId": "<ClusterInstanceId>",
  "LifeState": "<LifeState>",
  "LaunchConfigurationId": "<LaunchConfigurationId>"
}
```

### 资源检查

```bash
# 4. 确认集群为托管集群（MANAGED_CLUSTER），原生节点池仅支持托管集群
tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterType 为 MANAGED_CLUSTER，ClusterStatus 为 Running

# 5. 确认集群 K8s 版本 >= 1.24（原生节点池最低要求）
tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]' --Filters '[{"Name":"ClusterType","Values":["MANAGED_CLUSTER"]}]'
# expected: ClusterVersion >= 1.24.0

# 6. （如有存量原生节点池）查询现有 Management 配置作为参考
tccli tke DescribeClusterNodePoolDetail --region <Region> --ClusterId CLUSTER_ID --NodePoolId NODE_POOL_ID
# expected: 返回节点池详情，包含 Native 特有字段
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
| 查看节点池 Management 配置 | `DescribeClusterNodePoolDetail` | 是 |
| 创建时配置 Management | `CreateClusterNodePool` | 否 |
| 修改 Management 参数 | `ModifyClusterNodePool` | 否 |

## 关键字段说明

`Management` 参数在 `CreateClusterNodePool` 和 `ModifyClusterNodePool` 中均为可选参数。不传则使用默认值。完整结构如下：

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `Management.Nameservers` | Array of String | 否 | DNS 服务器地址列表，如 `["8.8.8.8", "114.114.114.114"]`。会覆盖节点 `/etc/resolv.conf` | DNS 地址不可达 → 节点内域名解析失败，影响 Pod 间通信和镜像拉取 |
| `Management.Hosts` | Array of String | 否 | Hosts 文件条目，格式 `"IP HOSTNAME"`。如 `["10.0.0.1 registry.internal"]`。会追加到 `/etc/hosts` | 格式错误（无空格或多余空格）→ 节点初始化可能失败 |
| `Management.KernelArgs` | Array of String | 否 | 内核启动参数，如 `["cgroup.memory=nokmem"]`。需重启节点生效 | 内核不支持该参数 → 节点无法正常启动；需逐节点测试 |
| `Management.KubeletArgs` | Array of String | 否 | Kubelet 额外启动参数，格式 `"KEY=VALUE"`。如 `["max-pods=110", "kube-reserved=cpu=500m,memory=1Gi"]` | 参数冲突（与系统默认值）→ kubelet 启动失败，节点 NotReady |
| `Management.AutoRepair` | Boolean | 否 | 默认 `false`。`true` 时启用自动修复：NodeNotReady 超时后触发自动重装/迁移 | 误开（测试环境无需自愈）→ 节点异常时产生非预期的重建操作 |
| `Management.HealthCheckPolicyName` | String | 否 | 健康检查策略名称，与[故障自愈规则](../故障自愈规则/tccli%20操作.md)联动。值由 `DescribeHealthCheckPolicy` 查询 | 策略名不存在 → `InvalidParameter.HealthCheckPolicyName` |

## 操作步骤

### 步骤 1：查看现有原生节点池的 Management 配置

```bash
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId CLUSTER_ID \
    --NodePoolId NODE_POOL_ID
# expected: exit 0，返回完整节点池配置，含 Native.Management 字段
```

**预期输出**（含 Management 的原生节点池）：

```json
{
    "NodePool": {
        "NodePoolId": "np-example",
        "Name": "native-pool-1",
        "Type": "Native",
        "ClusterId": "cls-example",
        "LifeState": "normal",
        "Native": {
            "InstanceChargeType": "POSTPAID_BY_HOUR",
            "Management": {
                "Nameservers": ["8.8.8.8"],
                "Hosts": ["10.0.0.1 registry.internal"],
                "KernelArgs": ["cgroup.memory=nokmem"],
                "KubeletArgs": ["max-pods=110"],
                "AutoRepair": true,
                "HealthCheckPolicyName": "default-policy"
            }
        }
    },
    "RequestId": "..."
}
```

**如返回中不含 `Management` 字段**，表示为空配置，所有子字段使用默认值。

### 步骤 2：创建含 Management 的原生节点池（最小配置）

#### 选择依据

- **AutoRepair**：如集群承载生产业务，建议设置为 `true`。自愈可自动修复 NotReady 节点，减少人工干预。测试环境不建议开启，避免非预期重建。
- **HealthCheckPolicyName**：与故障自愈规则联动。先通过控制台或 API 创建健康检查策略模板，再填入策略名。如果不填，即使 `AutoRepair=true` 也不会触发自愈动作。
- **Nameservers**：内网环境建议指定内部 DNS，公网环境可用公共 DNS。不指定则使用 VPC 默认 DNS。
- **KubeletArgs**：按节点规格合理设置。例如 `max-pods` 按 ENI/IP 配额调整。
- **KernelArgs**：仅当业务对内核参数有特殊要求时才设置。错误的参数会导致节点启动失败。

```bash
cat > native-pool-management-minimal.json <<'EOF'
{
  "ClusterId": "CLUSTER_ID",
  "Name": "NODE_POOL_NAME",
  "Type": "Native",
  "Native": {
    "InstanceChargeType": "POSTPAID_BY_HOUR",
    "SubnetIds": ["SUBNET_ID"],
    "InstanceTypes": ["CXM_INSTANCE_TYPE"],
    "Management": {
      "AutoRepair": true,
      "HealthCheckPolicyName": "POLICY_NAME"
    }
  }
}
EOF

tccli tke CreateClusterNodePool --region <Region> \
    --cli-input-json file://native-pool-management-minimal.json
# expected: exit 0，返回 NodePoolId
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `CLUSTER_ID` | 目标集群 ID | 托管集群，K8s >= 1.24 | `tccli tke DescribeClusters --region <Region>` |
| `NODE_POOL_NAME` | 节点池名称 | 长度 1-60 | 自定义 |
| `SUBNET_ID` | 子网 ID | 必须属于集群所在 VPC | `tccli vpc DescribeSubnets --region <Region>` |
| `CXM_INSTANCE_TYPE` | CXM 机型 | 原生节点必须用 CXM 机型 | `tccli cvm DescribeInstanceTypeConfigs` 查询 CXM 族 |
| `POLICY_NAME` | 健康检查策略名 | 已创建的健康检查策略 | 控制台创建或通过 `DescribeHealthCheckPolicy` 查询 |

### 步骤 3：增强 Management 配置（含 DNS、Hosts、内核和 Kubelet 参数）

```bash
cat > native-pool-management-enhanced.json <<'EOF'
{
  "ClusterId": "CLUSTER_ID",
  "Name": "NODE_POOL_NAME",
  "Type": "Native",
  "Native": {
    "InstanceChargeType": "POSTPAID_BY_HOUR",
    "SubnetIds": ["SUBNET_ID"],
    "InstanceTypes": ["CXM_INSTANCE_TYPE"],
    "Management": {
      "Nameservers": ["INTERNAL_DNS_1", "INTERNAL_DNS_2"],
      "Hosts": ["10.0.0.1 registry.internal"],
      "KernelArgs": ["cgroup.memory=nokmem", "transparent_hugepage=never"],
      "KubeletArgs": ["max-pods=110", "kube-reserved=cpu=500m,memory=1Gi"],
      "AutoRepair": true,
      "HealthCheckPolicyName": "POLICY_NAME"
    }
  }
}
EOF

tccli tke CreateClusterNodePool --region <Region> \
    --cli-input-json file://native-pool-management-enhanced.json
# expected: exit 0，返回 NodePoolId
```

### 步骤 4：修改已有节点池的 Management 参数

```bash
cat > modify-nodepool-management.json <<'EOF'
{
  "ClusterId": "CLUSTER_ID",
  "NodePoolId": "NODE_POOL_ID",
  "Native": {
    "Management": {
      "Nameservers": ["NEW_DNS_1", "NEW_DNS_2"],
      "KubeletArgs": ["max-pods=110", "kube-reserved=cpu=1,memory=2Gi"],
      "AutoRepair": false
    }
  }
}
EOF

tccli tke ModifyClusterNodePool --region <Region> \
    --cli-input-json file://modify-nodepool-management.json
# expected: exit 0

# 注意：KernelArgs 修改需要重启节点才生效
```

### 步骤 5：验证 Management 参数已生效

```bash
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId CLUSTER_ID \
    --NodePoolId NODE_POOL_ID \
    | jq '.NodePool.Native.Management'
# expected: 返回修改后的 Management 配置
```

**预期输出**：

```json
{
  "Nameservers": ["NEW_DNS_1", "NEW_DNS_2"],
  "KubeletArgs": ["max-pods=110", "kube-reserved=cpu=1,memory=2Gi"],
  "AutoRepair": false
}
```

## 验证

### 验证 Management 字段完整性

```bash
# 确认所有预期的 Management 子字段存在且值正确
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId CLUSTER_ID \
    --NodePoolId NODE_POOL_ID \
    | jq '.NodePool.Native.Management'

# expected: 各字段值与创建/修改参数一致
```

```json
{
  "NodePool": "<NodePool>",
  "NodePoolId": "<NodePoolId>",
  "Name": "<Name>",
  "ClusterInstanceId": "<ClusterInstanceId>",
  "LifeState": "<LifeState>",
  "LaunchConfigurationId": "<LaunchConfigurationId>"
}
```

### 验证 AutoRepair 状态

```bash
# 确认自愈开关状态
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId CLUSTER_ID \
    --NodePoolId NODE_POOL_ID \
    | jq '.NodePool.Native.Management.AutoRepair'
# expected: true 或 false，与配置一致
```

```json
{
  "NodePool": "<NodePool>",
  "NodePoolId": "<NodePoolId>",
  "Name": "<Name>",
  "ClusterInstanceId": "<ClusterInstanceId>",
  "LifeState": "<LifeState>",
  "LaunchConfigurationId": "<LaunchConfigurationId>"
}
```

### 验证 KubeletArgs 生效（需登录节点）

```bash
# SSH 登录到原生节点后执行
ps aux | grep kubelet | grep max-pods
# expected: 输出中包含 --max-pods=110 等配置的参数
```

| 验证维度 | 命令 | 预期 |
|---------|------|------|
| Management 存在 | `DescribeClusterNodePoolDetail` → `.Native.Management` | 非 null |
| Nameservers 正确 | 同上 | 与创建参数一致 |
| KubeletArgs 正确 | 同上 | 与创建参数一致 |
| AutoRepair 开关 | 同上 → `.AutoRepair` | 与创建参数一致 |

## 清理

本页为概念说明页，无资源需清理。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `CreateClusterNodePool` 返回 `InvalidParameter.Management` | 检查 `Management` 对象中 `HealthCheckPolicyName` 是否为空或不存在 | `AutoRepair=true` 但未指定 `HealthCheckPolicyName`，或策略名不存在 | 在控制台或 API 中创建健康检查策略后填入策略名；或暂将 `AutoRepair` 设为 `false` |
| `CreateClusterNodePool` 返回 `InvalidParameter.KubeletArgs` | 检查 `KubeletArgs` 数组中的参数格式 | KubeletArgs 格式错误（如缺少 `=`、值含非法字符）或参数与系统保留参数冲突 | 确保每个条目格式为 `KEY=VALUE`，如 `"max-pods=110"`。避免修改系统保留参数 |
| 创建节点池后节点 NotReady | `kubectl describe node NODE_NAME` 查看 Conditions。同时 `tccli tke DescribeClusterInstances --region <Region> --ClusterId CLUSTER_ID` 查看节点状态 | KernelArgs 或 KubeletArgs 中的参数导致 kubelet/内核启动失败 | 移除新增的 KernelArgs 和 KubeletArgs，逐个添加并重启节点，定位具体问题参数 |

### 配置未生效

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `ModifyClusterNodePool` 修改 Management 后节点未生效 | `tccli tke DescribeClusterNodePoolDetail --region <Region> --ClusterId CLUSTER_ID --NodePoolId NODE_POOL_ID` 查看当前配置 | 部分参数（如 KernelArgs）需要重启节点才能生效；Nameservers/Hosts/KubeletArgs 由节点管家自动同步，可能有延迟 | 等待节点管家同步（通常 3-5 分钟）；对于 KernelArgs，逐节点重启（先 cordon → drain → reboot） |
| Management 配置与节点实际状态不一致 | SSH 节点执行 `cat /etc/resolv.conf` 和 `cat /etc/hosts` 对比 | 节点管家同步失败或节点已脱管 | 查看节点管家日志：`kubectl logs -n kube-system -l app=node-operator`。如节点脱管，重建节点 |

## 下一步

- [新建原生节点](../新建原生节点/tccli%20操作.md) — 完整的原生节点池创建流程
- [故障自愈规则](../故障自愈规则/tccli%20操作.md) — page_id `78650`
- [声明式操作实践](../声明式操作实践/tccli%20操作.md) — page_id `78649`
- [修改原生节点](../修改原生节点/tccli%20操作.md) — page_id `103599`
- [Pod 原地升降配](../Pod%20原地升降配/tccli%20操作.md) — page_id `79697`

## 控制台替代

[TKE 控制台 → 集群 → 节点管理 → 原生节点池详情 → 运维管理](https://console.cloud.tencent.com/tke2/cluster)：可在节点池创建或编辑页面直接配置 DNS、Hosts、内核参数、自愈开关。
