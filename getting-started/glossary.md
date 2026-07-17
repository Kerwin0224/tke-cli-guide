---
doc_type: Reference
fused: false
---
# 术语表

> 本指南涉及的腾讯云与 Kubernetes 术语速查。文档中首次出现时链接到本表对应条目。
>
> 官方文档：[TKE 产品概述](https://cloud.tencent.com/document/product/457/6759) · [TCR 产品概述](https://cloud.tencent.com/document/product/1141/39288)

## 腾讯云

| 术语 | 英文 | 释义 |
|:-----|:-----|:-----|
| CAM | Cloud Access Management | 腾讯云访问管理。管理子账号、API 密钥、权限策略。TCCLI 凭证（SecretId/SecretKey）在此创建。 |
| SecretId / SecretKey | — | CAM API 密钥对。SecretId 是公开标识，SecretKey 是密钥（切勿泄露）。TCCLI 用它调用腾讯云 API。 |
| 主账号 | Root account | 腾讯云注册账号，拥有全部权限。其 API 密钥风险最高，生产环境用子账号。 |
| 子账号 | Sub-account / CAM 用户 | 主账号下创建的用户，授予最小权限。生产/CI/CD 应使用子账号密钥。 |
| profile | — | TCCLI 的命名配置集。一个 profile 存一组凭证+地域，用 `--profile <NAME>` 切换，支持多账号。 |
| 地域 | Region | 腾讯云数据中心的地理区域，如 `ap-guangzhou`（广州）、`ap-shanghai`（上海）。多数资源的创建、查询和默认端点具有地域作用域；复制、同步或网络互通等跨地域能力以具体产品功能为准。 |
| VPC | Virtual Private Cloud | 腾讯云上逻辑隔离的网络空间。TKE 节点位于用户选择的 VPC 子网；TCR 企业版是托管 Registry，可配置连接用户 VPC 的私网访问链路，不等于实例本身部署在用户 VPC 内。 |
| 子网 | Subnet | VPC 内的 IP 地址段。TKE 节点、CVM 实例从子网分配内网 IP。 |
| CIDR | Classless Inter-Domain Routing | 无类别域间路由，表示 IP 地址段（如 `10.0.0.0/16`）。TKE 集群的 Pod 网段、Service 网段用 CIDR 表示。 |
| CVM | Cloud Virtual Machine | 腾讯云服务器。TKE 工作节点本质是带 K8s 组件的 CVM。 |

## TKE（容器服务）

| 术语 | 英文 | 释义 |
|:-----|:-----|:-----|
| TKE | Tencent Kubernetes Engine | 腾讯云容器服务，托管 Kubernetes 集群。 |
| 集群 | Cluster | 一组 Master + 工作节点，运行 K8s。TKE 有托管（MANAGED_CLUSTER）和独立（INDEPENDENT_CLUSTER）两种。 |
| Master | Master node | K8s 控制面节点（运行 API Server/etcd/scheduler）。托管集群由腾讯云运维，独立集群自维护。 |
| 工作节点 | Worker node | 运行 Pod 的节点。TKE 工作节点本质是 CVM。 |
| 节点池 | Node pool | 一组同配置工作节点的集合，支持自动扩缩容。 |
| kubeconfig | — | kubectl 连接 K8s 集群的凭证文件（含证书+端点）。TKE 用 `DescribeClusterKubeconfig` 获取。 |
| Pod | Pod | K8s 最小调度单元，含一个或多个容器。 |
| RBAC | Role-Based Access Control | K8s 基于角色的访问控制。TKE 子账号权限通过 RBAC 授予。 |
| OIDC | OpenID Connect | 企业身份系统单点登录协议。TKE 支持用 OIDC 接企业 SSO 认证集群。 |
| 端点 | Endpoint | 集群 API 的访问入口。有内网端点（VPC 内访问）和公网端点（互联网访问）。 |
| GlobalRouter | GR | TKE 容器网络模型之一：为每个节点预分配独立于 VPC 的 Pod CIDR，Pod IP 来自该段（不占 VPC 子网 IP）；跨节点经 VPC 全局路由转发。创建时须设 `ClusterCIDR`，且不得与 VPC CIDR / 同 VPC 其他集群 Pod CIDR 重叠。 |
| VPC-CNI | VPC-CNI | TKE 推荐的容器网络模型：IPAMD 将弹性网卡（ENI）的辅助 IP 分配给 Pod，Pod 与节点同属 VPC 二层网络。**Pod IP 占用 VPC 子网地址**，须为容器预留子网（`EniSubnetIds`）；支持共享网卡 / 独立网卡及固定 Pod IP。 |
| ENI | Elastic Network Interface | 弹性网卡。CVM/节点的虚拟网卡，VPC-CNI 模式下 Pod 通过 ENI 辅助 IP 通信。 |
| EIP | Elastic IP | 弹性公网 IP。可绑定 CVM/CLB，提供互联网访问。公网端点常通过 EIP 暴露。 |
| CLB | Cloud Load Balancer | 腾讯云负载均衡。TKE Service 类型 LoadBalancer 自动创建 CLB。 |
| CBS | Cloud Block Storage | 腾讯云云硬盘。TKE 节点系统盘/数据盘、PV 持久化卷常用 CBS。 |
| L5 | L5 | **TKE 托管集群规格（ClusterLevel）** 的最低档（`L5`～`L5000`）。规格决定管理费与 K8s 资源配额（Pod/ConfigMap/CRD 等上限），见 [TKE 配额](../tke/reference/quotas.md)。与 CLB 产品无关。 |
| kubectl | kubectl | Kubernetes 命令行工具。操作集群资源（Pod/Service/Deployment）用，与 TCCLI（操作腾讯云资源）不同。安装见 [Kubernetes 官方文档](https://kubernetes.io/docs/tasks/tools/)。 |

## TCR（容器镜像服务）

| 术语 | 英文 | 释义 |
|:-----|:-----|:-----|
| TCR | Tencent Container Registry | 腾讯云容器镜像服务。有企业版（实例）和个人版。 |
| 实例 | Instance | TCR 企业版的独立镜像仓库实例，独立域名+独立配额。 |
| 命名空间 | Namespace | 实例内的隔离单元，如 `team-a`。仓库属于命名空间。 |
| 仓库 | Repository | 存放一个镜像各版本的单元，格式 `namespace/repo`。 |
| 镜像 | Image | 容器镜像，通过 `docker push/pull` 推送拉取。 |
| Token | — | TCR 临时访问凭证。`docker login` 用 Username + Token 登录仓库。 |
| imagePullSecret | — | K8s 中存镜像仓库凭证的 Secret。TKE 集群拉 TCR 镜像时配置。 |
| 个人版 | Personal | TCR 免费版，功能受限，无独立实例。 |

## tccli

| 术语 | 释义 |
|:-----|:-----|
| tccli | 腾讯云命令行工具。本指南所有操作的入口。 |
| Action | TCCLI 调用的 API 操作名，如 `CreateCluster`、`DescribeClusters`。 |
| `--generate-cli-skeleton` | 生成 Action 入参骨架 JSON 的命令（调试用；日常执行可直接按文档参数传参）。 |
| `--filter` | JMESPath 表达式，从响应中提取字段。如 `--filter "Clusters[0].ClusterId"`。 |
| `--version` | 指定 API 版本。TKE 有 2018-05-25（默认，全功能）和 2022-05-01（官方当前版）两版本。 |
| `--region` | 指定地域。如 `--region ap-guangzhou`。 |
| `# expected:` | 代码块内的注释，标注命令的成功信号。执行时保留该注释不影响命令。 |

## 下一步

- [安装 TCCLI](install.md) — 若尚未安装
- [配置凭证](credentials.md) — 获取 CAM 密钥并配置 TCCLI
- [TKE 概览](../tke/index.md) — 了解 TKE 对象模型
- [TCR 概览](../tcr/index.md) — 了解 TCR 对象模型
