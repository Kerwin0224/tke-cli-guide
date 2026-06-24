# CLB 类型 Ingress 概述（tccli）

> 对照官方：[概述](https://cloud.tencent.com/document/product/457/45685) · page_id `45685`

## 概述

Ingress 是允许访问到集群内 Service 的规则的集合，提供七层 HTTP/HTTPS 服务暴露能力。TKE 集群默认启用基于腾讯云 CLB 的 TKE Ingress Controller（`l7-lb-controller`），通过最终一致性持续同步 CLB 配置与 Ingress 声明。

### Ingress Controller 选择规则

TKE Ingress Controller 支持两种区分机制：

1. `kubernetes.io/ingress.class` annotation
2. `spec.ingressClassName` 字段

匹配规则：

| 条件 | 行为 |
|------|------|
| 未指定 annotation 和字段 | TKE Ingress Controller 管理 |
| annotation 或字段值为 `qcloud` | TKE Ingress Controller 管理 |
| 修改 annotation/字段内容 | TKE Ingress Controller 相应纳入或排除管理范围（涉及资源创建与释放） |

## 前置条件

- 已安装 kubectl 并连接目标 TKE 集群
- 了解 Ingress 生命周期管理标签（见下方）

### Ingress 生命周期管理标签

| 标签 | 说明 | 可修改 |
|------|------|--------|
| `tke-createdBy-flag = yes` | 标识资源由容器服务创建；带此标签的 Ingress 销毁时删除对应 CLB 资源 | 否 |
| `tke-clusterId = <ClusterId>` | 标识哪个集群使用此资源 | 否 |
| `tke-lb-ingress-uuid = <Ingress UUID>` | 标识哪个 Ingress 使用此资源 | 否 |
| `tke-lifecycle-owner = tke\|user` | CLB 生命周期管理权归属。`tke`=删除 Ingress 自动删 CLB；`user`=删除 Ingress 不删 CLB | 是 |

> **注意**：除 `tke-lifecycle-owner` 外，其他标签对用户只读。修改或删除可能导致资源泄漏。

### 禁用 TKE Ingress Controller

**方法一**：将 `kube-system:l7-lb-controller` Deployment 副本数缩为 0：

```bash
kubectl scale deploy l7-lb-controller -n kube-system --replicas=0
```

**方法二**：若 Deployment 不存在，在 ConfigMap `kube-system:tke-service-controller-config` 的 data 中将 `EnableIngressController` 设为 `false`：

```bash
kubectl edit cm tke-service-controller-config -n kube-system
```

> **警告**：禁用前确保集群中无 TKE Ingress Controller 管理的 Ingress 资源，避免 CLB 无法释放。

## 注意事项

1. **禁止在 CLB 控制台手动删除或修改 Ingress 关联的 CLB**——所有 CLB 操作应在 TKE 侧完成。若已误删，参考 FAQ 恢复。
2. **删除顺序**：删除 Ingress 时先删除自动创建的 CLB，再删除 Ingress。已有 CLB 场景下删除 Ingress 不会删除 CLB。
3. **容器业务不可与 CVM 业务共用一个 CLB**。
4. **CLB 删除保护/Private Link**：若 CLB 开启删除保护或使用 Private Link，删除 Service/Ingress 不会删除该 CLB。
5. **托管 Ingress Controller**：TKE Ingress Controller 将逐步托管至元集群，用户集群中不再可见；托管版本与 Service Controller 版本匹配。

### 查询 Ingress Controller 版本

```bash
kubectl -n kube-system get cm tke-ingress-controller-config \
    -o jsonpath='{.data.VERSION}'
```

预期输出：

```
v2.12.0
```

### 查询当前 Ingress 列表

```bash
kubectl get ingress --all-namespaces
```

预期输出：

```
NAMESPACE   NAME           CLASS    HOSTS              ADDRESS        PORTS   AGE
default     my-ingress     <none>   example.com        1.2.3.4        80      2d
ns-app      app-ingress    <none>   app.example.com    1.2.3.4        80      1h
```

### 查询 Ingress 关联的 CLB ID

```bash
kubectl get ingress <IngressName> -n <Namespace> \
    -o jsonpath='{.metadata.annotations.kubernetes\.io/ingress\.qcloud-loadbalance-id}'
```

预期输出：

```
lb-xxxxxxxx
```

### 查询 CLB 生命周期归属

```bash
kubectl get ingress <IngressName> -n <Namespace> \
    -o jsonpath='{.metadata.labels.tke-lifecycle-owner}'
```

预期输出：

```
tke
```

## 控制台与 CLI 参数映射

| 控制台操作 | kubectl/tccli 命令 | 幂等 |
|-----------|-------------------|:--:|
| 查看资源列表 | `kubectl get RESOURCE` | 是 |
| 创建资源 | `kubectl apply -f resource.yaml` | 否 |
| 删除资源 | `kubectl delete RESOURCE NAME` | 是 |

## 操作步骤

本页面为概念性说明，无操作步骤。具体操作指引请参考子页面 [Ingress 基本功能](../Ingress 基本功能/tccli 操作.md)、[Ingress 使用已有 CLB](../Ingress 使用已有 CLB/tccli 操作.md) 等。

## 验证

不适用（概述页面）。

## 清理

不适用（概述页面）。

## 排障

> **注意**：本页为概念参考页，排障请参见对应任务页的排障章节。

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `tccli` 未安装或版本过低 | `tccli --version` | 环境未准备 | 参见 [环境准备](../../../../../环境准备.md) |
| `Unable to connect to the server` | `kubectl cluster-info` | 公网端点被 CAM 策略 strategyId:240463971 以 `tke:clusterExtranetEndpoint=true` 条件拒绝 | 通过 IOA/VPN 连接内网，或使用同 VPC CVM 执行 kubectl |

## 下一步

- [Ingress 基本功能](../Ingress 基本功能/tccli 操作.md)
- [Ingress 使用已有 CLB](../Ingress 使用已有 CLB/tccli 操作.md)
- [Ingress 使用 TkeServiceConfig 配置 CLB](../Ingress 使用 TkeServiceConfig 配置 CLB/tccli 操作.md)
- [Ingress Annotation 说明](../Ingress Annotation 说明/tccli 操作.md)
- [多 Ingress 复用 CLB](../多 Ingress 复用 CLB/tccli 操作.md)

## 控制台替代

控制台入口：集群详情 > 服务与路由 > Ingress > 新建
