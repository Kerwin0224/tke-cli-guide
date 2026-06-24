# Service&Ingress 网络无法访问排障处理（tccli）

> 对照官方：[Service&Ingress 网络无法访问排障处理](https://cloud.tencent.com/document/product/457/80913) · page_id `80913`

## 概述

Service 或 Ingress 创建成功但流量无法到达后端 Pod。排查 CLB 健康检查、Endpoints、kube-proxy 规则、安全组等。通过 tccli 检查 CLB 后端状态，配合 kubectl 检查 Service/Endpoints 和网络连通性。

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
| 查看 CLB 后端健康 | `tccli clb DescribeTargetHealth --region ap-guangzhou --LoadBalancerId <lb-id>` | 是 |
| 查看 CLB 监听器 | `tccli clb DescribeListeners --region ap-guangzhou --LoadBalancerId <lb-id>` | 是 |
| 查看 Endpoints | `kubectl get endpoints <svc>`（需 VPN/IOA） | 是 |
| 集群内连通性测试 | `kubectl run test --image=busybox --rm -it -- <cmd>`（需 VPN/IOA） | 是 |

## 操作步骤

### 步骤1：控制面检查集群

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

### 步骤2：检查 Endpoints（需 VPN/IOA）

```bash
kubectl get endpoints <svc-name> -n <ns>
# 预期: ENDPOINTS 列有 Pod IP:Port，非 <none>
```

```text
NAME  STATUS  AGE
...
```

如果 Endpoints 为空：
- Service selector 不匹配任何 Pod
- Pod 未 Ready（readinessProbe 失败）
- 端口号不一致

### 步骤3：检查 CLB 后端健康（tccli）

```bash
tccli clb DescribeTargetHealth --region ap-guangzhou --LoadBalancerId <lb-id>
```

**预期输出**：

```json
{
  "LoadBalancers": [
    {
      "LoadBalancerId": "lb-xxxxxxxx",
      "Listeners": [
        {
          "ListenerId": "lbl-xxxxxxxx",
          "Rules": [
            {
              "Targets": [
                {
                  "TargetId": "ins-xxxxxxxx",
                  "Port": 80,
                  "HealthStatus": "Healthy"
                }
              ]
            }
          ]
        }
      ]
    }
  ]
}
```

### 步骤4：集群内连通性测试（需 VPN/IOA）

```bash
# 测试 Service ClusterIP
kubectl run test-svc --image=busybox --rm -it --restart=Never -- wget -O- -T3 http://<svc-name>.<ns>.svc.cluster.local:<port>
# 预期: 返回预期内容

# 直接测试 Pod IP
kubectl run test-pod --image=busybox --rm -it --restart=Never -- wget -O- -T3 http://<pod-ip>:<port>
# 预期: 返回预期内容
```

### 步骤5：诊断表

| 现象 | 诊断 | 根因 | 修复 |
|---|---|---|---|
| Endpoints 为空 | `kubectl get pod -l <selector>` 无匹配 Pod | selector 不匹配 | 修正 Service selector 标签 |
| Endpoints 有 IP 但 CLB 后端不健康 | `tccli clb DescribeTargetHealth` 显示 Unhealthy | Pod 端口不监听或健康检查路径错误 | 检查 Pod 端口监听；调整 CLB 健康检查路径 |
| ClusterIP 可达但 CLB VIP 不可达 | 集群内测试成功，外网测试失败 | CLB 安全组或监听器未正确配置 | 检查 CLB 安全组和监听器后端端口 |
| 所有通路正常但仍无响应 | `kubectl logs <pod>` 查看应用日志 | 应用层问题（如业务逻辑错误） | 排查应用日志 |

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
kubectl get endpoints -A --no-headers | grep -v "<none>"
# 预期: 所有 Service 有后端 Endpoints

curl -s -o /dev/null -w "%{http_code}" http://<CLB-VIP>:<port>/<healthz>
# 预期: 返回 200
```

```text
NAME  STATUS  AGE
...
```

## 清理

无需特殊清理（排障页）。测试用 Pod 使用 `--rm` 会自动清理。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|---|---|---|---|
| `tccli clb DescribeTargetHealth` 返回权限错误 | 检查当前 CAM 用户/角色策略 | 缺少 `clb:DescribeTargetHealth` 权限 | 添加 CLB 只读权限 `QcloudCLBReadOnlyAccess` |
| 集群内 wget 返回 `Connection Refused` | 目标端口上无进程监听 | 容器内部端口配置错误或进程未启动 | 检查容器内实际监听端口；`kubectl exec <pod> -- netstat -tlnp` |

### 操作成功但异常

| 现象 | 诊断 | 根因 | 修复 |
|---|---|---|---|
| 间歇性超时 | 持续压测观察超时分布 | 部分节点 kube-proxy 规则未同步 | 重启对应节点的 kube-proxy Pod |
| 长连接偶发断开 | 检查 CLB 空闲连接超时设置 | CLB 默认空闲超时 60 秒 | 在 CLB 控制台调整空闲连接超时时间 |

## 下一步

- [Service&Ingress 常见报错和处理](../Service&Ingress%20常见报错和处理/tccli%20操作.md) -- page_id `80913`
- [集群 Kube-Proxy 异常排障处理](../集群%20Kube-Proxy%20异常排障处理/tccli%20操作.md) -- page_id `79797`

## 控制台替代

[容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) -> 服务与路由 -> Service -> 后端列表；CLB 控制台 -> 监听器管理。
