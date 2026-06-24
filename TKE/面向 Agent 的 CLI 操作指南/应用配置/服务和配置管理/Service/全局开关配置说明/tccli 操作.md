# 全局开关配置说明（tccli）

> 对照官方：[全局开关配置说明](https://cloud.tencent.com/document/product/457/131136) · page_id `131136`

## 概述

TKE 接入层组件（service-controller 和 ingress-controller）通过 ConfigMap 管理全局开关配置，用于控制集群级别的特性行为。全局开关对整个集群生效，与资源级注解（[Service Annotation](https://cloud.tencent.com/document/product/457/51258) 和 [Ingress Annotation](https://cloud.tencent.com/document/product/457/56112)）不同，全局开关作用于集群内所有资源。

全局开关存储在 `kube-system` 命名空间下的 ConfigMap `tke-service-controller-config` 中，用户可通过修改该 ConfigMap 动态开启或关闭集群级别的特性。

## 前置条件

- [环境准备](../../../../环境准备.md)
- 已 [连接集群](../../../../集群配置/集群管理/连接集群/tccli%20操作.md)，kubectl 可正常访问 APIServer

## 控制台与 CLI 参数映射

| 控制台操作 | CLI |
|-----------|-----|
| 查看集群 | `tccli tke DescribeClusters` |
| 查看全局开关状态 | `kubectl get configmap tke-service-controller-config -n kube-system -o yaml` |
| 修改全局开关 | `kubectl edit configmap tke-service-controller-config -n kube-system` |

## 操作步骤

### ConfigMap 示例

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: tke-service-controller-config
  namespace: kube-system
data:
  VERSION: "v2.11.4"
  GlobalRouteDirectAccess: "true"
  EnableNodeGracefulDeletion: "false"
  EnablePodGracefulDeletion: "false"
  EnableDefaultModificationProtection: "true"
  EnableDefaultDirectAccess: "true"
```

### 用户可配置的全局开关

#### GlobalRouteDirectAccess — GR 直连访问

| 项目 | 说明 |
|------|------|
| **ConfigMap Key** | `GlobalRouteDirectAccess` |
| **作用** | 控制集群是否支持 GlobalRoute（GR）直连特性。开启后，使用 GR 网络模式的 Service/Ingress 支持使用直连访问模式 |
| **支持版本** | >= v1.6.0，支持 `true` / `false` |
| **新建集群默认值** | `true`（默认开启） |
| **注意事项** | 开启后，Service/Ingress 仍需配置 `service.cloud.tencent.com/direct-access: "true"` 或 `ingress.cloud.tencent.com/direct-access: "true"` 注解 |

```bash
# 查看
kubectl get configmap tke-service-controller-config -n kube-system -o yaml | grep GlobalRouteDirectAccess

# 修改
kubectl patch configmap tke-service-controller-config \
  -n kube-system \
  --type merge \
  -p '{"data":{"GlobalRouteDirectAccess":"true"}}'
```

```text
NAME  STATUS  AGE
...
```

#### EnableNodeGracefulDeletion — 节点优雅删除

| 项目 | 说明 |
|------|------|
| **ConfigMap Key** | `EnableNodeGracefulDeletion` |
| **作用** | 控制集群是否开启节点优雅删除特性。开启后，当节点被删除或缩容时，控制器会先将该节点上所有 CLB 后端权重调整为 0，等待流量排空后再解绑后端 |
| **支持版本** | >= v2.4.2，支持 `true` / `false` |
| **新建集群默认值** | `false`（默认关闭） |
| **注意事项** | 开启后节点删除可能需要等待一段时间；仅对 NodePort 模式 Service 生效；如节点下线紧急可关闭 |

```bash
# 查看
kubectl get configmap tke-service-controller-config -n kube-system -o yaml | grep EnableNodeGracefulDeletion

# 开启
kubectl patch configmap tke-service-controller-config \
  -n kube-system \
  --type merge \
  -p '{"data":{"EnableNodeGracefulDeletion":"true"}}'
```

```text
NAME  STATUS  AGE
...
```

#### EnablePodGracefulDeletion — Pod 优雅删除

| 项目 | 说明 |
|------|------|
| **ConfigMap Key** | `EnablePodGracefulDeletion` |
| **作用** | 控制集群是否开启 Pod 优雅删除特性。开启后，Pod 被删除时控制器先将 Pod 在 CLB 上的权重调整为 0，等待流量排空后再解绑后端 |
| **支持版本** | >= v2.8.0，支持 `true` / `false` |
| **新建集群默认值** | `false`（默认关闭） |
| **注意事项** | 仅在直连访问模式下生效。全局开关作为默认值，用户可通过注解 `service.cloud.tencent.com/enable-grace-deletion` 覆盖全局开关 |

```bash
# 查看
kubectl get configmap tke-service-controller-config -n kube-system -o yaml | grep EnablePodGracefulDeletion

# 开启
kubectl patch configmap tke-service-controller-config \
  -n kube-system \
  --type merge \
  -p '{"data":{"EnablePodGracefulDeletion":"true"}}'
```

```text
NAME  STATUS  AGE
...
```

#### EnableDefaultModificationProtection — 默认修改保护

| 项目 | 说明 |
|------|------|
| **ConfigMap Key** | `EnableDefaultModificationProtection` |
| **作用** | 控制集群内自动创建的 CLB 是否默认开启配置修改保护。开启后，自动创建的 CLB 将无法通过 CLB 控制台或 API 修改实例配置属性 |
| **支持版本** | >= v2.11.1，支持 `true` / `false` |
| **新建集群默认值** | `true`（v2.11.1 起默认开启） |
| **注意事项** | 仅对新增的自动创建 CLB 资源生效，对存量资源不回溯。可通过 Service/Ingress 注解在资源级别单独控制 |

```bash
# 查看
kubectl get configmap tke-service-controller-config -n kube-system -o yaml | grep EnableDefaultModificationProtection

# 修改
kubectl patch configmap tke-service-controller-config \
  -n kube-system \
  --type merge \
  -p '{"data":{"EnableDefaultModificationProtection":"true"}}'
```

```text
NAME  STATUS  AGE
...
```

#### EnableDefaultDirectAccess — 默认直连访问

| 项目 | 说明 |
|------|------|
| **ConfigMap Key** | `EnableDefaultDirectAccess` |
| **作用** | 控制集群内新建的 LoadBalancer 类型 Service/Ingress 是否默认开启直连访问模式。开启后，新建资源自动添加 `direct-access: "true"` 注解 |
| **支持版本** | >= v2.11.0，支持 `true` / `false` |
| **新建集群默认值** | 根据网络类型动态决定：TKE GlobalRoute 和 VPC-CNI 新建集群默认 `true`；其他网络类型无默认值 |
| **注意事项** | 仅影响新建的 Service/Ingress，对存量资源不回溯 |

```bash
# 查看
kubectl get configmap tke-service-controller-config -n kube-system -o yaml | grep EnableDefaultDirectAccess

# 修改
kubectl patch configmap tke-service-controller-config \
  -n kube-system \
  --type merge \
  -p '{"data":{"EnableDefaultDirectAccess":"true"}}'
```

```text
NAME  STATUS  AGE
...
```

### 只读字段

以下 ConfigMap 字段为系统自动写入的只读字段，**用户不应修改**：

| ConfigMap Key | 说明 |
|---------------|------|
| `VERSION` | 当前 controller 的版本号，由系统自动同步 |
| `REUSE_LOADBALANCER` | Service 支持复用 CLB 特性是否已开启，v2.5.1 起默认 `true`，由组件自动写入 |

> **注意**：系统会监听 ConfigMap 的变更事件，如果只读字段被用户修改，控制器会强制覆盖回正确的值。

## 验证

### 数据面（kubectl）

```bash
# 查看当前集群所有全局开关状态
kubectl get configmap tke-service-controller-config -n kube-system -o yaml
```

```text
NAME  STATUS  AGE
...
```

## 清理

不需要清理。如需恢复默认值，将对应开关值设为默认配置即可：

```bash
kubectl edit configmap tke-service-controller-config -n kube-system
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `Unable to connect to the server` | `kubectl cluster-info` | 公网端点被 CAM 策略 strategyId:240463971 以 `tke:clusterExtranetEndpoint=true` 条件拒绝 | 通过 IOA/VPN 连接内网，或使用同 VPC CVM 执行 kubectl |
| 修改后不生效 | `"true"` | 可配置类型的全局开关修改后立即生效 | 可配置类型的全局开关修改后立即生效；确认 key 名称拼写正确；确认值使用字符串 `"true"` 或 `"false"` |
| 存量集群升级后新增开关未自动开启 | `kubectl describe <resource>` | 存量集群升级时不会新增 ConfigMap 中的字段 | 存量集群升级时不会新增 ConfigMap 中的字段；需手动添加新开关并设置 |
| 全局开关被注解覆盖 | `EnablePodGracefulDeletion` | 部分开关作为默认值（如 `EnablePodGracefulDeletion`） | 部分开关作为默认值（如 `EnablePodGracefulDeletion`），资源级注解可覆盖 |
| 只读字段被覆盖 | `VERSION` | 系统会自动恢复 `VERSION` 等只读字段的正确值 | 系统会自动恢复 `VERSION` 等只读字段的正确值 |
| 修改 ConfigMap 后现有 Service 无变化 | `EnableDefaultDirectAccess` | 部分开关（如 `EnableDefaultDirectAccess`）仅影响新建资源 | 部分开关（如 `EnableDefaultDirectAccess`）仅影响新建资源，不影响存量资源 |

### 常见问题

**Q: 修改 ConfigMap 后多久生效？**
可配置类型的全局开关修改后立即生效，控制器监听 ConfigMap 变更事件并动态调整状态，无需重启组件。

**Q: 全局开关和注解（Annotation）有什么区别？**
全局开关是集群级别配置，对集群内所有资源生效；注解是资源级别配置，只对单个 Service/Ingress 生效。两者关系因开关而异：部分开关（如 `EnablePodGracefulDeletion`）作为默认值，注解可覆盖全局开关；部分开关（如 `EnableDefaultDirectAccess`）仅影响新建资源的默认行为。

**Q: 存量集群升级后，新增的全局开关是否会自动开启？**
不会。存量集群升级时 ConfigMap 中已存在的字段不会被覆盖，新增的开关只在 ConfigMap 首次创建时（新建集群）设置默认值。

**Q: 如何查看当前集群的全局开关状态？**
```bash
kubectl get configmap tke-service-controller-config -n kube-system -o yaml
```

```text
NAME  STATUS  AGE
...
```

**Q: 如何修改全局开关？**
```bash
kubectl edit configmap tke-service-controller-config -n kube-system
```

## 下一步

- [使用 LoadBalancer 直连 Pod 模式 Service](../使用%20LoadBalancer%20直连%20Pod%20模式%20Service/tccli%20操作.md)
- [Node 优雅下线](../Node%20优雅下线/tccli%20操作.md)
- [Pod 优雅删除](../Pod%20优雅删除/tccli%20操作.md)
- Service Annotation 说明请参见官方文档

## 控制台替代

[控制台 → 集群 → 配置管理 → ConfigMap](https://console.cloud.tencent.com/tke2/cluster) 编辑 `kube-system/tke-service-controller-config`。
