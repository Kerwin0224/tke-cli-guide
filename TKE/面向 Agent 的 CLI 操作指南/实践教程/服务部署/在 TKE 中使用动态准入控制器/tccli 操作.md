# 在 TKE 中使用动态准入控制器（tccli）

> 对照官方：[在 TKE 中使用动态准入控制器](https://cloud.tencent.com/document/product/457/51238) · page_id `51238`

## 概述

动态准入控制器（Admission Webhook）通过 MutatingWebhookConfiguration 和 ValidatingWebhookConfiguration 实现在资源创建/更新时的拦截和修改。TKE 集群默认启用 admission webhook。

## 前置条件

- [环境准备](../../../环境准备.md)
- Webhook 服务已部署（TLS 证书可用）

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 创建 MutatingWebhook | `kubectl apply -f mutating-webhook.yaml` | 是 |
| 创建 ValidatingWebhook | `kubectl apply -f validating-webhook.yaml` | 是 |
| 查看 webhook | `kubectl get mutatingwebhookconfigurations,validatingwebhookconfigurations` | 是 |
| 测试拦截 | `kubectl apply -f test-pod.yaml` | 是 |

## 操作步骤

### 1. 部署 Webhook 服务

```yaml
apiVersion: v1
kind: Service
metadata:
  name: webhook-svc
  namespace: default
spec:
  ports:
  - port: 443
    targetPort: 8443
  selector:
    app: webhook
```

### 2. 创建 MutatingWebhookConfiguration

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: pod-mutator
webhooks:
- name: pod-mutator.example.com
  clientConfig:
    service:
      name: webhook-svc
      namespace: default
      path: /mutate
    caBundle: <base64-CA-cert>
  rules:
  - operations: ["CREATE", "UPDATE"]
    apiGroups: [""]
    apiVersions: ["v1"]
    resources: ["pods"]
  sideEffects: None
  admissionReviewVersions: ["v1"]
```

```bash
kubectl apply -f mutating-webhook.yaml
```

```text
# command executed successfully
```

```output
mutatingwebhookconfiguration.admissionregistration.k8s.io/pod-mutator created
```

### 3. 创建 ValidatingWebhookConfiguration

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: pod-validator
webhooks:
- name: pod-validator.example.com
  clientConfig:
    service:
      name: webhook-svc
      namespace: default
      path: /validate
    caBundle: <base64-CA-cert>
  rules:
  - operations: ["CREATE"]
    apiGroups: [""]
    apiVersions: ["v1"]
    resources: ["pods"]
  failurePolicy: Fail
  sideEffects: None
  admissionReviewVersions: ["v1"]
```

## 验证

```bash
kubectl get mutatingwebhookconfigurations pod-mutator
kubectl get validatingwebhookconfigurations pod-validator
```

```text
NAME  STATUS  AGE
...
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl` 连接超时/拒绝 | `tccli tke DescribeClusters` 返回集群 Running 但 kubectl 不可达 | CAM 策略 240463971 拒绝公网访问 APIServer 端点 | 通过 VPN/IOA 接入内网后执行 kubectl 命令 |
| Webhook 未拦截请求 | `kubectl get mutatingwebhookconfigurations,validatingwebhookconfigurations` 检查配置 | WebhookConfiguration 的 rules 未匹配目标资源 | 确认 rules 中的 apiGroups/apiVersions/resources 覆盖目标资源 |
| `connection refused` 调用 webhook | `kubectl get svc webhook-svc` + `kubectl get endpoints webhook-svc` | Webhook 服务未部署或 Endpoints 为空 | 确认 Webhook 服务已部署且 TLS 证书可用 |
| `no endpoints` 错误 | `kubectl describe mutatingwebhookconfiguration <name>` 查看 clientConfig | service 选中 Pod 的 label 不匹配 | 确认 Service selector 与 Pod label 一致 |

## 清理

```bash
kubectl delete mutatingwebhookconfiguration pod-mutator
kubectl delete validatingwebhookconfiguration pod-validator
```

## 下一步

- [工作负载平滑升级](../工作负载平滑升级/tccli%20操作.md)
- [应用高可用部署](../应用高可用部署/tccli%20操作.md)

## 控制台替代

无控制台界面，通过 kubectl 管理。
