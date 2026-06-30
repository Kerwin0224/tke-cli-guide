# TencentOS Rainbow 镜像介绍

> 对照官方：[TencentOS Rainbow 镜像介绍](https://cloud.tencent.com/document/product/457/131166) · page_id `131166`

## 概述

TencentOS Rainbow 是腾讯云专为云原生场景设计的轻量级、安全可靠的容器操作系统，作为 TKE 官方公共镜像提供。采用不可变发行版设计，集成 Kernel、Linux 发行版和容器运行时，为 K8s 集群提供底层底座。

**当前限制：仅支持 K8s 1.30 版本。**

## 前置条件

- TKE 集群 >= v1.30
- 适用场景：对安全性、弹性扩缩容速度、节点一致性有极高要求的大规模 K8s 集群

## 核心架构

Rainbow 由四大核心模块构成：

| 模块 | 功能 | 特点 |
|------|------|------|
| **基础 OS**（不可变发行版） | 根文件系统只读，系统与用户数据分离 | 禁止 `dnf/yum` 安装软件，防止系统碎片化和配置漂移 |
| **升级管理**（OS Update Operator） | 基于 K8s Operator 的声明式升级 | A/B 双分区，原子化升级和回滚 |
| **配置管理**（OS Config Operator） | 集中式 GitOps 化管理 | 声明式配置下发，防篡改监控 |
| **运维管理**（Admin Container） | 按需启动的独立运维容器 | 特权模式，ssh + 调测工具，不污染基础 OS |

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 查看 Rainbow 镜像 | `tccli tke DescribeImages` | 是 |
| 创建集群使用 Rainbow | `tccli tke CreateCluster --ImageId <RainbowImageId>` | 否 |
| 查看节点 OS 升级状态 | `kubectl get nodeos` | 是 |
| 查看配置下发状态 | `kubectl get confos` | 是 |

## 与传统 TencentOS Server 的差异

| 对比维度 | TencentOS Server（如 3.1 / 4.4） | TencentOS Rainbow |
|---------|-------------------------------|-------------------|
| 系统架构 | 传统通用 Linux 发行版 | 不可变基础设施（Immutable OS），专用于容器化负载 |
| 软件包管理 | 完整 RPM 仓库，`yum`/`dnf` 安装 | 剔除包管理器，移除非必须组件和冗余系统服务 |
| 内核参数 | 通用配置，兼顾广泛场景 | 深度裁剪的云原生定制内核，针对高并发网络和容器隔离调优 |
| 配置管理 | SSH 登录手动修改或脚本批量执行 | K8s 原生声明式 API（ConfOS CRD），禁止手动热改 |
| 系统升级 | 逐个包更新，易中断，回滚困难 | A/B 分区原子级整包替换，一键回滚 |

## 产品优势

| 优势 | 数据 |
|------|------|
| 极致轻量 | 镜像大小约 393MB（相比 TS4 的 4.5G 缩减 90%+） |
| 内存优化 | 内存占用较 CentOS/Ubuntu/TS2/TS3 缩减 11.7%~32.1% |
| 节点启动 | 节点拉起耗时提升 66%~77% |
| 安全加固 | 根文件系统只读，收敛攻击面，配置防篡改 |
| K8s 原生管理 | 通过 `kubectl` 或 K8s API 完成 OS 升级、回滚和内核参数调整 |

## 验证

```bash
# 查看 TKE 当前支持的 Rainbow 镜像
tccli tke DescribeImages --region <Region> \
    | jq '.ImageInstanceSet[] | select(.ImageId | startswith("img-rainbow"))'
# expected: 返回 Rainbow 镜像列表
```

```json
{
  "ImageInfoList": "<ImageInfoList>",
  "Digest": "<Digest>",
  "Size": "<Size>",
  "ImageVersion": "<ImageVersion>",
  "UpdateTime": "<UpdateTime>",
  "Kind": "<Kind>"
}
```

## 清理

本页为概念介绍页，无资源需清理。

## 下一步

- [镜像概述](../../镜像概述/tccli 操作.md)
- [自定义镜像说明](../../自定义镜像说明/tccli 操作.md)

## 控制台替代

[TKE 控制台 → 创建集群/新建节点池](https://console.cloud.tencent.com/tke2/cluster)：操作系统选择 "TencentOS Rainbow"。
