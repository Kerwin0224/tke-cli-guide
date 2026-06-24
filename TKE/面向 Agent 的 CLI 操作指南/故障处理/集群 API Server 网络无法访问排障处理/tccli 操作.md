# 集群 API Server 网络无法访问排障处理（tccli）

> 对照官方：[集群 API Server 网络无法访问排障处理](https://cloud.tencent.com/document/product/457/80912) · page_id `80912`

## 概述

排查 TKE 集群 API Server 不可达问题：公网/内网端点状态、安全组、VPC 网络等。通过 tccli 检查集群端点和 kubeconfig 状态，不依赖 kubectl 即可完成控制面诊断。

> **kubectl 数据面不可达**：CAM拒绝公网端点(strategyId:240463971)，内网需VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

## 前置条件

- [环境准备](../../环境准备.md)
- 已获取集群 kubeconfig（如已下载）

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
| 查看集群端点 | `tccli tke DescribeClusterEndpoints --region ap-guangzhou --ClusterId cls-xxxxxxxx` | 是 |
| 查看集群状态 | `tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]'` | 是 |
| 获取 kubeconfig | `tccli tke DescribeClusterKubeconfig --region ap-guangzhou --ClusterId cls-xxxxxxxx` | 是 |
| 测试端口连通 | `telnet <APIServer> 443`（需 VPN/IOA） | 是 |

## 操作步骤

### 步骤1：控制面检查集群状态和端点

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
      "ClusterVersion": "1.30.0",
      "ClusterType": "MANAGED_CLUSTER"
    }
  ]
}
```

```bash
tccli tke DescribeClusterEndpoints --region ap-guangzhou --ClusterId cls-xxxxxxxx
```

**预期输出**：

```json
{
  "ClusterExternalEndpoint": "https://cls-xxxxxxxx.ccs.tencent-cloud.com",
  "ClusterInternalEndpoint": "https://cls-xxxxxxxx.ccs.tencent-cloud.com",
  "ExternalEndpointOpen": true,
  "InternalEndpointOpen": true
}
```

### 步骤2：获取 kubeconfig 测试

```bash
tccli tke DescribeClusterKubeconfig --region ap-guangzhou --ClusterId cls-xxxxxxxx
```

```json
{
  "Kubeconfig": "<Kubeconfig>",
  "RequestId": "<RequestId>"
}
```

**预期输出**：

- 成功时返回 Base64 编码的 kubeconfig 内容
- CAM 权限不足时返回 `UnauthorizedOperation` 错误

### 步骤3：连通性诊断

按以下顺序排查 API Server 不可达：

**第一步：确认集群状态**

集群必须处于 Running 状态。若为其他状态（Upgrading、Abnormal），等待恢复或联系售后。

**第二步：检查端点开启状态**

`DescribeClusterEndpoints` 返回 `ExternalEndpointOpen` 和 `InternalEndpointOpen` 状态。若公网端点未开启：
- 前往控制台开启公网访问
- 或使用 VPN/IOA 走内网端点

**第三步：检查安全组**

确认 API Server 443 端口在安全组中已放通：
- 集群安全组需允许客户端 IP 访问 443 端口
- 当前环境 CAM 拒绝公网端点 (strategyId:240463971)，需通过 VPN/IOA 连接内网端点

**第四步：检查 VPC 网络**

若使用内网端点：
- 确认客户端与集群 VPC 可通过专线/VPN/对等连接互通
- 确认 DNS 可解析内网端点域名

**第五步：检查 CAM 权限**

```bash
tccli tk

```json
{
  "Kubeconfig": "<Kubeconfig>",
  "RequestId": "<RequestId>"
}
```e DescribeClusterKubeconfig --region ap-guangzhou --ClusterId cls-xxxxxxxx
```

若返回权限错误，确认当前 CAM 子账号/角色有 `tke:DescribeClusterKubeconfig` 权限。

### 步骤4：数据面替代方案（不依赖 kubectl）

当 kubectl 不可达时，通过 tccli 仍可执行核心操作：

```bash
# 查看集群节点
tccli tke DescribeClusterInstances --region ap-guangzhou --ClusterId cls-xxxxxxxx
```

**预期输出**：

```json
{
  "TotalCount": 3,
  "InstanceSet": [
    {
      "InstanceId": "ins-xxxxxxxx",
      "InstanceRole": "WORKER",
      "InstanceState": "running",
      "LanIP": "10.0.1.10"
    }
  ]
}
```

## 验证

### 控制面（tccli）

```bash
tccli tke DescribeClusterKubeconfig --region ap-guangzhou --ClusterId cls-xxxxxxxx
# 预期: 成功返回 kubeconfig，无 CAM 错误
```

### 数据面（需 VPN/IOA）

```bash
kubectl get nodes
# 预期: 返回节点列表
```

## 清理

无需特殊清理（排障页）。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|---|---|---|---|
| `DescribeClusterKubeconfig` 返回 `UnauthorizedOperation` | 检查当前 CAM 策略 | 子账号缺少 `tke:DescribeClusterKubeconfig` 权限 | 添加策略：`tke:DescribeClusterKubeconfig` 或 `QcloudTKEFullAccess` |
| 公网端点可解析但 `telnet 443` 不通 | 检查安全组入站规则 | 安全组未放通 443 端口 | 在安全组添加入站规则：来源 `0.0.0.0/0`，端口 443 |
| 内网端点不可达 | 在内网/VPN 环境中测试 `telnet <内网IP> 443` | VPC 路由或 DNS 异常 | 检查 VPC 路由表；确认 DNS 解析正常 |

### 操作成功但异常

| 现象 | 诊断 | 根因 | 修复 |
|---|---|---|---|
| kubectl 间歇性超时 | 多次 `kubectl get nodes` 观察超时模式 | API Server 负载过高或网络抖动 | 检查集群节点数是否超出 API Server 规格；考虑升级集群规格 |
| CAM 拒绝公网端点 (strategyId:240463971) | API 返回策略拒绝信息 | 腾讯云 CAM 策略限制从公网访问集群数据面 | 必须通过 VPN/IOA 从内网端点访问 kubectl |

## 下一步

- [集群 DNS 解析异常排障处理](../集群%20DNS%20解析异常排障处理/tccli%20操作.md) -- page_id `80531`
- [集群 Kube-Proxy 异常排障处理](../集群%20Kube-Proxy%20异常排障处理/tccli%20操作.md) -- page_id `79797`

## 控制台替代

[容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) -> 集群 -> 基本信息 -> 集群 APIServer 信息 -> 开启/关闭公网/内网访问。
