# 创建注册节点(专线版)

> 对照官方：[创建注册节点(专线版)](https://cloud.tencent.com/document/product/457/57917) · page_id `57917`

## 概述

通过 **专线** 网络将 IDC 或第三方云主机注册到 TKE 集群，支持节点池管理、Label 与 Taint 调度约束、自定义数据注入。专线版要求 IDC 与腾讯云 VPC 间已建立专线连接，注册节点与集群控制面通过专线内网通信，延迟低、安全性高。

创建流程：确认集群已开启注册节点能力 → 创建注册节点池（指定运行时、标签、污点等） → 获取初始化脚本 → 在目标主机执行脚本完成纳管。

## 前置条件

每条可独立 COPY-PASTE 执行验证。

```bash
# 1. 环境配置
source ../../../../环境准备.md
# expected: 无报错，tccli 和 kubectl 均可正常执行

# 2. 确认集群已开启注册节点能力
tccli tke DescribeExternalNodeSupportConfig --ClusterId <ClusterId> --region <Region> --output json
# expected: "Enabled": true，且 "NetworkType" 已配置（HostNetwork 或 CiliumBGP）
```

```json
{
  "ClusterCIDR": "<ClusterCIDR>",
  "NetworkType": "<NetworkType>",
  "SubnetId": "<SubnetId>",
  "Enabled": "<Enabled>",
  "AS": "<AS>",
  "SwitchIP": "<SwitchIP>"
}
```

若 `Enabled` 为 `false`，需先完成 [网络模式（专线版）](../网络模式（专线版）/tccli%20操作.md) 中的 `EnableExternalNodeSupport` 操作。

```bash
# 3. 确认专线网络可达
# 从 IDC 主机测试到集群内网端点的连通性（端点地址从 DescribeExternalNodeSupportConfig 返回中获取）
curl -k https://cls-example.ccs.tencent-cloud.com
# expected: HTTP 响应（可能返回 401/403，说明网络可达）
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli | 幂等 |
|-----------|-------|------|
| 查看注册节点支持配置 | `tccli tke DescribeExternalNodeSupportConfig` | 是 |
| 创建注册节点池（名称） | `CreateExternalNodePool --Name` | 否(同名报错) |
| 选择运行时（Docker/containerd） | `CreateExternalNodePool --ContainerRuntime` + `--RuntimeVersion` | — |
| 添加标签（Label） | `CreateExternalNodePool --Labels` | — |
| 添加污点（Taint） | `CreateExternalNodePool --Taints` | — |
| 容器目录 | `InstanceAdvancedSettings.DockerGraphPath` | — |
| 自定义数据（UserScript） | `InstanceAdvancedSettings.UserScript` | — |
| 封锁（初始不可调度） | `InstanceAdvancedSettings.Unschedulable` | — |
| 删除保护 | `CreateExternalNodePool --DeletionProtection` | — |
| 节点类型 | `CreateExternalNodePool --NodeType` | — |
| GPU 驱动配置 | `InstanceAdvancedSettings.GPUArgs` | — |
| 数据盘挂载 | `InstanceAdvancedSettings.DataDisks` | — |
| 获取注册脚本 | `tccli tke DescribeExternalNodeScript` | 是 |
| 查询节点池列表 | `tccli tke DescribeExternalNodePools` | 是 |
| 删除注册节点池 | `tccli tke DeleteExternalNodePool` | 是(不存在不报错) |

## 操作步骤

### 1. 查询集群注册节点配置（确认已开启）

```bash
tccli tke DescribeExternalNodeSupportConfig --ClusterId <ClusterId> --region <Region> --output json
```

预期输出：

```json
{
    "Response": {
        "ClusterExternalConfig": {
            "Enabled": true,
            "NetworkType": "HostNetwork",
            "SubnetId": "subnet-example",
            "ClusterCIDR": ""
        },
        "RequestId": "00000000-0000-0000-0000-000000000000"
    }
}
```

### 2. 创建注册节点池

参数较多（>=4 必填），使用 `--cli-input-json file://` 格式。

准备 JSON 入参文件 `CreateExternalNodePool.json`：

```json
{
    "ClusterId": "<ClusterId>",
    "Name": "np-example",
    "ContainerRuntime": "containerd",
    "RuntimeVersion": "1.6.9",
    "Labels": [
        {
            "Name": "env",
            "Value": "idc"
        }
    ],
    "Taints": [
        {
            "Key": "dedicated",
            "Value": "idc",
            "Effect": "NoSchedule"
        }
    ],
    "InstanceAdvancedSettings": {
        "DockerGraphPath": "/var/lib/containerd",
        "Unschedulable": 0,
        "UserScript": "#!/bin/bash\necho 'node registered'"
    },
    "DeletionProtection": false,
    "NodeType": "CPU"
}
```

| 字段 | 必填 | 说明 |
|------|------|------|
| `ClusterId` | 是 | 目标集群 ID |
| `Name` | 是 | 节点池名称，集群内唯一 |
| `ContainerRuntime` | 是 | 容器运行时，可选 `docker` 或 `containerd` |
| `RuntimeVersion` | 是 | 运行时版本，需与集群运行时版本兼容 |
| `Labels` | 否 | 节点标签，调度依据 |
| `Taints` | 否 | 节点污点，`Effect` 可选 `NoSchedule`、`PreferNoSchedule`、`NoExecute` |
| `InstanceAdvancedSettings.DockerGraphPath` | 否 | 容器运行时数据目录，默认 `/var/lib/docker` |
| `InstanceAdvancedSettings.Unschedulable` | 否 | 初始是否封锁，`0` 可调度，`1` 不可调度 |
| `InstanceAdvancedSettings.UserScript` | 否 | 节点初始化后执行的脚本，存放于 `/usr/local/qcloud/tke/userscript` |
| `DeletionProtection` | 否 | 是否开启删除保护 |
| `NodeType` | 否 | 节点类型，可选 `CPU`、`GPU` |

提交创建：

```bash
tccli tke CreateExternalNodePool --cli-input-json file://CreateExternalNodePool.json --region <Region> --output json
```

预期输出：

```json
{
    "Response": {
        "NodePoolId": "np-example001",
        "RequestId": "00000000-0000-0000-0000-000000000000"
    }
}
```

记录返回的 `NodePoolId`，后续获取脚本时使用。

### 3. 获取注册节点初始化脚本

```bash
tccli tke DescribeExternalNodeScript --ClusterId <ClusterId> --NodePoolId np-example001 --Internal true --region <Region> --output json
```

| 参数 | 必填 | 说明 |
|------|------|------|
| `ClusterId` | 是 | 集群 ID |
| `NodePoolId` | 是 | 上一步返回的节点池 ID |
| `Internal` | 否 | 专线版设为 `true`，使用内网地址注册 |
| `Interface` | 否 | 指定网卡，如 `eth0` |
| `Name` | 否 | 预置节点名称 |

预期输出含 `Link` 和 `Script` 字段：

```json
{
    "Response": {
        "Link": "https://cls-example.ccs.tencent-cloud.com/...",
        "Script": "#!/bin/bash\n# TKE 注册节点初始化脚本\n...",
        "RequestId": "00000000-0000-0000-0000-000000000000"
    }
}
```

### 4. 在目标主机执行脚本纳管

将返回的 `Script` 内容保存并在 IDC 目标主机上执行：

```bash
# 在 IDC 主机上执行（需 root 权限）
bash tke-register-node.sh
# expected: agent 安装完成，节点注册成功
```

脚本主要完成：安装 TKE agent、配置 kubelet、建立与控制面的长连接。执行完成后节点将出现在集群节点列表中。

## 验证

### 控制面（tccli）

```bash
# 查看节点池列表，确认新池已创建
tccli tke DescribeExternalNodePools --ClusterId <ClusterId> --region <Region> --output json
```

预期输出含新建节点池：

```json
{
    "Response": {
        "NodePoolSet": [
            {
                "NodePoolId": "np-example001",
                "Name": "np-example",
                "ClusterId": "<ClusterId>",
                "LifeState": "normal",
                "ContainerRuntime": "containerd",
                "RuntimeVersion": "1.6.9"
            }
        ],
        "TotalCount": 1,
        "RequestId": "00000000-0000-0000-0000-000000000000"
    }
}
```

### 数据面（kubectl）

```bash
# 节点纳管完成后，检查节点状态
kubectl get nodes -l env=idc
# expected: 节点状态为 Ready

kubectl describe node <NodeName>
# expected: 节点 Labels 含 env=idc，Taints 含 dedicated=idc:NoSchedule
```

预期 kubectl 输出：

```
NAME               STATUS   ROLES    AGE   VERSION
10.0.0.100         Ready    <none>   2m    v1.30.0-tke.1
```

## 清理

> **副作用警告**：删除节点池将移除池中所有注册节点。若节点上运行工作负载，Pod 将被终止。建议先 `kubectl drain` 或 `DrainExternalNode` 排空节点。
>
> **计费警告**：注册节点本身不产生额外 TKE 费用，但专线带宽按量计费。删除节点池不释放专线资源。

### 控制面（tccli）

```bash
# 1. 查看节点池列表，确认待删除的 NodePoolId
tccli tke DescribeExternalNodePools --ClusterId <ClusterId> --region <Region> --output json

# 2. 删除注册节点池
tccli tke DeleteExternalNodePool --ClusterId <ClusterId> --NodePoolIds '["np-example001"]' --region <Region> --output json
```

预期输出：

```json
{
    "Response": {
        "RequestId": "00000000-0000-0000-0000-000000000000"
    }
}
```

```bash
# 3. 若需强制删除（跳过排空），加 --Force
tccli tke DeleteExternalNodePool --ClusterId <ClusterId> --NodePoolIds '["np-example001"]' --Force true --region <Region> --output json
```

### 数据面（kubectl）

```bash
# 删除池前，确认节点已排空
kubectl drain <NodeName> --ignore-daemonsets --delete-emptydir-data
kubectl delete node <NodeName>
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `enable external node support first` | `DescribeExternalNodeSupportConfig` 返回 `Enabled: false` | 集群未开启注册节点能力 | 执行 `EnableExternalNodeSupport`（参见[网络模式（专线版）](../网络模式（专线版）/tccli%20操作.md)） |
| `NodePool name already exists` | 同名节点池已存在 | 节点池名称集群内唯一 | 更换 `Name` 字段值，或先删除旧池 |
| `invalid ContainerRuntime` | 运行时版本与集群不兼容 | `ContainerRuntime` 或 `RuntimeVersion` 取值无效 | 参照集群运行时设置一致值，如 containerd 1.6.9 |
| `NetworkType mismatch` | 查询 `DescribeExternalNodeSupportConfig` 对比 | 节点池类型（专线/公网）与集群配置不一致 | 确认 `Internal: true` 与集群 NetworkType 匹配 |
| `Script link expired` | 重新调用 `DescribeExternalNodeScript` 仍失败 | 脚本有时效性 | 重新调用 `DescribeExternalNodeScript` 获取新脚本 |

### 创建成功但状态异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 节点长期 `NotReady` | `kubectl describe node <NodeName>` 查看 Conditions | Agent 未成功安装或与控制面断连 | 检查 IDC 主机到专线 CLB 的 443/9000 端口连通性；重新执行脚本 |
| 节点未出现在列表中 | `DescribeExternalNodePools` 有池但 `kubectl get nodes` 无节点 | 脚本未执行或执行环境网络不通 | 在目标主机重新执行 `DescribeExternalNodeScript` 返回的脚本 |
| 节点加入后频繁断连 | 检查 `kubectl get node <NodeName> -o yaml` 中的 `lastHeartbeatTime` | 专线网络不稳定或防火墙间歇性阻断 | 检查 IDC 防火墙对 UDP 8472、TCP 10250 等端口的放通策略 |
| Pod 无法调度到注册节点 | `kubectl describe pod` 查看 Events | 节点有 Taint 而 Pod 无对应 Toleration | 为 Pod 添加匹配 `Toleration`，或移除节点 `Taint` |

## 下一步

- [注册节点概述](../注册节点概述/tccli%20操作.md) — 了解注册节点能力边界与限制
- [编辑注册节点池](../编辑注册节点池/tccli%20操作.md) — 修改节点池配置
- [移除注册节点](../移除注册节点/tccli%20操作.md) — 删除节点或节点池
- [流量接入](../流量接入/tccli%20操作.md) — 为注册节点配置 Service/Ingress

## 控制台替代

容器服务控制台 → **集群** → 目标集群 ID → **节点管理** → **节点池** → **新建节点池**，节点类型选择 **注册节点（专线版）**，填写节点池名称、容器运行时、标签/污点等配置后创建。创建完成后按控制台提示复制初始化脚本，在 IDC 主机上执行以完成纳管。详见[官方文档](https://cloud.tencent.com/document/product/457/57917)。
