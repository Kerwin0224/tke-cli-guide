# CLB Ingress 创建报错排障处理（tccli）

> 对照官方：[CLB Ingress 创建报错排障处理](https://cloud.tencent.com/document/product/457/75757) · page_id `75757`

## 概述

TKE CLB 类型 Ingress 创建报错的常见原因：子网 IP 不足、安全组配置错误、CLB 配额耗尽、证书格式错误。通过 tccli 检查 CLB 配额和子网资源，配合 kubectl 查看 Ingress 事件定位根因。

> **kubectl 数据面不可达**：CAM拒绝公网端点(strategyId:240463971)，内网需VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

## 前置条件

- [环境准备](../../环境准备.md)
- 已获取集群 kubeconfig（如需 kubectl 操作）

### 环境检查

```bash
tccli --version
# 预期输出: tccli version X.X.X
```

### 资源检查

```bash
tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]'
```

**预期输出**：

```json
{
  "TotalCount": 1,
  "Clusters": [
    {
      "ClusterId": "cls-xxxxxxxx",
      "ClusterName": "my-test-cluster",
      "ClusterStatus": "Running",
      "ClusterVersion": "1.30.0",
      "ClusterType": "MANAGED_CLUSTER"
    }
  ]
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli / CLI | 幂等 |
|---|---|---|
| 查看集群状态 | `tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]'` | 是 |
| 查看 Ingress 事件 | `kubectl describe ingress <name>`（需 VPN/IOA） | 是 |
| 查看 CLB 配额 | `tccli clb DescribeQuota --region ap-guangzhou` | 是 |
| 查看子网 IP | `tccli vpc DescribeSubnets --region ap-guangzhou --SubnetIds '["<subnet-id>"]'` | 是 |

## 操作步骤

### 步骤1：控制面检查集群状态

```bash
tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]'
```

**预期输出**：

```json
{
  "Clusters": [
    {
      "ClusterId": "cls-xxxxxxxx",
      "ClusterStatus": "Running",
      "ClusterVersion": "1.30.0"
    }
  ]
}
```

### 步骤2：检查 CLB 配额

```bash
tccli clb DescribeQuota --region ap-guangzhou
```

**预期输出**：

```json
{
  "QuotaSet": [
    {
      "QuotaId": "TOTAL_OPEN_CLB_QUOTA",
      "QuotaLimit": 50,
      "QuotaCurrent": 12
    }
  ]
}
```

### 步骤3：数据面排障（需 VPN/IOA）

```bash
kubectl describe ingress <ingress-name> | grep -A20 Events
# 预期: 查看创建过程中的 Events 错误信息

kubectl get svc -n kube-system -l app=nginx-ingress-controller
# 预期: Ingress Controller Service 状态正常
```

```text
NAME  STATUS  AGE
...
```

### 常见报错速查表

| 错误信息 | 根因 | 修复 |
|---|---|---|
| `CLB quota exceeded` | CLB 实例数达到地域上限 | 提工单申请扩容或清理闲置 CLB |
| `Subnet IP insufficient` | 子网无可用 IP 分配给 CLB | 更换子网或扩大子网 CIDR |
| `SecurityGroup not found` | Ingress 注解引用的安全组已被删除 | 更新 Ingress 注解，使用有效的安全组 ID |
| `Certificate error` | TLS 证书格式错误或已过期 | `kubectl describe secret <tls-secret>` 检查证书内容 |

## 验证

### 控制面（tccli）

```bash
tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]'
```

**预期输出**：

```json
{
  "Clusters": [
    {
      "ClusterId": "cls-xxxxxxxx",
      "ClusterStatus": "Running"
    }
  ]
}
```

### 数据面（需 VPN/IOA）

```bash
kubectl get ingress <name> -o wide
# 预期: ADDRESS 列显示 CLB VIP，非空

kubectl describe ingress <name> | tail -20
# 预期: Events 中无错误信息
```

```text
NAME  STATUS  AGE
...
```

## 清理

```bash
# 删除测试 Ingress（需 VPN/IOA）
kubectl delete ingress <ingress-name>
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|---|---|---|---|
| Ingress 长时间无 ADDRESS | `kubectl describe ingress` Events 为空 | Ingress Controller 未安装或不支持 CLB 类型 | 安装 NginxIngress 组件；确认 Ingress className 正确 |
| `Failed to create CLB` | `kubectl describe ingress` 显示子网相关错误 | Ingress 注解中 subnet-id 所在可用区无可用 IP | 更换或添加 subnet-id 注解指定有空闲 IP 的子网 |

### 操作成功但异常

| 现象 | 诊断 | 根因 | 修复 |
|---|---|---|---|
| Ingress 创建成功但无法访问 | `tccli clb DescribeListeners` 查看监听器状态 | CLB 监听器后端健康检查失败 | 检查后端 Pod 是否 Ready；确认健康检查路径和端口配置正确 |
| 删除 Ingress 后 CLB 残留 | CLB 实例未被自动清理 | Ingress 删除时 CLB 解绑超时 | 在 CLB 控制台手动清理残留实例 |

## 下一步

- [Service&Ingress 常见报错和处理](../Service&Ingress%20常见报错和处理/tccli%20操作.md) -- page_id `80913`
- [Ingress 基本功能](../../../应用配置/服务和配置管理/Ingress/tccli%20操作.md)

## 控制台替代

[容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) -> 服务与路由 -> Ingress -> 查看事件。
