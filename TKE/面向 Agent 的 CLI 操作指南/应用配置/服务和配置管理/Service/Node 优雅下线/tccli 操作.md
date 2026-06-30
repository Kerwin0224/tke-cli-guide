# Node 优雅下线（tccli）

> 对照官方：[Node 优雅下线](https://cloud.tencent.com/document/product/457/113905) · page_id `113905`

## 概述

在非直连 Pod 场景中，Node 作为中间转发者将流量转发到 Pod。如果直接删除 Node，在多个 Service 共享一个 CLB 且下线 Node 数量较多时，Node 删除速度可能快于 CLB 解绑后端（RS）的速度。此时 Node 转发组件已被移除，但 Node 仍作为 RS 绑定在 CLB 上，CLB 会将流量转发给已失去转发能力的 Node。长连接场景下，这一过程可能持续到长连接超时。

开启 Node 优雅下线后，节点下线时会保证所有 CLB 先解绑该 Node，然后才销毁 Node。

## 前置条件

- [环境准备](../../../../环境准备.md)
- 已 [连接集群](../../../../集群配置/集群管理/连接集群/tccli%20操作.md)，kubectl 可正常访问 APIServer
- 仅针对**非直连场景**生效
- 在 TKE 层删除 Node 前，不能在云服务器控制台删除实际机器
- 长连接业务需能感知 RST（Connection Reset），及时重新建立连接
- 开启功能后，接入层组件会为所有作为 CLB 后端的 Node 打上保护性 Finalizer，这一过程需约 10 分钟，期间不要操作缩容

## 控制台与 CLI 参数映射

| 控制台操作 | CLI |
|-----------|-----|
| 查看集群 | `tccli tke DescribeClusters` |
| 查看 ConfigMap | `kubectl get configmap tke-service-controller-config -n kube-system -o yaml` |
| 开启节点优雅删除 | ConfigMap `EnableNodeGracefulDeletion: "true"` |
| 修改 ConfigMap | `kubectl edit configmap tke-service-controller-config -n kube-system` |

## 操作步骤

### 支持版本

- **service-controller**（针对 Service 场景）：>= v2.4.2
- **ingress-controller**（针对 Ingress 场景）：>= v2.4.2

通过查看 `kube-system` 下 ConfigMap 的 VERSION 字段确认版本：

```bash
kubectl get configmap tke-service-controller-config -n kube-system -o yaml | grep VERSION
kubectl get configmap tke-ingress-controller-config -n kube-system -o yaml | grep VERSION
```

```text
NAME  STATUS  AGE
...
```

### 方式1：通过控制台 ConfigMap 开启

1. 登录 [容器服务控制台](https://console.cloud.tencent.com/tke2)，在集群详情页选择 **配置管理 > ConfigMap**。
2. 在 `kube-system` 命名空间下找到 `tke-service-controller-config`。
3. 单击 **更新配置**，找到 `EnableNodeGracefulDeletion` 开关变量，将其值设为 `true`。

### 方式2：通过 kubectl 开启

```bash
# 查看当前 ConfigMap
kubectl -n kube-system get configmap tke-service-controller-config -o yaml
```

```bash
# 编辑 ConfigMap，将 EnableNodeGracefulDeletion 置为 true
kubectl -n kube-system edit configmap tke-service-controller-config
```

在 `data` 下设置：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: tke-service-controller-config
  namespace: kube-system
data:
  EnableNodeGracefulDeletion: "true"
  VERSION: v2.x.x
```

### 生效范围

| 节点类型 | 操作方式 | 是否支持 |
|----------|----------|----------|
| 普通节点池 | 直接删除节点池 | No |
| 普通节点池 | 单击"调整数量" | No |
| 普通节点池 | 选中一批节点，单击"移出" | Yes |
| 普通节点池 | 自动扩缩容 | Yes |
| 普通节点 | 单击"移出" | Yes |
| 原生节点池 | 所有操作 | No |

> 超级节点不存在"非直连"模式，不需要支持该能力。

## 验证

### 数据面（kubectl）

```bash
# 确认 ConfigMap 配置
kubectl get configmap tke-service-controller-config -n kube-system -o yaml | grep EnableNodeGracefulDeletion
```

```text
NAME  STATUS  AGE
...
```

## 清理

无需清理特殊资源。如需关闭该功能，将 `EnableNodeGracefulDeletion` 设为 `"false"`：

```bash
kubectl patch configmap tke-service-controller-config \
  -n kube-system \
  --type merge \
  -p '{"data":{"EnableNodeGracefulDeletion":"false"}}'
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `Unable to connect to the server` | `kubectl cluster-info` | 公网端点被 CAM 策略 strategyId:240463971 以 `tke:clusterExtranetEndpoint=true` 条件拒绝 | 通过 IOA/VPN 连接内网，或使用同 VPC CVM 执行 kubectl |
| 节点删除后仍连接旧 Node | `EnableNodeGracefulDeletion: "true"` | 已开启 `EnableNodeGracefulDeletion: "true"` | 确认已开启 `EnableNodeGracefulDeletion: "true"` |
| 功能不生效 | `kubectl describe <resource>` | service-controller/ingress-controller 版本 >= v2.4.2 | 确认 service-controller/ingress-controller 版本 >= v2.4.2 |
| 原生节点池缩容不优雅 | `kubectl describe <resource>` | 原生节点池不支持该功能 | 原生节点池不支持该功能 |
| 开启后缩容异常 | `kubectl describe <resource>` | 开启后需要约 10 分钟完成所有节点 Finalizer 标记 | 开启后需要约 10 分钟完成所有节点 Finalizer 标记，期间不要操作缩容 |
| 云服务器直接删除导致异常 | `kubectl describe <resource>` | TKE 层删除 Node 前 | TKE 层删除 Node 前，不能在云服务器控制台删除机器 |

## 下一步

- [Service 优雅停机](../Service%20优雅停机/tccli%20操作.md)
- [Pod 优雅删除](../Pod%20优雅删除/tccli%20操作.md)
- [全局开关配置说明](../全局开关配置说明/tccli%20操作.md)

## 控制台替代

[控制台 → 集群 → 配置管理 → ConfigMap](https://console.cloud.tencent.com/tke2/cluster) 更新 `kube-system/tke-service-controller-config`。
