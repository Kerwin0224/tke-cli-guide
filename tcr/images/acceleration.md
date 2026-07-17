---
doc_type: How-to
subtype: 6A
fused: true
---
# 镜像加速

> 为 TCR 企业版实例开启镜像加速服务，加速镜像拉取。
> 控制台: [容器镜像服务 - 镜像加速](https://console.cloud.tencent.com/tcr/acceleration)
> 官方文档: [产品服务层级与容量限制](https://cloud.tencent.com/document/product/1141/104731)（“按需加载容器镜像”仅企业版高级版支持；该页未说明 `CFSVIP` 的客户端用法）

## 触发条件

- 你需要为 TCR 企业版实例创建镜像加速服务 — 用本页步骤创建服务并通过 `DescribeImageAccelerateService` 查询开通状态
- 你遇到服务未开通或状态异常 — 看 [故障恢复](#故障恢复) 段诊断

## 准备工作

- 已创建 TCR 企业版实例，并已备 VPC、子网和 CFS 权限组
- 已配置 tccli 凭证 (见 [配置凭证](../../getting-started/credentials.md))



## 概述

`CreateImageAccelerationService` 在指定 VPC、子网和可用区中创建 CFS，并配置存储类型和权限组。`DescribeImageAccelerateService` 返回开通状态、镜像加速状态和 `CFSVIP`；该字段的接口说明仅为“CFS 的 VIP”。当前接口说明与产品服务层级文档均未说明客户端应将 `CFSVIP` 用作 Docker、containerd 或其他镜像客户端的拉取地址，因此不能据此改写镜像引用或替代 TCR 实例访问域名。

> **能力边界**：产品服务层级文档将“容器镜像缓存”列为企业版各规格支持，将“按需加载容器镜像”列为仅企业版高级版支持；该页面未说明本接口创建的 CFS 与“容器镜像缓存”的对应关系，也未提供 `CFSVIP` 的客户端配置步骤。创建接口可以证明会在指定 VPC/子网创建 CFS，但不能据此证明客户端直接通过 `CFSVIP` 拉取镜像。

## 决策依据

### 是否需要加速

| 场景 | 是否启用 | 说明 |
|:-----|:--------:|:-----|
| 需要创建接口所定义的镜像加速服务 | ✅ | 可通过接口创建并查询服务状态 |
| 仅需使用 TCR 实例域名正常推送、拉取 | ❌ | 当前接口与官方文档未要求将客户端切换到 `CFSVIP` |
| 需要确认客户端接入方式 | ⚠️ | 当前接口与服务层级文档未提供 `CFSVIP` 客户端配置方法 |

### CFS 存储选型

`CreateImageAccelerationService` 的 `StorageType` 决定 CFS 类型：

| StorageType | 含义 | 适用 |
|:------------|:-----|:-----|
| `SD` | 标准型存储 | 接口支持的取值 |
| `HP` | 性能存储 | 接口支持的取值 |

> `PGroupId` 是 CFS 权限组 ID。接口要求同时提供 `Zone` 和 `SubnetId`，但当前接口说明未声明二者必须位于同一可用区。

## 关键字段

| 参数 | 所属 Action | 必填 | 说明 |
|:-----|:-----------|:----:|:-----|
| `RegistryId` | 全部 | 是 | TCR 实例 ID |
| `VpcId` | Create | 是 | VPC ID（CFS 所在） |
| `SubnetId` | Create | 是 | 创建 CFS 所属子网 ID |
| `StorageType` | Create | 是 | CFS 类型 SD/HP |
| `PGroupId` | Create | 是 | CFS 权限组 ID |
| `Zone` | Create | 是 | 可用区名称 |

> `--PGroupId` 缺失报 `the following arguments are required: --PGroupId`（exit 252）。

## 操作步骤

### 步骤 1：查询加速服务现状

```bash
tccli tcr DescribeImageAccelerateService --region <REGION> --RegistryId <REGISTRY_ID>
# expected: exit 0, IsEnable 反映当前状态
```
```json
{
    "Status": "",
    "CFSVIP": "",
    "IsEnable": false,
    "RequestId": "..."
}
```

> tccli 默认输出的是 SDK 响应模型，已去除腾讯云 HTTP API 的外层 `Response` 包装；未使用 `--filter` 时，`Status`/`CFSVIP`/`IsEnable` 等字段位于 tccli JSON 的顶层。`IsEnable=false` 表示未开启。接口仅将 `CFSVIP` 定义为“CFS 的 VIP”，不将其定义为镜像客户端拉取地址。

### 步骤 2：创建加速服务

```bash
tccli tcr CreateImageAccelerationService --region <REGION> \
  --RegistryId <REGISTRY_ID> \
  --VpcId <VPC_ID> \
  --SubnetId <SUBNET_ID> \
  --StorageType SD \
  --PGroupId <PGROUP_ID> \
  --Zone <ZONE>
# expected: exit 0
```

### 步骤 3：查询确认开启

```bash
tccli tcr DescribeImageAccelerateService --region <REGION> --RegistryId <REGISTRY_ID>
# expected: exit 0；以 IsEnable 判断是否开通，CFSVIP 和 Status 按接口原样读取
```

## 验证

| 维度 | 命令 | 期望 |
|:-----|:-----|:-----|
| 服务已开启 | `DescribeImageAccelerateService` | `IsEnable=true` |
| 服务状态可查询 | `DescribeImageAccelerateService` | 读取 `Status` 和 `CFSVIP`；接口未公布 `Status` 枚举，也未规定 `CFSVIP` 非空可单独判定服务就绪 |
| 客户端拉取路径 | 使用 TCR 实例域名执行既有拉取流程 | 不把 `CFSVIP` 直接写入镜像引用；当前接口与官方文档未提供该用法 |

| 占位符 | 含义 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<REGISTRY_ID>` | TCR 实例 ID | 企业版 | `tccli tcr DescribeInstances --region <REGION>` |
| `<REGION>` | 地域 | 如 `ap-guangzhou` | `tccli tcr DescribeRegions` |
| `<VPC_ID>` | VPC ID | CFS 所在 VPC | `tccli vpc DescribeVpcs` |
| `<SUBNET_ID>` | 子网 ID | 选择创建 CFS 所需的子网；接口未声明其与 Zone 的跨字段约束 | `tccli vpc DescribeSubnets` |
| `<PGROUP_ID>` | CFS 权限组 ID | 已创建 | CFS 服务创建 |
| `<ZONE>` | 可用区 | 选择创建 CFS 所需的可用区；接口未声明其与 SubnetId 的跨字段约束 | `tccli tcr DescribeRegions` / `tccli cvm DescribeZones` |

## 清理

```bash
tccli tcr DeleteImageAccelerateService --region <REGION> --RegistryId <REGISTRY_ID>
# expected: exit 0
```

> `DeleteImageAccelerateService` 仅需 `RegistryId`。接口未说明删除后 CFS、缓存数据或客户端拉取行为如何变化。

## 副作用

- **创建加速服务**要求指定 VPC、子网、CFS 存储类型、权限组和可用区。
- 当前接口说明未给出计费、子网 IP 占用、删除后的 CFS 生命周期或缓存数据保留规则；执行前需从控制台或腾讯云支持确认。

## 故障恢复

| 现象 | 根因 | 修复 |
|:-----|:-----|:-----|
| `the following arguments are required: --PGroupId`（exit 252） | 缺 CFS 权限组 ID | 先在 CFS 服务创建权限组，传 `--PGroupId` |
| VPC/子网不存在 | `VpcId`/`SubnetId` 错误 | `tccli vpc DescribeVpcs` / `DescribeSubnets` 查真实 ID |
| 创建请求被服务端拒绝 | VPC、子网、权限组、可用区或存储类型不符合服务端约束 | 根据响应中的 `Error.Code` 检查对应参数；当前接口说明未列出跨字段约束 |
| `StorageType` 无效 | 非 SD/HP | 用 `SD` 或 `HP` |
| `IsEnable` 为 false | 服务未开通或创建尚未反映为已开通 | 查询 `DescribeImageAccelerateService`；接口未公布 `Status` 枚举，不能假定某个状态值 |
| 不确定客户端如何使用 `CFSVIP` | 接口仅说明该字段是“CFS 的 VIP” | 不将其用作镜像拉取地址；通过腾讯云支持确认具体接入方式 |

> 参数校验样本：缺 `--PGroupId` → `tccli: error: the following arguments are required: --PGroupId`（exit 252，客户端解析错误）。

## 收尾确认

```bash
# 查询镜像加速服务是否开通及服务端返回字段
tccli tcr DescribeImageAccelerateService --region <REGION> --RegistryId "<REGISTRY_ID>" \
  --filter "{enable:IsEnable,vip:CFSVIP,status:Status}"
# expected: enable=true；vip 和 status 按接口原样返回，不据其值推断客户端拉取路径
```

---

## 下一步

- 镜像推送拉取：[推送与拉取镜像](push-pull.md)
- 实例访问配置：[访问管理](../instances/manage-access.md)
- 实例同步（跨地域）：[实例同步](../replication/manage.md)
