# 超级节点可调度 Pod 说明

> 对照官方：[超级节点可调度 Pod 说明](https://cloud.tencent.com/document/product/457/74015) · page_id `74015`

## 概述

说明超级节点上可调度的 Pod **CPU、内存、GPU 规格范围**以及 Pod 类型限制。不同计费模式（包年包月 vs 按量）对应不同的规格约束。Pod 超出规格上限或类型不支持时将无法调度到超级节点。

| 维度 | 包年包月 | 按量计费 |
|------|---------|---------|
| CPU 范围 | 0.25C ~ 8C | 0.25C ~ 64C |
| CPU:内存比 | < 1:4 | ≤ 1:8 |
| GPU 支持 | 不支持 | 支持 |
| 系统盘 | 默认 20GiB，Annotation 自定义 | 默认 20GiB，Annotation 自定义 |
| 非标准规格 | 向上取整 | 向上取整 |
| 适用场景 | 常驻业务、可预估的资源用量 | 弹性扩缩、GPU 任务 |

**控制面验证**：通过 `DescribeClusterVirtualNodePools` 查看超级节点池计费模式。

## 前置条件

- [环境准备](../../../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查 CAM 权限 — 需要以下 Action 名
#    tke:DescribeClusterVirtualNodePools
# 验证：
tccli tke DescribeClusterVirtualNodePools --region <Region> --ClusterId CLUSTER_ID
# expected: exit 0，返回节点池列表（可为空）
```

```json
{
  "TotalCount": "<TotalCount>",
  "NodePoolSet": "<NodePoolSet>",
  "NodePoolId": "<NodePoolId>",
  "SubnetIds": "<SubnetIds>",
  "Name": "<Name>",
  "LifeState": "<LifeState>"
}
```

### 资源检查

```bash
# 3. 查询超级节点池计费模式（如已创建）
tccli tke DescribeClusterVirtualNodePools --region <Region> \
    --ClusterId CLUSTER_ID
# expected: exit 0，查看计费模式字段
```

```json
{
  "TotalCount": "<TotalCount>",
  "NodePoolSet": "<NodePoolSet>",
  "NodePoolId": "<NodePoolId>",
  "SubnetIds": "<SubnetIds>",
  "Name": "<Name>",
  "LifeState": "<LifeState>"
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看超级节点池计费模式 | `DescribeClusterVirtualNodePools` | 是 |
| 查看超级节点规格 | `DescribeClusterVirtualNodes` | 是 |
| 查看 Pod 资源 requests | `kubectl describe pod` | 是 |

## 关键字段说明

| 字段 | 类型 | 来源 | 取值与约束 | 错误后果 |
|------|------|------|------|------|
| CPU（核） | Float | Pod `resources.requests.cpu` 或 Annotation `cpu` | 包年包月：0.25~8，按量：0.25~64。非标准规格向上取整 | 超出范围 → Pod Pending |
| 内存（GiB） | String | Pod `resources.requests.memory` 或 Annotation `mem` | 包年包月：与 CPU 比 < 1:4，按量：与 CPU 比 ≤ 1:8 | 比例超标 → Pod Pending |
| GPU（卡） | Integer | Annotation `gpu-count` | 0 ~ 4（按量），包年包月不支持 | 包年包月声明 GPU → 被忽略 |
| 系统盘（GiB） | String | Annotation `root-cbs-size` | 默认 20，上限受节点池配置限制 | 超出上限 → Pod 调度失败 |

## 操作步骤

### 步骤 1：查看超级节点池计费模式和配额

```bash
tccli tke DescribeClusterVirtualNodePools --region <Region> \
    --ClusterId CLUSTER_ID
# expected: exit 0，返回节点池信息
```

**预期输出**（含 Quota）：

```json
{
    "TotalCount": 1,
    "NodePoolSet": [
        {
            "NodePoolId": "np-example",
            "Name": "NODE_POOL_NAME",
            "LifeState": "normal",
            "VirtualNodes": [
                {
                    "Quota": {
                        "Cpu": 4,
                        "Memory": 8
                    }
                }
            ]
        }
    ],
    "RequestId": "..."
}
```

| 字段 | 含义 |
|------|------|
| `Quota.Cpu` | 超级节点池默认 CPU 配额（核） |
| `Quota.Memory` | 超级节点池默认内存配额（GiB） |

### 步骤 2：验证 Pod 规格是否在可调度范围内

创建 Pod 前，检查其资源声明是否符合超级节点规格约束。

```bash
# 检查 Pod YAML 中的 resources.requests
grep -A3 "requests:" pod-spec.yaml
# expected: cpu/memory 在超级节点可调度范围内
```

**规格换算示例**：

| 声明值 | 实际分配 | 说明 |
|--------|---------|------|
| `cpu: "0.5"` | 0.5 核 | 按量支持 0.25C 粒度 |
| `cpu: "0.3"` | 0.5 核 | 非标准规格向上取整 |
| `cpu: "0.25", memory: "2Gi"` | 0.25C/2GiB | 按量：1:8，符合约束 |
| `cpu: "1", memory: "5Gi"` | 1C/5GiB | 包年包月：1:5，不符合 <1:4 约束 |

## 验证

### 确认可调度规格

```bash
# 查看节点池 Quota 确认规格上限
tccli tke DescribeClusterVirtualNodePools --region <Region> \
    --ClusterId CLUSTER_ID --NodePoolIds '["NODE_POOL_ID"]'
# expected: VirtualNodes[].Quota 为可调度规格上限
```

```json
{
  "TotalCount": "<TotalCount>",
  "NodePoolSet": "<NodePoolSet>",
  "NodePoolId": "<NodePoolId>",
  "SubnetIds": "<SubnetIds>",
  "Name": "<Name>",
  "LifeState": "<LifeState>"
}
```

```bash
# 查看实际 Pod 分配的资源
kubectl --kubeconfig ~/.kube/config-super describe node NODE_NAME | grep -A5 "Allocated resources"
# expected: 显示超级节点上已分配和可分配的资源
```

| 验证维度 | 命令 | 预期 |
|---------|------|------|
| 节点池 Quota | `DescribeClusterVirtualNodePools --ClusterId CLUSTER_ID` | Quota 字段非空 |
| 节点可分配资源 | `kubectl describe node NODE_NAME \| grep -A5 "Allocatable"` | CPU/内存与 Quota 一致 |
| Pod 资源匹配 | `kubectl describe pod POD_NAME \| grep -A3 "Requests"` | 资源数值在允许范围内 |

## 清理

本页为概念说明页，仅执行只读操作。无资源需清理。

> 如已创建测试 Pod 并需清理：
> ```bash
> kubectl --kubeconfig ~/.kube/config-super delete pod POD_NAME
> ```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `DescribeClusterVirtualNodePools` 返回 `InvalidParameter.ClusterId` | `tccli tke DescribeClusters --region <Region>` 列出集群 | 集群 ID 错误 | 用 `tccli tke DescribeClusters --region <Region>` 获取正确 ClusterId |

### Pod 调度失败（Pending）

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Pod Pending，Events 显示 `0/1 nodes are available` | `kubectl describe pod POD_NAME \| tail -20` | CPU 超出超级节点规格上限（包年包月 8C，按量 64C） | 降低 `resources.requests.cpu` 至允许范围 |
| Pod Pending，Events 显示 `insufficient memory` | 同上 | CPU:内存比超标（包年包月需 < 1:4，按量需 ≤ 1:8） | 降低内存 requests 或增加 CPU requests 以满足比例约束 |
| 包年包月节点池上 Pod 声明 GPU 后 Pending | `tccli tke DescribeClusterVirtualNodePools --region <Region> --ClusterId CLUSTER_ID` 查看计费模式 | 包年包月超级节点不支持 GPU | 改用按量计费超级节点池 |
| 非标准规格 Pod 分配资源与声明不符 | `kubectl get pod POD_NAME -o json \| jq '.spec.containers[0].resources.requests'` | 非标准规格（如 `"0.3"` 核）被向上取整 | 使用标准规格值（如 0.25, 0.5, 1, 2, 4, 8, 16, 32, 64） |

## 下一步

- [调度 Pod 至超级节点](https://cloud.tencent.com/document/product/457/74016) — 调度策略配置
- [指定资源规格](https://cloud.tencent.com/document/product/457/44174) — CPU/内存/GPU/系统盘 Annotation 详解
- [超级节点概述](https://cloud.tencent.com/document/product/457/74014) — 超级节点 vs 普通节点对比
- [超级节点常见问题](https://cloud.tencent.com/document/product/457/60411) — 常见 FAQ

## 控制台替代

[TKE 控制台 → 超级节点 → 购买页](https://console.cloud.tencent.com/tke2/cluster)：购买页展示当前地域支持的计费模式和规格范围。
