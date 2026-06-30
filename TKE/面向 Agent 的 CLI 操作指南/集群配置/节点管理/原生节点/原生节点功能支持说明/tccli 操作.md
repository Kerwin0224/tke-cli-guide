# 原生节点功能支持说明

> 对照官方：[原生节点功能支持说明](https://cloud.tencent.com/document/product/457/78222) · page_id `78222`

## 概述

原生节点相比普通节点新增声明式基础设施管理、Pod 原地升降配、故障自愈、内存压缩、qGPU 等能力。本页以功能矩阵对照 API 参数体现，并提供 `tccli` 查询命令验证集群是否具备相应的能力前置条件。

方案选择：需要精细化管理（声明式、自愈、原地升降配）的场景选原生节点；仅需基础容器运行的场景选普通节点。

## 前置条件

- [环境准备](../../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置，region 为目标地域

# 3. 检查 CAM 权限 — 需要以下 Action 名
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
    --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterStatus 为 "Running"，ClusterType 为 "MANAGED_CLUSTER" 或 "INDEPENDENT_CLUSTER"
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

原生节点通过 `CreateClusterNodePool` 的参数体现各项能力——`Type="Native"` 创建原生节点池，`Management`、`Annotations` 等参数控制具体行为。

| 能力 | 支持情况 | 对应 API 参数 / 操作 |
|------|----------|---------------------|
| 声明式管理 | 支持 | `Annotations` 字段写入声明式配置 |
| Pod 原地升降配 | 支持 | MachineSet CRD（数据面操作） |
| 故障自愈 | 支持 | `CreateNodePoolHealthCheckPolicy` 配置自愈规则 |
| 内存压缩 | 支持 | `Management` 参数中的压缩配置 |
| 节点自动伸缩 | 支持 | `EnableAutoscale` + 扩缩容策略 |
| 初始化脚本 | 支持（前/后两阶段） | `PreStartUserScript` / `UserScript` |
| 公网带宽 / EIP | 支持绑定 EIP | `InternetAccessible` 参数 |
| GPU / qGPU | 支持 | `GPUArgs` 参数 |
| SSH 登录 | 支持 | 节点创建后通过 OrcaTerm 或 SSH 登录 |
| 仅 containerd 运行时 | 是 | `RuntimeConfig` → `ContainerRuntime` = `containerd` |
| K8s 版本 | ≥ 1.16（部分能力有更高要求） | 集群 `ClusterVersion` 字段 |

### 查询集群版本（确认兼容性）

```bash
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]'
# expected: exit 0，返回集群信息含 ClusterVersion
```

预期输出：

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

### 查询节点池类型（区分原生/普通）

```bash
tccli tke DescribeClusterNodePools --region <Region> \
    --ClusterId CLUSTER_ID
# expected: exit 0，返回节点池列表，类型为 "Native" 的为原生节点池
```

预期输出：

```json
{
    "TotalCount": 2,
    "NodePoolSet": [
        {
            "NodePoolId": "np-example-native",
            "Name": "native-pool",
            "ClusterId": "cls-example",
            "LifeState": "normal",
            "NodePoolType": "Native"
        },
        {
            "NodePoolId": "np-example-regular",
            "Name": "regular-pool",
            "ClusterId": "cls-example",
            "LifeState": "normal",
            "NodePoolType": "Regular"
        }
    ]
}
```

### 查询原生节点池详情（查看已启用的能力）

```bash
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId CLUSTER_ID \
    --NodePoolId NODE_POOL_ID
# expected: exit 0，返回完整节点池配置，包含 Type、Management、Annotations
```

预期输出：

```json
{
    "NodePool": {
        "NodePoolId": "np-example-native",
        "Name": "native-pool",
        "ClusterId": "cls-example",
        "LifeState": "normal",
        "Type": "Native",
        "Management": {
            "Nameservers": ["183.60.83.19", "183.60.82.98"],
            "Hosts": [],
            "KernelArgs": [],
            "KubeletArgs": []
        },
        "Annotations": [
            {
                "Name": "node.tke.cloud.tencent.com/compression",
                "Value": "enable"
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

预期输出：

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
    --K8sVersion K8S_VERSION
# expected: exit 0，返回 containerd 可用版本列表
```

预期输出：

```json
{
    "RuntimeSet": [
        {
            "RuntimeType": "containerd",
            "RuntimeVersion": "1.6.28",
            "Status": "online"
        },
        {
            "RuntimeType": "containerd",
            "RuntimeVersion": "1.7.16",
            "Status": "online"
        }
    ]
}
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `CLUSTER_ID` | 集群 ID | 格式 `cls-xxxxxxxx`，集群必须处于 Running 状态 | `tccli tke DescribeClusters --region <Region>` |
| `NODE_POOL_ID` | 节点池 ID | 格式 `np-xxxxxxxx`，必须是原生类型（Type=Native） | `tccli tke DescribeClusterNodePools --region <Region> --ClusterId CLUSTER_ID` |
| `REGION` | 地域 | 如 `ap-guangzhou` | `tccli tke DescribeRegions` |
| `K8S_VERSION` | K8s 主版本 | 如 `1.30`，须与集群 ClusterVersion 匹配 | `tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'` |

## 验证

### 控制面（tccli）

```bash
# 验证集群信息和版本
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]'
# expected: exit 0，ClusterStatus 为 "Running"，ClusterVersion 明确

# 验证节点池类型
tccli tke DescribeClusterNodePools --region <Region> \
    --ClusterId CLUSTER_ID
# expected: exit 0，NodePoolSet 中包含 Type 字段

# 验证 OS 镜像可用
tccli tke DescribeOSImages --region <Region>
# expected: exit 0，OSImageSeriesSet 非空

# 验证运行时版本
tccli tke DescribeSupportedRuntime --region <Region> \
    --K8sVersion K8S_VERSION
# expected: exit 0，RuntimeSet 中包含 containerd
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

## 清理

本页为概念说明页，无资源需清理。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DescribeClusterNodePools` 返回空列表 | 执行 `tccli tke DescribeClusters --region <Region>` 确认集群状态 | 集群中尚未创建节点池 | 前往 [新建原生节点](../新建原生节点/tccli%20操作.md) 创建 |
| `DescribeSupportedRuntime` 返回 `InvalidParameter.K8sVersion` | 检查传入的版本号格式 | 传入了完整版本号（如 `1.30.0`）而非主版本号 | 改为传入主版本号，如 `--K8sVersion 1.30` |
| 某能力在表中标注"支持"但查询不到对应参数 | `DescribeClusterNodePoolDetail` 查看 `Management` 和 `Annotations` 字段 | K8s 版本未达到对应能力的最低要求 | 升级集群 K8s 版本，或确认该能力是否需要额外开通白名单 |

### 查询成功但结果不符合预期

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 节点池 Type 为 "Regular" 而非 "Native" | `DescribeClusterNodePools` 查看 `NodePoolType` | 该节点池为普通节点池，非原生节点池 | 普通节点池不具备原生节点能力；如需原生能力，创建新的 Type="Native" 节点池 |
| `DescribeOSImages` 返回的镜像列表不包含 tlinux4 | 在控制台确认目标地域的镜像市场 | 部分地域或机型可能不提供 tlinux4 镜像 | 换用其他可用地域，或选用 `DescribeOSImages` 返回的其他兼容镜像 |

## 下一步

- [新建原生节点](https://cloud.tencent.com/document/product/457/78198) — 创建原生节点池
- [原生节点概述](https://cloud.tencent.com/document/product/457/78197) — 原生节点概念与对比
- [故障自愈规则](https://cloud.tencent.com/document/product/457/78209) — 配置故障自愈策略
- [Pod 原地升降配](https://cloud.tencent.com/document/product/457/79785) — Pod 资源原地变更

## 控制台替代

[容器服务控制台 - 原生节点](https://console.cloud.tencent.com/tke2/native-node)
