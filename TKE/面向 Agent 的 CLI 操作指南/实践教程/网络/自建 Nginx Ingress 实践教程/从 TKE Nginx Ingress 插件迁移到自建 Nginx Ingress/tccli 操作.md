# 从 TKE Nginx Ingress 插件迁移到自建 Nginx Ingress（tccli）

> 对照官方：[从 TKE Nginx Ingress 插件迁移到自建 Nginx Ingress](https://cloud.tencent.com/document/product/457/104864) · page_id `104864`

## 概述

将 TKE 集群中通过 `InstallAddon` 安装的 Nginx Ingress 插件迁移到自建（Helm）Nginx Ingress Controller，获得更大的版本灵活性、自定义 CLB 配置能力和多实例支持。

**迁移策略对比**：

| 策略 | 步骤 | 停机时间 | 风险 |
|------|------|---------|------|
| 蓝绿切换 | 安装自建 → 验证 → 切换 DNS → 卸载插件 | 秒级（DNS TTL 内） | 低，可随时回滚 |
| 灰度切换 | 安装自建（新 IngressClass）→ 逐步迁移 Ingress 规则 | 无 | 低，但需要双 Controller 运行一段时间 |
| 原地替换 | 卸载插件 → 立即安装自建 → 重新创建 Ingress | 分钟级 | 中，需重建资源 |

**推荐蓝绿切换**：DNS 指向新 CLB 后验证通过再拆旧，最小停机时间且可回滚。

> **kubectl 数据面不可达**：CAM 拒绝公网端点（strategyId:240463971），内网需 VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

## 前置条件

- [环境准备](../../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId、secretKey、region 均已配置

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:DescribeClusters, tke:DescribeAddon, tke:UninstallAddon
#    tke:DescribeClusterKubeconfig
#    clb:DescribeLoadBalancers
# 验证
tccli tke DescribeClusters --region <Region>
# expected: exit 0

tccli tke DescribeAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName NginxIngress
# expected: exit 0，返回插件信息

# 4. 检查 kubectl 和 Helm（须 VPN/IOA）
kubectl version --client
# expected: Client Version v1.30+
helm version --short
# expected: v3.x
```

```json
{
  "TotalCount": "<TotalCount>",
  "Clusters": "<Clusters>",
  "ClusterId": "<ClusterId>",
  "ClusterName": "<ClusterName>",
  "ClusterDescription": "<ClusterDescription>",
  "ClusterVersion": "<ClusterVersion>"
}
```

### 资源检查

```bash
# 5. 确认 Nginx Ingress 插件当前状态
tccli tke DescribeAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName NginxIngress
# expected: Status 为 "Running"

# 6. 备份当前 Ingress 规则
kubectl get ingress -A -o yaml > ingress-backup-$(date +%Y%m%d%H%M%S).yaml
# expected: 文件生成，包含所有命名空间的 Ingress 规则

# 7. 获取当前插件使用的 CLB ID
kubectl -n kube-system get svc -l app.kubernetes.io/name=ingress-nginx -o yaml | grep loadbalance-id
# expected: 插件创建的 CLB ID

# 8. 查看当前 IngressClass
kubectl get ingressclass
# expected: 现有 IngressClass（通常为 nginx）
```

```json
{
  "Addons": "<Addons>",
  "AddonName": "<AddonName>",
  "AddonVersion": "<AddonVersion>",
  "RawValues": "<RawValues>",
  "Phase": "<Phase>",
  "Reason": "<Reason>"
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 查看已安装组件 | `tccli tke DescribeAddon` | 是 |
| 卸载组件 | `tccli tke UninstallAddon` | 是 |
| 查看集群 | `tccli tke DescribeClusters` | 是 |
| 备份 Ingress | `kubectl get ingress -A -o yaml` | 是 |
| 添加 Helm 仓库 | `helm repo add` | 否 |
| 安装 Helm Chart | `helm install` | 否 |
| 查看 CLB | `tccli clb DescribeLoadBalancers` | 是 |

## 关键字段说明

| 字段/操作 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `AddonName` | String | 是（卸载时） | `"NginxIngress"` 精确匹配 | 拼写错误 → `ResourceNotFound` |
| `ClusterId` | String | 是 | 集群 ID，格式 `cls-xxxxxxxx` | ID 错误 → `ResourceNotFound` |
| `ingressClassResource.name` | String | 迁移时推荐 | 蓝绿策略推荐使用 `nginx-selfbuilt` 与插件 `nginx` 区分 | 与现有 Class 重名 → 控制权冲突 |
| `DNS 指向` | 操作 | 切换时 | 将域名 A 记录从旧 CLB IP 改为新 CLB IP | 未更新 → 流量仍走旧入口 |

## 操作步骤

### 步骤 1：备份当前状态

#### 选择依据

- **全量备份**：备份所有命名空间的 Ingress 规则、当前插件配置、CLB 信息，确保迁移失败时可恢复。
- 不要在备份后修改任何 Ingress 规则，直到迁移验证通过。

```bash
# 备份 Ingress 规则
kubectl get ingress -A -o yaml > ingress-backup-$(date +%Y%m%d%H%M%S).yaml
# expected: 文件生成

# 备份插件当前配置
tccli tke DescribeAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName NginxIngress > addon-backup-$(date +%Y%m%d%H%M%S).json
# expected: 文件生成

# 记录当前 CLB IP
kubectl -n kube-system get svc -l app.kubernetes.io/name=ingress-nginx -o wide
# expected: 显示 EXTERNAL-IP
```

> **kubectl 数据面不可达**：CAM 拒绝公网端点（strategyId:240463971），内网需 VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

### 步骤 2：安装自建 Nginx Ingress Controller（蓝绿）

#### 选择依据

- **独立 Ingress Class**：使用 `nginx-selfbuilt` 作为 IngressClass，与插件默认的 `nginx` 区分。这样两个 Controller 可以同时运行。
- **独立命名空间**：安装在 `ingress-nginx-selfbuilt`，与插件的 `kube-system` 隔离。
- **CLB 配置**：如需复用旧的 CLB IP（减少 DNS 变更），可以先安装自建 Controller 并创建新 CLB，DNS 验证成功后再下线旧 CLB。

`nginx-ingress-migrate.yaml`：

```yaml
controller:
  ingressClassResource:
    name: nginx-selfbuilt
    controllerValue: k8s.io/ingress-nginx-selfbuilt
  ingressClass: nginx-selfbuilt
  electionID: ingress-nginx-selfbuilt-leader
  service:
    type: LoadBalancer
    annotations:
      service.kubernetes.io/qcloud-loadbalancer-internal-subnetid: "SUBNET_ID"
```

```bash
helm install nginx-ingress-selfbuilt ingress-nginx/ingress-nginx \
    --namespace ingress-nginx-selfbuilt \
    --create-namespace \
    -f nginx-ingress-migrate.yaml
# expected: STATUS: deployed
```

### 步骤 3：验证自建 Controller

```bash
# 获取新 CLB IP
kubectl -n ingress-nginx-selfbuilt get svc nginx-ingress-selfbuilt-ingress-nginx-controller
# expected: EXTERNAL-IP 不为空

# 验证新 Controller Pod 状态
kubectl -n ingress-nginx-selfbuilt get pods
# expected: Running

# 验证 IngressClass
kubectl get ingressclass nginx-selfbuilt
# expected: 存在且 Controller 值正确
```

```text
NAME  STATUS  AGE
...
```

### 步骤 4：迁移 Ingress 规则到自建 Controller

#### 选择依据

- **批量迁移**：将备份的 Ingress 规则中 `ingressClassName` 从 `nginx`（插件）改为 `nginx-selfbuilt`（自建）。
- **DNS 切换**：将域名指向新的 CLB IP。
- **灰度验证**：先迁移一个非关键 Ingress 测试，确认流量正常后再迁移其余。

```bash
# 方式 1：修改单个 Ingress 的 IngressClass
kubectl annotate ingress INGRESS_NAME -n NAMESPACE \
    kubernetes.io/ingress.class=nginx-selfbuilt --overwrite
# 并在 Ingress spec 中设置 ingressClassName

# 方式 2：批量更新（用 sed 修改备份文件中的 className）
sed 's/ingressClassName: nginx/ingressClassName: nginx-selfbuilt/g' ingress-backup.yaml | kubectl apply -f -
# expected: ingress configured (或 created)

# 验证迁移后的 Ingress 状态
kubectl get ingress -A | grep -E "nginx-selfbuilt"
# expected: 显示迁移后的 Ingress 规则
```

```text
NAME  STATUS  AGE
...
```

### 步骤 5：验证流量

```bash
# 获取新入口 IP
NEW_CLB_IP=$(kubectl -n ingress-nginx-selfbuilt get svc nginx-ingress-selfbuilt-ingress-nginx-controller -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
# expected: 获取新 IP

# 测试流量（手动修改本地 hosts 或 DNS）
# curl -H "Host: YOUR_DOMAIN" http://NEW_CLB_IP/
# expected: 正常返回

# 更新 DNS A 记录指向新 CLB IP
# （DNS 操作不在 tccli 范围内，通过域名服务商控制台或 tccli dnspod 完成）
```

### 步骤 6：卸载 TKE Nginx Ingress 插件

#### 选择依据

- **卸载时机**：确认 DNS 已切换到新 CLB，所有流量经由自建 Controller，且监控指标（QPS、延迟、错误率）正常。
- **不可逆**：插件卸载后无法快速恢复。保留插件至少 24 小时再卸载（如 CLB 计费成本可接受）。

```bash
# 卸载 Nginx Ingress 插件
tccli tke UninstallAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName NginxIngress
# expected: exit 0
```

```bash
# 验证插件已卸载
tccli tke DescribeAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName NginxIngress
# expected: ResourceNotFound 或状态显示未安装
```

```json
{
  "Addons": "<Addons>",
  "AddonName": "<AddonName>",
  "AddonVersion": "<AddonVersion>",
  "RawValues": "<RawValues>",
  "Phase": "<Phase>",
  "Reason": "<Reason>"
}
```

## 验证

### 控制面（tccli）

```bash
# 确认插件已卸载
tccli tke DescribeAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName NginxIngress
# expected: ResourceNotFound

# 确认集群状态正常
tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterStatus "Running"

# 确认新 CLB 状态
tccli clb DescribeLoadBalancers --region <Region> \
    --LoadBalancerIds '["NEW_CLB_ID"]'
# expected: Status 为 1
```

```json
{
  "Addons": "<Addons>",
  "AddonName": "<AddonName>",
  "AddonVersion": "<AddonVersion>",
  "RawValues": "<RawValues>",
  "Phase": "<Phase>",
  "Reason": "<Reason>"
}
```

### 数据面（需 VPN/IOA）

```bash
# 验证自建 Controller 运行
kubectl -n ingress-nginx-selfbuilt get pods
# expected: Running

# 验证 Ingress 规则已迁移
kubectl get ingress -A -o wide | grep nginx-selfbuilt
# expected: 所有迁移的 Ingress 规则使用 nginx-selfbuilt

# 验证旧插件相关资源已清理（kube-system 无残留 Service）
kubectl -n kube-system get svc -l app.kubernetes.io/name=ingress-nginx
# expected: No resources found

# 验证流量
curl -H "Host: YOUR_DOMAIN" https://NEW_CLB_IP/
# expected: 正常响应

# 递归 DNS 验证（确认域名解析到新 IP）
dig +short YOUR_DOMAIN
# expected: NEW_CLB_IP
```

```text
NAME  STATUS  AGE
...
```

## 清理

### 数据面（需 VPN/IOA）

> **⚠️ 警告**：如果插件创建了独立的 CLB 且未被复用，插件卸载时 CLB 可能已被自动删除。务必确认所有 Ingress 规则已迁移到自建 Controller 且流量正常后再卸载。

```bash
# 1. 清理旧 IngressClass（如不再需要）
kubectl delete ingressclass nginx
# expected: ingressclass deleted（仅当插件已卸载且无 Ingress 使用此 Class）

# 2. 删除备份文件（确认迁移成功后）
rm -f ingress-backup-*.yaml addon-backup-*.json
# expected: 文件删除
```

### 控制面（tccli）

```bash
# 验证旧 CLB 已释放（如果未被复用）
tccli clb DescribeLoadBalancers --region <Region>
# expected: 列表中不再包含插件创建的 CLB
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `UninstallAddon` 返回 `ResourceNotFound` | `tccli tke DescribeAddon --region <Region> --ClusterId CLUSTER_ID --AddonName NginxIngress` | 插件已卸载或 AddonName 拼写错误 | 确认 AddonName 为 `"NginxIngress"`（大小写敏感） |
| `UninstallAddon` 返回 `FailedOperation` | 检查集群状态和插件依赖 | 插件卸载过程中资源清理失败 | 登录控制台查看详细错误；必要时 `tccli tke DescribeAddon` 确认状态 |
| `helm install` 返回 `IngressClass already exists` | `kubectl get ingressclass` | 自建 Controller 的 IngressClass 与插件冲突 | 使用不同的 `ingressClassResource.name`（如 `nginx-selfbuilt`） |

### 迁移后功能异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 迁移后流量不通 | `kubectl -n ingress-nginx-selfbuilt logs -l app.kubernetes.io/name=ingress-nginx --tail=20` | IngressClass 不匹配或 Ingress 规则有 Webhook 校验失败 | 检查 `kubectl describe ingress <NAME>` 的 Events；确认 Ingress 的 `ingressClassName` 与新 Controller 的 `ingressClassResource.name` 一致 |
| DNS 切换后部分客户端仍走旧 IP | `dig +short YOUR_DOMAIN @8.8.8.8` | DNS TTL 未过期，缓存中仍是旧 IP | 等待 DNS TTL 过期（通常 5 分钟 - 1 小时）；或降低 TTL 提前等待过期 |
| 卸载插件后 CLB 被删导致流量全断 | `tccli clb DescribeLoadBalancers --region <Region>` | 插件创建的 CLB 随插件卸载被删除，而 DNS 仍指向旧 IP | 立即将 DNS 指向新 CLB IP 或恢复插件：`tccli tke InstallAddon --AddonName NginxIngress`（需确认 CLB 可以重建） |
| Webhook 拒绝 Ingress（validation admission error） | `kubectl describe ingress <NAME>` 查看 Events；`kubectl get validatingwebhookconfigurations` | 插件和自建 Controller 的 Webhook 同时存在，新 Ingress 被旧 Webhook 拒绝 | 卸载插件后旧 Webhook 自动移除；或手动删除：`kubectl delete validatingwebhookconfiguration <PLUGIN_WEBHOOK>` |
| 自建 Controller 不处理使用旧 Class 的 Ingress | `kubectl get ingress -A -o yaml | grep ingressClassName` | Ingress 仍使用 `ingressClassName: nginx`（插件 Class），但插件已卸载 | 将 `ingressClassName` 改为 `nginx-selfbuilt`，或修改自建 Controller 的 `ingressClassResource.name` 为 `nginx` |

## 下一步

- [快速开始](https://cloud.tencent.com/document/product/457/104857) — 自建 Nginx Ingress 基础操作
- [自定义负载均衡器](https://cloud.tencent.com/document/product/457/104858) — 迁移后进一步自定义 CLB 配置
- [安装多个 Nginx Ingress Controller](https://cloud.tencent.com/document/product/457/104863) — 多实例管理
- [高可用配置优化](https://cloud.tencent.com/document/product/457/104860) — 迁移后增强高可用

## 控制台替代

通过 [TKE 控制台 - 组件管理](https://console.cloud.tencent.com/tke2/cluster/sub/list/addon/cluster?rid=1) 卸载 Nginx Ingress 插件，通过 [CLB 控制台](https://console.cloud.tencent.com/clb) 管理负载均衡。
