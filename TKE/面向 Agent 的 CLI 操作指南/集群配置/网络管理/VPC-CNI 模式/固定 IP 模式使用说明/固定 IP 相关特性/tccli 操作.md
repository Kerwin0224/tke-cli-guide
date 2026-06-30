# 固定 IP 相关特性（tccli）

> 对照官方：[固定 IP 相关特性](https://cloud.tencent.com/document/product/457/50358) · page_id `50358`

## 概述

对照[官方 固定 IP 相关特性](https://cloud.tencent.com/document/product/457/50358)，保留产品与限制说明，并用 **tccli + kubectl** 完成固定 IP CRD 资源审计、回收策略配置与故障排查步骤。

## 前置条件

- [环境准备](../../../../../环境准备.md)：`tccli`、`kubectl`、地域 `ap-guangzhou` 与凭证已配置。
- 集群已开启 VPC-CNI 固定 IP 模式（见 [固定 IP 使用方法](../%E5%9B%BA%E5%AE%9A%20IP%20%E4%BD%BF%E7%94%A8%E6%96%B9%E6%B3%95/tccli%20%E6%93%8D%E4%BD%9C.md)）。
- 下载 kubeconfig 见 [连接集群](../../../../集群管理/连接集群/tccli%20操作.md)。

## 控制台与 CLI 参数映射

| 官方 / 控制台 | 含义 | tccli / 说明 | 幂等 |
|---------------|------|-------------|------|
| 查看 IP 使用情况 | 容器子网内 IP 占用 | `kubectl get vip` | 是 |
| 查看 IP Claim | CRD VpcIPClaim | `kubectl get vipc -n <namespace>` | 是 |
| 过期回收（eniipamd 组件配置） | 固定 IP 超时回收 | `--claim-expired-duration=1h`（≥5m） | 否 |
| 手动回收 IP | 立即释放固定 IP | `kubectl delete vipc <podname> -n <namespace>` | 否 |
| 级联回收（enable-ownerref） | 删除 Workload 时级联删除 IP | `--enable-ownerref`（eniipamd 启动参数） | 否 |
| 组件版本查看 | eniipamd 版本 | `tccli tke DescribeAddon` / `kubectl get deploy tke-eni-ipamd -o jsonpath='{..image}'` | 是 |

## 操作步骤

### CRD 对象模型

VPC-CNI 固定 IP 模式下，网络组件为每个固定 IP Pod 自动创建两个 CRD：

- **`VpcIPClaim`**（`vipc`）：同名于 Pod，描述 Pod 的 IP 需求。
- **`VpcIP`**（`vip`）：以实际分配的 IP 地址命名，表示 IP 占用状态。

#### kubectl：查看 IP 使用情况

```bash
kubectl get vip
```

```text
NAME           STATUS    IP             SUBNET
10.0.0.10      Attached  10.0.0.10      subnet-example
10.0.0.11      Free      10.0.0.11      subnet-example
10.0.0.12      Attached  10.0.0.12      subnet-example
```

# expected: exit 0

### 生命周期行为

| Pod 类型 | Pod 销毁时 | VpcIPClaim | VpcIP | 行为 |
|----------|-----------|------------|-------|------|
| 非固定 IP | 销毁 | 销毁 | 回收 | IP 立即释放 |
| 固定 IP | 销毁 | 保留 | 保留 | 同名新 Pod 复用同一 IP |

> **警告：** 默认永不回收固定 IP，未释放的 IP 会导致子网 IP 耗尽。

---

### 回收方式一：过期回收

#### tccli：查看 eniipamd 组件版本

```bash
tccli tke DescribeAddon --ClusterId <ClusterId> --region ap-guangzhou --output json \
  --filter "Addons[?AddonName=='eniipamd'] | [0].{AddonName:AddonName,AddonVersion:AddonVersion,Phase:Phase}"
```

```json
{
  "AddonName": "eniipamd",
  "AddonVersion": "v3.5.0",
  "Phase": "Running"
}
```

# expected: exit 0, contains "AddonVersion"

- **≥ v3.5.0**：组件管理 → eniipamd → 更新配置 → 填入**固定 IP 回收策略过期时间**。
- **< v3.5.0**：`kubectl edit deploy tke-eni-ipamd -n kube-system`，在 `args` 中增加 `--claim-expired-duration=1h`（最短 5m）。

### 回收方式二：手动回收

确定 Pod 命名空间和名称后执行：

```bash
kubectl delete vipc <podname> -n <namespace>
```

```text
vpcipclaim.networking.tke.cloud.tencent.com "<podname>" deleted
```

# expected: exit 0, contains "deleted"

> **注意：** 对应 Pod 必须已销毁，否则 Pod 网络会不可用。

### 回收方式三：级联回收

删除 Workload 时立即回收关联的固定 IP。仅对**增量** Workload 生效（存量不生效）。

- **≥ v3.5.0**：组件管理 → eniipamd → 更新配置 → 勾选**固定 IP 模式级联回收**。
- **< v3.5.0**：`kubectl edit deploy tke-eni-ipamd -n kube-system`，在 `args` 中增加 `--enable-ownerref`。

```bash
kubectl rollout status deploy tke-eni-ipamd -n kube-system
```

```text
deployment "tke-eni-ipamd" successfully rolled out
```

# expected: exit 0, contains "successfully"

---

### 常见问题排查

#### 问题一：节点无法分配 ENI，Pod 不能调度（共享网卡模式）

节点加入集群后 ipamd 尝试从同 AZ 子网绑定 ENI，失败原因：

- ipamd 异常
- 节点 AZ 内无容器子网
- VPC ENI 配额不足

```bash
kubectl get event
```

```text
NAME  STATUS  AGE
...
```

若出现 **ENILimit** 事件 → 申请提升 VPC ENI 配额。若同 AZ 子网 IP 耗尽 → 添加该 AZ 内子网到容器网段。

#### 问题二：节点 ENI 数量超限

症状：节点 ENI 绑定失败、nec 关联 vip 状态卡在 `Attaching`。

```bash
kubectl get nec -o yaml
kubectl get vip -o yaml
```

```text
Status: Attaching
Error: ENI quota exceeded
```

# expected: exit 0

**解决**：VPC 单地域默认最多 1000 ENI，可通过 [在线咨询](https://cloud.tencent.com/online-service) 申请提升。

## 验证

### 数据面（kubectl）

- `kubectl get vip` 返回固定 IP Pod 的 IP 占用记录。
- `kubectl describe deploy tke-eni-ipamd -n kube-system | grep claim-expired-duration` 确认回收策略已生效。

## 清理

本页为查询与配置说明；手动回收的 IP 不可恢复，操作前确认 Pod 已销毁。

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl get vip` 无输出 | 执行 `tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["<ClusterId>"]'` 检查 `ClusterNetworkSettings.EnableStaticIp` | 集群未开启 VPC-CNI 固定 IP 模式 | 通过 `tccli tke EnableVpcCniNetworkType` 命令开启固定 IP 模式，设置 `EnableStaticIp=true` |
| 固定 IP 未回收导致子网 IP 耗尽 | 执行 `kubectl get vip` 查看 IP 占用状态，关注 `Free` 状态的 IP 数量 | 未配置固定 IP 过期回收策略，也未手动回收 | 设置 `--claim-expired-duration` 过期时间（如 `1h`），或手动执行 `kubectl delete vipc <podname> -n <namespace>` |
| `kubectl edit deploy` 修改 eniipamd 参数后未生效 | 执行 `kubectl rollout status deploy tke-eni-ipamd -n kube-system` 检查 rollout 状态 | ipamd 的 Deployment 修改后 Pod 未重启，旧配置仍在生效 | 执行 `kubectl rollout restart deploy tke-eni-ipamd -n kube-system` 触发重启，或等待 rollout 自动完成 |
| NEC 状态为 Attaching 或空 | 执行 `kubectl get nec -o yaml` 查看 NEC 资源的 `Status` 和错误信息 | VPC ENI 配额超限，无法为节点分配更多 ENI | 通过 [在线咨询](https://cloud.tencent.com/online-service) 申请提升 VPC ENI 配额 |

## 下一步

- [固定 IP 使用方法](https://cloud.tencent.com/document/product/457/34994)（基础开启与 StatefulSet 创建）· [非固定 IP 模式使用说明](https://cloud.tencent.com/document/product/457/64940)

## 控制台替代

[控制台 → 集群](https://console.cloud.tencent.com/tke2/cluster) → 组件管理 → eniipamd → 更新配置（≥ v3.5.0）可配置回收策略与级联回收。
