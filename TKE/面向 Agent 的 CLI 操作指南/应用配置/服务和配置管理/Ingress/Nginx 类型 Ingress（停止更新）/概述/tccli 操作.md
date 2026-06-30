# Nginx 类型 Ingress 概述（tccli）

> 对照官方：[概述](https://cloud.tencent.com/document/product/457/50502) · page_id `50502`

## 概述

> **重要提示**：TKE 提供的 NginxIngress 扩展组件已停止更新和维护，但 TKE 与开源 Ingress-Nginx Controller 的兼容性不受影响。用户可通过自建部署或 TKE 应用市场继续使用。TKE 仅提供安装能力，不提供可用性保障。

NginxIngress 是以 Nginx 作为反向代理和负载均衡器的 Kubernetes Ingress 控制器。Nginx 可担任反向代理、负载均衡器和 HTTP 缓存角色。TKE 提供了产品化能力在集群内安装和使用 NginxIngress。

## NginxIngress 术语

| 术语 | 说明 |
|------|------|
| NginxIngress 组件 | TKE 中使用 NginxIngress 的入口，从集群组件页一键安装部署 |
| NginxIngress 实例 | 一组 CRD 对应一个实例，一个集群可部署多个实例（如公网、内网各一个）。创建实例自动创建 nginx-ingress-controller、Service、ConfigMap 等资源 |
| nginx-ingress-controller | 实际的 Nginx 负载，监控 Kubernetes Ingress 对象变更并更新 `nginx.conf` |

## 相关资源

- NginxIngress GitHub：https://github.com/kubernetes/ingress-nginx
- 官方文档：https://kubernetes.github.io/ingress-nginx/
- Annotations 参考：https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/
- ConfigMap 参考：https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/configmap/

## 与 CLB 类型 Ingress 的对比

参见 [Ingress Controllers 说明](../../Ingress Controllers 说明/tccli 操作.md) 中的功能对比表。核心差异：
- Nginx Ingress 支持更多协议（http2, grpc, tcp, udp）
- Nginx Ingress 支持 IP 收敛（多 Ingress 共用一个 CLB IP）
- Nginx Ingress 支持更丰富的特征路由（header, cookie）
- Nginx Ingress 支持流量行为控制（重定向、重写）
- Nginx Ingress 支持认证授权
- Nginx Ingress 需要自行运维组件（控制面 + 数据面）

## 前置条件

- [环境准备](../../../../../环境准备.md)：`tccli` 已配置，`kubectl` ≥ v1.28
- 已获取集群 kubeconfig：

```bash
tccli tke DescribeClusterKubeconfig --region <Region> --ClusterId CLUSTER_ID
# expected: 返回 Kubeconfig 配置
```
```json
{
    "RequestId": "00000000-0000-0000-0000-000000000000",
    ...
}
```


- 需要 CAM 权限：`tke:DescribeClusters`、`tke:DescribeClusterKubeconfig`


## 控制台与 CLI 参数映射

| 控制台操作 | kubectl/tccli 命令 | 幂等 |
|-----------|-------------------|:--:|
| 查看资源列表 | `kubectl get RESOURCE` | 是 |
| 创建资源 | `kubectl apply -f resource.yaml` | 否 |
| 删除资源 | `kubectl delete RESOURCE NAME` | 是 |

## 操作步骤

本页面为概念性说明，无操作步骤。具体操作指引请参考子页面 [安装 NginxIngress 实例](../安装 NginxIngress 实例/tccli 操作.md)、[使用 NginxIngress 对象接入集群外部流量](../使用 NginxIngress 对象接入集群外部流量/tccli 操作.md) 等。

## 验证

不适用（概述页面）。

## 清理

不适用（概述页面）。

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `Unable to connect to the server` | `kubectl cluster-info` | 公网端点被 CAM 策略 strategyId:240463971 以 `tke:clusterExtranetEndpoint=true` 条件拒绝 | 通过 IOA/VPN 连接内网，或使用同 VPC CVM 执行 kubectl |

## 下一步

- [安装 NginxIngress 实例](../安装 NginxIngress 实例/tccli 操作.md)
- [使用 NginxIngress 对象接入集群外部流量](../使用 NginxIngress 对象接入集群外部流量/tccli 操作.md)
- [NginxIngress 日志配置](../NginxIngress 日志配置/tccli 操作.md)
- [通过 Terraform 安装 Nginx 插件和实例](../通过 Terraform 安装 Nginx 插件和实例/tccli 操作.md)
- [Ingress Controllers 说明](../../Ingress Controllers 说明/tccli 操作.md)

## 控制台替代

控制台入口：集群详情 > 组件管理 > 新建 > NginxIngress
