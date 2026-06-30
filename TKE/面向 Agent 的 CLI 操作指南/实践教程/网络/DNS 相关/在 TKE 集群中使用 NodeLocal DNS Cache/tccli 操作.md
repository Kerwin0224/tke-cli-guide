# 在 TKE 集群中使用 NodeLocal DNS Cache（tccli）

> 对照官方：[在 TKE 集群中使用 NodeLocal DNS Cache](https://cloud.tencent.com/document/product/457/40613) · page_id `40613`

## 概述

NodeLocal DNS Cache 以 DaemonSet 形式在每个节点上运行 DNS 缓存代理，拦截 Pod 的 DNS 查询请求，本地缓存命中后直接返回结果，未命中再转发至 CoreDNS。这显著降低了 DNS 查询延迟和 CoreDNS 负载，减少 conntrack 表条目数量，避免高并发场景下的 DNS 解析超时。

NodeLocal DNS Cache 监听 `169.254.20.10:53`（链路本地地址），Pod 的 `/etc/resolv.conf` 指向该地址而非 CoreDNS Service ClusterIP，从而绕过 iptables DNAT 转发。

> **kubectl 数据面不可达**：CAM拒绝公网端点(strategyId:240463971)，内网需VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

## 前置条件

- [环境准备](../../../../环境准备.md)
- 集群类型为 MANAGED_CLUSTER，Kubernetes 版本 >= 1.30.0
- CoreDNS 组件已安装且运行正常

### 环境检查

```bash
# 1. 检查 tccli 版本和当前凭据
tccli --version
# expected: tccli version >= 1.0.0

tccli configure list
# expected: secretId, secretKey, region 均已配置

# 2. 检查 CAM 权限 — 需要以下 Action 名
#    tke:DescribeClusters, tke:InstallAddon, tke:DescribeAddon, tke:DeleteAddon, tke:DescribeAddonValues
# 验证：
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
            "ClusterStatus": "Running"
        }
    ]
}
```

### 资源检查

```bash
# 3. 确认 CoreDNS 组件已安装且运行正常
tccli tke DescribeAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName coredns
# expected: Phase: "running"
```

**预期输出**：

```json
{
    "AddonName": "coredns",
    "AddonVersion": "1.30.0",
    "Phase": "running",
    "RequestId": "..."
}
```

```bash
# 4. 查看节点数量（NodeLocal DNS 每节点部署一个 Pod，资源消耗随节点数线性增长）
tccli tke DescribeClusterInstances --region <Region> \
    --ClusterId CLUSTER_ID \
    --offset 0 --limit 100 \
    --output json | jq '.InstanceSet | length'
# expected: 节点数，用于评估资源开销
```

```json
{
  "TotalCount": "<TotalCount>",
  "InstanceSet": "<InstanceSet>",
  "InstanceId": "<InstanceId>",
  "InstanceRole": "<InstanceRole>",
  "FailedReason": "<FailedReason>",
  "InstanceState": "<InstanceState>"
}
```

```bash
# 5. 确认 NodeLocal DNS Cache 未安装
tccli tke DescribeAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName NodeLocalDNS
# expected: 如已安装返回组件详情；未安装返回 ResourceNotFound
```

**预期输出**（未安装时）：

```json
{
    "Error": {
        "Code": "ResourceNotFound",
        "Message": "Addon not found"
    }
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli / kubectl 命令 | 幂等 |
|-----------|----------------------|:--:|
| 安装 NodeLocal DNS Cache | `tccli tke InstallAddon --cli-input-json file://install.json` | 是（已安装则忽略） |
| 查看组件状态 | `tccli tke DescribeAddon --ClusterId CLUSTER_ID --AddonName NodeLocalDNS` | 是 |
| 查看组件可用参数 | `tccli tke DescribeAddonValues --ClusterId CLUSTER_ID --AddonName NodeLocalDNS` | 是 |
| 卸载组件 | `tccli tke DeleteAddon --ClusterId CLUSTER_ID --AddonName NodeLocalDNS` | 否 |
| 查看 DaemonSet | `kubectl get ds node-local-dns -n kube-system`（需 VPN/IOA） | 是 |
| 查看 Pod resolv.conf | `kubectl exec POD_NAME -- cat /etc/resolv.conf`（需 VPN/IOA） | 是 |
| 查看 CoreDNS ConfigMap | `kubectl get cm coredns -n kube-system -o yaml`（需 VPN/IOA） | 是 |

## 关键字段说明

### InstallAddon 参数

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `ClusterId` | String | 是 | 目标集群 ID，格式 `cls-xxxxxxxx` | 集群不存在 → `InvalidParameter.ClusterId` |
| `AddonName` | String | 是 | `NodeLocalDNS`，大小写敏感 | 名称错误 → `InvalidParameter.AddonName` |
| `AddonVersion` | String | 否 | 组件版本号。不指定则安装最新版 | 版本不存在 → `InvalidParameter.AddonVersion` |
| `RawValues` | String | 否 | addon 参数，base64 编码的 JSON 字符串 | 格式错误 → `InvalidParameter.RawValues` |

### RawValues 参数

| 字段 | 类型 | 必填 | 取值与约束 | 默认值 |
|------|------|:--:|------|--------|
| `forward.upstreamServers` | Array | 否 | 上游 DNS 服务器地址列表 | CoreDNS Service ClusterIP |
| `prometheus.scrape` | Boolean | 否 | 是否开启 Prometheus 指标采集 | `false` |

## 操作步骤

### 步骤 1：安装 NodeLocal DNS Cache 组件

#### 选择依据

- **NodeLocal DNS Cache** 在节点本地监听 `169.254.20.10:53`，Pod DNS 请求先命中本地缓存（命中率通常 80%+），未命中再转发 CoreDNS。
- **适用场景**：集群节点数 > 20，或 CoreDNS 频繁出现高负载/查询超时，或应用对 DNS 延迟敏感（如微服务高频服务发现）。
- **不适用**：集群节点数 < 10（每节点额外消耗约 50MB 内存 + 0.1 CPU，小集群资源开销不划算）；已使用自定义 DNS 方案且不兼容 NodeLocal DNS Cache 的场景。
- **安装时机**：建议在集群已运行 CoreDNS 且无 DNS 异常时安装，作为预防性优化而非故障应急。

#### 最小安装（仅必填字段）

```bash
tccli tke InstallAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName NodeLocalDNS
# expected: exit 0，返回 RequestId，组件开始安装
```

**预期输出**：

```json
{
    "RequestId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
}
```

#### 增强配置（自定义 DNS 上游 + Prometheus）

构建 RawValues JSON：

```json
{
    "forward": {
        "upstreamServers": ["CLUSTER_DNS_IP"]
    },
    "prometheus": {
        "scrape": true
    }
}
```

将上述 JSON 做 base64 编码后赋值给 `RawValues`。

`install-nodelocaldns-enhanced.json`：

```json
{
    "ClusterId": "CLUSTER_ID",
    "AddonName": "NodeLocalDNS",
    "RawValues": "BASE64_ENCODED_RAW_VALUES"
}
```

> `BASE64_ENCODED_RAW_VALUES` = `echo -n '{"forward":{"upstreamServers":["CLUSTER_DNS_IP"]},"prometheus":{"scrape":true}}' | base64`

```bash
tccli tke InstallAddon --region <Region> \
    --cli-input-json file://install-nodelocaldns-enhanced.json
# expected: exit 0，返回 RequestId
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `CLUSTER_ID` | 目标集群 ID | 格式 `cls-xxxxxxxx` | `tccli tke DescribeClusters --region <Region>` |
| `REGION` | 地域 | 如 `ap-guangzhou` | `tccli configure list` |
| `CLUSTER_DNS_IP` | CoreDNS Service ClusterIP | 如 `10.247.0.10` | `kubectl get svc -n kube-system coredns -o jsonpath='{.spec.clusterIP}'` |
| `ADDON_VERSION` | NodeLocal DNS 版本 | 如 `1.22.20` | `tccli tke DescribeAddonValues --ClusterId CLUSTER_ID --AddonName NodeLocalDNS` |

### 步骤 2：轮询组件安装状态

```bash
tccli tke DescribeAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName NodeLocalDNS
# expected: Phase: "running"
```

**预期输出**：

```json
{
    "AddonName": "NodeLocalDNS",
    "AddonVersion": "1.22.20",
    "Phase": "running",
    "RawValues": "...",
    "RequestId": "..."
}
```

> 安装通常需要 30-60 秒，期间 `Phase` 可能为 `creating`。如超过 5 分钟仍为 `failed`，参见排障章节。

### 步骤 3：验证节点部署和数据面配置

数据面操作（需 VPN/IOA）：

```bash
# 确认 DaemonSet 在所有节点上运行
kubectl get ds node-local-dns -n kube-system
# expected: DESIRED == CURRENT == READY == UP-TO-DATE == AVAILABLE == 节点数
```

**预期输出**：

```text
NAME             DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
node-local-dns   5         5         5       5            5           <none>          2m
```

```bash
# 确认 Pod resolv.conf 指向 NodeLocal DNS Cache
kubectl run -it --rm dns-test --image=busybox:1.28 --restart=Never -- \
    cat /etc/resolv.conf
# expected: nameserver 169.254.20.10
```

**预期输出**：

```text
nameserver 169.254.20.10
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

## 验证

### 控制面（tccli）

```bash
# 维度 1：确认组件状态正常
tccli tke DescribeAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName NodeLocalDNS
# expected: Phase: "running"
```

**预期输出**：

```json
{
    "AddonName": "NodeLocalDNS",
    "AddonVersion": "1.22.20",
    "Phase": "running",
    "RequestId": "..."
}
```

### 数据面（需 VPN/IOA）

```bash
# 维度 2：确认 DaemonSet 每节点一个 Pod
kubectl get pods -n kube-system -l k8s-app=node-local-dns -o wide
# expected: 每个节点一个 Pod，STATUS 为 Running
```

**预期输出**：

```text
NAME                   READY   STATUS    RESTARTS   AGE   IP              NODE         NOMINATED NODE   READINESS GATES
node-local-dns-abc12   1/1     Running   0          3m    169.254.20.10   node-ex-001   <none>           <none>
node-local-dns-def34   1/1     Running   0          3m    169.254.20.10   node-ex-002   <none>           <none>
```

```bash
# 维度 3：测试 DNS 解析走本地缓存
kubectl run -it --rm dns-test --image=busybox:1.28 --restart=Never -- \
    nslookup kubernetes.default.svc.cluster.local
# expected: 解析成功，返回 ClusterIP
```

**预期输出**：

```text
Server:        169.254.20.10
Address:       169.254.20.10:53

Name:   kubernetes.default.svc.cluster.local
Address: 10.247.0.1
```

```bash
# 维度 4：确认 CoreDNS ConfigMap forward 配置
kubectl get configmap coredns -n kube-system -o jsonpath='{.data.Corefile}'
# expected: Corefile forward 指向 CoreDNS Service ClusterIP（NodeLocal DNS 转发上游）
```

```text
NAME  STATUS  AGE
...
```

| 验证维度 | 命令 | 预期 |
|---------|------|------|
| 组件状态 | `DescribeAddon --AddonName NodeLocalDNS` | `Phase: "running"` |
| DaemonSet 部署 | `kubectl get ds node-local-dns -n kube-system` | DESIRED == READY == 节点数 |
| Pod resolv.conf | `kubectl exec POD_NAME -- cat /etc/resolv.conf` | `nameserver 169.254.20.10` |
| DNS 解析 | `kubectl run dns-test -- nslookup kubernetes.default.svc.cluster.local` | 解析成功 |
| CoreDNS forward | `kubectl get cm coredns -n kube-system` | forward 指向 CoreDNS ClusterIP |

## 清理

> **副作用警告**：卸载 NodeLocal DNS Cache 后，新建 Pod 的 `/etc/resolv.conf` 会恢复指向 CoreDNS Service ClusterIP。已有 Pod 需重启才会恢复。卸载期间 DNS 解析不中断（CoreDNS 仍正常运行）。
> **资源开销警告**：NodeLocal DNS Cache 每节点占用约 50MB 内存 + 0.1 CPU。卸载后这些资源释放回节点可分配池。

### 控制面（tccli）

```bash
# 清理前状态检查
tccli tke DescribeAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName NodeLocalDNS
# expected: 确认组件当前状态为 running
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
# 卸载组件
tccli tke DeleteAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName NodeLocalDNS
# expected: exit 0，返回 RequestId
```

**预期输出**：

```json
{
    "RequestId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
}
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

### 数据面（需 VPN/IOA）

```bash
# 确认 DaemonSet 已删除
kubectl get ds node-local-dns -n kube-system
# expected: Error from server (NotFound)
```

**预期输出**：

```text
Error from server (NotFound): daemonsets.apps "node-local-dns" not found
```

```bash
# 确认新建 Pod 的 resolv.conf 已恢复指向 CoreDNS
kubectl run -it --rm dns-test --image=busybox:1.28 --restart=Never -- \
    cat /etc/resolv.conf
# expected: nameserver 指向 CoreDNS Service ClusterIP（如 10.247.0.10）
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl` 命令超时或连接被拒绝 | `tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]'`（控制面返回 Running） | CAM 策略 strategyId:240463971 拒绝公网端点，数据面需内网访问 | 通过 VPN/IOA 接入 VPC 内网后执行 kubectl；或使用控制台替代 |
| `InstallAddon` 返回 `InvalidParameter.AddonName` | 检查 AddonName 大小写 | AddonName 大小写不匹配 | 使用精确名称 `NodeLocalDNS`（N、L、D 大写） |
| `InstallAddon` 返回 `FailedOperation.AddonAlreadyExists` | `tccli tke DescribeAddon --ClusterId CLUSTER_ID --AddonName NodeLocalDNS` | 组件已安装 | 如状态异常，先 `DeleteAddon` 卸载后重新安装 |
| `InstallAddon` 返回 `LimitExceeded` | `tccli tke DescribeClusters` 检查集群状态 | 集群资源不足或配额达上限（此为环境限制，非命令错误） | 扩容集群或等待其他安装任务完成 |
| `DescribeAddon` 返回 `Phase: "failed"` | 查看 `Reason` 字段 | 节点资源不足或端口冲突（9253 端口被占用） | 扩容节点或排查端口占用 |

### 安装成功但 DNS 解析异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| node-local-dns Pod CrashLoopBackOff | `kubectl logs -n kube-system -l k8s-app=node-local-dns --tail=20` | 9253 端口被占用，或节点 169.254.20.10 地址冲突 | 检查 `kubectl describe pod -n kube-system -l k8s-app=node-local-dns` Events；确认 9253 端口未占用 |
| 新建 Pod 仍使用 CoreDNS 直连 | `kubectl exec POD_NAME -- cat /etc/resolv.conf` | Pod `dnsPolicy` 设为 `Default` 或 `None`，未走集群 DNS | 设置 `dnsPolicy: ClusterFirst` 或 `ClusterFirstWithHostNet`；NodeLocal DNS Cache 仅对 ClusterFirst 生效 |
| DNS 解析偶发超时 | `kubectl logs -n kube-system -l k8s-app=node-local-dns --tail=50` | conntrack 表满或上游 CoreDNS 负载高 | 检查 `forward.upstreamServers` 配置；确认 CoreDNS 副本数充足 |
| 解析结果不一致 | `kubectl exec -n kube-system NODELOCAL_POD -- nslookup test.example.com` | NodeLocal DNS Cache 缓存 TTL 过长 | 检查 CoreDNS ConfigMap 中 `cache` TTL 设置；降低缓存时间 |
| 已有 Pod 未使用 NodeLocal DNS | `kubectl exec EXISTING_POD -- cat /etc/resolv.conf` | 已有 Pod 的 `/etc/resolv.conf` 在安装前已生成，不会自动更新 | 重启已有 Pod（`kubectl rollout restart deployment/DEPLOYMENT_NAME`）使其使用新的 resolv.conf |

## 下一步

- [TKE DNS 最佳实践](../TKE%20DNS%20最佳实践/tccli%20操作.md) -- page_id `78005`
- [在 TKE 中实现自定义域名解析](../在%20TKE%20中实现自定义域名解析/tccli%20操作.md) -- page_id `50865`
- [CoreDNS 日志仪表盘使用指南](../CoreDNS%20日志仪表盘使用指南/tccli%20操作.md) -- page_id `109060`
- [集群 DNS 解析异常排障处理](../../../../故障处理/集群%20DNS%20解析异常排障处理/tccli%20操作.md) -- page_id `80531`

## 控制台替代

登录 [容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) → 选择集群 → **组件管理** → **新建** → 搜索 `NodeLocalDNS` → 选择版本并安装 → 在 NodeLocal DNS Cache 详情页配置上游 DNS 和 Prometheus 采集。
