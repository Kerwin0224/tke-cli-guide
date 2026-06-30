# Pod 优雅删除（tccli）

> 对照官方：[Pod 优雅删除](https://cloud.tencent.com/document/product/457/109977) · page_id `109977`

## 概述

在 CLB 直连场景下，当用户发起工作负载滚动更新时，会同时触发两个并行的工作流：
1. Service/Ingress 控制器调整 CLB 后端 Pod 的权重为 0，并后续删除该 Pod。
2. 运行时（kubelet）终止并删除 Pod。

在某些极端场景（如多 Service 复用同一 CLB、单 Service 下 Pod 数量非常多），可能出现某些 Pod 已被运行时删除，但 Service 控制器尚未完成 CLB 后端该 Pod 权重调整的情况，导致流量仍被发往已不存在的 Pod IP，引起业务异常。

**Pod 优雅删除特性**将两个并行工作流协调为串行：先让控制器完成 CLB 后端 Pod 权重调整为 0，再让运行时删除该 Pod。

## 前置条件

- [环境准备](../../../../环境准备.md)
- 已 [连接集群](../../../../集群配置/集群管理/连接集群/tccli%20操作.md)，kubectl 可正常访问 APIServer
- 仅适用于 [直连场景](../使用%20LoadBalancer%20直连%20Pod%20模式%20Service/tccli%20操作.md)
- 确保集群节点 kubelet 版本支持 Pod 删除保护功能
- Service/Ingress 控制器版本 >= v2.5.0
- 超级节点请提工单咨询确认

## 控制台与 CLI 参数映射

| 控制台操作 | CLI |
|-----------|-----|
| 查看集群 | `tccli tke DescribeClusters` |
| 查看组件版本 | `kubectl get configmap tke-service-controller-config -n kube-system -o yaml \| grep VERSION` |
| 开启 Service Pod 优雅删除（注解） | annotation `service.cloud.tencent.com/enable-grace-deletion: "true"` |
| 开启 Ingress Pod 优雅删除（注解） | annotation `ingress.cloud.tencent.com/enable-grace-deletion: "true"` |
| 全局开关（所有 Service/Ingress 默认开启） | ConfigMap `EnablePodGracefulDeletion: "true"` |
| 开启直连访问 | annotation `service.cloud.tencent.com/direct-access: "true"` 或 `ingress.cloud.tencent.com/direct-access: "true"` |

## 操作步骤

### 步骤1：确保版本满足要求

```bash
# 检查 service-controller 版本
kubectl get configmap tke-service-controller-config -n kube-system -o yaml | grep VERSION

# 检查 ingress-controller 版本
kubectl get configmap tke-ingress-controller-config -n kube-system -o yaml | grep VERSION
```

确保版本 >= v2.5.0。

### 步骤2：开启 Pod 优雅删除

#### 方式一：使用注解（单 Service/Ingress 级别）

**Service YAML 示例：**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: anyserver
  annotations:
    service.cloud.tencent.com/direct-access: "true"
    service.cloud.tencent.com/enable-grace-deletion: "true"  # 开启 Pod 优雅删除
spec:
  ports:
  - name: nginx
    port: 80
    targetPort: 80
  selector:
    app: anyserver
  type: NodePort
```

```bash
kubectl apply -f anyserver-service-grace-deletion.yaml
```

**Ingress YAML 示例：**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: anyserver
  annotations:
    ingress.cloud.tencent.com/direct-access: "true"
    ingress.cloud.tencent.com/enable-grace-deletion: "true"
spec:
  rules:
  - host: example.io
    http:
      paths:
      - backend:
          service:
            name: anyserver
            port:
              number: 80
        path: /
        pathType: ImplementationSpecific
```

```bash
kubectl apply -f anyserver-ingress-grace-deletion.yaml
```

#### 方式二：使用全局开关（集群级别）

修改 `kube-system/tke-service-controller-config` ConfigMap：

```yaml
apiVersion: v1
data:
  EnablePodGracefulDeletion: "true"
  VERSION: v2.7.0
kind: ConfigMap
metadata:
  name: tke-service-controller-config
  namespace: kube-system
```

```bash
kubectl patch configmap tke-service-controller-config \
  -n kube-system \
  --type merge \
  --patch '{"data":{"EnablePodGracefulDeletion":"true"}}'
```

> **注意**：全局开关的优先级低于注解的优先级。当手动在特定 Service/Ingress 上设置 Pod 优雅删除注解时，全局开关不生效。

## 验证

### 数据面（kubectl）

```bash
# 查看 Service 注解
kubectl get service anyserver -o yaml | grep enable-grace-deletion

# 查看 Ingress 注解
kubectl get ingress anyserver -o yaml | grep enable-grace-deletion

# 查看全局开关
kubectl get configmap tke-service-controller-config -n kube-system -o yaml | grep EnablePodGracefulDeletion
```

```text
NAME  STATUS  AGE
...
```

## 清理

### 数据面（kubectl）

```bash
kubectl delete service anyserver --ignore-not-found
kubectl delete ingress anyserver --ignore-not-found

# 如需关闭全局开关
kubectl patch configmap tke-service-controller-config \
  -n kube-system \
  --type merge \
  --patch '{"data":{"EnablePodGracefulDeletion":"false"}}'
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `Unable to connect to the server` | `kubectl cluster-info` | 公网端点被 CAM 策略 strategyId:240463971 以 `tke:clusterExtranetEndpoint=true` 条件拒绝 | 通过 IOA/VPN 连接内网，或使用同 VPC CVM 执行 kubectl |
| 功能不生效 | `kubectl describe <resource>` | kubelet 版本满足要求； controller 版本 >= v2.5.0 | 确认 kubelet 版本满足要求；确认 controller 版本 >= v2.5.0 |
| 仅注解模式不生效 | `direct-access: "true"` | 已同时设置 `direct-access: "true"` 和 `enable-grace-deletion: "... | 确认已同时设置 `direct-access: "true"` 和 `enable-grace-deletion: "true"` |
| 全局开关被覆盖 | `kubectl describe <resource>` | 全局开关优先级低于注解 | 全局开关优先级低于注解，当 Service 有显式注解时以注解为准 |
| 超级节点不生效 | `kubectl describe <resource>` | 超级节点需提工单咨询 | 超级节点需提工单咨询确认 |
| 非直连场景不生效 | `kubectl describe <resource>` | 该功能仅适用于直连场景 | 该功能仅适用于直连场景 |

## 下一步

- [使用 LoadBalancer 直连 Pod 模式 Service](../使用%20LoadBalancer%20直连%20Pod%20模式%20Service/tccli%20操作.md)
- [Service 优雅停机](../Service%20优雅停机/tccli%20操作.md)
- [Node 优雅下线](../Node%20优雅下线/tccli%20操作.md)
- [全局开关配置说明](../全局开关配置说明/tccli%20操作.md)

## 控制台替代

[控制台 → 集群 → 服务与路由 → Service](https://console.cloud.tencent.com/tke2/cluster) 编辑 Annotation，添加 `service.cloud.tencent.com/enable-grace-deletion: "true"`。
