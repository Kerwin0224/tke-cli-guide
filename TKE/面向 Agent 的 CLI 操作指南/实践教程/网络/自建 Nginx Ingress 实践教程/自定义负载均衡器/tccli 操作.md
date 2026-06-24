# 自定义负载均衡器（tccli）

> 对照官方：[自定义负载均衡器](https://cloud.tencent.com/document/product/457/104858) · page_id `104858`

## 概述

通过 Helm Values 中的 Service Annotation 自定义 Nginx Ingress Controller 使用的 CLB 实例属性，包括：CLB 带宽上限、网络类型（公网/内网）、已有 CLB 复用、跨可用区容灾等。

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
#    tke:DescribeClusters, clb:CreateLoadBalancer, clb:DescribeLoadBalancers
#    clb:ModifyLoadBalancerAttributes, clb:DeleteLoadBalancer
# 验证
tccli tke DescribeClusters --region <Region>
# expected: exit 0

tccli clb DescribeLoadBalancers --region <Region>
# expected: exit 0，返回 CLB 列表（可为空）

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
# 5. 确认目标集群
tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterStatus "Running"

# 6. 查询 CLB 配额
tccli clb DescribeQuota --region <Region>
# expected: 剩余 CLB 配额 ≥ 1

# 7. 如复用已有 CLB，确认其状态
tccli clb DescribeLoadBalancers --region <Region> \
    --LoadBalancerIds '["CLB_ID"]'
# expected: Status 为 1（正常），Isolation 为 0（未隔离）
```

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 查看 CLB 配额 | `tccli clb DescribeQuota` | 是 |
| 查看 CLB 列表 | `tccli clb DescribeLoadBalancers` | 是 |
| 创建 CLB | `tccli clb CreateLoadBalancer` | 否 |
| 修改 CLB 属性 | `tccli clb ModifyLoadBalancerAttributes` | 否 |
| 修改 CLB 带宽 | `tccli clb ModifyLoadBalancerSla` | 否 |
| Helm 安装/升级 | `helm install` / `helm upgrade` | 否 |

## 关键字段说明

以下说明 Helm Values 中 `controller.service.annotations` 的常用 CLB Annotation 及对应 tccli 操作。

| 注解（Annotation） | 说明 | 取值与约束 | 对应 tccli 参数 |
|------|------|------|------|
| `service.kubernetes.io/qcloud-loadbalancer-internet-charge-type` | 公网 CLB 计费类型 | `TRAFFIC_POSTPAID_BY_HOUR`（按流量）或 `BANDWIDTH_POSTPAID_BY_HOUR`（按带宽） | `CreateLoadBalancer.InternetAccessible.InternetChargeType` |
| `service.kubernetes.io/qcloud-loadbalancer-internet-max-bandwidth-out` | 公网 CLB 带宽上限（Mbps） | 1-2048 | `CreateLoadBalancer.InternetAccessible.InternetMaxBandwidthOut` |
| `service.kubernetes.io/qcloud-loadbalancer-internal-subnetid` | 内网 CLB 子网 ID | 格式 `subnet-xxxxxxxx`，`tccli vpc DescribeSubnets` 获取 | `CreateLoadBalancer.SubnetId` |
| `service.kubernetes.io/loadbalance-id` | 复用已有 CLB（指定已有 CLB ID 时不创建新 CLB） | 格式 `lb-xxxxxxxx` | 直接使用 `--LoadBalancerIds` 查询 |
| `service.kubernetes.io/qcloud-loadbalancer-master-zone-id` | 主可用区 ID | `ap-guangzhou-3` 等 | `CreateLoadBalancer.MasterZoneId` |
| `service.kubernetes.io/qcloud-loadbalancer-slave-zone-id` | 备可用区 ID（跨可用区容灾） | `ap-guangzhou-4` 等 | `CreateLoadBalancer.SlaveZoneId` |

## 操作步骤

### 步骤 1：手动创建 CLB（可选，不指定已有 CLB 时此步省略）

#### 选择依据

- **带宽计费模式**：测试环境选按流量（`TRAFFIC_POSTPAID_BY_HOUR`），生产环境高流量选按带宽（`BANDWIDTH_POSTPAID_BY_HOUR`）。
- **网络类型**：对外服务选公网 CLB，内部服务选内网 CLB 并绑定子网。

#### 创建公网 CLB

`clb-create.json`：

```json
{
  "LoadBalancerType": "OPEN",
  "LoadBalancerName": "CLB_NAME",
  "VpcId": "VPC_ID",
  "InternetAccessible": {
    "InternetChargeType": "TRAFFIC_POSTPAID_BY_HOUR",
    "InternetMaxBandwidthOut": 100
  }
}
```

```bash
tccli clb CreateLoadBalancer --region <Region> --cli-input-json file://clb-create.json
# expected: exit 0，返回 LoadBalancerIds
```

```json
{
  "LoadBalancerIds": ["lb-example"],
  "DealName": "20240617XXXX"
}
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `CLB_NAME` | CLB 实例名称 | 长度 1-60 | 自定义 |
| `VPC_ID` | VPC ID | 必须与集群 VPC 一致 | `tccli tke DescribeClusters --ClusterIds '["CLUSTER_ID"]' | jq '.Clusters[0].ClusterNetworkSettings.VpcId'` |

#### 创建内网 CLB

`clb-create-internal.json`：

```json
{
  "LoadBalancerType": "INTERNAL",
  "LoadBalancerName": "CLB_NAME",
  "VpcId": "VPC_ID",
  "SubnetId": "SUBNET_ID"
}
```

```bash
tccli clb CreateLoadBalancer --region <Region> --cli-input-json file://clb-create-internal.json
# expected: exit 0，返回 LoadBalancerIds
```

### 步骤 2：配置 Helm Values 自定义 CLB

#### 场景 A：自定义公网 CLB 带宽

`nginx-ingress-clb.yaml`：

```yaml
controller:
  service:
    type: LoadBalancer
    annotations:
      service.kubernetes.io/qcloud-loadbalancer-internet-charge-type: "TRAFFIC_POSTPAID_BY_HOUR"
      service.kubernetes.io/qcloud-loadbalancer-internet-max-bandwidth-out: "500"
```

```bash
helm upgrade nginx-ingress ingress-nginx/ingress-nginx \
    --namespace ingress-nginx \
    -f nginx-ingress-clb.yaml \
    --reuse-values
# expected: STATUS: deployed
```

> **kubectl 数据面不可达**：CAM 拒绝公网端点（strategyId:240463971），内网需 VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

#### 场景 B：使用内网 CLB

`nginx-ingress-clb-internal.yaml`：

```yaml
controller:
  service:
    type: LoadBalancer
    annotations:
      service.kubernetes.io/qcloud-loadbalancer-internal-subnetid: "SUBNET_ID"
```

```bash
helm upgrade nginx-ingress ingress-nginx/ingress-nginx \
    --namespace ingress-nginx \
    -f nginx-ingress-clb-internal.yaml \
    --reuse-values
# expected: STATUS: deployed
```

#### 场景 C：复用已有 CLB

`nginx-ingress-clb-existing.yaml`：

```yaml
controller:
  service:
    type: LoadBalancer
    annotations:
      service.kubernetes.io/loadbalance-id: "CLB_ID"
```

```bash
helm upgrade nginx-ingress ingress-nginx/ingress-nginx \
    --namespace ingress-nginx \
    -f nginx-ingress-clb-existing.yaml \
    --reuse-values
# expected: STATUS: deployed
```

#### 场景 D：跨可用区容灾（主备 CLB）

`nginx-ingress-clb-multi-az.yaml`：

```yaml
controller:
  service:
    type: LoadBalancer
    annotations:
      service.kubernetes.io/qcloud-loadbalancer-master-zone-id: "ZONE_ID_MASTER"
      service.kubernetes.io/qcloud-loadbalancer-slave-zone-id: "ZONE_ID_SLAVE"
```

```bash
helm upgrade nginx-ingress ingress-nginx/ingress-nginx \
    --namespace ingress-nginx \
    -f nginx-ingress-clb-multi-az.yaml \
    --reuse-values
# expected: STATUS: deployed
```

### 步骤 3：修改已有 CLB 属性（使用 tccli）

```bash
# 修改带宽上限
tccli clb ModifyLoadBalancerSla --region <Region> \
    --LoadBalancerId CLB_ID \
    --SlaType clb.c3.medium
# expected: exit 0

# 修改 CLB 名称
tccli clb ModifyLoadBalancerAttributes --region <Region> \
    --LoadBalancerId CLB_ID \
    --LoadBalancerName NEW_CLB_NAME
# expected: exit 0
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `CLB_ID` | CLB 实例 ID | 格式 `lb-xxxxxxxx` | `kubectl get svc -n ingress-nginx` 注解或 `tccli clb DescribeLoadBalancers` |
| `SUBNET_ID` | 内网 CLB 子网 ID | 格式 `subnet-xxxxxxxx` | `tccli vpc DescribeSubnets --region <Region>` |
| `ZONE_ID_MASTER` | 主可用区 | 如 `ap-guangzhou-3` | `tccli clb DescribeMasterZones --region <Region>` |
| `ZONE_ID_SLAVE` | 备可用区 | 如 `ap-guangzhou-4` | 同上 |

## 验证

### 控制面（tccli）

```bash
# 验证 CLB 属性
tccli clb DescribeLoadBalancers --region <Region> \
    --LoadBalancerIds '["CLB_ID"]'
# expected: 返回 LoadBalancerSet，确认 InternetMaxBandwidthOut、LoadBalancerType 等属性与配置一致
```

```json
{
  "LoadBalancerSet": [
    {
      "LoadBalancerId": "lb-example",
      "LoadBalancerName": "nginx-ingress",
      "LoadBalancerType": "OPEN",
      "Status": 1,
      "InternetAccessible": {
        "InternetChargeType": "TRAFFIC_POSTPAID_BY_HOUR",
        "InternetMaxBandwidthOut": 500
      },
      "MasterZone": {"Zone": "ap-guangzhou-3"},
      "LoadBalancerVips": ["EXTERNAL_IP"]
    }
  ]
}
```

### 数据面（需 VPN/IOA）

```bash
# 验证 Service 注解生效
kubectl -n ingress-nginx get svc nginx-ingress-ingress-nginx-controller -o yaml | grep -A5 annotations
# expected: 包含配置的 qcloud-loadbalancer 注解

# 验证 Service 获得 CLB IP
kubectl -n ingress-nginx get svc nginx-ingress-ingress-nginx-controller
# expected: EXTERNAL-IP 不为空

# 验证 CLB 关联的 Service
kubectl -n ingress-nginx describe svc nginx-ingress-ingress-nginx-controller
# expected: Events 无异常
```

## 清理

### 数据面（需 VPN/IOA）

> **⚠️ 警告**：如果使用了 `loadbalance-id` 注解复用了已有 CLB，卸载不会删除该 CLB。如果未使用该注解（自动创建 CLB），卸载 Service 时会自动删除 CLB。

```bash
# 如使用了自动创建的 CLB，先记录 CLB ID
kubectl -n ingress-nginx get svc nginx-ingress-ingress-nginx-controller \
    -o jsonpath='{.metadata.annotations.service\.kubernetes\.io/loadbalance-id}'
# expected: CLB ID

# 卸载 Nginx Ingress（会删除自动创建的 CLB）
helm uninstall nginx-ingress -n ingress-nginx
# expected: release "nginx-ingress" uninstalled
```

### 控制面（tccli）

```bash
# 如果使用的是已有 CLB（不受 Helm 卸载影响），手动清理监听器
tccli clb DescribeListeners --region <Region> --LoadBalancerId CLB_ID
# 记录 80/443 监听器的 ListenerId

tccli clb DeleteListener --region <Region> \
    --LoadBalancerId CLB_ID \
    --ListenerId LISTENER_ID
# expected: exit 0

# 验证 CLB 监听器已删除
tccli clb DescribeListeners --region <Region> --LoadBalancerId CLB_ID
# expected: Listeners 为空或仅含非 Nginx Ingress 的监听器
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `CreateLoadBalancer` 返回 `LimitExceeded` | `tccli clb DescribeQuota --region <Region>` | CLB 配额达到上限（此为环境限制，非命令错误） | 删除不再使用的 CLB 后重试，或提交工单申请配额提升 |
| `CreateLoadBalancer` 返回 `InvalidParameterValue.SubnetId` | `tccli vpc DescribeSubnets --region <Region> --SubnetIds '["SUBNET_ID"]'` | 子网 ID 不存在或格式错误 | 使用正确的 `subnet-xxxxxxxx` 格式 |
| `helm upgrade` 后 Service 未更新 | `kubectl -n ingress-nginx get svc -o yaml` 对比注解 | `--reuse-values` 未传递或 values 文件路径错误 | 确认 values 文件路径正确，无需 `--reuse-values` 时使用 `--reset-values` |

### 配置生效后功能异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| CLB EXTERNAL-IP 一直 `<pending>` | `kubectl -n ingress-nginx describe svc` 查看 Events | 子网 IP 不足或 CLB 创建失败 | 检查 Events 中的错误信息；`tccli vpc DescribeSubnets` 检查 `AvailableIpCount` |
| 带宽修改不生效 | `tccli clb DescribeLoadBalancers --region <Region> --LoadBalancerIds '["CLB_ID"]'` 查看 InternetAccessible | `ModifyLoadBalancerSla` 需要 CLB 实例支持 SLA 规格变更 | 确认 CLB 实例类型支持 SLA 变更，或使用 `ModifyLoadBalancerAttributes` 修改 |
| 跨可用区配置无效 | `tccli clb DescribeLoadBalancers --region <Region> --LoadBalancerIds '["CLB_ID"]'` | CLB 类型不支持主备可用区（内网 CLB 不支持） | 公网 CLB 方可配置主备可用区 |
| 复用已有 CLB 后访问失败 | `tccli clb DescribeListeners --region <Region> --LoadBalancerId CLB_ID` | 已有监听器端口冲突 | 删除冲突监听器或更换 CLB |

## 下一步

- [启用 CLB 直连](https://cloud.tencent.com/document/product/457/104866) — CLB 直连 Pod，跳过 NodePort 转发
- [高可用配置优化](https://cloud.tencent.com/document/product/457/104860) — 多副本、PDB、健康检查
- [接入腾讯云 WAF](https://cloud.tencent.com/document/product/457/104862) — 为 Ingress 接入 WAF 防护

## 控制台替代

通过 [TKE 控制台 - 组件管理](https://console.cloud.tencent.com/tke2/cluster/sub/list/addon/cluster?rid=1) 或 [CLB 控制台](https://console.cloud.tencent.com/clb) 管理负载均衡器配置。
