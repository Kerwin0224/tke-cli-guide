# 镜像和内核参数说明

> 对照官方：[镜像和内核参数说明](https://cloud.tencent.com/document/product/457/124612) · page_id `124612`

## 概述

原生节点使用 TencentOS Server（tlinux）系列镜像，支持通过 `Management` 参数配置 Kubelet 参数、内核参数、hosts、nameservers 等。创建节点池时通过 `ImageId` 指定镜像，通过 `DescribeOSImages` 查询可用镜像列表。

方案选择：
- **TencentOS Server 4.0**：推荐，原生节点默认镜像，内核与运行时兼容性最佳
- **TencentOS Server 4.0 (tkernel5)**：含 tkernel5 内核，适合需要新内核特性的场景
- **TencentOS Server 3.1**：兼容旧版，部分老集群迁移使用

不支持自定义镜像或非 TencentOS 系列镜像。

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

# 3. 检查 CAM 权限 -- 需要以下 Action 名
#    tke:DescribeClusters, tke:DescribeClusterNodePools, tke:DescribeClusterNodePoolDetail
#    tke:DescribeOSImages, tke:DescribeSupportedRuntime
# 验证：执行 DescribeOSImages 确认权限
tccli tke DescribeOSImages --region <Region>
# expected: exit 0，返回 OS 镜像列表（可为空）
```

```json
{
  "OSImageSeriesSet": "<OSImageSeriesSet>",
  "SeriesName": "<SeriesName>",
  "Alias": "<Alias>",
  "OsName": "<OsName>",
  "OsCustomizeType": "<OsCustomizeType>",
  "Status": "<Status>"
}
```

### 资源检查

```bash
# 4. 确认目标集群存在
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterStatus 为 "Running"

# 5. 确认已有原生节点池（如需查看现有配置）
tccli tke DescribeClusterNodePools --region <Region> \
    --ClusterId CLUSTER_ID
# expected: exit 0，返回节点池列表，原生类型含 NodePoolType="Native"
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
| 查看可用 OS 镜像 | `DescribeOSImages` | 是 |
| 查看节点池 OS 和运行时配置 | `DescribeClusterNodePoolDetail` | 是 |
| 查看支持的 containerd 版本 | `DescribeSupportedRuntime` | 是 |

## 操作步骤

### 支持的 OS 镜像

原生节点仅支持 TencentOS Server 系列镜像：

| 镜像系列 | 别名示例 | 架构 | 说明 |
|---------|---------|------|------|
| TencentOS Server 4.0 | `tlinux4_x86_64_public` | x86_64 | 推荐，默认镜像 |
| TencentOS Server 4.0 (tkernel5) | `tlinux4.0(tkernel5)x86_64` | x86_64 | 含 tkernel5 内核 |
| TencentOS Server 3.1 | `tlinux3.1_x86_64` | x86_64 | 兼容老版本 |

### 查询可用 OS 镜像

```bash
tccli tke DescribeOSImages --region <Region>
# expected: exit 0，返回所有可用 OS 镜像系列
```

**预期输出**：

```json
{
    "OSImageSeriesSet": [
        {
            "ImageId": "img-example-tlinux4",
            "OsName": "TencentOS Server 4.0",
            "Alias": "tlinux4_x86_64_public",
            "Arch": "x86_64",
            "OsCustomizeType": "GENERAL",
            "Status": "online",
            "Type": "GENERAL"
        },
        {
            "ImageId": "img-example-tkernel5",
            "OsName": "TencentOS Server 4.0 (tkernel5)",
            "Alias": "tlinux4.0(tkernel5)x86_64",
            "Arch": "x86_64",
            "OsCustomizeType": "GENERAL",
            "Status": "online",
            "Type": "GENERAL"
        },
        {
            "ImageId": "img-example-tlinux3",
            "OsName": "TencentOS Server 3.1",
            "Alias": "tlinux3.1_x86_64",
            "Arch": "x86_64",
            "OsCustomizeType": "GENERAL",
            "Status": "online",
            "Type": "GENERAL"
        }
    ]
}
```

### 查看节点池当前镜像与内核配置

```bash
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId CLUSTER_ID \
    --NodePoolId NODE_POOL_ID
# expected: exit 0，NodePool 中返回 NodePoolOs、RuntimeConfig、Management 字段
```

**预期输出**：

```json
{
    "NodePool": {
        "NodePoolId": "np-example-native",
        "Name": "native-pool",
        "ClusterId": "cls-example",
        "LifeState": "normal",
        "Type": "Native",
        "NodePoolOs": "tlinux4_x86_64_public",
        "OsCustomizeType": "GENERAL",
        "RuntimeConfig": {
            "ContainerRuntime": "containerd",
            "RuntimeVersion": "1.6.28"
        },
        "Management": {
            "Nameservers": ["183.60.83.19", "183.60.82.98"],
            "Hosts": [],
            "KernelArgs": [
                {
                    "Key": "vm.swappiness",
                    "Value": "0"
                }
            ],
            "KubeletArgs": [
                {
                    "Key": "max-pods",
                    "Value": "110"
                }
            ]
        }
    }
}
```

### 查询支持的 containerd 运行时版本

```bash
tccli tke DescribeSupportedRuntime --region <Region> \
    --K8sVersion K8S_VERSION
# expected: exit 0，RuntimeSet 中返回 containerd 可用版本
```

**预期输出**：

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

### Management 参数说明

创建原生节点池时，通过 `Management` 参数配置节点的运维参数：

| 子字段 | 类型 | 说明 | 示例值 |
|-------|------|------|--------|
| `Nameservers` | String[] | DNS 服务器列表 | `["183.60.83.19", "183.60.82.98"]` |
| `Hosts` | Object[] | /etc/hosts 自定义条目 | `[{"Key": "10.0.0.1", "Value": "my-registry.local"}]` |
| `KernelArgs` | Object[] | 内核启动参数 | `[{"Key": "vm.swappiness", "Value": "0"}]` |
| `KubeletArgs` | Object[] | Kubelet 启动参数 | `[{"Key": "max-pods", "Value": "110"}]` |

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `CLUSTER_ID` | 集群 ID | 格式 `cls-xxxxxxxx` | `tccli tke DescribeClusters --region <Region>` |
| `NODE_POOL_ID` | 原生节点池 ID | 格式 `np-xxxxxxxx`，Type 须为 Native | `tccli tke DescribeClusterNodePools --region <Region> --ClusterId CLUSTER_ID` |
| `REGION` | 地域 | 如 `ap-guangzhou` | `tccli tke DescribeRegions` |
| `K8S_VERSION` | K8s 主版本号 | 如 `1.30` | `tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'` |

## 验证

### 控制面（tccli）

```bash
# 验证 OS 镜像列表可查
tccli tke DescribeOSImages --region <Region>
# expected: exit 0，OSImageSeriesSet 非空

# 验证节点池 OS 配置
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId CLUSTER_ID \
    --NodePoolId NODE_POOL_ID
# expected: exit 0，NodePoolOs 非空，Management 含配置项

# 验证运行时版本
tccli tke DescribeSupportedRuntime --region <Region> \
    --K8sVersion K8S_VERSION
# expected: exit 0，RuntimeSet 中 containerd Status 为 "online"
```

```json
{
  "OSImageSeriesSet": "<OSImageSeriesSet>",
  "SeriesName": "<SeriesName>",
  "Alias": "<Alias>",
  "OsName": "<OsName>",
  "OsCustomizeType": "<OsCustomizeType>",
  "Status": "<Status>"
}
```

## 清理

本页为概念说明页，无资源需清理。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DescribeOSImages` 返回的镜像不包含 tlinux4 系列 | 检查 region 参数和目标地域 | 该地域可能不提供 TencentOS Server 4.0 镜像 | 尝试其他地域，或使用 `DescribeOSImages --region <Region>` 查看该地域提供的所有镜像，选择 Status="online" 的 TencentOS 镜像 |
| `DescribeSupportedRuntime` 返回 `InvalidParameter.K8sVersion` | 检查传入的 `--K8sVersion` 参数值 | 传入了带补丁号的完整版本（如 `1.30.0`） | 改为传入主版本号，如 `--K8sVersion 1.30` |
| `DescribeClusterNodePoolDetail` 返回的 `Management` 为空 | 检查节点池类型 | 该节点池可能为普通节点池（Type="Regular"），不支持 Management 参数 | 确认节点池 Type 为 "Native"，普通节点池无 Management 配置 |

### 查询成功但结果不符合预期

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `NodePoolOs` 字段与创建时指定的 `ImageId` 不一致 | 执行 `DescribeClusterNodePoolDetail` 对比 `NodePoolOs` 和 `OsCustomizeType` | 节点池创建后 OS 不可变更，该字段反映实际运行的 OS | 如需更换 OS，需创建新节点池并指定新 `ImageId` |
| `Management.KernelArgs` 配置未生效 | 登录节点执行 `cat /proc/cmdline` 对比内核参数 | 内核参数可能与该 OS 镜像版本不兼容而被忽略 | 检查内核参数是否在该 OS 版本的支持范围内；参考官方文档中支持的内核参数列表 |

## 下一步

- [新建原生节点](https://cloud.tencent.com/document/product/457/78198) -- 创建原生节点池时指定镜像
- [原生节点概述](https://cloud.tencent.com/document/product/457/78197) -- 原生节点概念
- [修改原生节点](https://cloud.tencent.com/document/product/457/103599) -- 修改 Management 配置

## 控制台替代

[容器服务控制台 - 节点池运维参数](https://console.cloud.tencent.com/tke2/cluster)
