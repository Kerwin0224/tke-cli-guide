# 地域和可用区（tccli）

> 对照官方：[地域和可用区](https://cloud.tencent.com/document/product/457/58172) · page_id `58172`

## 概述

TKE Serverless 集群支持多个地域（Region）和可用区（Availability Zone）。地域是物理数据中心的地理区域，不同地域间完全隔离。可用区是同一地域内独立电力与网络的物理数据中心，故障隔离，同一 VPC 内的不同可用区间可通过内网互通。

> **重要提示：** Serverless 容器服务已升级为 TKE 标准集群 + 超级节点模式，新建 Serverless 集群入口已关闭。存量用户集群不受影响。需要超级节点的用户应创建 TKE 标准集群并添加超级节点。

## 前置条件

- [环境准备](../../环境准备.md)
- 已注册腾讯云账号并完成实名认证

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|------|
| 查询产品支持的地域列表 | `tke DescribeEKSClusters --region` 或 `api DescribeRegions` | 是 |
| 查询地域下的可用区列表 | `api DescribeZones` | 是 |
| 查看可创建 Serverless 集群的地域 | `tke DescribeEKSClusters --filter` | 是 |

## 操作步骤

### 查询 TKE Serverless 支持的地域

通过 `DescribeEKSClusters` 在不同地域查询，返回不为空的即表示该地域支持 Serverless 集群（即使无实例，API 返回空列表表明地域可达）。

```bash
tccli tke DescribeEKSClusters \
    --region ap-guangzhou \
    --output json
```

示例输出（广州地域存在集群时）：

```json
{
    "TotalCount": 3,
    "Clusters": [
        {
            "ClusterId": "cls-example01",
            "ClusterName": "my-serverless-cluster",
            "ClusterDesc": "示例集群",
            "VpcId": "vpc-example",
            "SubnetIds": ["subnet-example"],
            "Status": "Running",
            "CreatedTime": "2024-01-15T10:30:00Z",
            "ClusterVersion": "1.28",
            "K8SVersion": "1.28.3"
        }
    ],
    "RequestId": "abc123-def456-ghi789"
}
```

### 查询可用区列表

使用 `DescribeZones` 获取指定地域的所有可用区：

```bash
tccli api DescribeZones \
    --Product tke \
    --region ap-guangzhou \
    --output json
```

若不支持 `--Product` 过滤，可查询全量可用区：

```bash
tccli api DescribeZones \
    --region ap-guangzhou \
    --output json
```

示例输出：

```json
{
    "TotalCount": 4,
    "ZoneSet": [
        {
            "Zone": "ap-guangzhou-3",
            "ZoneName": "广州三区",
            "ZoneState": "AVAILABLE"
        },
        {
            "Zone": "ap-guangzhou-4",
            "ZoneName": "广州四区",
            "ZoneState": "AVAILABLE"
        },
        {
            "Zone": "ap-guangzhou-6",
            "ZoneName": "广州六区",
            "ZoneState": "AVAILABLE"
        },
        {
            "Zone": "ap-guangzhou-7",
            "ZoneName": "广州七区",
            "ZoneState": "AVAILABLE"
        }
    ],
    "RequestId": "abc123-def456-ghi789"
}
```

### TKE Serverless 支持的地域与可用区一览

| 地域 | Region ID | 可用区 |
|------|----------|--------|
| 华南地区（广州） | `ap-guangzhou` | ap-guangzhou-3, 4, 6, 7 |
| 华南地区（深圳金融） | `ap-shenzhen-fsi` | ap-shenzhen-fsi-1, 2, 3 |
| 华东地区（上海） | `ap-shanghai` | ap-shanghai-2, 3, 4, 5 |
| 华东地区（上海金融） | `ap-shanghai-fsi` | ap-shanghai-fsi-1, 2, 3 |
| 华东地区（南京） | `ap-nanjing` | ap-nanjing-1, 2 |
| 华北地区（北京） | `ap-beijing` | ap-beijing-3, 4, 5, 6, 7 |
| 华北地区（北京金融） | `ap-beijing-fsi` | ap-beijing-fsi-1, 2, 3 |
| 西南地区（成都） | `ap-chengdu` | ap-chengdu-1, 2 |
| 港澳台地区（中国香港） | `ap-hongkong` | ap-hongkong-2 |
| 亚太东南（新加坡） | `ap-singapore` | ap-singapore-1, 2 |
| 亚太东南（雅加达） | `ap-jakarta` | ap-jakarta-1 |
| 亚太东北（首尔） | `ap-seoul` | ap-seoul-1, 2 |
| 亚太东北（东京） | `ap-tokyo` | ap-tokyo-2 |
| 亚太东南（曼谷） | `ap-bangkok` | ap-bangkok-1 |
| 美国东部（弗吉尼亚） | `na-ashburn` | na-ashburn-1, 2 |
| 欧洲地区（法兰克福） | `eu-frankfurt` | eu-frankfurt-1 |

> **金融专区说明：** 深圳金融、上海金融、北京金融为合规专区，面向金融行业客户，需提交申请并通过审核后方可使用。

### 跨地域通信说明

- 不同地域间默认不能通过内网通信，如需互通需使用**云联网（Cloud Connect Network）**或公网 IP
- 负载均衡默认同地域转发，支持**跨地域绑定**功能
- 同一地域、同一 VPC、同一账号下的不同可用区可直接通过内网 IP 互通
- 跨账号资源即使在同一地域、同一 VPC 也完全隔离

## 验证

| 验证项 | 命令 | 期望结果 |
|--------|------|---------|
| 广州地域 API 可达 | `tke DescribeEKSClusters --region ap-guangzhou` | 返回 `TotalCount` 和 `Clusters` 列表（空列表亦可） |
| 广州可用区列表 | `api DescribeZones --region ap-guangzhou` | `ZoneSet` 包含 ap-guangzhou-3/4/6/7 |

## 清理

无需清理。本页为参考概念页，不创建任何资源。

## 排障

| 现象 | 原因 | 处理 |
|------|------|------|
| `DescribeEKSClusters` 报 `AuthFailure` | 未授权或账号未实名 | 完成实名认证，确保 CAM 权限包含 `tke:DescribeEKSClusters` |
| 某些可用区不在 `DescribeZones` 返回中 | 可用区可能已下线或该地域不包含该可用区 | 以 `DescribeZones` 实时返回为准 |
| `DescribeZones --Product tke` 不生效 | API 版本不支持 Product 过滤 | 去掉 `--Product` 参数，查全量后按需筛选 |

## 下一步

- [创建集群](../../TKE%20Serverless%20集群管理/创建集群/tccli%20操作.md) — 在选定的地域和可用区创建 Serverless 集群
- [计费概述](../计费概述/tccli%20操作.md) — 了解 TKE Serverless 的计费模式

## 控制台替代

[容器服务控制台 → 集群 → 新建 Serverless 集群](https://console.cloud.tencent.com/tke2/ecluster/create) — 创建页面的地域下拉框展示所有可选地域。
