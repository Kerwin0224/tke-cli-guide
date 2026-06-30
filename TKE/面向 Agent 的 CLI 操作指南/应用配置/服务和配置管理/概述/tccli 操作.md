# 服务和配置管理概述（tccli）

> 对照官方：[概述](https://cloud.tencent.com/document/product/457/31700) · page_id `31700`

## 概述

服务和配置管理涵盖 Kubernetes 中与应用程序运行直接相关的对象管理。Kubernetes 对象是集群中的持久化实体，承载着运行中的业务负载。不同 K8s 对象表达不同的意图：运行什么应用、应用可用哪些资源、以及应用关联何种策略。

对象管理可通过 [TKE 控制台](https://console.cloud.tencent.com/tke2/overview) 或 Kubernetes API（如 kubectl）操作。

### 对象分类

| 类别 | 对象 | 说明 |
|------|------|------|
| **服务** | Service | 提供 Pod 访问的 K8s 对象，支持 ClusterIP、NodePort、LoadBalancer、ExternalName 四种访问类型 |
| **服务** | Ingress | 管理集群中 Service 的外部访问，提供 L7 HTTP/HTTPS 负载均衡 |
| **配置** | ConfigMap | 保存非敏感配置信息，通过数据卷或环境变量挂载到 Pod |
| **配置** | Secret | 保存密码、Token、密钥等敏感信息，数据以 Base64 编码 |
| **存储** | Volume / PV / PVC / StorageClass | 容器数据存储及持久化卷管理 |
| **其他** | Namespaces | 单集群内的逻辑环境分区，实现多团队/项目隔离 |

### 资源限制

TKE 使用 `ResourceQuota/tke-default-quota` 对所有**托管集群**施加资源限制。详见 [Kubernetes 资源配额说明](https://cloud.tencent.com/document/product/457/9087)。

## 前置条件

- [环境准备](../../../环境准备.md)
- 已 [连接集群](../../../集群配置/集群管理/连接集群/tccli 操作.md)，kubectl 可正常访问 APIServer
- CAM 权限：`tke:DescribeClusters`

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 查看集群列表 | `tccli tke DescribeClusters` | 是 |
| 查看命名空间 | `kubectl get namespaces` | 是 |
| 查看 Service | `kubectl get services` | 是 |
| 查看 Ingress | `kubectl get ingress` | 是 |
| 查看 ConfigMap | `kubectl get configmaps` | 是 |
| 查看 Secret | `kubectl get secrets` | 是 |
| 查看资源配额 | `kubectl get resourcequotas` | 是 |

## 操作步骤

### 查看集群信息

```bash
tccli tke DescribeClusters \
    --region <Region> \
    --ClusterIds '["<ClusterId>"]'
```

<details>
<summary>输出示例</summary>

```json
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "cls-example",
            "ClusterName": "example-cluster",
            "ClusterVersion": "1.28",
            "ClusterType": "MANAGED_CLUSTER",
            "ClusterNetworkSettings": {
                "VpcId": "vpc-example",
                "ClusterCIDR": "10.0.0.0/16"
            }
        }
    ]
}
```
</details>

### 查看所有命名空间

```bash
kubectl get namespaces
```

```
NAME              STATUS   AGE
default           Active   30d
kube-node-lease   Active   30d
kube-public       Active   30d
kube-system       Active   30d
```

### 查看所有资源配额

```bash
kubectl get resourcequotas --all-namespaces
```

```
NAMESPACE   NAME                  AGE   REQUEST                                      LIMIT
default     tke-default-quota     30d   cpu: 0/1000, memory: 0/200Gi                ...
```

## 验证

### 数据面（kubectl）

```bash
kubectl api-resources --namespaced=true
```

列出所有命名空间级别的 API 资源。

### 控制面（tccli）

```bash
tccli tke DescribeClusters --region <Region>
```
```json
{
    "RequestId": "00000000-0000-0000-0000-000000000000",
    ...
}
```


## 清理

不适用（概述页面）。

## 排障

> **注意**：本页为概念参考页，排障请参见对应任务页的排障章节。

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `tccli` 未安装或版本过低 | `tccli --version` | 环境未准备 | 参见 [环境准备](../../../环境准备.md) |
| `Unable to connect to the server` | `kubectl cluster-info` | 公网端点被 CAM 策略 strategyId:240463971 以 `tke:clusterExtranetEndpoint=true` 条件拒绝 | 通过 IOA/VPN 连接内网，或使用同 VPC CVM 执行 kubectl |

## 下一步

- [Namespaces](../Namespaces/tccli 操作.md)
- [ConfigMap](../ConfigMap/tccli 操作.md)
- [Secret](../Secret/tccli 操作.md)
- [Service 概述](../Service/概述/tccli 操作.md)
- [Service 基本功能](../Service/Service 基本功能/tccli 操作.md)

## 控制台替代

[控制台 → 集群 → 详情](https://console.cloud.tencent.com/tke2/cluster) 查看各类对象。
