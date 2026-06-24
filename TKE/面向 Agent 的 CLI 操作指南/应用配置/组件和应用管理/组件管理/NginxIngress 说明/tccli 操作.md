# NginxIngress 说明（tccli）

> 对照官方：[NginxIngress 说明](https://cloud.tencent.com/document/product/457/51260) · page_id `51260`

## 概述

NginxIngress 是基于 Nginx 的 Ingress Controller 组件，支持更丰富的七层路由功能、自定义配置和性能调优。

### 功能特性

- 支持丰富的 Annotation 自定义路由规则
- 支持 TCP/UDP 四层转发
- 支持 Canary 灰度发布
- 支持自定义 Nginx 配置模板
- 支持 Prometheus 监控指标

### 与 CLB Ingress 对比

| 特性 | NginxIngress | CLB Ingress |
|------|-------------|-------------|
| 实现方式 | Nginx Pod（自管理） | 腾讯云 CLB（托管） |
| 自定义程度 | 高（Nginx 配置） | 中（Annotation） |
| 运维成本 | 自行运维 Nginx Pod | 腾讯云托管 |
| 性能 | 取决于 Node 性能 | CLB 性能规格 |
| 四层转发 | 支持 TCP/UDP | 不支持 |

## 前置条件

- [环境准备](../../../../环境准备.md)
- 熟悉 Kubernetes 相关概念，已了解 TKE 集群架构
- 集群状态 `Running`

```bash
# 检查集群状态
tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'
# expected: ClusterStatus "Running"
```

```json
{
  "TotalCount": "<TotalCount>",
  "Clusters": "<Clusters>",
  "ClusterId": "<ClusterId>",
  "ClusterName": "<ClusterName>",
  "ClusterDescription": "<ClusterDescription>",
  "ClusterVersion": "<ClusterVersion>"
}
```

- 需要 CAM 权限：`tke:DescribeClusters`、`tke:DescribeAddon`、`tke:InstallAddon`、`tke:UpdateAddon`、`tke:DeleteAddon`

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 查看集群信息 | `tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'` | 是 |
| 查看 NginxIngress 组件详情 | `tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName NginxIngress` | 是 |
| 安装 NginxIngress 组件 | `tccli tke InstallAddon --region <Region> --ClusterId <ClusterId> --AddonName NginxIngress` | 否 |
| 升级 NginxIngress 组件 | `tccli tke UpdateAddon --region <Region> --ClusterId <ClusterId> --AddonName NginxIngress --AddonVersion <AddonVersion>` | 否 |
| 卸载 NginxIngress 组件 | `tccli tke DeleteAddon --region <Region> --ClusterId <ClusterId> --AddonName NginxIngress` | 是 |
| 查看 Ingress | `kubectl get ingress` | 是 |
| 查看 NginxIngress Controller Pod | `kubectl get pods -n kube-system -l app.kubernetes.io/name=ingress-nginx` | 是 |

## 操作步骤

### 步骤 1：查看 NginxIngress 组件信息（控制面）

```bash
tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName NginxIngress \
    | jq '.Addons[0] | {AddonName, AddonVersion, Status}'
# expected: 返回 NginxIngress 组件详细信息
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

### 步骤 2：安装 NginxIngress 组件（控制面）

```bash
tccli tke InstallAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName NginxIngress
# expected: exit 0，组件安装请求已提交
```

### 步骤 3：查看 NginxIngress 运行状态（数据面）

```bash
kubectl get pods -n kube-system -l app.kubernetes.io/name=ingress-nginx
# expected: ingress-nginx-controller Pod Running
```

```text
NAME  STATUS  AGE
...
```

> **注意**：kubectl 数据面当前因公网端点被 CAM 策略 strategyId:240463971 拒绝而不可达。以下 kubectl 命令为参考格式，需在内网/VPN 环境执行。

### 步骤 4：卸载 NginxIngress 组件（控制面）

```bash
tccli tke DeleteAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName NginxIngress
# expected: exit 0，组件卸载请求已提交
```

## 验证

### 控制面（tccli）

```bash
tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName NginxIngress \
    | jq '.Addons[0] | {AddonName, Status, AddonVersion}'
# expected: Status "Running"
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

### 数据面（kubectl）

```bash
kubectl get pods -n kube-system -l app.kubernetes.io/name=ingress-nginx
# expected: ingress-nginx-controller Pod Running

kubectl get svc -n kube-system ingress-nginx-controller
# expected: LoadBalancer 类型 Service 存在
```

```text
NAME  STATUS  AGE
...
```

> **注意**：kubectl 数据面当前因公网端点被 CAM 策略 strategyId:240463971 拒绝而不可达。以下 kubectl 命令为参考格式，需在内网/VPN 环境执行。

## 清理

```bash
# 卸载组件
tccli tke DeleteAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName NginxIngress
# expected: exit 0

# 验证已卸载
tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName NginxIngress \
    | jq '.Addons[] | select(.AddonName == "NginxIngress")'
# expected: 空结果
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

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `InstallAddon` 返回 `FailedOperation.AddonAlreadyInstalled` | `tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId>` 检查已安装组件 | 组件已安装 | 无需重复安装；如需重装请先 `DeleteAddon` |
| 组件安装失败 | `tccli tke DescribeAddon --region <Region> --ClusterId <ClusterId> --AddonName NginxIngress` 查看 Status 与 reason | 集群版本不兼容或资源不足 | 检查集群版本要求；扩容节点后重试 |
| Ingress 规则不生效 | `kubectl get ingress`、`kubectl describe ingress <name>` 查看 Events | NginxIngress Controller Pod 异常或 Annotation 配置错误 | 检查 Controller Pod 状态；确认 Annotation 配置 |
| `Unable to connect to the server` | `kubectl cluster-info` 检查连通性 | 公网端点被 CAM 策略 strategyId:240463971 以 `tke:clusterExtranetEndpoint=true` 条件拒绝 | 通过 IOA/VPN 连接内网，或使用同 VPC CVM 执行 kubectl 命令 |

## 下一步

- [组件的生命周期管理](../组件的生命周期管理/tccli 操作.md)
- [扩展组件概述](../扩展组件概述/tccli 操作.md)
- [eNP（ebpf Network Policies）使用说明](../eNP（ebpf%20Network%20Policies）使用说明/tccli 操作.md)

## 控制台替代

[TKE 控制台 - 组件管理](https://console.cloud.tencent.com/tke2/cluster) → 集群 → 组件管理。
