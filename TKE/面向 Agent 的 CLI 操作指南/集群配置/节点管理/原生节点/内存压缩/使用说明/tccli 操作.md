# 内存压缩使用说明

> 对照官方：[使用说明](https://cloud.tencent.com/document/product/457/102456) · page_id `102456`

## 概述

原生节点 QosAgent 组件提供**内存压缩**能力，通过主动回收 PageCache、控制内存水位和识别冷热内存，在不扩容节点的情况下提升内存利用率。开启后，QosAgent 在节点内存压力升高时自动触发压缩，将不活跃内存页回收或压缩，释放更多可用内存给业务容器。

| 方案 | 适用场景 | 压缩行为 |
|------|---------|---------|
| 按节点池默认开启 | 测试/开发环境快速验证 | 使用推荐的默认压缩阈值 |
| 自定义压缩参数 | 生产环境精细管控 | 自定义 PageCache 回收比例、内存水位线和回收间隔 |

## 前置条件

- [环境准备](../../../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置，region 为目标地域

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:DescribeClusterNodePools, tke:DescribeClusterNodePoolDetail
#    tke:ModifyClusterNodePool, tke:DescribeClusterInstances
# 验证：执行 DescribeClusterNodePools 确认权限
tccli tke DescribeClusterNodePools --region <Region> --ClusterId CLUSTER_ID
# expected: exit 0，返回节点池列表（可为空）

# 4. 检查 kubectl 可用
kubectl version --client
# expected: Client Version 显示版本号
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
# 5. 确认集群为托管集群且版本 >= 1.24
tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterType 为 MANAGED_CLUSTER，ClusterVersion >= 1.24.0

# 6. 确认原生节点池存在且状态正常
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId CLUSTER_ID \
    --NodePoolId NODE_POOL_ID
# expected: Type 为 "Native"，LifeState 为 "normal"

# 7. 确认节点已就绪
kubectl get nodes
# expected: 原生节点 STATUS 为 Ready

# 8. 确认 QosAgent 组件已部署
kubectl get pods -n kube-system -l app=qos-agent
# expected: 返回 Running 状态的 QosAgent Pod 列表
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
| 查看节点池 | `DescribeClusterNodePoolDetail` | 是 |
| 开启内存压缩 | `ModifyClusterNodePool --Annotations` | 是 |
| 查看节点列表 | `DescribeClusterInstances` | 是 |
| 查看 QosAgent 状态 | kubectl get pods -n kube-system | 是 |

## 关键字段说明

内存压缩通过节点池 `Annotations` 声明开启，主要 Annotation 如下：

| 字段（Annotation Name） | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `tke.cloud.tencent.com/memory-compression-enabled` | String | 是 | `"true"` 开启，`"false"` 关闭 | 拼写错误或不传 → 压缩不生效 |
| `tke.cloud.tencent.com/memory-compression-threshold` | String | 否 | 内存水位百分比，如 `"80"`。节点可用内存低于此比例时触发压缩 | 阈值过高 → 频繁压缩影响性能；阈值过低 → 压缩不及时 |
| `tke.cloud.tencent.com/memory-compression-period` | String | 否 | 压缩检查周期，如 `"30s"`、`"1m"` | 周期过短 → 节点管家频繁检查消耗 CPU |

## 操作步骤

### 步骤 1：控制面 — 查看当前节点池配置

```bash
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId CLUSTER_ID \
    --NodePoolId NODE_POOL_ID
# expected: 返回节点池完整配置，含 Annotations 数组
```

**预期输出**：

```json
{
    "NodePool": {
        "NodePoolId": "np-example",
        "Name": "native-pool-example",
        "ClusterId": "cls-example",
        "LifeState": "normal",
        "Type": "Native",
        "Annotations": []
    },
    "RequestId": "..."
}
```

### 步骤 2：控制面 — 为节点池开启内存压缩

#### 选择依据

- **默认开启**：使用默认阈值，QosAgent 自动管理压缩行为，适合快速验证和通用场景。
- **自定义参数**：根据业务内存使用特征调整阈值和周期。例如，内存密集型应用可将阈值设为 `"70"`（更早触发压缩），周期设为 `"30s"`（更高频检测）。
- **Annotation 仅对新节点生效**：修改节点池 Annotations 后，存量节点需手动添加对应 Annotation。

#### 最小配置（默认参数开启）

`enable-memory-compression-minimal.json`：

```json
{
  "ClusterId": "CLUSTER_ID",
  "NodePoolId": "NODE_POOL_ID",
  "Annotations": [
    {
      "Name": "tke.cloud.tencent.com/memory-compression-enabled",
      "Value": "true"
    }
  ]
}
```

```bash
tccli tke ModifyClusterNodePool --region <Region> \
    --cli-input-json file://enable-memory-compression-minimal.json
# expected: exit 0，返回 RequestId
```

**预期输出**：

```json
{
    "Response": {
        "RequestId": "xxxx-xxxx-xxxx-xxxx"
    }
}
```

#### 增强配置（自定义阈值和周期）

`enable-memory-compression-enhanced.json`：

```json
{
  "ClusterId": "CLUSTER_ID",
  "NodePoolId": "NODE_POOL_ID",
  "Annotations": [
    {
      "Name": "tke.cloud.tencent.com/memory-compression-enabled",
      "Value": "true"
    },
    {
      "Name": "tke.cloud.tencent.com/memory-compression-threshold",
      "Value": "80"
    },
    {
      "Name": "tke.cloud.tencent.com/memory-compression-period",
      "Value": "60s"
    }
  ]
}
```

```bash
tccli tke ModifyClusterNodePool --region <Region> \
    --cli-input-json file://enable-memory-compression-enhanced.json
# expected: exit 0，返回 RequestId
```

### 步骤 3：数据面 — 给存量节点打 Annotation

存量节点不会自动继承节点池 Annotation，需手动添加：

```bash
# 查看原生节点名称
kubectl get nodes -l node.tke.cloud.tencent.com/type=native
# expected: 返回原生节点列表

# 给节点打内存压缩 Annotation
kubectl annotate node NODE_NAME \
    tke.cloud.tencent.com/memory-compression-enabled=true
# expected: node/NODE_NAME annotated
```

```text
NAME  STATUS  AGE
...
```

### 步骤 4：数据面 — 开启 QosAgent 的压缩功能

确认 QosAgent 配置已启用内存压缩：

```bash
# 查看 QosAgent 配置
kubectl get configmap -n kube-system qos-agent-config -o yaml
# expected: 配置中包含 memory_compression 相关字段

# 确认 QosAgent Pod 已重启并加载配置
kubectl rollout status daemonset/qos-agent -n kube-system
# expected: daemon set successfully rolled out
```

```text
NAME  STATUS  AGE
...
```

## 验证

### 控制面（tccli）

| 维度 | 检查内容 | 预期 |
|------|---------|------|
| 节点池 Annotation | `DescribeClusterNodePoolDetail` → `Annotations` | 包含 `tke.cloud.tencent.com/memory-compression-enabled: "true"` |

### 数据面（kubectl）

| 维度 | 检查内容 | 预期 |
|------|---------|------|
| 节点 Annotation | `kubectl get node NODE_NAME -o jsonpath='{.metadata.annotations}'` | 包含 `tke.cloud.tencent.com/memory-compression-enabled: "true"` |
| QosAgent 运行 | `kubectl get pods -n kube-system -l app=qos-agent` | 所有 Pod 状态为 Running |
| QosAgent 日志 | `kubectl logs -n kube-system -l app=qos-agent --tail=20` | 含 memory compression 相关日志 |
| 压缩效果 | 参见 [压缩监控](../压缩监控/tccli%20操作.md) | 压缩率和收益指标正常 |

```bash
# 验证节点池 Annotation 已更新
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId CLUSTER_ID \
    --NodePoolId NODE_POOL_ID \
    | jq '.NodePool.Annotations'
# expected: 输出含 memory-compression-enabled 的 Annotation 列表

# 验证节点 Annotation
kubectl get node NODE_NAME -o jsonpath='{.metadata.annotations.tke\.cloud\.tencent\.com/memory-compression-enabled}'
# expected: true
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

## 清理

### 控制面（tccli）

```bash
# 移除节点池的内存压缩 Annotation
tccli tke ModifyClusterNodePool --region <Region> \
    --ClusterId CLUSTER_ID \
    --NodePoolId NODE_POOL_ID \
    --Annotations '[]'
# expected: exit 0
```

### 数据面（kubectl）

```bash
# 移除节点的内存压缩 Annotation
kubectl annotate node NODE_NAME \
    tke.cloud.tencent.com/memory-compression-enabled-
# expected: node/NODE_NAME annotated
```

> **注意**：关闭内存压缩后，已回收的内存不会自动释放，节点按正常内核内存管理策略运行。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `ModifyClusterNodePool` 返回 `InvalidParameterValue.Annotation` | 检查 Annotation Name 是否以 `tke.cloud.tencent.com/` 开头 | Annotation Key 格式不符合 TKE 命名规范 | 使用 `tke.cloud.tencent.com/` 前缀 |
| `ModifyClusterNodePool` 返回 `ResourceNotFound.NodePoolNotFound` | `tccli tke DescribeClusterNodePoolDetail --region <Region> --ClusterId CLUSTER_ID --NodePoolId NODE_POOL_ID` | NodePoolId 不存在或非原生类型 | 确认节点池存在且 Type 为 `Native` |

### 配置成功但压缩不生效

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 节点池 Annotation 已设置但存量节点未压缩 | `kubectl get node NODE_NAME -o jsonpath='{.metadata.annotations}'` | 节点池级 Annotation 仅对新节点生效 | 手动在存量节点上 `kubectl annotate node` 添加同样的 Annotation |
| QosAgent 运行但压缩日志为空 | `kubectl logs -n kube-system -l app=qos-agent --tail=50` | 节点内存未达到触发阈值，或压缩周期未到 | 等待内存使用率升高后观察；降低阈值加速触发 |
| QosAgent Pod 频繁重启 | `kubectl describe pod -n kube-system -l app=qos-agent` | QosAgent 配置有误或节点资源不足 | 检查 qos-agent-config ConfigMap，确认参数合法 |
| 压缩后内存释放不明显 | 对比压缩前后 `kubectl top node NODE_NAME` | PageCache 占比本身较低，压缩收益有限 | 确认业务内存使用模式；压缩主要回收 PageCache，非 RSS |

## 下一步

- [压缩监控](../压缩监控/tccli%20操作.md) — page_id `102625`
- [原生节点功能支持说明](../../原生节点功能支持说明/tccli%20操作.md)
- [声明式操作实践](../../声明式操作实践/tccli%20操作.md) — page_id `78649`

## 控制台替代

[容器服务控制台 → 集群 → 节点管理 → 原生节点池 → 详情 → 运维信息](https://console.cloud.tencent.com/tke2/cluster)
