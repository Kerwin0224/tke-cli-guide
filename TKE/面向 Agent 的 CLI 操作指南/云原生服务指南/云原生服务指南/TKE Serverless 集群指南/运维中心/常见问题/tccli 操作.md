# 常见问题（tccli）

> 对照官方：[常见问题](https://cloud.tencent.com/document/product/457/54778) · page_id `54778`

## 概述

本文汇总 TKE Serverless 集群使用过程中的常见问题，按类别组织并从 CLI 操作视角给出参考方案。各 FAQ 类别包含典型问题场景及应使用的 tccli 命令组合。

## 前置条件

- [环境准备](../../../环境准备.md)
- 已配置 `tccli`，可用 `tccli tke help` 验证
- 已有 TKE Serverless 集群或容器实例（部分查询需要资源 ID）

> **注意：** kubectl 不可达（公网端点 CAM 拒绝 strategyId:240463971）。以下所有排查和诊断均通过 tccli API 完成。

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 查看集群信息 | `tccli tke DescribeEKSClusters --ClusterIds '["cls-example"]'` | 是 |
| 查看容器实例 | `tccli tke DescribeEKSContainerInstances --EksCiIds '["eksci-example"]'` | 是 |
| 查看实例事件 | `tccli tke DescribeEKSContainerInstanceEvent --EksCiId "eksci-example"` | 是 |
| 查看监控指标 | `tccli monitor DescribeStatisticData --Namespace QCE/EKS --MetricName <Metric>` | 是 |
| 检索事件日志 | `tccli cls SearchLog --TopicId "topic-example" --Query "<Query>"` | 是 |

## 操作步骤

### TKE Serverless 集群相关问题

#### Q: 如何查看集群当前状态和基本信息？

```bash
tccli tke DescribeEKSClusters \
    --ClusterIds '["cls-example"]' \
    --region ap-guangzhou \
    | jq '.Clusters[0] | {ClusterId, ClusterName: .ClusterDesc, Status, VpcId, ClusterVersion, CreatedTime}'
```

```json
{
  "TotalCount": 0,
  "Clusters": [],
  "ClusterId": "<ClusterId>",
  "ClusterName": "<ClusterName>",
  "VpcId": "<VpcId>",
  "SubnetIds": [],
  "K8SVersion": "<K8SVersion>",
  "Status": "<Status>"
}
```

```output
{
    "ClusterId": "cls-example",
    "ClusterName": "my-serverless-cluster",
    "Status": "Running",
    "VpcId": "vpc-example",
    "ClusterVersion": "1.28",
    "CreatedTime": "2024-07-01T10:00:00Z"
}
```

#### Q: 集群中 Pod 一直处于 Pending 状态怎么办？

1. 通过事件日志检索 Pending 原因：

```bash
tccli cls SearchLog \
    --TopicId "topic-example" \
    --Query "name:<PodName> AND reason:*" \
    --From $(date -d '1 hour ago' +%s) \
    --To $(date +%s) \
    --Limit 10 \
    --Sort "desc" \
    --region ap-guangzhou
```

2. 常见原因及对应排查：

- **子网 IP 不足：** 检查子网剩余 IP，扩容子网或更换子网
- **资源不足：** 调整 CPU/内存 Request，或使用更大规格
- **镜像拉取失败：** 检查镜像地址、TCR 访问权限、网络连通性

#### Q: TKE Serverless 集群是否支持多可用区部署？

支持。创建容器实例时，通过 `SubnetId` 参数指定子网即可决定部署可用区。

#### Q: 集群如何设置资源配额？

通过容器实例的 `Cpu` 和 `Memory` 参数限制单实例规格。集群级别资源配额需通过云监控 `QCE/EKS` 指标配合告警策略实现预警。

### 负载均衡相关问题

#### Q: 如何为容器实例关联 CLB？

创建 Service（通过 super-node kubectl）或在创建容器实例时配置 EIP。CLB 相关 API 为 `tccli clb` 系列：

```bash
# 查询已有 CLB 实例
tccli clb DescribeLoadBalancers --region ap-guangzhou

# 创建 CLB 实例
tccli clb CreateLoadBalancer \
    --LoadBalancerType "OPEN" \
    --VpcId "vpc-example" \
    --SubnetId "subnet-example" \
    --LoadBalancerName "eksci-clb-example" \
    --region ap-guangzhou
```

#### Q: CLB 健康检查失败如何处理？

1. 确认容器内服务端口正常监听
2. 确认安全组 `sg-example` 入站规则已放通 CLB 健康检查源 IP 和端口
3. 检查 `RestartPolicy` 配置是否合理（`Always` 策略可自动恢复异常容器）

#### Q: 如何配置 HTTPS 监听器？

CLB HTTPS 监听器需上传 SSL 证书。结合 `tccli clb` 和 `tccli ssl`：

```bash
# 查看已有证书
tccli ssl DescribeCertificates --region ap-guangzhou

# 创建 CLB HTTPS 监听器
tccli clb CreateListener \
    --LoadBalancerId "lb-example" \
    --Protocol "HTTPS" \
    --Port 443 \
    --CertificateId "cert-example" \
    --region ap-guangzhou
```

### 超级节点相关问题

#### Q:

```json
{
  "TotalCount": 0,
  "Clusters": [],
  "ClusterId": "<ClusterId>",
  "ClusterName": "<ClusterName>",
  "VpcId": "<VpcId>",
  "SubnetIds": [],
  "K8SVersion": "<K8SVersion>",
  "Status": "<Status>"
}
``` 什么是超级节点？如何查看超级节点信息？

超级节点是 TKE Serverless 集群的调度资源池，每个超级节点对应一个可用区的资源池。

```bash
tccli tke DescribeEKSClusters \
    --ClusterIds '["cls-example"]' \
    --region ap-guangzhou \
    | jq '.Clusters[0].SubnetIds'
```

```json
{
  "RequestId": "..."
}
```

#### Q: 超级节点资源不足时会怎样？

容器实例创建请求会返回 `Pending` 状态或报错 `InsufficientResource`。排查步骤：

1. 检查子网是否有可用 IP
2. 尝试更换子网（对应不同可用区的超级节点）
3. 确认实例规格（CPU/内存组合）在超级节点支持的范围内

#### Q: 如何查看超级节点的资源使用情况？

通过云监控 `QCE/EKS` 命名空间的集群级指标查看：

```bash
tccli monitor DescribeStatisticData \
    --Module "monitor" \
    --Namespace "QCE/EKS" \
    --MetricName "K8sClusterCpuCoreUsed" \
    --Period "300" \
    --StartTime "$(date -d '1 hour ago' +%Y-%m-%dT%H:%M:%S+08:00)" \
    --EndTime "$(date +%Y-%m-%dT%H:%M:%S+08:00)" \
    --Conditions '[{"Key":"tke_cls_instance_id","Operator":"=","Value":["cls-example"]}]' \
    --region ap-guangzhou
```

### 镜像仓库相关问题

#### Q: 如何配置容器实例使用 TCR 镜像仓库？

1. 在创建时指定完整的镜像地址（含 TCR 域名）：

```json
{
    "Containers": [
        {
            "Image": "ccr.ccs.tencentyun.com/<namespace>/<repo>:<tag>",
            "Name": "my-app"
        }
    ]
}
```

2. 若 TCR 实例为私有，需配置访问凭证。在创建容器实例时通过 `ImageRegistryCredentials` 字段指定：

```json
{
    "ImageRegistryCredentials": [
        {
            "Name": "tcr-cred",
            "Server": "ccr.ccs.tencentyun.com",
            "Username": "<TCR用户名>",
            "Password": "<TCR密码或临时Token>"
        }
    ]
}
```

#### Q: 镜像拉取失败（ImagePullBackOff）如何排查？

1. 检查镜像地址是否正确（拼写、tag、命名空间）
2. 确认 TCR 仓库权限：如为私有仓库，检查 `ImageRegistryCredentials` 配置
3. 检查网络连通性：
   - VPC 内网拉取 TCR 需 VPC 已关联 TCR 实例的私有域解析
   - 公网拉取需容器实例所在子网有 NAT 网关或公网路由
4. 查看容器实例事件获取详细错误信息：

```bash
tccli tke DescribeEKSContainerIn

```json
{
  "Data": "<Data>",
  "TotalCount": 0,
  "RepoInfo": [],
  "RepoName": "<RepoName>",
  "RepoType": "<RepoType>",
  "TagCount": 0,
  "Public": 0,
  "IsUserFavor": true
}
```stanceEvent \
    --EksCiId "eksci-example" \
    --region ap-guangzhou \
    | jq '.Events[] | select(.Reason | contains("Pull"))'
```

#### Q: TCR 实例相关 CLI 操作

```bash
# 查询 TCR 个人版仓库列表
tccli tcr DescribeRepositoryOwnerPersonal --region ap-guangzhou

# 查询 TCR 企业版实例
tccli tcr DescribeInstances --region ap-guangzhou

# 创建 TCR 长期访问凭证
tccli tcr CreateInstanceToken \
    --RegistryId "tcr-example" \
    --region ap-guangzhou
```

### 日志采集相关问题

#### Q: 如何开启容器日志采集到 CLS？

容器实例日志采集通过创建实例时配置 `LogCollectConfig` 启用。也可以使用独立的日志采集配置：

```bash
# 查看已配置的日志采集规则
tccli tke DescribeLogSwitches \
    --ClusterIds '["cls-example"]' \
    --ClusterType "eks" \
    --region ap-guangzhou
```

#### Q: 日志采集到 CLS 后如何检索？

```bash
tccli cls SearchLog \
    --TopicId "<LogTopicId>" \
    --Query "container_name:nginx-container" \
    --From $(date -d '1 hour ago' +%s) \
    --To $(date +%s) \
    --Limit 20 \
    --region ap-guangzhou
```

#### Q: 日志未采集到 CLS 如何排查？

1. 确认日志采集开关已开启（`DescribeLogSwitches`）
2. 检查容器实例状态为 `Succeeded`（`Running`）
3. 确认容器日志输出到 stdout/stderr（CLS 默认采集标准输出）
4. 检查 CLS 日志主题是否存在（`tccli cls DescribeTopics`）
5. 查看容器实例事件，确认日志 Agent 是否成功启动

#### Q: 如何配置自定义日志路径采集？

在日志采集配置中指定容器内文件路径。需确保：
- 容器拥有日志目录的写权限
- 日志文件支持轮转（避免单文件过大）
- 采集路径为绝对路径

```bash
# 查看日志采集配置详情
tccli tke DescribeEksLogConfig \
    --ClusterId "cls-example" \
    --region ap-guangzhou
```

### Prometheus 监控相关问题

#### Q: TKE Serverless 如何接入 Prometheus 监控服务？

1. 创建 Prometheus 监控实例（`tccli monitor CreatePrometheusInstance` 或控制台创建）
2. 关联 Serverless 集群：

```bash
# 查询可关联的集群
tccli tke DescribeEKSClusters \
    --ClusterIds '["cls-example"]' \
    --region ap-guangzhou \
    | jq '.Clusters[0] | {ClusterId, Status}'
```

3. 在 Prometheus 实例中配置数据采集规则（关联命名空间和工作负载）

#### Q: 默认监控指标和 Prometheus 自定义指标的区别？

| 维度 | 云监控 QCE/EKS | Prometheus 监控服务 |
|------|---------------|-------------------|
| 指标来源 | 云监控自动采集 | Prometheus Agent 采集 |
| 指标类型 | 预定义资源指标 | 自定义业务指标 |
| 数据粒度 | 60s / 300s | 15s - 60s（可配置） |
| 存储时长 | 免费 30 天 | 按实例规格 |
| 告警 | 云监控告警策略 | Prometheus Rule + AlertManager |
| CLI | `tccli monitor DescribeStatisticData` | API 需通过 Prometheus 实例的 Remote Read |

#### Q: Prometheus 采集自定义指标的方法？

1. 容器需暴露 `/metrics` 端点
2. 在 Prometheus 监控服务中创建采集配置，可通过 `tccli monitor` 的 Prometheus 相关接口管理抓取任务
3. 确认安全组放通 Prometheus 实例到容器实例的端口

```bash
# 查看 Prometheus 实例的抓取配置
tccli monitor DescribePrometheusScrapeJobs \
    --InstanceId "prom-example" \
    --region ap-guangzhou
```

#### Q: 如何创建基于 Prometheus 指标的告警规则？

```bash
tccli monitor CreatePrometheusAlertRule \
    --InstanceId "prom-example" \
    --RuleName "High CPU Alert" \
    --Expr "rate(container_cpu_usage_seconds_total[5m]) > 0.8" \
    --Duration "5m" \
    --region ap-guangzhou
```

### 通用排查方法

#### Q: 容器实例操作失败，如何定位根因？

按以下顺序排查，每一步都可通过 tccli 完成：

1. **查看实例状态：** `tccli tke DescribeEKSContainerInstances --EksCiIds`
2. **查看实例事件：** `tccli tke DescribeEKSContainerInstanceEvent --EksCiId`
3. **查看监控指标：** `tccli monitor DescribeStatisticData --Namespace QCE/EKS`
4. **查看集群事件日志（如已开启）：** `tccli cls SearchLog --TopicId`

#### Q: 如何批量查询所有容器实例？

```bash
tccli tke DescribeEKSContainerInstances \
    --Limit 100 \
    --region ap-guangzhou \
    | jq '{TotalCount, Instances: [.EksCis[]? | {EksCiId, EksCiName, Status, VpcId, CreationTime}]}'
```

#### Q: 如何获取集群级别的汇总信息？

```bash
# 集群基本信息
tccli tke DescribeEKSClusters \
    --ClusterIds '["cls-example"]' \
    --region ap-guangzhou \
    | jq '.Clusters[0] | {ClusterId, ClusterDesc, Status, ClusterVersion, VpcId, CreatedTime}'

# 集群监控概览
tccli monitor DescribeStatisticData \
    --Module "monitor" \
    --Namespace "QCE/EKS" \
    --MetricName "K8sClusterCpuCoreUsed" \
    --Period "300" \
    --StartTime "$(date -d '24 hours ago' +%Y-%m-%dT%H:%M:%S+08:00)" \
    --EndTime "$(date +%Y-%m-%dT%H:%M:%S+08:00)" \
    --Conditions '[{"Key":"tke_cls_instance_id","Operator":"=","Value":["cls-example"]}]' \
    --region ap-guangzhou
```

## 验证

本页面为常见问题参考，无需独立验证。各问题的排查步骤可在实际场景中验证。

## 清理

无需清理。本页面为参考类文档，不创建任何云资源。

## 排障

| 现象 | 参考章节 |
|------|---------|
| 容器实例创建失败 | 容器实例相关问题及通用排查方法 |
| 镜像拉取失败 | 镜像仓库相关问题 |
| 服务无法访问 | 负载均衡相关问题 |
| 资源调度失败 / Pending | 超级节点相关问题 |
| 日志无法查看 | 日志采集相关问题 |
| 监控无数据 | Prometheus 监控相关问题 |

## 下一步

- [容器实例](../容器实例/tccli%20操作.md) — 容器实例全生命周期 CLI 操作
- [查看监控](../监控和告警/查看监控/tccli%20操作.md) — 云监控指标查询和告警策略管理
- [集群事件](../事件管理/集群事件/tccli%20操作.md) — 通过 CLS 检索集群事件日志
- [日志采集](../日志采集/开通日志采集/tccli%20操作.md) — 开通日志采集功能

## 控制台替代

[容器服务控制台](https://console.cloud.tencent.com/tke2) → 弹性容器 / 集群管理 → 对应资源详情页。云监控控制台 → 告警管理。CLS 控制台 → 检索分析。相关问题也可参考控制台右上角的帮助文档入口。
