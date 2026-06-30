# Pod直接绑定弹性公网IP使用说明（tccli）

> 对照官方：[Pod直接绑定弹性公网IP使用说明](https://cloud.tencent.com/document/product/457/64886) · page_id `64886`

## 概述

VPC-CNI 模式下，Pod 可以通过 `tke.cloud.tencent.com/eip-attributes` 注解直接绑定弹性公网 IP（EIP），使 Pod 无需 NodePort 或 LoadBalancer 即可直接从互联网访问。EIP 可使用已有实例或自动创建，带宽可按需配置。此功能要求集群已开启 VPC-CNI 网络模式。

## 前置条件

- 集群已开启 VPC-CNI 网络模式（通过 EnableVpcCniNetworkType 开通）
- 已安装 eniipamd 组件（开通 VPC-CNI 时自动安装）
- 若使用已有 EIP，需确保 EIP 处于未绑定状态
- EIP 配额充足（默认每个地域 20 个，不足可申请提升）

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 查询集群 VPC-CNI 状态 | `tccli tke DescribeIPAMD --ClusterId cls-xxxxxxxx --region ap-guangzhou` | 是 |
| 查询 EIP 列表 | `tccli vpc DescribeAddresses --region ap-guangzhou` | 是 |
| 创建 EIP | `tccli vpc AllocateAddresses --region ap-guangzhou --AddressCount 1` | 否 |
| 释放 EIP | `tccli vpc ReleaseAddresses --region ap-guangzhou --AddressIds '["eip-xxxxxxxx"]'` | 否 |
| 查询 Pod 注解 | `kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.metadata.annotations}'` | 是 |

## 操作步骤

### 1. 查询集群 VPC-CNI 状态

```bash
tccli tke DescribeIPAMD \
  --ClusterId cls-xxxxxxxx \
  --region ap-guangzhou
```

```json
{
  "EnableIPAMD": true,
  "EnableCustomizedPodCidr": true,
  "DisableVpcCniMode": true,
  "Phase": "<Phase>",
  "Reason": "<Reason>",
  "SubnetIds": [],
  "ClaimExpiredDuration": "<ClaimExpiredDuration>",
  "EnableTrunkingENI": true
}
```

当前集群 cls-xxxxxxxx 为 GR 模式，EnableIPAMD 为 false。若返回 `EnableIPAMD: true` 则表示 VPC-CNI 已启用，可以继续后续步骤。若未启用，需先通过 EnableVpcCniNetworkType 开通。

### 2. 查询已有 EIP

```bash
tccli vpc DescribeAddresses \
  --region ap-guangzhou
```

从返回结果中查看已有 EIP 的 AddressId、AddressIp 及 AddressStatus。状态为 `UNBIND` 的 EIP 可用于绑定 Pod。若无需使用已有 EIP，可跳过此步直接创建新的。

### 3. 创建 EIP（可选）

```bash
tccli vpc AllocateAddresses \
  --region ap-guangzhou \
  --AddressCount 1 \
  --InternetChargeType TRAFFIC_POSTPAID_BY_HOUR \
  --InternetMaxBandwidthOut 10
```

可选参数说明：
- `--InternetChargeType`：计费类型，可选 TRAFFIC_POSTPAID_BY_HOUR（按流量）、BANDWIDTH_POSTPAID_BY_HOUR（按带宽）
- `--InternetMaxBandwidthOut`：带宽上限（Mbps）
- `--AddressName`：EIP 名称（便于识别）

记录返回的 AddressId（格式：`eip-xxxxxxxx`）。

### 4. 创建带 EIP 注解的 Pod

使用已有 EIP 绑定：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-eip
  namespace: default
  annotations:
    tke.cloud.tencent.com/eip-attributes: '{"Bandwidth":"10","ChargeType":"","DeletePolicy":""}'
    tke.cloud.tencent.com/eip-id: "eip-xxxxxxxx"
spec:
  containers:
  - name: nginx
    image: nginx:latest
```

自动创建并绑定 EIP：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-eip-auto
  namespace: default
  annotations:
    tke.cloud.tencent.com/eip-attributes: '{"Bandwidth":"10","ChargeType":"TRAFFIC_POSTPAID_BY_HOUR","DeletePolicy":"Delete"}'
spec:
  containers:
  - name: nginx
    image: nginx:latest
```

注解说明：
- `Bandwidth`：EIP 带宽上限（Mbps）
- `ChargeType`：计费类型；自动创建时需指定
- `DeletePolicy`：Pod 删除时是否同步释放 EIP，设为 `Delete` 自动释放，留空保留

```bash
kubectl apply -f nginx-eip-pod.yaml
```

```text
# command executed successfully
```

### 5. 验证 Pod EIP 绑定

```bash
# 查看 Pod 详细信息
kubectl describe pod nginx-eip

# 查看 Pod 注解中的 EIP 信息
kubectl get pod nginx-eip -o jsonpath='{.metadata.annotations}'

# 通过 VPC API 确认 EIP 绑定状态
tccli vpc DescribeAddresses \
  --region ap-guangzhou \
  --Filters '[{"Name":"instance-id","Values":["<pod-ip>"]}]'
```

```text
NAME  STATUS  AGE
...
```

## 验证

- 通过 `kubectl describe pod nginx-eip` 查看 Pod 事件，确认 EIP 绑定成功
- 通过 `tccli vpc DescribeAddresses` 查询 EIP 的 AddressStatus 是否为 `BIND`
- 从公网环境 ping 或 curl EIP 地址，验证 Pod 可被公网访问
- 在 Pod 内执行 `curl ifconfig.me` 确认出网 IP 为绑定的 EIP

## 清理

- 删除测试 Pod（若 DeletePolicy 为 Delete，EIP 会自动释放）：

  ```bash
  kubectl delete pod nginx-eip
  ```

- 若 DeletePolicy 为保留，需手动释放 EIP：

  ```bash
  tccli vpc ReleaseAddresses \
    --region ap-guangzhou \
    --AddressIds '["eip-xxxxxxxx"]'
  ```

- 在释放 EIP 前确保其已解绑 Pod（Pod 删除后自动解绑）

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| EIP 未绑定到 Pod | 检查 Pod 事件 `kubectl describe pod <name>` | 注解格式错误或 eniipamd 组件异常 | 确认注解 JSON 格式正确，检查 eniipamd DaemonSet 状态 |
| EIP 创建失败 | 检查 `tccli vpc DescribeAddressQuota` | EIP 配额不足或计费类型不支持 | 申请提升 EIP 配额或切换计费类型 |
| Pod 创建后无法绑定已有 EIP | 检查 EIP 状态 | EIP 已被其他资源绑定或状态异常 | 使用 `tccli vpc DescribeAddresses` 确认 EIP 为 UNBIND 状态 |
| EnableVpcCniNetworkType 失败 | 查看返回错误信息 `FailedOperation.EnableVPCCNIFailed` | 集群不是 VPC-CNI 集群或子网配置错误 | 确认 EnableVpcCniNetworkType 参数正确，检查子网可用性 |
| 公网无法访问 Pod | 检查安全组和 EIP 带宽 | 安全组未放行公网流量或带宽为 0 | 配置安全组入站规则放行所需端口，调整 EIP 带宽 |

## 下一步

- [VPC-CNI模式安全组使用说明](https://cloud.tencent.com/document/product/457/50360) -- page_id `50360`
- [VPC-CNI模式介绍](https://cloud.tencent.com/document/product/457/50355) -- page_id `50355`

## 控制台替代

在 TKE 控制台进入集群详情，左侧导航栏选择"网络" > "VPC-CNI 模式"，在工作负载创建时可通过控制台界面为 Pod 配置 EIP 绑定。EIP 的管理也可在 VPC 控制台的"弹性公网 IP"页面进行可视化操作。
