# qGPU 概述

> 对照官方：[qGPU 概述](https://cloud.tencent.com/document/product/457/61448) · page_id `61448`

## 概述

TKE qGPU 是针对原生节点推出的 GPU 容器虚拟化产品，支持多个容器共享 GPU 卡并支持容器间算力和显存精细隔离。同时提供在离线混部能力，在精细切分 GPU 资源的基础上最大限度提高 GPU 利用率。

**重要：qGPU 仅针对 TKE 原生节点开放，其他节点类型不提供服务保障。**

## 前置条件

- 原生节点（搭载 FinOps 理念）
- 支持的 GPU 架构：Volta（V100）、Turing（T4）、Ampere（A100/A10）
- TKE 版本 >= v1.14.x

## 产品优势

| 优势 | 说明 |
|------|------|
| 灵活性 | 精细配置 GPU 算力占比和显存大小 |
| 强隔离 | 支持显存和算力的严格隔离 |
| 在离线 | 支持业界唯一在离线混部能力，GPU 利用率压榨到最大化 |
| 覆盖度 | 支持 Volta、Turing、Ampere 主流架构 |
| 云原生 | 支持标准 Kubernetes 和 NVIDIA Docker |
| 兼容性 | 业务不重编、CUDA 库不替换、业务无感 |

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 安装 qGPU 组件 | `tccli tke InstallAddon --AddonName qgpu` | 否 |
| 查看 qGPU 组件状态 | `tccli tke DescribeAddon` | 是 |
| 开启节点池 qGPU 共享 | 创建原生节点池时设置 Label `tke.cloud.tencent.com/qgpu-schedule-policy` | 否（新建节点池） |

## qGPU 隔离策略

| Label 值 | 缩写 | 中文名 | 含义 |
|---|---|---|---|
| `best-effort`（默认） | `be` | 争抢模式 | 各 Pod 不限制算力，卡上有剩余算力即可使用 |
| `fixed-share` | `fs` | 固定配额 | 每个 Pod 有固定算力配额，无法超过，即使 GPU 有空闲算力 |
| `burst-share` | `bs` | 保证配额加弹性 | 调度器保证保底算力，空闲算力可被使用，被分配后回退到配额 |

## qGPU 资源声明

```yaml
spec:
  containers:
    resources:
      limits:
        tke.cloud.tencent.com/qgpu-memory: "5"   # 显存（GB），最小 1G
        tke.cloud.tencent.com/qgpu-core: "30"     # 算力（%），5-100，精度 5
```

- 显存分配粒度 1G
- 算力分配粒度 5%（即 5、10、15...100）
- 整卡分配：`qgpu-core: 100` 或 N * 100
- 一个 GPU 卡最多 16 个 qGPU 设备

## 验证

```bash
# 确认 qGPU 组件状态
tccli tke DescribeAddon --region <Region> \
    --ClusterId <ClusterId> \
    | jq '.Addons[] | select(.AddonName == "qgpu") | {AddonName, AddonVersion, Status}'
# expected: Status 为 "Enabled"
```

```json
{
  "Addons": "<Addons>",
  "AddonName": "<AddonName>",
  "AddonVersion": "<AddonVersion>",
  "RawValues": "<RawValues>",
  "Phase": "<Phase>",
  "Reason": "<Reason>"
}
```

## 清理

本页为概念说明页，无资源需清理。

## 下一步

- [使用 qGPU](../使用 qGPU/tccli 操作.md)
- [qGPU 离在线混部说明](../qGPU 离在线混部/qGPU 离在线混部说明/tccli 操作.md)
- [GPU 故障检测与自愈](../../GPU 故障检测与自愈/tccli 操作.md)

## 控制台替代

[TKE 控制台 → 集群详情 → 组件管理](https://console.cloud.tencent.com/tke2/cluster)：安装 QGPU 组件；节点池创建时开启 qGPU 共享。
