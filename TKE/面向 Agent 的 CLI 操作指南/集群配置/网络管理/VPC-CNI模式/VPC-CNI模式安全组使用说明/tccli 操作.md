# VPC-CNI模式安全组使用说明（tccli）

> 对照官方：[VPC-CNI模式安全组使用说明](https://cloud.tencent.com/document/product/457/50360) · page_id `50360`

## 概述

VPC-CNI 模式下，Pod 可以使用 VPC 安全组实现 Pod 级别的精细化网络访问控制。每个 Pod 的 ENI 可以绑定到安全组，从而独立控制入站和出站流量。默认情况下 Pod 继承节点安全组；通过 `tke.cloud.tencent.com/security-group-id` 注解可自定义 Pod 级别安全组。TKE 通过 eniipamd 组件管理安全组绑定。

## 前置条件

- 集群已开启 VPC-CNI 网络模式（通过 EnableVpcCniNetworkType 开通）
- 已安装 eniipamd 组件（开通 VPC-CNI 时自动安装）
- 已创建目标安全组及规则（通过 tccli vpc CreateSecurityGroup 创建）

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 查询集群 VPC-CNI 状态 | `tccli tke DescribeIPAMD --ClusterId cls-xxxxxxxx --region ap-guangzhou` | 是 |
| 查询安全组 | `tccli vpc DescribeSecurityGroups --region ap-guangzhou` | 是 |
| 查询安全组规则 | `tccli vpc DescribeSecurityGroupPolicies --region ap-guangzhou --SecurityGroupId sg-xxxxxxxx` | 是 |
| 创建安全组 | `tccli vpc CreateSecurityGroup --region ap-guangzhou --GroupName <name> --GroupDescription <desc>` | 否 |
| 添加安全组规则 | `tccli vpc CreateSecurityGroupPolicies --region ap-guangzhou --SecurityGroupId sg-xxxxxxxx --SecurityGroupPolicySet '{...}'` | 否 |
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

当前集群 cls-xxxxxxxx 为 GR 模式，EnableIPAMD 为 false。若返回 `EnableIPAMD: true` 则表示 VPC-CNI 已启用。若返回 `FailedOperation.EnableVPCCNIFailed`，需先执行 EnableVpcCniNetworkType 开通 VPC-CNI。

### 2. 查询已有安全组

```bash
tccli vpc DescribeSecurityGroups \
  --region ap-guangzhou
```

从返回结果中确认目标安全组的 SecurityGroupId 和规则配置。

### 3. 创建安全组（可选）

若尚无合适的安全组，创建一个新的：

```bash
tccli vpc CreateSecurityGroup \
  --region ap-guangzhou \
  --GroupName "pod-sg-demo" \
  --GroupDescription "Security group for VPC-CNI Pods"
```

记录返回的 SecurityGroupId。

### 4. 添加安全组规则（可选）

```bash
tccli vpc CreateSecurityGroupPolicies \
  --region ap-guangzhou \
  --SecurityGroupId sg-xxxxxxxx \
  --SecurityGroupPolicySet '{
    "Ingress": [
      {
        "Protocol": "TCP",
        "Port": "80",
        "CidrBlock": "0.0.0.0/0",
        "Action": "ACCEPT",
        "PolicyDescription": "Allow HTTP"
      }
    ]
  }'
```

根据实际需求配置入站/出站规则。

### 5. 创建带安全组注解的 Pod

在 Pod 模板中添加注解 `tke.cloud.tencent.com/security-group-id`：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-sg
  namespace: default
  annotations:
    tke.cloud.tencent.com/security-group-id: "sg-xxxxxxxx"
spec:
  containers:
  - name: nginx
    image: nginx:latest
```

```bash
kubectl apply -f nginx-sg-pod.yaml
```

```text
# command executed successfully
```

Pod 创建后，eniipamd 组件会自动将指定的安全组绑定到该 Pod 的 ENI。

### 6. 查询安全组规则

```bash
tccli vpc DescribeSecurityGroupPolicies \
  --region ap-guangzhou \
  --SecurityGroupId sg-xxxxxxxx
```

验证规则是否已正确配置和应用。

## 验证

- 通过 `kubectl describe pod nginx-sg` 查看 Pod 的 ENI 和安全组绑定状态
- 通过 `tccli vpc DescribeSecurityGroupPolicies` 确认安全组规则配置
- 从同 VPC 内的其他 Pod 或节点发起网络连通性测试，验证安全组规则生效
- 通过 `kubectl get pod nginx-sg -o jsonpath='{.metadata.annotations}'` 验证注解已正确设置

## 清理

- 删除测试 Pod：`kubectl delete pod nginx-sg`
- 删除不再使用的安全组规则：`tccli vpc DeleteSecurityGroupPolicies`
- 删除不再使用的安全组（需先解绑所有关联资源）：`tccli vpc DeleteSecurityGroup`

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Pod 无法被访问 | 检查安全组入站规则是否放行对应端口和来源 IP | 安全组入站规则缺失或配置错误 | 通过 CreateSecurityGroupPolicies 添加正确的入站规则 |
| Pod 无法访问外部 | 检查安全组出站规则是否放行目标 IP 和端口 | 安全组出站规则限制 | 通过 CreateSecurityGroupPolicies 添加出站规则 |
| 安全组未应用到 Pod | 检查 eniipamd DaemonSet Pod 状态 `kubectl get pods -n kube-system -l app=eni-ipamd` | eniipamd 组件异常或注解格式错误 | 检查 eniipamd 日志，确认注解 key 为 `tke.cloud.tencent.com/security-group-id` |
| DescribeSecurityGroups 返回空 | 确认区域参数 `--region` 与安全组所在区域一致 | 查询区域不正确 | 指定正确的 `--region` 参数 |
| Pod 创建后无法获取 IP | 检查 VPC-CNI 子网 IP 是否充足 | 子网 IP 耗尽或安全组配额不足 | 通过 AddVpcCniSubnets 添加子网，或释放不再使用的安全组 |

## 下一步

- [VPC-CNI模式介绍](https://cloud.tencent.com/document/product/457/50355) -- page_id `50355`
- [多Pod共享网卡](https://cloud.tencent.com/document/product/457/50356) -- page_id `50356`
- [VPC-CNI（eniipamd）组件介绍](https://cloud.tencent.com/document/product/457/64919) -- page_id `64919`

## 控制台替代

在 TKE 控制台进入集群详情，左侧导航栏选择"网络" > "VPC-CNI 模式"，可查看和管理 VPC-CNI 相关配置。安全组的创建和管理也可在 VPC 控制台的"安全组"页面通过可视化界面操作。
