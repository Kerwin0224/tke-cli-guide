# 安装 NginxIngress 实例（tccli）

> 对照官方：[安装 NginxIngress 实例](https://cloud.tencent.com/document/product/457/50503) · page_id `50503`

## 概述

> **注意：** NginxIngress 扩展组件已停止更新，详情见 [NginxIngress 扩展组件停止更新公告](https://cloud.tencent.com/document/product/457/108517)。

NginxIngress 是基于开源 Ingress-Nginx Controller 的 TKE 扩展组件，支持 DaemonSet 节点池部署、Deployment + HPA 部署、及 CLB 直通等多种安装方案。

## 前置条件

- [环境准备](../../../../../环境准备.md)
- 已 [连接集群](../../../../../集群配置/集群管理/连接集群/tccli 操作.md)，kubectl 可正常访问 APIServer

## 控制台与 CLI 参数映射

| Console 操作 | CLI 等价操作 | 幂等 | 说明 |
|---|---|---|---|------|
| 安装 NginxIngress 组件 | 控制台 **组件管理 > 新建 > NginxIngress** | 否 | 集群插件安装，无直接 kubectl 等价 |
| 新增 NginxIngress 实例 | 控制台 **服务与路由 > NginxIngress > 新增** | 否 | 通过 CRD NginxIngress 创建 |
| 编辑 Nginx 配置 | `kubectl edit configmap <nginx-instance-config>` | 否 | ConfigMap 编辑 |
| 查看实例 YAML | `kubectl get nginxingress <name> -o yaml` | 是 | 查看 CRD 资源 |

## 操作步骤

### 方案 1：DaemonSet 节点池部署（推荐）

通过 TKE 控制台在 **服务与路由 > NginxIngress > 新增 NginxIngress 实例** 中选择"指定节点池 DaemonSet 部署"。需准备专用节点池并设置污点。

### 方案 2：Deployment + HPA 部署

在 TKE 控制台选择"自定义 Deployment+HPA 部署"，支持 HPA 弹性伸缩。

如需排除注册节点：

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: node.kubernetes.io/instance-type
            operator: NotIn
            values:
            - external
```

### 方案 3：Nginx 前端接入 LB

| 集群网络模式 | 推荐方案 |
|---|---|
| VPC-CNI | CLB 直通 Nginx Service（推荐，性能最优） |
| GlobalRouter | LoadBalancer 类型 Service（经过 NodePort，多一层转发） |
| HostNetwork | 手动创建 CLB 绑定节点端口 |

### Nginx 配置参数（ConfigMap）

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: <nginx-instance-name>-nginx-controller
  namespace: kube-system
data:
  keep-alive-requests: "10000"
  max-worker-connections: "65536"
  upstream-keepalive-connections: "200"
```

> **注意：** 请勿修改 `access-log-path`、`error-log-path`、`log-format-upstream`，否则会影响 CLS 日志采集。

### 组件与 K8s 版本兼容性

| K8s 版本 | 支持组件版本 | Nginx 实例镜像版本 |
|---|---|---|
| ≤ 1.18 | 1.1.0 ~ 1.5.1 | v0.41.0, v0.49.3 |
| 1.20 | 1.1.0 ~ 1.5.1 | v1.1.3 |
| 1.22 | 1.1.0 ~ 1.5.1 | v1.1.3 |
| 1.24 | 1.1.0 ~ 1.5.1 | v1.1.3, v1.6.4 |
| 1.26 | 1.3.0 ~ 1.5.1 | v1.1.3, v1.6.4, v1.9.5 |
| 1.28 | 1.5.1 | v1.9.5 |

## 验证

### 数据面（kubectl）

```bash
kubectl get pods -l app=nginx-ingress -n kube-system
kubectl get nginxingress -A
```

```text
NAME  STATUS  AGE
...
```

## 清理

### 数据面（kubectl）

通过控制台 **服务与路由 > NginxIngress** 删除实例，再在 **组件管理** 中卸载组件。

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `Unable to connect to the server` | `kubectl cluster-info` | 公网端点被 CAM 策略 strategyId:240463971 以 `tke:clusterExtranetEndpoint=true` 条件拒绝 | 通过 IOA/VPN 连接内网，或使用同 VPC CVM 执行 kubectl |
| HPA 不生效 | `kubectl get hpa` | 已通过 HPA 开关开启（禁用后系统会重新下发 HPA） | 确保已通过 HPA 开关开启（禁用后系统会重新下发 HPA） |
| Nginx 调度到注册节点 | `node.kubernetes.io/instance-type: external` | 使用节点亲和性排除 `node.kubernetes.io/instance-type: external` | 使用节点亲和性排除 `node.kubernetes.io/instance-type: external` |
| 日志采集异常 | `access-log-path` | 修改 ConfigMap 中 `access-log-path` 等日志字段 | 请勿修改 ConfigMap 中 `access-log-path` 等日志字段 |

## 下一步

- [使用 NginxIngress 对象接入集群外部流量](../使用%20NginxIngress%20对象接入集群外部流量/tccli 操作.md)
- [NginxIngress 日志配置](../NginxIngress%20日志配置/tccli 操作.md)
- [Ingress Controllers 说明](../../Ingress%20Controllers%20说明/tccli 操作.md)

## 控制台替代

在控制台 **组件管理 > 新建 > NginxIngress** 安装组件，再在 **服务与路由 > NginxIngress > 新增** 创建实例。
