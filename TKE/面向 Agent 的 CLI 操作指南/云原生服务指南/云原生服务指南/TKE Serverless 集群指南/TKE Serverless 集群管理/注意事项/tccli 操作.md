# 注意事项（tccli）

> 对照官方：[注意事项](https://cloud.tencent.com/document/product/457/39815) · page_id `39815`

## 概述

使用 TKE Serverless 集群时需注意一些与标准 TKE 集群不同的特性和限制。这些差异源自 Serverless 集群的无节点架构和 VPC 网络直通模式。

> **重要提示：** Serverless 容器服务已升级为 TKE 标准集群 + 超级节点模式，新建 Serverless 集群入口已关闭。存量用户集群不受影响。

## 前置条件

- [环境准备](../../环境准备.md)

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|------|
| 查看集群信息 | `DescribeEKSClusters` | 是 |
| 查询集群（验证配置是否合规） | `DescribeEKSClusters` | 是 |

## 操作步骤

### 1. Pod 规格配置

Pod 的资源规格决定了可用的运行时资源和计费。规格通过以下方式指定：

- **YAML 中的 `resources.requests` 和 `resources.limits`**
- **TKE Serverless 特有 Annotation**：参见 [Annotation 说明](../../Kubernetes%20对象管理/Annotation%20说明/tccli%20操作.md)

示例：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  annotations:
    eks.tke.cloud.tencent.com/cpu: "2"
    eks.tke.cloud.tencent.com/mem: "4Gi"
spec:
  containers:
  - name: app
    image: nginx:latest
    resources:
      requests:
        cpu: "2"
        memory: "4Gi"
      limits:
        cpu: "2"
        memory: "4Gi"
```

### 2. Pod 临时存储

| 特性 | 说明 |
|------|------|
| 临时存储容量 | 每个 Pod 创建时获得**最多 20 GiB** 的免费临时镜像存储 |
| 可用空间 | 实际可用空间**小于 20 GiB**（镜像层占用部分空间） |
| 生命周期 | 临时存储在 Pod 生命周期结束时被**删除**，不可用于持久性数据 |
| 推荐方案 | 重要数据或大容量需求应使用持久化存储卷（CBS/CFS） |

### 3. Kubernetes 版本

- 不支持 **v1.14 以下**的 Kubernetes 版本
- 建议使用 v1.16+ 版本，推荐 v1.28+

### 4. 节点（Node）特性

| 特性 | 说明 |
|------|------|
| 物理节点 | 无法添加或管理物理节点 |
| 超级节点 | 集群以超级节点替代物理节点，每个超级节点映射到一个 VPC 子网 |
| 超级节点操作 | 支持节点亲和性、污点（Taint）、Cordon 操作 |
| Pod 容量 | 超级节点上可调度的 Pod 数量仅受对应 VPC 子网的 IP 数量限制 |
| `kubectl get nodes` | 返回的是超级节点（虚拟节点），非物理机 |

### 5. 容器网络

- 使用 **VPC 网络**，与 CVM、数据库等云产品处于同一网络平面
- 每个 Pod 占用一个 VPC 子网 IP
- 同一 VPC 内 Pod 之间、Pod 与云产品之间的通信**无性能开销**

### 6. Pod 隔离

- Pod 具有与 CVM 同等的安全隔离级别
- 底层物理服务器使用虚拟化技术隔离 Pod

### 7. Service 限制

| 特性 | 限制 |
|------|------|
| NodePort Service | **不支持** — Serverless 集群无实体节点 |
| ClusterIP Service | 支持，消耗 Service CIDR 子网 IP |
| LoadBalancer Service | 支持公网和内网 CLB |

### 8. 卷（Volume）限制

| 特性 | 说明 |
|------|------|
| HostPath | 支持但可能达不到预期效果 — **不同 Pod 可能在不同宿主机上** |
| HostNetwork: true | 兼容但不一定满足预期效果 |
| DnsPolicy: ClusterFirstWithHostNet | 兼容但不一定满足预期效果 |
| 持久化存储 | CBS（云硬盘）和 CFS（文件存储）正常工作，推荐使用 |
| Host 相关参数强依赖 | 不应强制依赖 Host 相关参数（如 HostPath 数据共享） |

### 9. 端口限制

| 端口 | 说明 |
|------|------|
| **9100** | 系统预留端口，不可在 Pod 中使用 |

### 10. 内核参数限制

- 仅以 `net` 开头的内核参数可以自定义（如 `net.core.somaxconn`）
- 其他内核参数由系统统一管理，不可修改

### 查询集群以验证配置

```bash
tccli tke DescribeEKSClusters \
    --ClusterIds '["cls-example"]' \
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
            "ClusterName": "my-serverless-cluster",
            "Status": "Running",
            "VpcId": "vpc-example",
            "SubnetIds": ["subnet-example-a", "subnet-example-b"],
            "K8SVersion": "1.28.3",
            "CreatedTime": "2024-01-15T10:30:00Z"
        }
    ],
    "RequestId": "abc123-def456-ghi789"
}
```

## 验证

| 验证项 | 命令 | 期望结果 |
|--------|------|---------|
| 集群状态 | `tke DescribeEKSClusters --region ap-guangzhou` | `Status: "Running"` |
| K8S 版本 >= 1.14 | 同上，检查 `K8SVersion` 字段 | 版本号 >= 1.14.0 |

## 清理

无需清理。本页为概念参考页，不创建任何资源。

## 排障

| 现象 | 原因 | 处理 |
|------|------|------|
| NodePort Service 不生效 | Serverless 集群不支持 NodePort | 改用 ClusterIP 或 LoadBalancer 类型 Service |
| HostPath 数据无法在 Pod 间共享 | 不同 Pod 可能在不同宿主机上 | 改用 CBS 或 CFS 持久化存储 |
| Pod 使用 9100 端口失败 | 9100 端口为系统预留 | 改用其他端口 |
| Pod 内核参数修改不生效 | 仅 `net.*` 参数可自定义 | 确认修改的参数是否以 `net` 开头 |
| 临时存储不足 | 镜像占用部分 20 GiB 空间 | 减小镜像体积，或使用持久化存储卷 |

## 下一步

- [工作负载管理](../../Kubernetes%20对象管理/工作负载管理/tccli%20操作.md) — 部署工作负载到 Serverless 集群
- [Annotation 说明](../../Kubernetes%20对象管理/Annotation%20说明/tccli%20操作.md) — 了解通过 Annotation 自定义超级节点配置

## 控制台替代

[容器服务控制台 → 集群 → 目标集群 → 基本信息](https://console.cloud.tencent.com/tke2/cluster) — 查看集群配置详情。
