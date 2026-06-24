# TKE · 面向 Agent 的 CLI 操作指南

腾讯云容器服务（TKE）的 `tccli` 命令行操作指南，面向 AI Agent 和运维人员，覆盖 460+ 个 CLI 操作场景。

## 📖 内容概览

| 分类 | 说明 |
|------|------|
| 🚀 快速入门 | 入门示例（Nginx、Hello World、WordPress 等）、快速创建集群 |
| ⚙️ 集群配置 | 集群管理、网络配置（GlobalRouter / VPC-CNI / Cilium-Overlay）、节点管理 |
| 📦 应用配置 | 组件和应用管理、容器登录方式 |
| 🎯 调度配置 | Crane 调度器、QoS 感知、CPU Burst、内存精细调度等 |
| 🔧 故障处理 | 常见集群和网络问题排障指南 |
| 📊 Data 应用实践 | TKE 集群混部 Spark 等大数据实践 |

## 🚀 快速开始

```bash
# 环境准备 - 安装和配置 tccli
# 详见：环境准备 章节

# 创建你的第一个集群
tccli tke CreateCluster --ClusterName "my-cluster" ...
```

## 📂 目录

👉 查看 [完整文档目录](./SUMMARY.md)（460+ 操作场景）
