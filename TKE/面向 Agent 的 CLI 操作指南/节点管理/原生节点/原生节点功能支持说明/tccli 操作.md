# 原生节点功能支持说明

> 对照官方：[原生节点功能支持说明](https://cloud.tencent.com/document/product/457/78222) · page_id `78222` · tccli ≥3.1.107.1 · API 2018-05-25

## 概述

原生节点相比普通节点新增声明式基础设施管理、Pod 原地升降配、故障自愈、内存压缩、qGPU 等高级能力。本页以功能矩阵对照 API 参数，并通过 `tccli` 查询命令验证集群是否具备相应能力的前置条件。

方案选择：
- **原生节点**：需要声明式管理、故障自愈、原地升降配、精细调度 -- 选此
- **普通节点**：仅需基础容器运行，用户自行管理节点生命周期 -- 选此
- **超级节点**：无节点运维、按 Pod 计费 -- 选此

## 前置条件

<!-- sync: 此相对链接需在 iWiki/写写 发布时转换为平台兼容格式（docId/nodeId） -->
- [环境准备](../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置

# 3. 检查 CAM 权限 -- 需要以下 Action 名
#    tke:DescribeClusters, tke:DescribeClusterNodePools, tke:DescribeClusterNodePoolDetail
#    tke:DescribeOSImages, tke:DescribeSupportedRuntime
# 验证：执行 DescribeClusters 确认权限
tccli tke DescribeClusters --region <Region>
# expected: exit 0，返回集群列表（可为空）
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
# 4. 确认目标集群存在且状态正常
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["<ClusterId>"]'
# expected: ClusterStatus 为 "Running"
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
| 查看集群信息 | `DescribeClusters` | 是 |
| 查看节点池列表 | `DescribeClusterNodePools` | 是 |
| 查看节点池详情 | `DescribeClusterNodePoolDetail` | 是 |
| 查看可用 OS 镜像 | `DescribeOSImages` | 是 |
| 查看支持的运行时 | `DescribeSupportedRuntime` | 是 |

## 操作步骤

### 功能支持对照

原生节点通过 `CreateClusterNodePool` 的参数体现各项能力 -- `Type="Native"` 创建原生节点池，`Management`、`Annotations` 等参数控制具体行为。

| 能力 | 支持情况 | 对应 API 参数 / 操作 |
|------|----------|---------------------|
| 声明式管理 | 支持 | `Annotations` 字段写入声明式配置 |
| Pod 原地升降配 | 支持 | MachineSet CRD（数据面操作），需节点池启用 in-place update 注解 |
| 故障自愈 | 支持 | `CreateNodePoolHealthCheckPolicy` 配置自愈规则 |
| 内存压缩 | 支持 | `Management` 参数中的压缩配置 |
| 节点自动伸缩 | 支持 | `EnableAutoscale` + 扩缩容策略 |
| 初始化脚本 | 支持（前/后两阶段） | `PreStartUserScript` / `UserScript` 参数 |
| 公网带宽 / EIP | 支持绑定 EIP | `InternetAccessible` 参数 |
| GPU / qGPU | 支持 | `GPUArgs` 参数 |
| SSH 登录 | 支持 | 节点创建后通过 OrcaTerm 或 SSH 登录 |
| 仅 containerd 运行时 | 是 | `RuntimeConfig` 中 `ContainerRuntime` = `containerd` |
| K8s 最低版本 | >= 1.16 | 集群 `ClusterVersion` 字段（部分能力有更高版本要求） |

### 查询集群版本（确认兼容性）

```bash
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["<ClusterId>"]'
# expected: exit 0，返回集群信息含 ClusterVersion
```

**预期输出**：

```json
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "cls-example",
            "ClusterName": "example-cluster",
            "ClusterVersion": "1.30.0",
            "ClusterStatus": "Running",
            "ClusterType": "MANAGED_CLUSTER"
        }
    ]
}
```

### 查询节点池列表（查看节点池配置）

```bash
tccli tke DescribeClusterNodePools --region <Region> \
    --ClusterId <ClusterId>
# expected: exit 0，返回节点池列表
```

**预期输出**：

```json
{
    "TotalCount": 1,
    "NodePoolSet": [
        {
            "NodePoolId": "np-example",
            "Name": "example-pool",
            "ClusterInstanceId": "cls-example",
            "LifeState": "normal",
            "AutoscalingGroupStatus": "disabled",
            "MaxNodesNum": 5,
            "MinNodesNum": 0,
            "DesiredNodesNum": 2,
            "RuntimeConfig": {
                "RuntimeType": "containerd",
                "RuntimeVersion": "1.6.9"
            },
            "NodePoolOs": "ubuntu22.04x86_64",
            "OsCustomizeType": "GENERAL",
            "DeletionProtection": false
        }
    ]
}
```

### 查询原生节点池详情（查看已启用的能力）

```bash
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId <ClusterId> \
    --NodePoolId <NodePoolId>
# expected: exit 0，返回完整节点池配置，包含 RuntimeConfig、Annotations
```

**预期输出**：

```json
{
    "NodePool": {
        "NodePoolId": "np-example",
        "Name": "example-pool",
        "ClusterInstanceId": "cls-example",
        "LifeState": "normal",
        "RuntimeConfig": {
            "RuntimeType": "containerd",
            "RuntimeVersion": "1.6.9"
        },
        "NodePoolOs": "ubuntu22.04x86_64",
        "OsCustomizeType": "GENERAL",
        "Annotations": [],
        "Labels": [],
        "Taints": [],
        "DeletionProtection": false,
        "Tags": [
            {
                "Key": "billing",
                "Value": "kerwinwjyan"
            }
        ]
    }
}
```

### 查询可用 OS 镜像（确认原生节点兼容镜像）

```bash
tccli tke DescribeOSImages --region <Region>
# expected: exit 0，返回 OS 镜像系列列表
```

**预期输出**：

```json
{
    "OSImageSeriesSet": [
        {
            "ImageId": "img-example001",
            "OsName": "TencentOS Server 4.0",
            "Arch": "x86_64",
            "Status": "online",
            "Type": "GENERAL"
        },
        {
            "ImageId": "img-example002",
            "OsName": "tlinux4_x86_64_public",
            "Arch": "x86_64",
            "Status": "online",
            "Type": "GENERAL"
        }
    ]
}
```

### 查询支持的 containerd 运行时版本

```bash
tccli tke DescribeSupportedRuntime --region <Region> \
    --K8sVersion 1.32.2
# expected: exit 0，返回 containerd 可用版本列表
```

**预期输出**：

```json
{
    "OptionalRuntimes": [
        {
            "RuntimeType": "containerd",
            "RuntimeVersions": ["1.6.9", "1.7.28"],
            "DefaultVersion": "1.6.9"
        },
        {
            "RuntimeType": "docker",
            "RuntimeVersions": ["20.10"],
            "DefaultVersion": "20.10"
        }
    ]
}
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<ClusterId>` | 集群 ID | 格式 `cls-xxxxxxxx`，集群必须处于 Running 状态 | `tccli tke DescribeClusters --region <Region>` |
| `<NodePoolId>` | 节点池 ID | 格式 `np-xxxxxxxx` | `tccli tke DescribeClusterNodePools --region <Region> --ClusterId <ClusterId>` |
| `<Region>` | 地域 | 如 `ap-guangzhou` | `tccli tke DescribeRegions` |

## 验证

### 控制面（tccli）

```bash
# 验证集群信息和版本
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["<ClusterId>"]'
# expected: exit 0，ClusterStatus 为 "Running"，ClusterVersion 明确
```

**预期输出**：

```json
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "cls-example",
            "ClusterName": "example-cluster",
            "ClusterVersion": "1.32.2",
            "ClusterStatus": "Running",
            "ClusterType": "MANAGED_CLUSTER"
        }
    ]
}
```

```bash
# 验证节点池列表
tccli tke DescribeClusterNodePools --region <Region> \
    --ClusterId <ClusterId>
# expected: exit 0，返回节点池列表
```

**预期输出**：

```json
{
    "TotalCount": 1,
    "NodePoolSet": [
        {
            "NodePoolId": "np-example",
            "Name": "example-pool",
            "ClusterInstanceId": "cls-example",
            "LifeState": "normal",
            "RuntimeConfig": {
                "RuntimeType": "containerd",
                "RuntimeVersion": "1.6.9"
            }
        }
    ]
}
```

```bash
# 验证 OS 镜像可用
tccli tke DescribeOSImages --region <Region>
# expected: exit 0，OSImageSeriesSet 非空
```

**预期输出**：

```json
{
    "OSImageSeriesSet": [
        {
            "ImageId": "img-example",
            "OsName": "TencentOS Server 4.0",
            "Arch": "x86_64",
            "Status": "online",
            "Type": "GENERAL"
        }
    ]
}
```

```bash
# 验证运行时版本
tccli tke DescribeSupportedRuntime --region <Region> \
    --K8sVersion 1.32.2
# expected: exit 0，OptionalRuntimes 中包含 containerd
```

**预期输出**：

```json
{
    "OptionalRuntimes": [
        {
            "RuntimeType": "containerd",
            "RuntimeVersions": ["1.6.9", "1.7.28"],
            "DefaultVersion": "1.6.9"
        }
    ]
}
```

## 清理

本页为概念说明页，无资源需清理。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DescribeClusterNodePools` 返回空列表 | 执行 `tccli tke DescribeClusters --region <Region>` 确认集群状态 | 集群中尚未创建节点池 | 前往 [新建原生节点](../新建原生节点/tccli%20操作.md) 创建 |
| `DescribeSupportedRuntime` 返回 `InvalidParameter.K8sVersion` 或 `InternalError` | 检查传入的版本号格式，确认是完整版本号（x.y.z） | 传入了主版本号（如 `1.30`）或格式不对的版本号 | 使用 `DescribeClusters` 输出中的完整 `ClusterVersion` 值（如 `1.32.2`）, 如 `--K8sVersion 1.32.2` |
| 某能力在表中标注"支持"但查询不到对应参数 | `DescribeClusterNodePoolDetail` 查看 `RuntimeConfig` 和 `Annotations` 字段 | K8s 版本未达到对应能力的最低要求，或该能力字段仅在创建请求中可见 | 升级集群 K8s 版本，或确认该能力是否需要额外开通白名单 |

### 查询成功但结果不符合预期

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 节点池不具备原生节点能力（如无法原地升降配） | `DescribeClusterNodePoolDetail` 查看节点池配置 | 该节点池为普通节点池，非原生节点池 | 普通节点池不具备原生节点能力；如需要，创建新的原生节点池（Type="Native"） |
| `DescribeOSImages` 返回的镜像列表不包含 tlinux4 | 检查 region 参数与目标地域 | 部分地域可能不提供 tlinux4 镜像 | 换用其他可用地域，或选用 `DescribeOSImages` 返回的其他兼容镜像 |

## 下一步

- [新建原生节点](https://cloud.tencent.com/document/product/457/78198) -- 创建原生节点池
- [原生节点概述](https://cloud.tencent.com/document/product/457/78197) -- 原生节点概念与对比
- [故障自愈规则](https://cloud.tencent.com/document/product/457/78209) -- 配置故障自愈策略
- [Pod 原地升降配](https://cloud.tencent.com/document/product/457/79785) -- Pod 资源原地变更

## 控制台替代

[容器服务控制台 - 原生节点](https://console.cloud.tencent.com/tke2/native-node)
