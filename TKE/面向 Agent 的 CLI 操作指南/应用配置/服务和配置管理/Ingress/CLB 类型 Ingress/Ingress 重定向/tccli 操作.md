# Ingress 重定向（tccli）

> 对照官方：[Ingress 重定向](https://cloud.tencent.com/document/product/457/59096) · page_id `59096`

## 概述

通过 CLB Ingress 的注解，支持 HTTP→HTTPS 自动重定向及自定义 URL 间的手动重定向，适用于网站 HTTP/HTTPS 双栈接入、域名迁移等场景。

## 前置条件

- [环境准备](../../../../../环境准备.md)
- 已 [连接集群](../../../../../集群配置/集群管理/连接集群/tccli 操作.md)，kubectl 可正常访问 APIServer

## 控制台与 CLI 参数映射

| Console 操作 | CLI 等价操作（Annotation） | 幂等 | 说明 |
|---|---|---|---|------|
| 无重定向 | 不配置重定向相关注解 | 默认行为 | 默认行为 |
| 自动重定向 | `ingress.cloud.tencent.com/auto-rewrite: "true"` + `rewrite-support: "true"` | 是(apply) | HTTP 自动跳转 HTTPS |
| 手动重定向 | `ingress.cloud.tencent.com/rewrite-support: "true"` + 在规则中添加 `rewrite` 字段 | 是(apply) | 自定义来源与目标 |
| 自定义重定向码 | `ingress.cloud.tencent.com/auto-rewrite-code` | 是(apply) | 需配合自动重定向 |

## 操作步骤

> **注意：**
> - 若没有重定向注解声明，可在 CLB 控制台配置重定向规则，但 TKE 不会管理
> - 若使用 TKE 重定向注解，CLB 下所有重定向规则由 TKE 管理，CLB 控制台修改会被覆盖
> - 基于 CLB 的 TKE Ingress 不支持 nginx-ingress 的 `rewrite-target`（路径重写）
> - A 已重定向至 B 后，A 不能再重定向至 C（除非先删除旧关系）；B 不能重定向至其他地址；A 不能重定向到 A
> - 关闭重定向功能需显式设置注解为 `"false"`，不能直接删除注解

### 自动重定向：HTTP → HTTPS

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    ingress.cloud.tencent.com/rewrite-support: "true"
    ingress.cloud.tencent.com/auto-rewrite: "true"
  name: my-ingress
  namespace: default
spec:
  rules:
    - host: <domain>
      http:
        paths:
          - backend:
              service:
                name: <service-name>
                port:
                  number: 443
            path: /
            pathType: ImplementationSpecific
  tls:
    - hosts:
      - <domain>
      secretName: <cert-secret>
```

配置后所有 HTTPS 监听器中的转发规则都将被对应到 HTTP:80 监听器作为重定向规则，域名与路径保持一致。

### 手动重定向

手动重定向在 `http-rules` 或 `https-rules` 的规则中增加 `rewrite` 字段：

```json
{
  "host": "<domain>",
  "path": "<path>",
  "backend": {
    "serviceName": "<service name>",
    "servicePort": "<service port>"
  },
  "rewrite": {
    "port": "<rewrite-port>",
    "host": "<rewrite-domain>",
    "path": "<rewrite-path>"
  }
}
```

#### 示例：路径迁移 `/v2/path2` → `/path2`

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    ingress.cloud.tencent.com/rewrite-support: "true"
    kubernetes.io/ingress.class: qcloud
    kubernetes.io/ingress.http-rules: '[{"path":"/path1","backend":{"serviceName":"path1","servicePort":"80"}},{"path":"/path2","backend":{"serviceName":"path2","servicePort":"80"}},{"path":"/v1/path1","rewrite":{"port":80,"path":"/path1"}},{"path":"/v2/path2","rewrite":{"port":80,"path":"/path2"}}]'
    kubernetes.io/ingress.https-rules: "null"
    kubernetes.io/ingress.rule-mix: "true"
  name: test
  namespace: default
spec:
  rules:
    - http:
        paths:
          - backend:
              service:
                name: path1
                port:
                  number: 80
            path: /path1
            pathType: ImplementationSpecific
    - http:
        paths:
          - backend:
              service:
                name: path2
                port:
                  number: 80
            path: /path2
            pathType: ImplementationSpecific
```

```bash
kubectl create -f redirect-ingress.yaml
```

## 验证

### 数据面（kubectl）

```bash
kubectl get ingress test
kubectl describe ingress test
```

```text
NAME  STATUS  AGE
...
```

确认 annotations 中包含重定向规则，并确保 CLB 监听器上已创建对应重定向策略。

## 清理

### 数据面（kubectl）

```bash
kubectl delete ingress test
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `Unable to connect to the server` | `kubectl cluster-info` | 公网端点被 CAM 策略 strategyId:240463971 以 `tke:clusterExtranetEndpoint=true` 条件拒绝 | 通过 IOA/VPN 连接内网，或使用同 VPC CVM 执行 kubectl |
| 重定向不生效 | `rewrite-support: "true"` | `rewrite-support: "true"` 已设置 | 确认 `rewrite-support: "true"` 已设置 |
| 关闭重定向后仍生效 | `"false"` | 需显式设置注解为 `"false"` 而非删除 | 需显式设置注解为 `"false"` 而非删除 |
| CLB 重定向规则被覆盖 | `tccli clb DescribeLoadBalancers` | TKE 管理的重定向会在 CLB 控制台覆盖手动修改 | TKE 管理的重定向会在 CLB 控制台覆盖手动修改 |
| 转发配置 B 无法删除 | `kubectl describe <resource>` | 先删除指向 B 的重定向规则再删除 B | 先删除指向 B 的重定向规则再删除 B |

## 下一步

- [Ingress 混合使用 HTTP 及 HTTPS 协议](../Ingress%20混合使用%20HTTP%20及%20HTTPS%20协议/tccli 操作.md)
- [Ingress Annotation 说明](../Ingress%20Annotation%20说明/tccli 操作.md)

## 控制台替代

在控制台 **新建 Ingress > 转发配置 > 重定向** 中选择"手动"或"自动"模式。
