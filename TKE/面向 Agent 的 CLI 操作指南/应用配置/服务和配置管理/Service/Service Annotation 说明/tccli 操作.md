# Service Annotation 说明（tccli）

> 对照官方：[Service Annotation 说明](https://cloud.tencent.com/document/product/457/51258) · page_id `51258`

## 概述

TKE LoadBalancer Service 支持丰富的 Annotation 控制 CLB 行为，包括带宽、健康检查、会话保持、HTTPS 证书、直连 Pod 等。

## 关键 Annotation

| Annotation | 说明 | 示例 |
|-----------|------|------|
| `service.kubernetes.io/qcloud-loadbalancer-internal-subnetid` | 内网 CLB 子网 | `subnet-xxxxx` |
| `service.kubernetes.io/tke-existed-lbid` | 绑定已有 CLB | `lb-xxxxx` |
| `service.kubernetes.io/qcloud-loadbalancer-internet-charge-type` | 计费方式 | `TRAFFIC_POSTPAID_BY_HOUR` |
| `service.kubernetes.io/qcloud-loadbalancer-internet-max-bandwidth-out` | 带宽上限(Mbps) | `10` |
| `service.cloud.tencent.com/direct-access` | 直连 Pod 模式 | `true` |
| `service.cloud.tencent.com/local-svc-only-bind-node-with-pod` | 仅绑定有 Pod 的节点 | `true` |
| `service.cloud.tencent.com/tke-service-config` | 引用 TkeServiceConfig | `nginx-config` |
| `service.cloud.tencent.com/loadbalance-id` | 指定 CLB ID | `lb-xxxxx` |
| `service.kubernetes.io/loadbalance-id` | 复用已有 CLB（公网） | `lb-xxxxx` |

## 使用示例

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
  annotations:
    service.kubernetes.io/qcloud-loadbalancer-internet-charge-type: "TRAFFIC_POSTPAID_BY_HOUR"
    service.kubernetes.io/qcloud-loadbalancer-internet-max-bandwidth-out: "10"
    service.cloud.tencent.com/direct-access: "true"
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
```

## 前置条件

- [环境准备](../../../../环境准备.md)：`tccli` 已配置，`kubectl` ≥ v1.28
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

本页面为 Annotation 参考说明。各 Annotation 的具体使用示例请参见上方"关键 Annotation"和"使用示例"章节，完整操作指引请参考 [Service 基本功能](../Service 基本功能/tccli 操作.md)、[Service 负载均衡配置](../Service 负载均衡配置/tccli 操作.md)。

## 验证

不适用（参考说明页面）。

## 清理

不适用（参考说明页面）。

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `Unable to connect to the server` | `kubectl cluster-info` | 公网端点被 CAM 策略 strategyId:240463971 以 `tke:clusterExtranetEndpoint=true` 条件拒绝 | 通过 IOA/VPN 连接内网，或使用同 VPC CVM 执行 kubectl |

## 下一步

- [Service 基本功能](../Service 基本功能/tccli 操作.md)
- [Service 负载均衡配置](../Service 负载均衡配置/tccli 操作.md)
- [Ingress Annotation 说明](../../Ingress/CLB 类型 Ingress/Ingress Annotation 说明/tccli 操作.md)

## 控制台替代

控制台详情参见 [Service Annotation 说明（控制台）](https://cloud.tencent.com/document/product/457/51258)
