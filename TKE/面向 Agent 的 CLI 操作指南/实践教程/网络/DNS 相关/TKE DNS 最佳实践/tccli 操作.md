# TKE DNS 最佳实践（tccli）

> 对照官方：[集群 DNS 最佳实践](https://cloud.tencent.com/document/product/457/78005) · page_id `78005`

## 概述

TKE 集群中 DNS 解析是应用通信的核心基础设施。本文涵盖 TKE 集群 DNS 的三大最佳实践：CoreDNS 高可用配置、NodeLocal DNS Cache 加速、以及 CoreDNS 性能调优。涉及控制面组件安装（tccli）和数据面配置（kubectl）。

> **kubectl 数据面不可达**：CAM 拒绝公网端点（strategyId:240463971），内网需 VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

## 前置条件

- [环境准备](../../../../../../../环境准备.md)
- 集群已安装 CoreDNS（TKE 托管集群默认安装）

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:DescribeClusters, tke:InstallAddon, tke:DescribeAddon,
#    tke:UpdateAddon, tke:DescribeAddonValues
tccli tke DescribeClusters --region <Region>
# expected: exit 0，返回集群列表

# 4. 检查 kubectl 版本（数据面操作需要）
kubectl version --client
# expected: Client Version >= v1.30.0
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
# 5. 确认目标集群存在且状态正常
tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
# expected: TotalCount >= 1，ClusterStatus: "Running"
```

**预期输出**：

```json
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "cls-example",
            "ClusterStatus": "Running",
            "ClusterType": "MANAGED_CLUSTER",
            "K8sVersion": "1.30.0"
        }
    ]
}
```

```bash
# 6. 需 kubectl 可达环境 — 确认 CoreDNS 已安装且正常运行
kubectl get deploy -n kube-system coredns
# expected: READY 副本数 >= 2
```

**预期输出**：

```text
NAME      READY   UP-TO-DATE   AVAILABLE   AGE
coredns   2/2     2            2           30d
```

```bash
# 7. 需 kubectl 可达环境 — 确认 CoreDNS ConfigMap 存在
kubectl get cm -n kube-system coredns
# expected: NAME: coredns
```

```text
NAME  STATUS  AGE
...
```

### 版本与规格选择

- CoreDNS 版本：通过 `tccli tke DescribeAddonValues --region <Region> --ClusterId CLUSTER_ID --AddonName coredns` 查询可用版本。
- NodeLocal DNS Cache 版本：通过 `tccli tke DescribeAddonValues --region <Region> --ClusterId CLUSTER_ID --AddonName NodeLocalDNS` 查询可用版本。

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看 CoreDNS 组件状态 | `DescribeAddon --AddonName coredns` | 是 |
| 升级 CoreDNS 版本 | `UpdateAddon --AddonName coredns` | 否 |
| 安装 NodeLocal DNS Cache | `InstallAddon --AddonName NodeLocalDNS` | 否 |
| 查看 NodeLocal DNS Cache 状态 | `DescribeAddon --AddonName NodeLocalDNS` | 是 |
| 配置 CoreDNS 副本数/资源 | 控制台操作（组件管理 → CoreDNS → 更新配置），对应 `UpdateAddon` | 否 |

## 操作步骤

### 步骤 1：CoreDNS 高可用配置 — 副本数调整

#### 选择依据

CoreDNS 默认副本数为 2。生产环境建议按以下规则调整：
- 集群节点数 <= 5：保持 2 副本。
- 集群节点数 6~50：设置 3~5 副本。
- 集群节点数 > 50：按 `ceil(节点数 / 10)` 计算，上限 20。
- 多副本增加 DNS 查询吞吐量和容错能力，但消耗更多节点资源。

#### 最小配置（调整副本数）

```bash
tccli tke UpdateAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName coredns \
    --AddonVersion COREDNS_VERSION \
    --RawValues '{"replicas":3}'
# expected: exit 0，返回 RequestId
```

**预期输出**：

```json
{
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

#### 增强配置（副本数 + 资源限制 + 反亲和策略）

```bash
tccli tke UpdateAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName coredns \
    --AddonVersion COREDNS_VERSION \
    --RawValues '{
  "replicas": 3,
  "resources": {
    "limits": {"cpu": "500m", "memory": "256Mi"},
    "requests": {"cpu": "200m", "memory": "128Mi"}
  },
  "affinity": {
    "podAntiAffinity": {
      "preferredDuringSchedulingIgnoredDuringExecution": [
        {
          "weight": 100,
          "podAffinityTerm": {
            "topologyKey": "kubernetes.io/hostname",
            "labelSelector": {
              "matchLabels": {"k8s-app": "kube-dns"}
            }
          }
        }
      ]
    }
  }
}'
# expected: exit 0，返回 RequestId
```

| 占位符 | 说明 | 获取方式 |
|--------|------|---------|
| `COREDNS_VERSION` | CoreDNS 当前版本 | `tccli tke DescribeAddon --region <Region> --ClusterId CLUSTER_ID --AddonName coredns` |

### 步骤 2：安装 NodeLocal DNS Cache

#### 选择依据

NodeLocal DNS Cache 以 DaemonSet 形式在每个节点运行本地 DNS 缓存代理，Pod DNS 查询通过本地缓存代理转发到 CoreDNS，命中缓存则直接返回（无网络跳转），降低 DNS 查询延迟（从 ~10ms 降至 ~1ms），减少 CoreDNS 负载，提升集群 DNS 可用性。

**适用场景**：
- 集群规模 > 20 节点
- 应用 DNS 查询密集（如微服务互相调用频繁）
- 对 DNS 延迟敏感的业务

**不适用场景**：
- 小型测试集群（节点数 <= 5）
- DNS 查询量极低的场景

#### 最小安装（不指定版本，使用默认版本）

```bash
tccli tke InstallAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName NodeLocalDNS
# expected: exit 0，返回 RequestId
```

**预期输出**：

```json
{
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

#### 增强配置（指定版本 + 自定义缓存参数）

```bash
tccli tke InstallAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName NodeLocalDNS \
    --AddonVersion NODELOCALDNS_VERSION
# expected: exit 0，返回 RequestId
```

```bash
# 需 kubectl 可达环境 — 安装后检查 CoreDNS ConfigMap，确认 upstream 配置指向 NodeLocal DNS Cache
kubectl get cm -n kube-system coredns -o yaml | grep -A5 "169.254.20.10"
# expected: 输出包含 169.254.20.10
```

```text
NAME  STATUS  AGE
...
```

| 占位符 | 说明 | 获取方式 |
|--------|------|---------|
| `NODELOCALDNS_VERSION` | NodeLocal DNS Cache 版本 | `tccli tke DescribeAddonValues --region <Region> --ClusterId CLUSTER_ID --AddonName NodeLocalDNS` |

### 步骤 3：轮询组件安装状态

```bash
tccli tke DescribeAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName NodeLocalDNS
# expected: Phase: "Succeeded", Status: "Running"
```

**预期输出**：

```json
{
    "Addon": {
        "AddonName": "NodeLocalDNS",
        "AddonVersion": "1.0.0",
        "Phase": "Succeeded",
        "Status": "Running"
    }
}
```

## 验证

### 控制面（tccli）

| 维度 | 命令 | 预期 |
|------|------|------|
| CoreDNS 状态 | `tccli tke DescribeAddon --region <Region> --ClusterId CLUSTER_ID --AddonName coredns` | `Status: "Running"` |
| NodeLocalDNS 状态 | `tccli tke DescribeAddon --region <Region> --ClusterId CLUSTER_ID --AddonName NodeLocalDNS` | `Status: "Running"` |
| 集群状态 | `tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'` | `ClusterStatus: "Running"` |

### 数据面（需 VPN/IOA）

```bash
# 需 kubectl 可达环境 — 维度 1：确认 CoreDNS Pod 全部 Running
kubectl get pods -n kube-system -l k8s-app=kube-dns
# expected: 所有 Pod STATUS: Running，副本数与配置一致

# 需 kubectl 可达环境 — 维度 2：确认 NodeLocal DNS Cache DaemonSet 全部 Ready
kubectl get ds -n kube-system node-local-dns
# expected: DESIRED = CURRENT = READY = UP-TO-DATE = AVAILABLE

# 需 kubectl 可达环境 — 维度 3：从 Pod 内测试 DNS 解析
kubectl run dns-test --image=busybox:1.36 --rm -it --restart=Never -- nslookup kubernetes.default
# expected: 返回 kubernetes.default.svc.cluster.local 的 Service IP
```

```text
NAME  STATUS  AGE
...
```

| 维度 | 命令 | 预期 |
|------|------|------|
| CoreDNS Pod | `kubectl get pods -n kube-system -l k8s-app=kube-dns` | 全部 Running |
| NodeLocalDNS DS | `kubectl get ds -n kube-system node-local-dns` | DESIRED = READY |
| DNS 解析 | `nslookup kubernetes.default` | 返回 Service IP |

## 清理

> **警告**：卸载 NodeLocal DNS Cache 将导致集群所有 DNS 查询回退到 CoreDNS 直接处理，DNS 延迟可能增加。生产环境卸载前需确认业务可容忍延迟增加。

### 数据面（需 VPN/IOA）

```bash
# 需 kubectl 可达环境 — 清理前确认 NodeLocal DNS Cache 状态
kubectl get ds -n kube-system node-local-dns
# 记录当前状态
```

```text
NAME  STATUS  AGE
...
```

### 控制面（tccli）

```bash
# 清理前状态检查
tccli tke DescribeAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName NodeLocalDNS
# expected: 确认是待清理的目标组件
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

```bash
# 卸载 NodeLocal DNS Cache
tccli tke DeleteAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName NodeLocalDNS
# expected: exit 0，返回 RequestId
```

```bash
# 验证已卸载
tccli tke DescribeAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName NodeLocalDNS
# expected: ResourceNotFound 或 AddonName 不在列表中
```

**预期输出**：

```json
{
    "Error": {
        "Code": "ResourceNotFound",
        "Message": "Addon not found"
    }
}
```

> **计费提醒**：NodeLocal DNS Cache 作为 DaemonSet 在每个节点运行一个 Pod，每个 Pod 消耗 CPU/内存资源（默认 requests: cpu=25m, memory=20Mi），节点数越多额外资源消耗越大。清理后可释放这部分资源。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl` 命令超时或连接被拒绝 | `tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]'`（控制面返回 Running） | CAM 策略 strategyId:240463971 拒绝公网端点，数据面需内网访问 | 通过 VPN/IOA 接入 VPC 内网后执行 kubectl；或使用控制台替代 |
| `InstallAddon NodeLocalDNS` 返回 `InvalidParameter.AddonName` | 检查 AddonName 大小写：`NodeLocalDNS` | AddonName 大小写不匹配 | 使用精确名称 `NodeLocalDNS`（N、L、D、N、S 大写） |
| `UpdateAddon coredns` 返回 "addon version not found" | `tccli tke DescribeAddonValues --region <Region> --ClusterId CLUSTER_ID --AddonName coredns` 检查版本 | 指定的 AddonVersion 不存在 | 从 DescribeAddonValues 输出获取正确版本号 |
| `InstallAddon` 返回 `FailedOperation.AddonAlreadyInstalled` | `tccli tke DescribeAddon --region <Region> --ClusterId CLUSTER_ID --AddonName NodeLocalDNS` | 组件已安装 | 如需重新安装，先 `DeleteAddon` 再 `InstallAddon` |

### DNS 解析异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Pod 内 DNS 解析超时 | `kubectl logs -n kube-system -l k8s-app=kube-dns --tail=50` | CoreDNS 副本不足或资源限制过低 | 增加 CoreDNS 副本数，提升 CPU/内存 limits |
| NodeLocal DNS Cache 安装后 DNS 查询仍走 CoreDNS | `kubectl get cm -n kube-system coredns -o yaml \| grep -A10 proxy` | CoreDNS ConfigMap 未正确配置 upstream | 确认 ConfigMap 中 proxy 段指向 169.254.20.10 |
| NodeLocal DNS Cache Pod CrashLoopBackOff | `kubectl logs -n kube-system node-local-dns-POD_NAME` | 端口冲突或 conntrack 表满 | 检查节点 53 端口占用；增大 `nf_conntrack_max` |
| CoreDNS OOMKilled | `kubectl describe pod -n kube-system coredns-POD_NAME` | 内存限制过低或 DNS 查询量过大 | 增加 memory limits 至 512Mi；考虑启用 cache 插件加大缓存 |
| DNS 查询返回 NXDOMAIN 但 Service 存在 | `kubectl get svc --all-namespaces \| grep SERVICE_NAME` | CoreDNS 缓存了过期的 NXDOMAIN 记录 | `kubectl rollout restart deploy -n kube-system coredns` 重启 CoreDNS |

## 下一步

- [在 TKE 集群中使用 NodeLocal DNS Cache](../在%20TKE%20集群中使用%20NodeLocal%20DNS%20Cache/tccli%20操作.md) — NodeLocal DNS Cache 详细配置
- [在 TKE 中实现自定义域名解析](../在%20TKE%20中实现自定义域名解析/tccli%20操作.md) — CoreDNS 自定义域名配置
- [CoreDNS 日志仪表盘使用指南](../CoreDNS%20日志仪表盘使用指南/tccli%20操作.md) — DNS 可观测性

## 控制台替代

登录 [容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) → 选择集群 → **组件管理** → 找到 `coredns` 组件 → **更新配置** 调整副本数/资源 → 找到 `NodeLocalDNS` 组件 → **安装**。
