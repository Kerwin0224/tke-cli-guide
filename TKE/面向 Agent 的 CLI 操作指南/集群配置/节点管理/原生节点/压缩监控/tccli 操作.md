# 压缩监控

> 对照官方：[压缩监控](https://cloud.tencent.com/document/product/457/102625) · page_id `102625`

## 概述

原生节点支持**内存压缩**功能，通过 `Management` 参数配置压缩阈值和策略，在节点池创建或修改时启用。开启后可通过云监控或 Prometheus 查看压缩率、压缩收益等指标。本页说明如何通过 API 查询压缩功能状态和节点池配置。

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
# 验证：执行 DescribeClusterNodePools 确认权限
tccli tke DescribeClusterNodePools --region <Region> \
    --ClusterId CLUSTER_ID
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
# 4. 确认目标集群和原生节点池存在
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterStatus 为 "Running"

tccli tke DescribeClusterNodePools --region <Region> \
    --ClusterId CLUSTER_ID
# expected: exit 0，至少有一个 Type="Native" 的节点池
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
| 查看节点池列表 | `DescribeClusterNodePools` | 是 |
| 查看节点池详情（含 Management 配置） | `DescribeClusterNodePoolDetail` | 是 |
| 启用/配置内存压缩 | `CreateClusterNodePool` 或 `ModifyClusterNodePool` 中配置 Management | 否 |

## 操作步骤

### 内存压缩功能说明

内存压缩在原生节点上通过回收冷内存页，提升节点整体的内存利用率。功能通过节点池的 `Management` 参数控制，配置方式包括：

- **创建时启用**：在 `CreateClusterNodePool` 的 `Management` 参数中配置
- **运行时修改**：通过 `ModifyClusterNodePool` 更新已存在节点池的 `Management` 参数
- **声明式管理**：通过 `Annotations` 写入声明式配置

启用后，压缩指标可通过以下方式查看：
- 云监控控制台 → 原生节点内存压缩相关指标
- 集群 Prometheus → 按官方文档配置采集规则

### 查询压缩监控相关配置

查看节点池的 Management 参数以确认压缩功能是否已启用：

```bash
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId CLUSTER_ID \
    --NodePoolId NODE_POOL_ID
# expected: exit 0，Management 字段中包含压缩相关参数
```

预期输出（压缩已启用的节点池）：

```json
{
    "NodePool": {
        "NodePoolId": "np-example-native",
        "Name": "native-pool-compression",
        "ClusterId": "cls-example",
        "LifeState": "normal",
        "Type": "Native",
        "NodeCountSummary": {
            "ManuallyAdded": {
                "Joining": 0,
                "Initializing": 0,
                "Normal": 5,
                "Abnormal": 0
            },
            "AutoscalingAdded": {
                "Joining": 0,
                "Initializing": 0,
                "Normal": 0,
                "Abnormal": 0
            }
        },
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
            },
            {
                "Name": "node.tke.cloud.tencent.com/compression-threshold",
                "Value": "80"
            }
        ]
    }
}
```

### Management 参数中与压缩相关的配置

| 参数位置 | 字段 | 类型 | 说明 |
|---------|------|------|------|
| `Management` | `KernelArgs` | Object[] | 可配置内存压缩相关内核参数，如 `vm.watermark_scale_factor` 等 |
| `Annotations` | `node.tke.cloud.tencent.com/compression` | String | 启用/禁用压缩：`enable` / `disable` |
| `Annotations` | `node.tke.cloud.tencent.com/compression-threshold` | String | 压缩触发阈值百分比，如 `80` |

### 查询节点池列表（确认哪些节点池启用了压缩）

```bash
tccli tke DescribeClusterNodePools --region <Region> \
    --ClusterId CLUSTER_ID
# expected: exit 0，返回节点池列表
```

预期输出：

```json
{
    "TotalCount": 2,
    "NodePoolSet": [
        {
            "NodePoolId": "np-example-compression",
            "Name": "native-pool-compression",
            "ClusterId": "cls-example",
            "LifeState": "normal",
            "NodePoolType": "Native",
            "DesiredNodesNum": 5
        },
        {
            "NodePoolId": "np-example-no-compression",
            "Name": "native-pool-no-compression",
            "ClusterId": "cls-example",
            "LifeState": "normal",
            "NodePoolType": "Native",
            "DesiredNodesNum": 3
        }
    ]
}
```

进一步对每个原生节点池执行 `DescribeClusterNodePoolDetail` 查看其 `Annotations` 和 `Management` 配置即可确认压缩功能是否已启用。

### 控制台概念与 API 参数映射

| 控制台配置项 | API 字段 | 取值 |
|------------|---------|------|
| 内存压缩开关 | `Annotations["node.tke.cloud.tencent.com/compression"]` | `enable` / `disable` |
| 压缩阈值（%） | `Annotations["node.tke.cloud.tencent.com/compression-threshold"]` | 数值字符串，如 `80` |
| 内核参数 | `Management.KernelArgs` | `[{"Key": "kernel.param", "Value": "value"}]` |
| kubelet 参数 | `Management.KubeletArgs` | `[{"Key": "param-name", "Value": "param-value"}]` |

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `CLUSTER_ID` | 集群 ID | 格式 `cls-xxxxxxxx` | `tccli tke DescribeClusters --region <Region>` |
| `NODE_POOL_ID` | 原生节点池 ID | 格式 `np-xxxxxxxx`，Type 须为 Native | `tccli tke DescribeClusterNodePools --region <Region> --ClusterId CLUSTER_ID` |
| `REGION` | 地域 | 如 `ap-guangzhou` | `tccli tke DescribeRegions` |

## 验证

### 控制面（tccli）

```bash
# 验证节点池详情含压缩配置
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId CLUSTER_ID \
    --NodePoolId NODE_POOL_ID
# expected: exit 0；若压缩已启用，Annotations 中应包含 compression 相关键值

# 验证节点池列表
tccli tke DescribeClusterNodePools --region <Region> \
    --ClusterId CLUSTER_ID
# expected: exit 0，NodePoolSet 中包含 Type="Native" 的节点池
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

### 数据面（节点内验证，需 kubectl 可达）

```bash
# 登录原生节点后，检查内存压缩功能是否生效
cat /proc/sys/vm/watermark_scale_factor
# expected: 返回配置的内核参数值

# 检查压缩相关组件是否运行
kubectl get pods -n kube-system | grep compression
# expected: 如有压缩组件 Pod，显示 Running 状态
```

```text
NAME  STATUS  AGE
...
```

## 清理

本页为概念说明页，无资源需清理。如需关闭压缩功能，通过 `ModifyClusterNodePool` 更新 `Annotations` 或 `Management` 参数。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DescribeClusterNodePoolDetail` 返回 `ResourceNotFound.NodePoolNotFound` | 检查 NODE_POOL_ID 是否正确 | 节点池 ID 不存在或不属于指定集群 | 通过 `tccli tke DescribeClusterNodePools --region <Region> --ClusterId CLUSTER_ID` 获取正确的 NODE_POOL_ID |
| `DescribeClusterNodePools` 无 Type="Native" 的节点池 | 查看返回的 NodePoolType 字段 | 集群中仅有普通节点池（Type="Regular"），不支持内存压缩 | 创建原生节点池（Type="Native"）才能启用压缩功能 |

### 查询成功但压缩未生效

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Annotations 中 compression 为 `enable` 但节点未压缩 | 登录节点执行 `cat /proc/sys/vm/watermark_scale_factor` 和检查压缩组件日志 | 压缩组件可能未正常启动或配置未下发到节点 | 检查 Annotations 是否正确应用到节点池；查看节点事件和组件日志确认下发状态 |
| Annotations 中无 compression 相关键 | `DescribeClusterNodePoolDetail` 查看 Annotations 列表 | 节点池创建时未配置压缩功能，或配置被修改移除 | 通过 `ModifyClusterNodePool` 或声明式 API 添加压缩相关 Annotations |
| 压缩启用但监控平台无数据 | 检查云监控或 Prometheus 的指标采集配置 | 监控采集组件未安装或未配置原生节点压缩指标 | 确保集群已安装监控组件（如 Prometheus）；按官方文档配置原生节点内存压缩指标采集规则 |

## 下一步

- [使用说明（内存压缩）](https://cloud.tencent.com/document/product/457/102456) — 内存压缩使用说明
- [原生节点功能支持说明](https://cloud.tencent.com/document/product/457/78222) — 功能矩阵
- [新建原生节点](https://cloud.tencent.com/document/product/457/78198) — 创建节点池时启用压缩

## 控制台替代

[容器服务控制台 - 节点池运维参数](https://console.cloud.tencent.com/tke2/cluster)
