# 关联集群（tccli）

> 对照官方：[关联集群](https://cloud.tencent.com/document/product/457/71898) · page_id `71898`

## 概述

通过 `tccli monitor CreatePrometheusClusterAgent` 将 TKE 集群关联到 Prometheus 监控实例。关联成功后，采集插件会自动部署到集群，可编辑数据采集规则。支持跨 VPC 关联，实现同一监控实例管理不同地域和 VPC 下的集群。

解除关联时，采集插件随之删除。

## 前置条件

- [环境准备](../../../../环境准备.md)
- 已创建 TKE 集群（标准集群、Serverless 集群、边缘集群或注册集群）
- 已创建 Prometheus 监控实例且状态为「运行中」（参见 [创建监控实例](../创建监控实例/tccli%20操作.md)）

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 关联集群 | `tccli monitor CreatePrometheusClusterAgent --cli-input-json file://associate.json` | 否（重复关联同一集群会报错） |
| 查看已关联集群 | `tccli monitor DescribePrometheusClusterAgents --InstanceId <InstanceId>` | 是 |
| 解除关联 | `tccli monitor DeletePrometheusClusterAgent --InstanceId <InstanceId> --ClusterId <ClusterId>` | 否 |
| 查看关联进度 | `tccli monitor DescribeClusterAgentCreatingProgress --InstanceId <InstanceId> --ClusterId <ClusterId>` | 是 |

## 操作步骤

### 1. 确认 Prometheus 实例状态

```bash
tccli monitor DescribePrometheusInstances --region ap-guangzhou --InstanceIds '["prom-example01"]' --filter "InstanceSet[0].InstanceStatus"
```

确保返回 `InstanceStatus` 为 `2`（运行中）。

### 2. 查看当前已关联集群（可选）

```bash
tccli monitor DescribePrometheusClusterAgents --region ap-guangzhou --InstanceId prom-example01 --filter "Agents[].ClusterId, Agents[].ClusterName, Agents[].Status"
```

### 3. 准备关联参数

**基础关联（同 VPC 同地域）：**

```json
{
    "InstanceId": "prom-example01",
    "Agents": [
        {
            "ClusterId": "cls-example",
            "ClusterType": "tke",
            "Region": "ap-guangzhou",
            "EnableExternal": false
        }
    ],
    "OpenDefaultRecord": true
}
```

**跨 VPC 关联（需公网 CLB 或已建立云联网）：**

```json
{
    "InstanceId": "prom-example01",
    "Agents": [
        {
            "ClusterId": "cls-example-cross",
            "ClusterType": "tke",
            "Region": "ap-shanghai",
            "EnableExternal": true,
            "NotScrape": false,
            "ExternalLabels": [
                {"Name": "cluster", "Value": "shanghai-prod"},
                {"Name": "env", "Value": "production"}
            ]
        }
    ],
    "OpenDefaultRecord": true
}
```

参数说明：

| 参数 | 必填 | 说明 |
|------|------|------|
| `InstanceId` | 是 | Prometheus 实例 ID |
| `Agents[].ClusterId` | 是 | 目标 TKE 集群 ID |
| `Agents[].ClusterType` | 是 | 集群类型：`tke`（标准）、`eks`（Serverless）、`edge`（边缘）、`external`（注册） |
| `Agents[].Region` | 是 | 集群所在地域 |
| `Agents[].EnableExternal` | 否 | 是否开启公网访问，跨 VPC 且未建立云联网时需设为 `true` |
| `Agents[].NotScrape` | 否 | 设为 `true` 则关联后不开启默认采集，需手动配置采集规则 |
| `Agents[].ExternalLabels` | 否 | 全局标记，该集群所有指标都会附加这些标签 |
| `Agents[].InClusterPodConfig` | 否 | 采集器 Pod 部署参数（容忍度、节点选择器、HostNet 模式） |
| `Agents[].NotInstallBasicScrape` | 否 | `true` 则不安装基础采集 |
| `OpenDefaultRecord` | 否 | 是否开启默认预聚合规则，首次关联建议设为 `true` |

### 4. 执行关联

```bash
tccli monitor CreatePrometheusClusterAgent --region ap-guangzhou --cli-input-json file://associate.json
```

```json
{
    "Response": {
        "RequestId": "ghi789-..."
    }
}
```

### 5. 验证关联结果

```bash
tccli monitor DescribePrometheusClusterAgents --region ap-guangzhou --InstanceId prom-example01
```

```json
{
    "Response": {
        "Agents": [
            {
                "ClusterId": "cls-example",
                "ClusterName": "demo-cluster",
                "ClusterType": "tke",
                "Status": "normal",
                "Region": "ap-guangzhou",
                "VpcId": "vpc-example",
                "DesiredAgentNum": 2,
                "ReadyAgentNum": 2,
                "ExternalLabels": []
            },
            {
                "ClusterId": "cls-example-cross",
                "ClusterName": "prod-shanghai",
                "ClusterType": "tke",
                "Status": "normal",
                "Region": "ap-shanghai",
                "VpcId": "vpc-sh-example",
                "EnableExternal": true,
                "DesiredAgentNum": 2,
                "ReadyAgentNum": 2,
                "ExternalLabels": [
                    {"Name": "cluster", "Value": "shanghai-prod"},
                    {"Name": "env", "Value": "production"}
                ]
            }
        ],
        "Total": 2,
        "IsFirstBind": false,
        "ImageNeedUpdate": false,
        "RequestId": "abc123-..."
    }
}
```

Agent 状态：`normal` = 正常，`abnormal` = 异常。`DesiredAgentNum` 与 `ReadyAgentNum` 一致说明采集组件全部就绪。

### 6. 解除关联（如需）

```bash
tccli monitor DeletePrometheusClusterAgent --region ap-guangzhou --InstanceId prom-example01 --ClusterId cls-example
```

## 验证

```bash
# 确认关联状态为 normal 且 ReadyAgentNum 等于 DesiredAgentNum
tccli monitor DescribePrometheusClusterAgents --region ap-guangzhou --InstanceId prom-example01 --filter "Agents[].ClusterId, Agents[].Status, Agents[].ReadyAgentNum, Agents[].DesiredAgentNum"
```

## 清理

解除关联用 `DeletePrometheusClusterAgent`。注意解除关联后，集群中采集插件会被删除，该集群不再上报监控数据。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `CreatePrometheusClusterAgent` 返回重复关联错误 | `tccli monitor DescribePrometheusClusterAgents --InstanceId <InstanceId>` 检查是否已关联 | 同一集群已关联到该实例，重复关联会报错 | 先执行 `DeletePrometheusClusterAgent` 解除旧关联，再重新关联 |
| `CreatePrometheusClusterAgent` 返回集群类型不支持 | 检查 `ClusterType` 参数值 | `ClusterType` 枚举值无效 | 使用有效值：`tke`（标准）、`eks`（Serverless）、`edge`（边缘）、`external`（注册） |

### 关联成功但状态异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `Status` 为 `abnormal` | `tccli monitor DescribePrometheusClusterAgents --InstanceId <InstanceId>` 查看 Agent 详情 | 跨 VPC 关联时云联网未建立或公网 CLB 不可达 | 检查集群网络连通性与云联网/CLB 配置；同 VPC 关联检查安全组规则 |
| `ReadyAgentNum` < `DesiredAgentNum` | `kubectl get pods -n prom-<InstanceId>` 检查采集器 Pod | 采集器 Pod 调度失败，可能是集群资源不足或污点未容忍 | 检查集群节点资源和污点配置；适当增加 `InClusterPodConfig` 容忍度 |
| 关联创建超时 | `tccli monitor DescribeClusterAgentCreatingProgress --InstanceId <InstanceId> --ClusterId <ClusterId>` 查看进度 | 安装采集组件耗时较长或失败 | 等待数分钟；如持续异常，保留 InstanceId、ClusterId、RequestId 联系工单 |
| `IsFirstBind: true` 后预聚合规则未生效 | `tccli monitor DescribePrometheusConfig` 检查预聚合规则 | 首次关联需安装预聚合规则，有一定延迟 | 等待 3-5 分钟后重新检查 |

## 下一步

- [数据采集配置](../数据采集配置/tccli%20操作.md) 配置 ServiceMonitor / PodMonitor 采集规则
- [精简监控指标](../精简监控指标/tccli%20操作.md) 优化采集指标以控制成本

## 控制台替代

控制台：Prometheus 监控 → 点击实例名 → 集群监控 → 关联集群 → 选择集群类型/地域/集群 → 确定。
