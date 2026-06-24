# Ingress Annotation 说明（tccli）

> 对照官方：[Ingress Annotation 说明](https://cloud.tencent.com/document/product/457/56112) · page_id `56112`

## 概述

TKE CLB 类型 Ingress 通过 Annotations 扩展了原生 Kubernetes Ingress 的能力，支持指定 Ingress 类型、复用已有 CLB、创建内网 CLB、配置付费类型、安全组、重定向、混合协议等功能。

## 前置条件

- [环境准备](../../../../../环境准备.md)
- 已 [连接集群](../../../../../集群配置/集群管理/连接集群/tccli 操作.md)，kubectl 可正常访问 APIServer

## 控制台与 CLI 参数映射

| Console 操作 | Annotation | 类型 | 幂等 | 版本要求 |
|---|---|---|---|---|---|
| 指定 Ingress 类型 | `kubernetes.io/ingress.class` | string | 否(首次) | ≥ v1.0.0 |
| 使用已有 CLB | `kubernetes.io/ingress.existLbId` | string | 否(首次) | ≥ v1.0.0 |
| 创建内网 CLB | `kubernetes.io/ingress.subnetId` | string | 否(首次) | ≥ v1.0.0 |
| 指定付费类型 | `kubernetes.io/ingress.internetChargeType` | enum | 是(apply) | ≥ v1.0.0 |
| 公网带宽上限 | `kubernetes.io/ingress.internetMaxBandwidthOut` | int | 是(apply) | ≥ v1.0.0 |
| CLB 拓展参数 | `kubernetes.io/ingress.extensiveParameters` | json | 否(首次) | ≥ v1.0.0 |
| 安全组放通 | `ingress.cloud.tencent.com/pass-to-target` | bool | 是(apply) | ≥ v1.8.3 |
| 绑定安全组 | `ingress.cloud.tencent.com/security-groups` | string | 是(apply) | ≥ v1.8.3 |
| 修改保护 | `ingress.cloud.tencent.com/modification-protection` | bool | 是(apply) | ≥ v1.7.3 |
| 保留 CLB | `ingress.cloud.tencent.com/loadbalance-retain` | bool | 是(apply) | ≥ v2.9.0 |
| 自定义监听端口 | `ingress.cloud.tencent.com/listen-ports` | json | 否(首次) | ≥ v2.4.1 |
| 拓展配置 | `ingress.cloud.tencent.com/tke-service-config` | string | 是(apply) | ≥ v1.3.0 |
| 自动拓展配置 | `ingress.cloud.tencent.com/tke-service-config-auto` | bool | 是(apply) | ≥ v1.3.0 |
| 自动重定向 | `ingress.cloud.tencent.com/auto-rewrite` | bool | 是(apply) | ≥ v1.3.0 |
| 重定向支持 | `ingress.cloud.tencent.com/rewrite-support` | bool | 是(apply) | ≥ v1.3.0 |
| 重定向码 | `ingress.cloud.tencent.com/auto-rewrite-code` | json | 是(apply) | ≥ v2.11.0 |
| 混合协议 | `kubernetes.io/ingress.rule-mix` | bool | 是(apply) | ≥ v1.3.0 |
| 混合协议全映射 | `kubernetes.io/ingress.rule-mix-both` | bool | 是(apply) | ≥ v2.8.0 |
| HTTP 规则 | `kubernetes.io/ingress.http-rules` | json | ≥ v1.3.0 |
| HTTPS 规则 | `kubernetes.io/ingress.https-rules` | json | ≥ v1.3.0 |
| 开启直连 | `ingress.cloud.tencent.com/direct-access` | bool | ≥ v1.3.0 |
| 优雅停机 | `ingress.cloud.tencent.com/enable-grace-shutdown` | bool | ≥ v1.5.0 |
| 优雅停机 tkex | `ingress.cloud.tencent.com/enable-grace-shutdown-tkex` | bool | ≥ v1.5.0 |
| 优雅删除 | `ingress.cloud.tencent.com/enable-grace-deletion` | bool | ≥ v2.4.0 |
| 自定义后端权重 | `ingress.cloud.tencent.com/lb-rs-weight` | json | ≥ v1.6.0 |
| 后端 IP 版本偏好 | `ingress.cloud.tencent.com/mix-target-prefer` | enum | ≥ v2.10.0 |
| 删除保护 | `ingress.cloud.tencent.com/deletion-protection` | bool | ≥ v2.5.3 |
| 注解配置证书 | `ingress.cloud.tencent.com/certificate` | json | ≥ v2.8.0 |
| 多 Ingress 复用 CLB | `ingress.cloud.tencent.com/enable-group` | bool | ≥ v2.10.0 |
| 同步模式 | `ingress.cloud.tencent.com/mode` | enum | ≥ v2.10.0 (skip) |

> 查看当前集群 ingress-controller 版本：
> ```bash
> kubectl -n kube-system get cm tke-ingress-controller-config -o jsonpath='{.data.VERSION}'
> ```

## 操作步骤

以下列出各 Annotation 的使用示例。

### 基础配置

#### 指定 Ingress 类型

```yaml
annotations:
  kubernetes.io/ingress.class: "qcloud"
```

可选值：`qcloud`（CLB 类型）、`nginx`（nginx-ingress）、`traefik`。

#### 只读注解 - 负载均衡 ID

```yaml
# 只读，由组件提供当前 Ingress 引用的 CLB ID
kubernetes.io/ingress.qcloud-loadbalance-id
```

### CLB 管理

#### 使用已有 CLB

```yaml
annotations:
  kubernetes.io/ingress.existLbId: "lb-xxxxxxxx"
```

#### 创建内网 CLB

```yaml
annotations:
  kubernetes.io/ingress.subnetId: "subnet-xxxxxxxx"
```

#### 付费类型 + 带宽

```yaml
annotations:
  kubernetes.io/ingress.internetChargeType: "BANDWIDTH_POSTPAID_BY_HOUR"
  kubernetes.io/ingress.internetMaxBandwidthOut: "10"
```

| 付费类型 | 说明 |
|---|---|
| `BANDWIDTH_POSTPAID_BY_HOUR` | 按带宽按小时后计费 |
| `TRAFFIC_POSTPAID_BY_HOUR` | 按流量按小时后计费 |

> **注意：** 付费类型和带宽仅在创建时配置生效，创建后修改无效。

#### 拓展参数（IP 版本、规格、运营商）

```yaml
annotations:
  kubernetes.io/ingress.extensiveParameters: '{"VipIsp":"CTCC","AddressIPVersion":"IPV4","SlaType":"clb.c2.medium","InternetAccessible":{"InternetChargeType":"TRAFFIC_POSTPAID_BY_HOUR","InternetMaxBandwidthOut":10}}'
```

#### 同步模式 skip

```yaml
annotations:
  ingress.cloud.tencent.com/mode: "skip"
```

skip 模式下删除资源时不删除 CLB 实例、监听器，不解绑 RS。限制：不支持复用 CLB、不支持多 Ingress 复用。

### 安全组

#### 安全组默认放通

```yaml
annotations:
  ingress.cloud.tencent.com/pass-to-target: "true"
```

#### 绑定安全组

```yaml
# 绑定
ingress.cloud.tencent.com/security-groups: "sg-xxxxx,sg-yyyyy,sg-zzzzz"
# 解绑部分
ingress.cloud.tencent.com/security-groups: "sg-xxxxx"
# 解绑全部（≥ v2.5.2）
ingress.cloud.tencent.com/security-groups: ""
```

单个 CLB 最多绑定 5 个安全组。

### 保护与保留

#### 修改保护

```yaml
annotations:
  ingress.cloud.tencent.com/modification-protection: "true"
```

#### 删除保护

```yaml
annotations:
  ingress.cloud.tencent.com/deletion-protection: "true"
```

#### 保留 CLB

```yaml
annotations:
  ingress.cloud.tencent.com/loadbalance-retain: "true"
```

> **注意：** 使用前需关闭 CLB 配置保护。

### 监听端口

#### 自定义监听端口

```yaml
annotations:
  ingress.cloud.tencent.com/listen-ports: '[{"HTTP": 80}, {"HTTPS": 443}, {"HTTP": 8080}, {"HTTPS": 8443}]'
```

> 自动重定向仅在 HTTP:80 和 HTTPS:443 均存在时生效。

### 拓展配置

#### 引用 TkeServiceConfig

```yaml
annotations:
  ingress.cloud.tencent.com/tke-service-config: "config-name"
```

#### 自动生成 TkeServiceConfig

```yaml
annotations:
  ingress.cloud.tencent.com/tke-service-config-auto: "true"
```

### 重定向

#### 自动重定向

```yaml
annotations:
  ingress.cloud.tencent.com/auto-rewrite: "true"
  ingress.cloud.tencent.com/rewrite-support: "true"
```

关闭重定向需显式设为 `"false"`，不能直接删除注解。

#### 设置重定向码

```yaml
annotations:
  ingress.cloud.tencent.com/auto-rewrite-code: '{"defaultRewriteCode": 302, "rewriteCodeInfos": [{"host": "<domain>","rewriteCode": 307}]}'
  ingress.cloud.tencent.com/auto-rewrite: "true"
```

### 协议混合

#### 混合协议

```yaml
annotations:
  kubernetes.io/ingress.rule-mix: "true"
  kubernetes.io/ingress.http-rules: '[{"host":"<domain>","path":"/","backend":{"serviceName":"example","servicePort":"80"}}]'
  kubernetes.io/ingress.https-rules: '[{"host":"<domain>","path":"/","backend":{"serviceName":"example","servicePort":"80"}}]'
```

#### 混合协议全映射

```yaml
annotations:
  kubernetes.io/ingress.rule-mix-both: "true"
```

> 配置 `rule-mix-both` 后不能再配置 `rule-mix`、`rewrite-support` 或 `auto-rewrite`。

### 直连与优雅停机

#### 直连 Pod

```yaml
annotations:
  ingress.cloud.tencent.com/direct-access: "true"
```

#### 优雅停机

```yaml
annotations:
  ingress.cloud.tencent.com/enable-grace-shutdown: "true"
  ingress.cloud.tencent.com/direct-access: "true"
```

> v2.2.0 起废弃，默认开启。仅在直连模式下支持。

#### 优雅停机 tkex

```yaml
annotations:
  ingress.cloud.tencent.com/enable-grace-shutdown-tkex: "true"
```

> v2.2.0 起废弃，默认开启。

#### 优雅删除

```yaml
annotations:
  ingress.cloud.tencent.com/enable-grace-deletion: "true"
```

### 其他

#### 自定义后端权重

```yaml
annotations:
  ingress.cloud.tencent.com/lb-rs-weight: '{"defaultWeight":10,"groups":[{"key":{"proto":"TCP","port":80},"statefulSets":[{"name":"sts-v1","weights":[{"weight":0,"podIndexes":[0]}]}]}]}'
```

#### 后端 IP 版本偏好

```yaml
annotations:
  ingress.cloud.tencent.com/mix-target-prefer: "ipv4"
```

可选值：`ipv4`、`ipv6`（默认）。

#### 注解配置证书（替代 spec.tls）

```yaml
annotations:
  ingress.cloud.tencent.com/certificate: '[{"hosts": ["a.tencent.com"],"qcloud_cert_id":["certA"]},{"hosts": ["b.tencent.com"],"qcloud_cert_id":["certB"]}]'
```

> 不能和 `spec.tls` 一起使用。更换证书需手动更新注解中的证书 ID。

#### 多 Ingress 复用 CLB

```yaml
annotations:
  ingress.cloud.tencent.com/enable-group: "true"
```

> 需在创建时与 `existLbId` 同时指定。

## 验证

### 数据面（kubectl）

```bash
kubectl get ingress <name> -o yaml
kubectl describe ingress <name>
```

```text
NAME  STATUS  AGE
...
```

## 清理

不适用（参考说明页面，不涉及资源创建）。

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `Unable to connect to the server` | `kubectl cluster-info` | 公网端点被 CAM 策略 strategyId:240463971 以 `tke:clusterExtranetEndpoint=true` 条件拒绝 | 通过 IOA/VPN 连接内网，或使用同 VPC CVM 执行 kubectl |
| Annotation 不生效 | `kubectl -n kube-system get cm tke-ingress-controller-config -o jsonpath='{.data.VERSION}'` 检查版本 | ingress-controller 版本不满足该 Annotation 的版本要求 | 升级 ingress-controller 至要求版本 |
| 创建时注解无效 | `kubectl get ingress <name> -o yaml` 确认注解 | 部分注解（如付费类型、带宽、拓展参数）仅在创建时生效 | 删除 Ingress 重建，创建时即指定注解 |
| skip 模式限制 | `kubectl get ingress <name> -o yaml` 确认 `mode: skip` | skip 模式不支持复用 CLB 和多 Ingress 复用 | 移除 `mode: skip` 注解或避免复用场景 |
| 解绑安全组不生效 | `kubectl -n kube-system get cm tke-ingress-controller-config -o jsonpath='{.data.VERSION}'` 检查版本 | ingress-controller 版本 < v2.5.2 | 升级组件至 ≥ v2.5.2 |

## 下一步

- [Ingress 基本功能](../Ingress%20基本功能/tccli 操作.md)
- [Ingress 重定向](../Ingress%20重定向/tccli 操作.md)
- [多 Ingress 复用 CLB](../多%20Ingress%20复用%20CLB/tccli 操作.md)

## 控制台替代

控制台可通过表单配置部分 Annotation 对应的功能，完整 Annotation 需通过 YAML 编辑。
