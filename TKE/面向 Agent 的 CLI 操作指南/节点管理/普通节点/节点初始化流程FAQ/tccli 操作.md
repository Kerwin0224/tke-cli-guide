# 节点初始化流程 FAQ

> 对照官方：[节点初始化流程 FAQ](https://cloud.tencent.com/document/product/457/96860) · page_id `96860`

## 概述

TKE 节点初始化分为三个阶段：安装节点组件（Cloud Init 执行、系统包安装、资源下载、节点检查、服务启动、挂盘）、等待节点注册（Kubelet 启动与注册）、以及 GPU 相关初始化（Device Plugin 安装、GPU 资源注册）。本页将流程说明转为 CLI 可操作的监控与排障指南。

以本集群 `cls-xxxxxxxx`（`ap-guangzhou`，托管集群，Kubernetes 1.30.0，containerd 1.6.9）为例，其节点池 `np-lmvsqkuu`（`kerwinwjyan-rewrite-s3-np`）中节点若新加入，将经历上述三阶段初始化流程。

## 前置条件

- [环境准备](../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:DescribeClusterInstances, tke:DescribeClusterNodePools
#    验证：查询集群内节点确认权限
tccli tke DescribeClusterInstances --region <Region> \
    --ClusterId <ClusterId>
# expected: exit 0，返回 InstanceSet（可为空）
```

```json
{
  "TotalCount": "<TotalCount>",
  "InstanceSet": "<InstanceSet>",
  "InstanceId": "<InstanceId>",
  "InstanceRole": "<InstanceRole>",
  "FailedReason": "<FailedReason>",
  "InstanceState": "<InstanceState>"
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看节点状态 | `tke DescribeClusterInstances` | 是 |
| 查看节点初始化进度 | `tke DescribeClusterInstances --Filters instance-state` | 是 |
| 查看节点池创建进度 | `tke DescribeClusterNodePoolDetail` | 是 |
| SSH 登录节点排查 | 数据面操作 | — |

## 关键字段说明

`DescribeClusterInstances` 返回结果中与节点初始化相关的核心字段：

| 字段 | 类型 | 说明 |
|------|------|------|
| `InstanceId` | String | CVM 实例 ID，格式 `ins-xxxxxxxx` |
| `InstanceRole` | String | 节点角色：`WORKER` |
| `InstanceState` | String | 节点状态：`initializing` / `running` / `abnormal` / `failed` |
| `FailedReason` | String | 失败原因（仅在状态 `failed` 时有值） |
| `LanIP` | String | 节点内网 IP |
| `InstanceType` | String | 机型 |
| `NodePoolId` | String | 所属节点池 ID |
| `CreatedTime` | String | 创建时间 |

## 操作步骤

### 步骤 1：监控节点初始化状态

```bash
# 查看集群所有节点及其初始化状态
tccli tke DescribeClusterInstances --region <Region> \
    --ClusterId <ClusterId>
# expected: exit 0，返回 InstanceSet
```

```json
{
  "TotalCount": "<TotalCount>",
  "InstanceSet": "<InstanceSet>",
  "InstanceId": "<InstanceId>",
  "InstanceRole": "<InstanceRole>",
  "FailedReason": "<FailedReason>",
  "InstanceState": "<InstanceState>"
}
```

以 `cls-xxxxxxxx` 为例：

```bash
tccli tke DescribeClusterInstances \
    --ClusterId cls-xxxxxxxx \
    --region ap-guangzhou
```

**预期输出**（节点初始化中）：

```json
{
    "InstanceSet": [
        {
            "InstanceId": "ins-example",
            "InstanceRole": "WORKER",
            "InstanceState": "initializing",
            "FailedReason": "",
            "LanIP": "172.24.0.10",
            "CreatedTime": "2026-06-17T10:00:00Z"
        }
    ],
    "TotalCount": 1
}
```

### 步骤 2：查看节点初始化失败详情

```bash
# 筛选状态异常的节点
tccli tke DescribeClusterInstances --region <Region> \
    --ClusterId <ClusterId> \
    --Filters '[
      {"Name":"instance-state","Values":["failed","abnormal"]}
    ]' \
    | jq '.InstanceSet[] | {InstanceId, InstanceState, FailedReason}'
# expected: 列出所有异常节点及其失败原因
```

```json
{
  "TotalCount": "<TotalCount>",
  "InstanceSet": "<InstanceSet>",
  "InstanceId": "<InstanceId>",
  "InstanceRole": "<InstanceRole>",
  "FailedReason": "<FailedReason>",
  "InstanceState": "<InstanceState>"
}
```

### 步骤 3：检查节点池内节点初始化进度

```bash
# 查看节点池详情，确认有多少节点处于初始化中
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId <ClusterId> \
    --NodePoolId <NodePoolId> \
    | jq '.NodePool.NodeCount.Summary'
# expected: 返回各状态节点数，如 {"Total": 3, "Initializing": 2, "Running": 1}
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

以 `np-lmvsqkuu` 为例：

```bash
tccli tke DescribeClusterNodePoolDetail \
    --ClusterId cls-xxxxxxxx \
    --NodePoolId np-lmvsqkuu \
    --region ap-guangzhou \
    | jq '.NodePool.NodeCount.Summary'
```

```json
{
  "NodePool": "<NodePool>",
  "NodePoolId": "<NodePoolId>",
  "Name": "<Name>",
  "ClusterInstanceId": "<ClusterInstanceId>",
  "LifeState": "<LifeState>",
  "LaunchConfigurationId": "<LaunchConfigurationId>",
  "AutoscalingGroupId": "<AutoscalingGroupId>",
  "Labels": []
}
```

### 步骤 4：通过 CVM 控制面确认底层实例状态

```bash
# 获取节点池内节点 CVM 实例 ID
tccli tke DescribeClusterInstances --region <Region> \
    --ClusterId <ClusterId> \
    --Filters '[{"Name":"node-pool-id","Values":["<NodePoolId>"]}]' \
    | jq '.InstanceSet[].InstanceId'
# expected: 返回实例 ID 列表

# 查询 CVM 实例状态，确认底层实例是否正常运行
tccli cvm DescribeInstances --region <Region> \
    --InstanceIds '["<INSTANCE_ID>"]' \
    | jq '.InstanceSet[] | {InstanceId, InstanceState, LatestOperation}'
# expected: 返回 CVM 实例状态和最近操作
```

```json
{
  "TotalCount": "<TotalCount>",
  "InstanceSet": "<InstanceSet>",
  "InstanceId": "<InstanceId>",
  "InstanceRole": "<InstanceRole>",
  "FailedReason": "<FailedReason>",
  "InstanceState": "<InstanceState>"
}
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<ClusterId>` | 集群 ID | 格式 `cls-xxxxxxxx` | `tccli tke DescribeClusters --region <Region>` |
| `<Region>` | 地域 | 如 `ap-guangzhou` | `tccli tke DescribeRegions` |
| `<NodePoolId>` | 节点池 ID | 格式 `np-xxxxxxxx` | `tccli tke DescribeClusterNodePools --region <Region> --ClusterId <ClusterId>` |
| `<INSTANCE_ID>` | CVM 实例 ID | 格式 `ins-xxxxxxxx` | `DescribeClusterInstances` 返回 |

## 验证

### 控制面（tccli）

```bash
# 验证 1：所有节点 InstanceState 为 running
tccli tke DescribeClusterInstances --region <Region> \
    --ClusterId <ClusterId> \
    | jq '[.InstanceSet[] | select(.InstanceState != "running")] | length'
# expected: 返回 0，表示没有非 running 状态的节点

# 验证 2：节点已分配 IP
tccli tke DescribeClusterInstances --region <Region> \
    --ClusterId <ClusterId> \
    | jq '.InstanceSet[] | {InstanceId, LanIP} | select(.LanIP == "")'
# expected: 空，所有节点均已分配内网 IP

# 验证 3：FailedReason 为空
tccli tke DescribeClusterInstances --region <Region> \
    --ClusterId <ClusterId> \
    | jq '[.InstanceSet[] | select(.FailedReason != "")] | length'
# expected: 返回 0
```

```json
{
  "TotalCount": "<TotalCount>",
  "InstanceSet": "<InstanceSet>",
  "InstanceId": "<InstanceId>",
  "InstanceRole": "<InstanceRole>",
  "FailedReason": "<FailedReason>",
  "InstanceState": "<InstanceState>"
}
```

| 验证维度 | 命令 | 预期 |
|---------|------|------|
| 节点状态 | `DescribeClusterInstances` 检查 `InstanceState` | 所有节点 `InstanceState: "running"` |
| 内网 IP | 同上，检查 `LanIP` | 所有节点均有非空内网 IP |
| 失败原因 | 同上，检查 `FailedReason` | 所有节点 `FailedReason` 为空 |
| CVM 底层 | `cvm DescribeInstances` 检查 `InstanceState` | CVM 实例为 `RUNNING` |

## 清理

无需清理。本页为只读查询，不创建任何资源。

## 排障

### 节点初始化失败（有三阶段）

节点初始化分为三个阶段，每阶段的常见故障如下：

**阶段一：安装节点组件**

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `InstanceState: "failed"` 且 `FailedReason` 含 `Cloud init failed` | `tccli cvm DescribeInstances --region <Region> --InstanceIds '["<INSTANCE_ID>"]'` 检查 CVM 状态 | 节点有特殊安全组或网络限制 | 检查安全组是否放通必要的出站规则；自定义镜像需清理 cloud init |
| `FailedReason` 含 `Install package failed` | SSH 登录节点执行 `systemctl status kubelet` 和 `journalctl -u kubelet -n 100` | 安全组阻止软件源访问，或自定义镜像修改过 yum/apt 源 | 检查安全组出站规则；还原 yum/apt 源配置 |
| `FailedReason` 含 `Download resource failed` | `ping mirrors.tencentyun.com`（从同 VPC 内其他节点测试） | 节点无法访问软件源/镜像仓库，可能是 DNS 或网络问题 | 检查 VPC DNS 配置和 NAT 网关 |
| `FailedReason` 含 `Node check failed` | `nslookup common-server.tencentyun.com` | Agent Server 不可达，安全组或 DNS 问题 | 确认节点出站连通性和 DNS 配置正常 |
| `FailedReason` 含 `Mount disk failed` | SSH 登录节点执行 `lsblk` 和 `dmesg \| grep -i mount` | 磁盘挂载失败，可能磁盘未正确初始化或格式化 | 检查数据盘是否存在且状态正常；根据 dmesg 定位具体错误 |

**阶段二：等待节点注册**

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 节点长时间 `initializing`（> 10 分钟） | `DescribeClusterInstances` 持续监控 `InstanceState` | Kubelet 未能成功注册到集群 | 保留 region、ClusterId、InstanceId、RequestId → 登录控制台查看详细状态 → 仍无法解决则提交工单 |
| `FailedReason` 含 `Kubelet start failed: unknown policy` | 检查节点池自定义参数是否有 `--cgroup-driver` 等 Kubelet 参数 | Kubelet 参数设置错误（如 `--feature-gates` 参数格式应为 `key=value` 而非 `key: value`） | 在 `ModifyClusterNodePool` 中修正 Kubelet 自定义参数，格式为 `string=bool` |
| `FailedReason` 含 `failed to parse kubelet flag: malformed pair` | 同上，检查自定义参数格式 | `--feature-gates` 参数格式错误 | 确认格式为 `--feature-gates=FeatureA=true,FeatureB=false` |

**阶段三：GPU 相关**

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `FailedReason` 含 `GPU component not found` | 检查是否开启了 qGPU：`DescribeClusterNodePoolDetail` → `GPUArgs` | GPU Device Plugin 未安装（如开启 qGPU）或运行中被删除 | 确保对应 GPU 组件已安装，在节点池配置中正确设置 `GPUArgs` |
| `FailedReason` 含 `GPU resource register failed` | SSH 登录节点执行 `nvidia-smi` | 节点层面看不到 GPU → 机型或驱动问题；Device Plugin 层面看不到 → 检查 `NVIDIA_DRIVER_CAPABILITIES` 环境变量 | 先在节点执行 `nvidia-smi` 判断是硬件/驱动问题还是 runtime 配置问题。GPU inforom 损坏不可自修复，需重新购买实例 |
| `FailedReason` 含 `GPU component start failed` | 检查节点资源是否充足：SSH 登录后执行 `free -h` 和 `nvidia-smi` | 节点资源不足或 Device Plugin 的 node selector 被修改 | 保证 Device Plugin 能调度到节点上；确认 GPU 驱动和 nvidia-runtime 配置正确 |

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DescribeClusterInstances` 返回 `ResourceNotFound.ClusterNotFound` | `DescribeClusters` 确认集群存在 | 集群 ID 错误 | 使用正确的 `<ClusterId>` |
| 状态为 `abnormal` 但 `FailedReason` 为空 | 保留 region、ClusterId、InstanceId、RequestId、创建 JSON | 通常是底层 CVM 或 VPC 配置问题 | 登录 [TKE 控制台](https://console.cloud.tencent.com/tke2/cluster) 查看节点事件详情 |

### 常见前置条件问题

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 使用自定义镜像初始化失败 | 检查自定义镜像是否清理了 `/var/lib/cloud/` 下的 cloud-init 数据 | 自定义镜像残留 cloud init 配置，导致 TKE 初始化脚本执行冲突 | 使用标准镜像或确保自定义镜像已清理 cloud init。ARM/海光机型注意镜像兼容性限制 |
| 消费级 GPU 卡（如 GC49）初始化失败 | 检查驱动安装情况 | 消费级卡需自行安装驱动 | 在创建节点时指定驱动安装脚本或使用预装驱动的自定义镜像 |

## 下一步

- [新增游离普通节点](../新增游离普通节点/tccli%20操作.md) —— 手动添加节点到集群
- [创建节点池](../创建节点池/tccli%20操作.md) —— 通过节点池批量创建节点
- [节点池 FAQ](../节点池FAQ/tccli%20操作.md) —— 节点池常见问题

## 控制台替代

在 [TKE 控制台](https://console.cloud.tencent.com/tke2/cluster) 进入目标集群 → **节点管理** → **节点** → 查看节点状态和事件日志。
