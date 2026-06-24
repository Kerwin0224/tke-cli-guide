# 计费概述（tccli）

> 对照官方：[计费概述](https://cloud.tencent.com/document/product/457/39807) · page_id `39807`

## 概述

TKE Serverless 集群不收取集群管理费用，仅对集群内运行的 Pod 资源进行计费。以**超级节点**为资源维度，提供四种计费模式，适用于不同的业务场景。

> **重要提示：** Serverless 容器服务已升级为 TKE 标准集群 + 超级节点模式，新建 Serverless 集群入口已关闭。存量用户集群不受影响。

## 前置条件

- [环境准备](../../环境准备.md)
- 已注册腾讯云账号并完成实名认证

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|------|
| 查看集群列表（获取计费相关状态） | `DescribeEKSClusters` | 是 |
| 查看超级节点配置 | `DescribeEKSClusters --filter` 查看 `SubnetIds`、`Status` | 是 |

## 操作步骤

### 查询集群及其计费相关属性

通过 `DescribeEKSClusters` 查看集群列表，了解集群运行状态（只有 Running 状态的集群才会产生 Pod 资源费用）：

```bash
tccli tke DescribeEKSClusters \
    --region ap-guangzhou \
    --output json
```

示例输出：

```json
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "cls-example",
            "ClusterName": "production-serverless",
            "ClusterDesc": "生产环境 Serverless 集群",
            "VpcId": "vpc-example",
            "SubnetIds": ["subnet-example-a", "subnet-example-b"],
            "Status": "Running",
            "CreatedTime": "2024-01-15T10:30:00Z",
            "K8SVersion": "1.28.3",
            "ClusterVersion": "1.28"
        }
    ],
    "RequestId": "abc123-def456-ghi789"
}
```

### 计费模式说明

TKE Serverless 以超级节点（Super Node）为资源维度，提供四种计费模式：

| 计费模式 | 说明 | 适用场景 |
|---------|------|---------|
| **按量计费（Pay-as-you-go）** | 按 Pod 实际申请的规格（CPU/内存）和运行时长计费，秒级计费、按小时结算 | 临时测试、弹性伸缩、不确定的资源用量 |
| **包年包月（Monthly）** | 预购超级节点总规格（CPU/内存），在预购范围内不额外按量收费 | 长期稳定的生产环境 |
| **预留券（Reserved Vouchers）** | 承诺一定时长的资源用量，获取折扣价格 | 有稳定长期资源需求，需降低单位成本 |
| **竞价模式（Spot）** | 以较低价格竞拍空闲资源，可能被回收 | 无状态、可中断的批处理/分析任务 |

#### 按量计费

- 所有地域均支持按量计费模式
- 支持多个子网，每个子网创建一个超级节点
- 创建时不收费，Pod 调度运行后按 Pod 规格和运行时长计费
- 计算公式：费用 = Pod 申请 CPU 核数 x 单价 x 运行秒数 + Pod 申请内存 GiB x 单价 x 运行秒数

#### 包年包月

- 仅支持**北京、上海、广州、南京**四个地域
- 仅支持**单子网**
- 配置参数：
  - **CPU 总规格**：50–5000 核（超级节点可调度的 Pod CPU 总数）
  - **CPU:内存配比**：1:2 或 1:4（总 CPU 与总内存比例，支持调度 1:1 至 1:4 的 Pod）
  - **购买时长**和**自动续费**设置

### 其他关联产品费用

使用 TKE Serverless 时可能产生以下关联产品费用：

| 产品 | 计费触发条件 | 参考文档 |
|------|------------|---------|
| **负载均衡 CLB** | 创建 LoadBalancer 类型 Service 或 Ingress | CLB [计费概述](https://cloud.tencent.com/document/product/214/42934) |
| **云硬盘 CBS** | 创建 PersistentVolumeClaim 并绑定 CBS | CBS [计费概述](https://cloud.tencent.com/document/product/362/2345) |
| **文件存储 CFS** | 创建 PersistentVolumeClaim 并绑定 CFS | CFS [计费概述](https://cloud.tencent.com/document/product/582/9553) |

各产品按各自计费规则独立计费。

## 验证

| 验证项 | 命令 | 期望结果 |
|--------|------|---------|
| 集群列表可查 | `tke DescribeEKSClusters --region ap-guangzhou` | 返回集群列表 |
| 集群状态为 Running | 同上，查看 `Status` 字段 | 值为 `"Running"` 时 Pod 运行会产生费用 |

## 清理

无需清理。本页为计费概念参考页，不创建任何资源。

## 排障

| 现象 | 原因 | 处理 |
|------|------|------|
| 不确定费用来源 | Pod 规格 x 运行时长 + 关联产品 | 前往[费用中心](https://console.cloud.tencent.com/expense/bill)按产品查看明细 |
| Pod 未运行但产生了费用 | 可能是关联产品（CLB/CBS/CFS）的费用 | 检查集群中是否创建了 LoadBalancer Service 或持久化存储 |

## 下一步

- [欠费说明](../欠费说明/tccli%20操作.md) — 了解账户欠费对 Serverless 集群的影响
- [购买限制](../购买限制/tccli%20操作.md) — 了解资源配额限制

## 控制台替代

[容器服务控制台 → 集群 → 新建 Serverless 集群](https://console.cloud.tencent.com/tke2/ecluster/create) — 创建页面展示计费模式选择及预估费用。
