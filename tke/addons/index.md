---
doc_type: Overview
---
# 集群插件

> TKE 集群插件管理——扩展集群功能的组件，如 CBS CSI、Ingress Controller、eniipamd。

## 这是什么

插件是封装好的 Kubernetes 组件，一键安装到集群。TKE 提供官方插件列表，每个插件有版本与配置。

## 核心概念

| 概念 | 含义 | 为什么重要 |
|:-----|:-----|:-----|
| Addon | 集群插件（封装的 Helm Chart） | 扩展集群功能 |
| AddonName | 插件名（如 `eniipamd`） | 安装/查询的标识 |
| AddonVersion | 插件版本 | 决定功能与兼容性 |
| Phase | 插件状态 | 判断插件是否正常运行 |
| RawValues | 插件配置（base64 编码的 JSON） | 自定义插件行为 |

## 常见插件

| 插件 | 作用 |
|:-----|:-----|
| `eniipamd` | VPC-CNI 弹性网卡管理 |
| `cbs-csi` | CBS 云硬盘 CSI 存储 |
| `cionfig` | 集群配置管理 |
| `kubejarvisservice` | 集群巡检 |
| `gatekeeper` | OPA 策略引擎 |

> 插件本质是 Helm Release，用 `DescribeClusterReleases` 也能查到。见 [应用发布](../releases/manage.md)。

## 不适用场景

- 不需扩展集群功能 → 跳过插件
- 用 Helm 直接装应用 → 看 [应用发布](../releases/manage.md)
- 需要非官方插件 → 用 Helm Release 直接部署

## 快速检查

```bash
# 查看某插件状态（实测 eniipamd Phase=Succeeded）
tccli tke DescribeAddon --region <REGION> --ClusterId "<CLUSTER_ID>" --AddonName eniipamd \
  --filter "Addons[].{name:AddonName,ver:AddonVersion,phase:Phase}"
# expected: 插件信息，Phase 含 Succeeded
```

## 文档

- [插件管理](manage.md) — 安装/更新/卸载插件
- [应用发布](../releases/manage.md) — Helm Release（插件本质）
- [故障排查](../troubleshooting.md) — 插件 Phase 异常诊断
