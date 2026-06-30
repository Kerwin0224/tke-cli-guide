# CLB 目标组（tccli）

> 对照官方：[CLB 目标组](https://cloud.tencent.com/document/product/457/122522) · page_id `122522`

## 概述

TKE 通过 Kubernetes 自定义资源 TargetGroup 实现对 CLB 新版本目标组的创建、复用与管理。用户可通过为 Service 添加目标组 Annotation，将指定目标组绑定到 LB Service 端口对应的监听器上。

目标组允许将 Pod 直接作为后端 RS 绑定到 CLB 目标组上，支持四层（TCP/UDP/TCP_SSL/QUIC）和七层（HTTP/HTTPS）协议。

## 前置条件

- [环境准备](../../../../环境准备.md)
- 已 [连接集群](../../../../集群配置/集群管理/连接集群/tccli%20操作.md)，kubectl 可正常访问 APIServer
- 已在 CLB 侧开启**新版本四层目标组**及**支持新版本目标组挂载弹性网卡 IP** 白名单（需提工单）
- 如使用七层目标组能力，需开启**七层目标组**白名单
- service-controller 和 ingress-controller 版本 >= v2.8.0

## 控制台与 CLI 参数映射

| 控制台操作 | CLI |
|-----------|-----|
| 查看集群 | `tccli tke DescribeClusters` |
| 纳管已有目标组 | TargetGroup CRD `spec.id: lbtg-<id>` |
| 新建目标组 | TargetGroup CRD 完整 `spec` 字段 |
| Service 绑定目标组 | annotation `service.cloud.tencent.com/target-groups` |
| ClusterIP Service 绑定目标组 | annotation `service.cloud.tencent.com/target-groups`（name 必填） |

## 操作步骤

### 使用前提

在 CLB 侧开启相关白名单后，确保集群 controller 版本 >= v2.8.0：

```bash
kubectl get configmap tke-service-controller-config -n kube-system -o yaml | grep VERSION
```

### TargetGroup CRD

#### 方式一：纳管 CLB 侧已有目标组

已在 CLB 控制台创建目标组后，在 TKE 集群创建 TargetGroup 资源并填写 `spec.id`：

```yaml
apiVersion: cloud.tencent.com/v1alpha1
kind: TargetGroup
metadata:
  name: existing-tg
spec:
  id: lbtg-<TargetGroupId>
```

```bash
kubectl apply -f targetgroup-existing.yaml
```

> **注意**：纳管模式下暂不支持填写 `spec` 下其他字段。同步完成后，用户可在 TKE 侧对 CLB 目标组属性进行配置。同步期间禁止修改 TargetGroup 资源配置。

#### 方式二：新建 CLB 目标组

```yaml
apiVersion: cloud.tencent.com/v1alpha1
kind: TargetGroup
metadata:
  name: target-group
spec:
  protocol: TCP
  name: test
```

```bash
kubectl apply -f targetgroup-new.yaml
```

> **注意**：通过 TargetGroup CRD 新建的 CLB 目标组在集群销毁后不会被回收，需用户于 CLB 控制台手动删除。

### 在 LB Service 上使用目标组

在 Service 上添加 Annotation 为指定端口开启目标组能力：

```yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    service.cloud.tencent.com/target-groups: |
      [
        {"protocol": "TCP", "port": 80, "name": "test-target-group"}
      ]
  name: my-service
spec:
  ports:
  - port: 80
    targetPort: 80
    protocol: TCP
  selector:
    app: my-app
  type: LoadBalancer
```

```bash
kubectl apply -f service-targetgroup.yaml
```

字段说明：

| 字段 | 说明 |
|------|------|
| `protocol` | 需要开启目标组能力的端口协议，**必填** |
| `port` | 需要开启目标组能力的端口，**必填** |
| `name` | 该端口关联的 TargetGroup CR 资源名称，选填 |

- 指定 `name` 时：对应的 TargetGroup CR 资源必须提前创建；TKE 自动将监听器关联到指定 CLB 目标组。
- 未指定 `name` 时：TKE 自动为用户创建一个 TargetGroup 资源，自动名称格式为 `<namespace>.<service-name>.<protocol>.<port>`。
- 完成绑定后，TKE 将 Service 选中的所有 Pod 作为 RS 绑定到 CLB 目标组。
- 通过更新 `name` 字段可切换对应监听器绑定的 CLB 目标组。

> **警告**：切换 CLB 目标组涉及 RS 切换，可能有断流风险，请谨慎操作。

### 在 ClusterIP Service 上使用目标组

```yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    service.cloud.tencent.com/target-groups: |
      [
        {"protocol": "TCP", "port": 80, "name": "test-target-group"}
      ]
  name: my-clusterip-service
spec:
  ports:
  - port: 80
    targetPort: 80
    protocol: TCP
  selector:
    app: my-app
  type: ClusterIP
```

> ClusterIP Service 使用目标组时，`name` 字段必填。TKE 仅将 Service 选中的所有 Ready Pod 作为 RS 绑定到指定的 CLB 目标组上。

### TargetGroup CRD 完整字段说明

```yaml
apiVersion: cloud.tencent.com/v1alpha1
kind: TargetGroup
metadata:
  name: test-target-group           # 用户自行创建的 name 不能包含点（.）
spec:
  keepAlive: false                  # 是否开启长连接，仅 HTTP/HTTPS 监听器。选填，默认 false
  sessionExpireTime: 30             # 会话保持时间：30-3600。选填，不填写默认关闭
  scheduleAlgorithm: WRR            # 七层目标组负载均衡算法：WRR|LEAST_CONN|IP_HASH。默认 WRR
  healthCheck:                      # 健康检查配置，选填
    protocol: TCP                   # 健康检查协议：TCP|HTTP|HTTPS|PING|CUSTOM|GRPC。必填
    healthyThresholdCount: 3        # 健康阈值：2-10。选填，默认 3
    interval: 5                     # 检查间隔：3-300。选填，默认 5
    jumboFrame: false               # 是否开启巨帧。选填，默认 false
    port: 80                        # 健康检查端口。选填，默认为 RS 注册端口
    timeout: 2                      # 超时时间。选填，默认 2
    unhealthyThresholdCount: 3      # 检查失败阈值：2-10。选填，默认 3
    extendedCode: "12"              # GRPC 健康检查状态码。选填，默认 "12"
    custom:                         # 当健康检查类型为 CUSTOM 时可配置
      contextType: TEXT             # 文本类型：HEX|TEXT。必填
      sendContext: "xxxxx"
      receiveContext: "xxxxx"
    http:                           # 当健康检查类型为 HTTP 时可配置
      domain: test.com              # 健康检查域名。选填
      path: /                       # 健康检查路径。选填
      method: GET                   # HTTP 请求方式：GET|HEAD。选填，默认 GET
      version: HTTP/1.0             # HTTP 版本：HTTP/1.0|HTTP/1.1。选填，默认 HTTP/1.0
      statusCodes:                  # 健康检查正常状态码：1-5。选填，默认 1,2,3,4
        - 1                         # 1xx
        - 2                         # 2xx
  name: test-tg                     # CLB 侧目标组资源名称。选填，默认 <集群ID>.<k8s资源名称>
  protocol: TCP                     # 协议类型：TCP|UDP|HTTP|HTTPS|TCP_SSL|QUIC。必填
  id: lbtg-xxxxx                    # CLB 目标组 ID。填写 id 时不能填写其他字段。选填
```

### TargetGroup 示例

#### 创建 TCP 目标组（含会话保持）

```yaml
apiVersion: cloud.tencent.com/v1alpha1
kind: TargetGroup
metadata:
  name: test-target-group
spec:
  healthCheck:
    protocol: TCP
  sessionExpireTime: 1000
  name: test
  protocol: TCP
```

```bash
kubectl apply -f targetgroup-tcp.yaml
```

#### 创建 HTTP 目标组（含自定义健康检查）

```yaml
apiVersion: cloud.tencent.com/v1alpha1
kind: TargetGroup
metadata:
  name: test-target-group
spec:
  healthCheck:
    protocol: HTTP
    healthyThresholdCount: 10
    interval: 10
    timeout: 5
    unhealthyThresholdCount: 10
    http:
      domain: test.com
      path: /api
      statusCodes:
        - 1
        - 2
        - 3
        - 4
        - 5
  name: test
  protocol: HTTP
```

```bash
kubectl apply -f targetgroup-http.yaml
```

```text
# command executed successfully
```

### 相关限制

| 限制 | 说明 |
|------|------|
| 不支持回滚 | Service 端口开启目标组后，不支持回滚到监听器直接绑定 RS 模式 |
| 不支持重复纳管 | 不支持多个 TargetGroup 资源纳管同一个 CLB 目标组 |
| VPC 限制 | 不支持纳管与当前集群所在 VPC 不同的 CLB 目标组 |
| 端口唯一性 | 不支持单个 Service 不同端口使用相同 TargetGroup 资源 |
| 单引用 | 不支持多个 Service 引用相同 TargetGroup 资源 |
| 仅弹性网卡 IP | 仅支持绑定弹性网卡 IP 的 Service 开启目标组（非直连或 GR 直连不支持） |
| 自定义权重冲突 | 不支持配置了 `service.cloud.tencent.com/lb-rs-weight` 的 Service 使用目标组 |
| 单端口单目标组 | 不支持一个 Service 端口配置多个 TargetGroup 资源 |

### 风险提示

- 开启或切换目标组涉及 RS 的绑定与解绑，可能有断流风险。
- 请勿在 CLB 控制台对 TKE 管理的目标组进行修改，否则会有配置覆盖风险。

## 验证

### 数据面（kubectl）

```bash
# 查看 TargetGroup 资源
kubectl get targetgroups.cloud.tencent.com

# 查看 TargetGroup 详情
kubectl get targetgroups.cloud.tencent.com test-target-group -o yaml

# 查看 Service 目标组配置
kubectl get service my-service -o yaml | grep target-groups

# 确认 Pod 已作为 RS 注册
kubectl get pods -o wide -l app=my-app
```

```text
NAME  STATUS  AGE
...
```

## 清理

### 数据面（kubectl）

```bash
# 删除 Service
kubectl delete service my-service --ignore-not-found

# 删除 ClusterIP Service
kubectl delete service my-clusterip-service --ignore-not-found

# 删除 TargetGroup CR（不会自动删除 CLB 侧目标组）
kubectl delete targetgroups.cloud.tencent.com test-target-group existing-tg --ignore-not-found
```

> 通过 TargetGroup CRD 新建的 CLB 目标组在集群销毁后不会被回收，需在 CLB 控制台手动删除。

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `Unable to connect to the server` | `kubectl cluster-info` | 公网端点被 CAM 策略 strategyId:240463971 以 `tke:clusterExtranetEndpoint=true` 条件拒绝 | 通过 IOA/VPN 连接内网，或使用同 VPC CVM 执行 kubectl |
| 目标组创建失败 | `tccli clb DescribeLoadBalancers` | 已开启 CLB 目标组白名单； controller 版本 >= v2.8.0 | 确认已开启 CLB 目标组白名单；确认 controller 版本 >= v2.8.0 |
| 非直连模式不生效 | `kubectl describe <resource>` | 仅支持绑定弹性网卡 IP 的 Service | 仅支持绑定弹性网卡 IP 的 Service |
| Service 端口开启目标组后无法回滚 | `kubectl describe svc` | 目标组开启后不支持回滚到直接绑定 RS 模式 | 目标组开启后不支持回滚到直接绑定 RS 模式，属于设计限制 |
| 配置被覆盖 | `tccli clb DescribeLoadBalancers` | 请勿在 CLB 控制台修改 TKE 管理的目标组配置 | 请勿在 CLB 控制台修改 TKE 管理的目标组配置 |
| 纳管已有目标组同步失败 | `kubectl describe <resource>` | 同步期间不要修改 TargetGroup 资源配置 | 同步期间不要修改 TargetGroup 资源配置 |
| 自定义权重不生效 | `lb-rs-weight` | 配置了 `lb-rs-weight` 的 Service 不支持目标组 | 配置了 `lb-rs-weight` 的 Service 不支持目标组 |

## 下一步

- [Service 基本功能](../Service%20基本功能/tccli%20操作.md)
- [Service 扩展协议](../Service%20扩展协议/tccli%20操作.md)
- [全局开关配置说明](../全局开关配置说明/tccli%20操作.md)
- Service Annotation 说明请参见官方文档

## 控制台替代

CLB 目标组通过 TargetGroup CRD + Service Annotation 配置管理；目标组本身可在 [CLB 控制台](https://console.cloud.tencent.com/clb) 创建和管理。
