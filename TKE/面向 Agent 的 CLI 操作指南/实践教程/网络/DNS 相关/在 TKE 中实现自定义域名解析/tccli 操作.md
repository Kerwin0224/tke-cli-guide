# 在 TKE 中实现自定义域名解析（tccli）

> 对照官方：[在 TKE 中实现自定义域名解析](https://cloud.tencent.com/document/product/457/50865) · page_id `50865`

## 概述

通过 CoreDNS 的 `hosts` 插件实现集群内自定义域名解析，将指定域名映射到任意 IP。无需修改应用代码，即可在集群内实现内网服务发现、私有 DNS 覆盖、灰度域名等场景。

核心操作路径：备份 CoreDNS ConfigMap → 编辑 ConfigMap 添加 `hosts` 插件条目 → CoreDNS 自动 reload 或手动重启 → 验证解析。

> **kubectl 数据面不可达**：CAM拒绝公网端点(strategyId:240463971)，内网需VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

## 前置条件

- [环境准备](../../../../环境准备.md)
- 集群类型为 MANAGED_CLUSTER，Kubernetes 版本 >= 1.30.0
- CoreDNS 组件已安装且运行正常
- kubectl 已配置且可连接集群 APIServer（需 VPN/IOA 环境）

### 环境检查

```bash
# 1. 检查 tccli 版本和当前凭据
tccli --version
# expected: tccli version >= 1.0.0

tccli configure list
# expected: secretId, secretKey, region 均已配置

# 2. 检查 CAM 权限 — 需要以下 Action 名
#    tke:DescribeClusters, tke:DescribeAddon, tke:DescribeClusterKubeconfig
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
# 4. 获取 kubeconfig（需 VPN/IOA 环境执行 kubectl）
tccli tke DescribeClusterKubeconfig --region <Region> --ClusterId CLUSTER_ID \
    --output json | jq -r '.Kubeconfig' | base64 -d > ~/.kube/config-CLUSTER_ID
export KUBECONFIG=~/.kube/config-CLUSTER_ID
kubectl cluster-info
# expected: Kubernetes control plane 可达
```

```json
{
  "Kubeconfig": "<Kubeconfig>",
  "RequestId": "<RequestId>"
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli / kubectl 命令 | 幂等 |
|-----------|----------------------|:--:|
| 查看 CoreDNS 组件状态 | `tccli tke DescribeAddon --ClusterId CLUSTER_ID --AddonName coredns` | 是 |
| 获取 kubeconfig | `tccli tke DescribeClusterKubeconfig --ClusterId CLUSTER_ID` | 是 |
| 查看 CoreDNS ConfigMap | `kubectl get cm coredns -n kube-system -o yaml`（需 VPN/IOA） | 是 |
| 备份 CoreDNS ConfigMap | `kubectl get cm coredns -n kube-system -o yaml > coredns-backup.yaml`（需 VPN/IOA） | 是 |
| 编辑 CoreDNS ConfigMap | `kubectl apply -f coredns-custom.yaml`（需 VPN/IOA） | 是 |
| 重启 CoreDNS | `kubectl rollout restart deployment coredns -n kube-system`（需 VPN/IOA） | 否 |
| 测试域名解析 | `kubectl run dns-test --image=busybox -- nslookup HOST`（需 VPN/IOA） | 是 |
| 恢复 CoreDNS 配置 | `kubectl apply -f coredns-backup.yaml`（需 VPN/IOA） | 是 |

## 操作步骤

### 步骤 1：备份 CoreDNS ConfigMap

#### 选择依据

- CoreDNS ConfigMap 是集群 DNS 解析的核心配置，误改会导致全集群 DNS 解析中断。**编辑前必须备份**。
- 备份为 YAML 文件，可随时回滚。

数据面操作（需 VPN/IOA）：

```bash
kubectl get configmap coredns -n kube-system -o yaml > coredns-backup.yaml
# expected: exit 0，文件 coredns-backup.yaml 已生成
```

```text
NAME  STATUS  AGE
...
```

```bash
# 确认备份文件包含 Corefile
grep "Corefile" coredns-backup.yaml
# expected: 返回 Corefile 行
```

### 步骤 2：查看当前 Corefile 配置

```bash
kubectl get configmap coredns -n kube-system -o jsonpath='{.data.Corefile}'
# expected: 返回当前 Corefile 内容
```

**预期输出**：

```text
.:53 {
    errors
    health
    kubernetes cluster.local in-addr.arpa ip6.arpa {
        pods insecure
        fallthrough in-addr.arpa ip6.arpa
        ttl 30
    }
    prometheus :9153
    forward . /etc/resolv.conf
    cache 30
    loop
    reload
    loadbalance
}
```

### 步骤 3：添加 hosts 插件实现自定义域名解析

#### 选择依据

- **hosts 插件**：适合固定 IP 的小规模域名映射（如内部 API 网关、私有服务地址），配置简单，CoreDNS 自动 reload。
- **forward 插件**：适合需上游 DNS 递归查询的场景（如企业内网 DNS），上游 DNS 需可达。
- **rewrite 插件**：适合域名重写（如 `service.ns` 映射到 `service.ns.svc.cluster.local`）。
- **hosts 插件位置**：必须放在 `kubernetes` 插件**之前**，确保自定义域名优先于集群 DNS 解析，避免被 kubernetes 插件拦截后 NXDOMAIN。
- **fallthrough**：在 hosts 块末尾添加 `fallthrough`，使未匹配的域名继续传递给后续插件（如 kubernetes、forward），不影响正常解析。

数据面操作（需 VPN/IOA）：

`coredns-custom.yaml`：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health
        hosts {
            10.0.100.50 internal.example.com
            10.0.100.51 api.internal.example.com
            fallthrough
        }
        kubernetes cluster.local in-addr.arpa ip6.arpa {
            pods insecure
            fallthrough in-addr.arpa ip6.arpa
            ttl 30
        }
        prometheus :9153
        forward . /etc/resolv.conf
        cache 30
        loop
        reload
        loadbalance
    }
```

| 参数 | 说明 | 推荐值 |
|------|------|--------|
| `hosts` 块位置 | 放在 `kubernetes` 插件之前 | — |
| `10.0.100.50 internal.example.com` | IP 域名映射条目 | 自定义 IP 和域名 |
| `fallthrough` | 未匹配域名传递给后续插件 | 必须添加，否则非自定义域名解析失败 |

```bash
kubectl apply -f coredns-custom.yaml
# expected: configmap/coredns configured
```

**预期输出**：

```text
configmap/coredns configured
```

> CoreDNS 默认开启 `reload` 插件，ConfigMap 修改后约 30 秒内自动 reload，无需手动重启。如需立即生效，可手动触发 rollout restart。

### 步骤 4：手动重启 CoreDNS（可选）

#### 选择依据

- 如果 CoreDNS `reload` 插件正常工作，步骤 3 的 ConfigMap 修改会在 30 秒内自动生效。
- 如需立即生效（如紧急排障场景），手动 rollout restart。
- **风险**：rollout restart 会逐个重建 CoreDNS Pod，期间有短暂的 DNS 解析中断（约 10-30 秒）。

```bash
kubectl rollout restart deployment coredns -n kube-system
# expected: deployment.apps/coredns restarted
```

```bash
# 等待 rollout 完成
kubectl rollout status deployment coredns -n kube-system
# expected: deployment "coredns" successfully rolled out
```

### 步骤 5：验证自定义域名解析

数据面操作（需 VPN/IOA）：

```bash
# 测试 hosts 插件映射的域名
kubectl run -it --rm dns-test --image=busybox:1.28 --restart=Never -- \
    nslookup internal.example.com
# expected: 返回 10.0.100.50
```

**预期输出**：

```text
Server:        10.247.0.10
Address:       10.247.0.10:53

Name:   internal.example.com
Address: 10.0.100.50
```

```bash
# 测试第二个映射域名
kubectl run -it --rm dns-test2 --image=busybox:1.28 --restart=Never -- \
    nslookup api.internal.example.com
# expected: 返回 10.0.100.51
```

**预期输出**：

```text
Server:        10.247.0.10
Address:       10.247.0.10:53

Name:   api.internal.example.com
Address: 10.0.100.51
```

```bash
# 确认集群内部域名解析不受影响
kubectl run -it --rm dns-test3 --image=busybox:1.28 --restart=Never -- \
    nslookup kubernetes.default.svc.cluster.local
# expected: 返回 kubernetes Service ClusterIP
```

## 验证

### 控制面（tccli）

```bash
# 维度 1：确认 CoreDNS 组件状态正常
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

### 数据面（需 VPN/IOA）

```bash
# 维度 2：确认 Corefile 包含 hosts 插件
kubectl get configmap coredns -n kube-system -o jsonpath='{.data.Corefile}'
# expected: Corefile 中含 hosts 块和自定义域名映射
```

```text
NAME  STATUS  AGE
...
```

```bash
# 维度 3：确认 CoreDNS Pod 运行正常
kubectl get pods -n kube-system -l k8s-app=kube-dns
# expected: 所有 Pod STATUS 为 Running，READY 1/1
```

**预期输出**：

```text
NAME                       READY   STATUS    RESTARTS   AGE
coredns-xxxxxxxxxx-aaaaa   1/1     Running   0          5m
coredns-xxxxxxxxxx-bbbbb   1/1     Running   0          5m
```

```bash
# 维度 4：解析自定义域名
kubectl run -it --rm dns-verify --image=busybox:1.28 --restart=Never -- \
    nslookup internal.example.com
# expected: 返回配置的 IP 10.0.100.50
```

```bash
# 维度 5：确认 fallthrough 不影响其他域名
kubectl run -it --rm dns-verify2 --image=busybox:1.28 --restart=Never -- \
    nslookup kubernetes.default.svc.cluster.local
# expected: 返回 kubernetes Service ClusterIP，未受 hosts 插件影响
```

| 验证维度 | 命令 | 预期 |
|---------|------|------|
| 组件状态 | `DescribeAddon --AddonName coredns` | `Phase: "running"` |
| Corefile 配置 | `kubectl get cm coredns -n kube-system` | 含 hosts 块和自定义映射 |
| CoreDNS Pod | `kubectl get pods -n kube-system -l k8s-app=kube-dns` | Running，READY 1/1 |
| 自定义域名解析 | `nslookup internal.example.com` | 返回 10.0.100.50 |
| fallthrough | `nslookup kubernetes.default.svc.cluster.local` | 返回 Service ClusterIP |

## 清理

> **副作用警告**：删除 hosts 配置后，已缓存的自定义域名解析结果会在 CoreDNS cache TTL（默认 30 秒）过期后失效。期间应用可能仍解析到旧 IP。
> **操作顺序**：先备份当前配置，再恢复为原始 Corefile，最后确认 CoreDNS Pod 正常运行。切勿直接删除 ConfigMap。

### 数据面（需 VPN/IOA）

#### 1. 恢复 CoreDNS ConfigMap

使用步骤 1 的备份文件恢复：

```bash
kubectl apply -f coredns-backup.yaml
# expected: configmap/coredns configured
```

**预期输出**：

```text
configmap/coredns configured
```

```bash
# 确认 Corefile 已恢复（不含 hosts 块）
kubectl get configmap coredns -n kube-system -o jsonpath='{.data.Corefile}'
# expected: Corefile 不含 hosts 块
```

```text
NAME  STATUS  AGE
...
```

#### 2. 重启 CoreDNS 使配置生效

```bash
kubectl rollout restart deployment coredns -n kube-system
# expected: deployment.apps/coredns restarted

kubectl rollout status deployment coredns -n kube-system
# expected: deployment "coredns" successfully rolled out
```

#### 3. 验证自定义域名已失效

```bash
kubectl run -it --rm dns-cleanup --image=busybox:1.28 --restart=Never -- \
    nslookup internal.example.com
# expected: 解析失败（NXDOMAIN），说明 hosts 配置已移除
```

**预期输出**：

```text
Server:        10.247.0.10
Address:       10.247.0.10:53

*** Can't find internal.example.com: No answer
```

### 控制面（tccli）

> 控制面无需清理操作。自定义域名解析通过修改 CoreDNS ConfigMap 实现，不涉及 TKE 组件安装或卸载。

```bash
# 确认 CoreDNS 组件状态正常（清理后）
tccli tke DescribeAddon --region <Region> \
    --ClusterId CLUSTER_ID \
    --AddonName coredns
# expected: Phase: "running"
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

## 排障

### 配置修改后解析异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl` 命令超时或连接被拒绝 | `tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]'`（控制面返回 Running） | CAM 策略 strategyId:240463971 拒绝公网端点，数据面需内网访问 | 通过 VPN/IOA 接入 VPC 内网后执行 kubectl；或使用控制台替代 |
| 自定义域名解析不到 | `kubectl run dns-test -- nslookup internal.example.com` | CoreDNS 未加载新的 hosts 配置 | 确认 `reload` 插件在 Corefile 中；或 `kubectl rollout restart deployment coredns -n kube-system` 手动重启 |
| CoreDNS Pod CrashLoopBackOff | `kubectl logs -n kube-system -l k8s-app=kube-dns --tail=20` | Corefile 语法错误（大括号不匹配、插件顺序错误） | 用备份恢复：`kubectl apply -f coredns-backup.yaml`；修正 Corefile 后重新 apply |
| 集群内所有域名解析失败 | `kubectl run dns-test -- nslookup kubernetes.default.svc.cluster.local` | hosts 块缺少 `fallthrough`，未匹配域名被拦截返回 NXDOMAIN | 在 hosts 块末尾添加 `fallthrough`，使未匹配域名继续传递给 kubernetes 插件 |
| hosts 插件覆盖了集群 Service 域名 | `kubectl run dns-test -- nslookup kubernetes.default.svc.cluster.local` | hosts 条目与集群域名冲突，hosts 优先级高于 kubernetes 插件 | 检查 hosts 条目，避免使用 `*.svc.cluster.local` 域名；调整 hosts 块位置 |
| Corefile reload 不生效 | `kubectl logs -n kube-system -l k8s-app=kube-dns \| grep reload` | Corefile 中未配置 `reload` 插件，或 ConfigMap 未被 CoreDNS 挂载 | 确认 Corefile 含 `reload` 插件；手动 `kubectl rollout restart deployment coredns -n kube-system` |
| hosts 条目 IP 不通 | `kubectl run dns-test -- wget -qO- http://internal.example.com` | IP 已失效或网络不通 | 更新 hosts 条目中的 IP 为有效地址；确认安全组规则允许访问 |

### YAML 格式错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl apply` 返回 YAML 解析错误 | `kubectl apply -f coredns-custom.yaml --dry-run=client` | Corefile 中缩进错误或 Tab 字符 | 使用空格而非 Tab；确保 Corefile 内容缩进一致（每级 4 空格） |
| `kubectl apply` 返回 `configmap "coredns" was not deleted` | `kubectl get cm coredns -n kube-system` | ConfigMap 已存在但 YAML 格式不匹配 | 先 `kubectl get cm coredns -n kube-system -o yaml > backup.yaml` 备份，再 `kubectl apply --force -f coredns-custom.yaml` |

## 下一步

- [TKE DNS 最佳实践](../TKE%20DNS%20最佳实践/tccli%20操作.md) -- page_id `78005`
- [在 TKE 集群中使用 NodeLocal DNS Cache](../在%20TKE%20集群中使用%20NodeLocal%20DNS%20Cache/tccli%20操作.md) -- page_id `40613`
- [CoreDNS 日志仪表盘使用指南](../CoreDNS%20日志仪表盘使用指南/tccli%20操作.md) -- page_id `109060`
- [集群 DNS 解析异常排障处理](../../../../故障处理/集群%20DNS%20解析异常排障处理/tccli%20操作.md) -- page_id `80531`

## 控制台替代

登录 [容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) → 选择集群 → **组件管理** → **CoreDNS** → **更新配置** → 在 Corefile 编辑器中添加 hosts 插件配置 → 保存。或在控制台中使用 kubectl 终端直接编辑 ConfigMap。
