# 从 kube-dns 切换到 CoreDNS（tccli）

> 对照官方：[从 kube-dns 切换到 CoreDNS](https://cloud.tencent.com/document/product/457/97873) · page_id `97873`

## 概述

CoreDNS 是 Kubernetes 1.11+ 推荐的集群 DNS 服务，已替代 kube-dns。本文介绍在 TKE 中将集群 DNS 从 kube-dns 切换到 CoreDNS 的操作步骤及回滚方案。

> **kubectl 数据面不可达**：CAM拒绝公网端点(strategyId:240463971)，内网需VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

## 前置条件

### 环境检查

```bash
tccli --version
# expected: tccli version >= 1.0.0

tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterStatus "Running", ClusterVersion >= 1.11
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

## 控制台与 CLI 参数映射

| 控制台操作 | tccli/kubectl 命令 | 幂等 |
|-----------|-------------------|:--:|
| 查看当前 DNS 组件 | `tccli tke DescribeAddon --ClusterId CLUSTER_ID` | 是 |
| 卸载 kube-dns | `tccli tke DeleteAddon --AddonName kube-dns` | 是 |
| 安装 CoreDNS | `tccli tke InstallAddon --AddonName coredns` | 否 |
| 验证 DNS 功能 | `kubectl run dns-test --image=busybox -- nslookup kubernetes`（需 VPN/IOA） | 是 |
| 回滚到 kube-dns | `tccli tke DeleteAddon --AddonName coredns` + `InstallAddon --AddonName kube-dns` | 否 |

## 操作步骤

### 1. 切换前检查

#### 选择依据

- CoreDNS 内存占用更低（~30Mi vs kube-dns ~70Mi），性能更好，社区活跃
- kube-dns 已停止维护，新版本 Kubernetes 不再支持 kube-dns
- 切换前务必备份 Corefile 和 kube-dns 自定义配置
- **切换有短暂 DNS 中断**（约 10-30 秒），建议在维护窗口执行

```bash
# 控制面：确认当前 DNS 组件
tccli tke DescribeAddon --region <Region> --ClusterId CLUSTER_ID --AddonName kube-dns
# expected: 返回 kube-dns 当前配置信息

# 数据面：备份当前 DNS 相关资源（需 VPN/IOA）
kubectl get deploy kube-dns -n kube-system -o yaml > kube-dns-backup.yaml
kubectl get svc kube-dns -n kube-system -o yaml > kube-dns-svc-backup.yaml
kubectl get cm kube-dns -n kube-system -o yaml > kube-dns-cm-backup.yaml
# expected: 备份文件生成
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

### 2. 卸载 kube-dns

```bash
tccli tke DeleteAddon --region <Region> --ClusterId CLUSTER_ID --AddonName kube-dns
# expected: exit 0
```

预期输出：

```json
{
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

### 3. 安装 CoreDNS

```bash
cat > install-coredns.json <<'EOF'
{
    "ClusterId": "CLUSTER_ID",
    "AddonName": "coredns",
    "AddonVersion": "ADDON_VERSION"
}
EOF
tccli tke InstallAddon --region <Region> --cli-input-json file://install-coredns.json
# expected: exit 0, 返回 RequestId
```

### 4. 等待 CoreDNS 就绪

```bash
tccli tke DescribeAddon --region <Region> --ClusterId CLUSTER_ID --AddonName coredns
# expected: Status "Running"
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

数据面验证（需 VPN/IOA）：

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
# expected: CoreDNS Pod Running

kubectl run -it --rm dns-test --image=busybox:1.28 --restart=Never -- nslookup kubernetes.default
# expected: 正常解析
```

```text
NAME  STATUS  AGE
...
```

## 验证

### 控制面（tccli）

```bash
tccli tke DescribeAddon --region <Region> --ClusterId CLUSTER_ID --AddonName coredns
# expected: Status "Running"

tccli tke DescribeAddon --region <Region> --ClusterId CLUSTER_ID --AddonName kube-dns
# expected: AddonNotFound 或 Status 不为 "Running"
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
kubectl get svc kube-dns -n kube-system
# expected: ClusterIP 未变

kubectl run -it --rm dns-test --image=busybox:1.28 --restart=Never -- nslookup kubernetes.default.svc.cluster.local
# expected: 正常解析

kubectl get pods -n kube-system | grep -E "coredns|kube-dns"
# expected: 只有 coredns Pod，无 kube-dns Pod
```

```text
NAME  STATUS  AGE
...
```

## 清理

切换后清理备份文件（可选）：

```bash
rm -f kube-dns-backup.yaml kube-dns-svc-backup.yaml kube-dns-cm-backup.yaml
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl` 命令超时或连接被拒绝 | `tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]'`（控制面返回 Running） | CAM 策略 strategyId:240463971 拒绝公网端点，数据面需内网访问 | 通过 VPN/IOA 接入 VPC 内网后执行 kubectl；或使用控制台替代 |
| CoreDNS 安装后 DNS 解析失败 | `kubectl logs -n kube-system -l k8s-app=kube-dns` | Corefile 配置或 upstream DNS 不正确 | 检查 ConfigMap coredns 的 forward 插件配置 |
| Service ClusterIP 变化 | `kubectl get svc kube-dns -n kube-system` | 卸载时自动清理了 kube-dns Service，安装 CoreDNS 时重建 | 确认 ClusterIP 被应用正确引用（通常自动处理） |
| 回滚 kube-dns 失败 | `tccli tke DescribeAddon --AddonName kube-dns` | kube-dns 版本不兼容当前 K8s 版本 | 检查可用 Addon 版本列表 |
| Pod 临时解析失败 | 检查切换时间窗口内的 dns-test 结果 | DNS 切换需 10-30 秒 | 已内置在 Kubernetes 重试机制中，应用会自动重试 |

## 下一步

- [TKE DNS 最佳实践](../TKE%20DNS%20最佳实践/tccli%20操作.md) -- page_id `78005`
- [CoreDNS 升级常见错误和处理](../../../../故障处理/CoreDNS%20升级常见错误和处理/tccli%20操作.md) -- page_id `130406`
- [集群 DNS 解析异常排障处理](../../../../故障处理/集群%20DNS%20解析异常排障处理/tccli%20操作.md) -- page_id `80531`

## 控制台替代

[容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) -> 集群 -> 组件管理 -> 安装 CoreDNS。
