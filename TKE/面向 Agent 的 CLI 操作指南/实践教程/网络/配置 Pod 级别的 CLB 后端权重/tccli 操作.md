# 配置 Pod 级别的 CLB 后端权重（tccli）

> 对照官方：[配置 Pod 级别的 CLB 后端权重](https://cloud.tencent.com/document/product/457/128647) · page_id `128647`

## 概述

通过 Pod 注解 `tke.cloud.tencent.com/clb-target-weight` 配置单个 Pod 在 CLB 后端的流量权重，实现细粒度的流量分发。

> **kubectl 数据面不可达**：CAM拒绝公网端点(strategyId:240463971)，内网需VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

## 前置条件

### 环境检查

```bash
tccli --version
# expected: tccli version >= 1.0.0

tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterStatus "Running"

tccli clb DescribeLoadBalancers --region <Region>
# expected: exit 0, CLB 已创建
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli/kubectl 命令 | 幂等 |
|-----------|-------------------|:--:|
| 查看 CLB 后端权重 | `tccli clb DescribeTargets --LoadBalancerId CLB_ID` | 是 |
| 设置 Pod 权重注解 | `kubectl annotate pod POD_NAME tke.cloud.tencent.com/clb-target-weight=WEIGHT`（需 VPN/IOA） | 是 |
| 修改 CLB 权重 | `tccli clb ModifyTargetWeight` | 否 |
| 查看 Pod 注解 | `kubectl describe pod POD_NAME`（需 VPN/IOA） | 是 |

## 操作步骤

### 步骤 1：创建带权重的 Service

#### 选择依据

- Pod 权重控制 vs CLB 直接配置：Pod 注解方式声明式，随 Pod 生命周期自动管理
- 权重范围：0-100，0 表示不接收流量（用于灰度），100 表示全量
- 适用场景：金丝雀发布、A/B 测试、环境隔离

```yaml
apiVersion: v1
kind: Service
metadata:
  name: weighted-svc
  namespace: NAMESPACE
  annotations:
    service.kubernetes.io/loadbalance-id: "CLB_ID"
spec:
  type: LoadBalancer
  selector:
    app: backend
  ports:
  - port: 80
    targetPort: 8080
```

```bash
kubectl apply -f weighted-svc.yaml
# expected: service/weighted-svc created
```

### 步骤 2：为 Pod 设置权重注解

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: backend-canary
  namespace: NAMESPACE
  labels:
    app: backend
  annotations:
    tke.cloud.tencent.com/clb-target-weight: "10"
spec:
  containers:
  - name: backend
    image: nginx:latest
    ports:
    - containerPort: 8080
---
apiVersion: v1
kind: Pod
metadata:
  name: backend-stable
  namespace: NAMESPACE
  labels:
    app: backend
  annotations:
    tke.cloud.tencent.com/clb-target-weight: "90"
spec:
  containers:
  - name: backend
    image: nginx:latest
    ports:
    - containerPort: 8080
```

```bash
kubectl apply -f pods-weighted.yaml
# expected: pod/backend-canary created, pod/backend-stable created
```

### 步骤 3：验证 CLB 后端权重

```bash
# 查看 CLB 后端列表
tccli clb DescribeTargets --region <Region> --LoadBalancerId CLB_ID
# expected: Targets 含两个后端，Weight 分别为 10 和 90
```

预期输出：

```json
{
    "Listeners": [{
        "Targets": [
            {"EniIp": "10.0.1.10", "Weight": 10, "Port": 8080},
            {"EniIp": "10.0.1.11", "Weight": 90, "Port": 8080}
        ]
    }]
}
```

### 步骤 4：动态调整权重

```bash
# 灰度放量：调整 canary 权重从 10 到 50
kubectl annotate pod backend-canary -n NAMESPACE \
    tke.cloud.tencent.com/clb-target-weight=50 --overwrite
# expected: pod/backend-canary annotated

kubectl annotate pod backend-stable -n NAMESPACE \
    tke.cloud.tencent.com/clb-target-weight=50 --overwrite
# expected: pod/backend-stable annotated

# 验证 CLB 权重已更新
tccli clb DescribeTargets --region <Region> --LoadBalancerId CLB_ID
# expected: Weight 均为 50
```

## 验证

### 控制面（tccli）

```bash
tccli clb DescribeTargets --region <Region> --LoadBalancerId CLB_ID
# expected: 各后端 Weight 与 Pod 注解一致
```

### 数据面（需 VPN/IOA）

```bash
kubectl get pods -n NAMESPACE -l app=backend -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.metadata.annotations.tke\.cloud\.tencent\.com/clb-target-weight}{"\n"}{end}'
# expected: 各 Pod 含权重注解
```

```text
NAME  STATUS  AGE
...
```

## 清理

```bash
kubectl delete pods backend-canary backend-stable -n NAMESPACE
kubectl delete svc weighted-svc -n NAMESPACE
# expected: 资源已删除，CLB 后端自动移除
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 权重注解不生效 | `kubectl describe pod POD_NAME` | Service 类型非 LoadBalancer 或 CLB 未关联 | 确认 Service type=LoadBalancer，CLB annotation 正确 |
| CLB 权重与注解不一致 | `tccli clb DescribeTargets --LoadBalancerId CLB_ID` | CLB 控制器同步延迟 | 等待 10-30 秒后重查 |
| Pod 权重为 0 仍收到流量 | 检查 CLB 健康检查 | 健康检查认为 Pod 正常，0 权重仅降低优先 | 将 Pod 从 Service selector 中移除以彻底停止流量 |
| 权重值超出范围 | `kubectl describe pod POD_NAME` | 权重注解值 > 100 或为负数 | 使用 0-100 之间的整数 |

## 下一步

- [在 TKE 上使用负载均衡直连 Pod](../在%20TKE%20上使用负载均衡直连%20Pod/tccli%20操作.md) -- page_id `48793`
- [TKE 基于弹性网卡直连 Pod 的网络负载均衡](../TKE%20基于弹性网卡直连%20Pod%20的网络负载均衡/tccli%20操作.md) -- page_id `48768`

## 控制台替代

[CLB 控制台](https://console.cloud.tencent.com/clb) -> 监听器 -> 后端服务 -> 修改权重。
