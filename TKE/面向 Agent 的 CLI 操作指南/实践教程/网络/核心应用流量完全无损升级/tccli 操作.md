# 核心应用流量完全无损升级（tccli）

> 对照官方：[核心应用流量完全无损升级](https://cloud.tencent.com/document/product/457/104996) · page_id `104996`

## 概述

核心应用滚动更新时，若 Pod 直接 SIGTERM 而 CLB 仍向其转发流量，会出现 502/连接重置。本文通过 preStop Hook、readinessProbe、terminationGracePeriodSeconds、PodDisruptionBudget 与 CLB 连接优雅退出配合，实现零流量丢失的滚动更新。控制面用 tccli 验证集群与 CLB 状态，数据面用 kubectl 配置 Deployment 并触发更新。

> **kubectl 数据面不可达**：CAM拒绝公网端点(strategyId:240463971)，内网需VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

## 前置条件

- [环境准备](../../../环境准备.md)：`tccli` 已配置，`kubectl` 已在内网/VPN 环境就绪
- 集群为 MANAGED_CLUSTER，K8s 1.30.0，ap-guangzhou
- 目标应用已通过 LoadBalancer/Ingress 暴露，CLB 已创建
- 应用支持健康检查端点（如 `/healthz`）并支持优雅退出（收到 SIGTERM 后关闭监听）

### 环境检查

```bash
tccli --version
# expected: tccli version >= 1.0.0

tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterStatus "Running"
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

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 查看集群状态 | `tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'` | 是 |
| 查看 CLB 监听器 | `tccli clb DescribeListeners --region <Region> --LoadBalancerId CLB_ID` | 是 |
| 查看 CLB 后端健康 | `tccli clb DescribeTargetHealth --region <Region> --LoadBalancerId CLB_ID` | 是 |
| 修改 CLB 健康检查 | `tccli clb ModifyListener --region <Region> --cli-input-json file://clb-health.json` | 是 |
| 滚动更新 Deployment | `kubectl set image deployment/APP_NAME ...`（需 VPN/IOA） | 否 |
| 配置 PDB | `kubectl apply -f pdb.yaml`（需 VPN/IOA） | 是 |
| 查看 rollout 状态 | `kubectl rollout status deployment/APP_NAME`（需 VPN/IOA） | 是 |

## 操作步骤

### 步骤 1：配置 preStop Hook（需 VPN/IOA）

#### 选择依据

- **preStop sleep 30s**：Pod 收到删除信号后先 sleep，给 CLB 健康检查（默认 5s 间隔 × 3 次不健康 = 15s）足够时间摘除该 Pod，避免 CLB 仍向即将退出的 Pod 转发新连接
- **sleep 时长 > CLB 健康检查摘除时间**：本例 CLB 不健康阈值 3 × 间隔 5s = 15s，preStop sleep 30s 留出 2 倍余量
- 用 `kubectl patch` 增量修改，无需重写整个 Deployment

```bash
kubectl patch deployment APP_NAME -n NAMESPACE --type=strategic -p '{
  "spec": {
    "template": {
      "spec": {
        "terminationGracePeriodSeconds": 60,
        "containers": [
          {
            "name": "app",
            "lifecycle": {
              "preStop": {
                "exec": {
                  "command": ["/bin/sh", "-c", "sleep 30"]
                }
              }
            }
          }
        ]
      }
    }
  }
}'
# expected: deployment.apps/APP_NAME patched
```

> 注意：patch 会触发滚动更新。`terminationGracePeriodSeconds: 60` 必须大于 preStop sleep（30s）+ 应用退出时间，否则 Pod 被 SIGKILL。

### 步骤 2：配置 readinessProbe（需 VPN/IOA）

#### 选择依据

- **readinessProbe** 决定 Pod 是否进入 Service Endpoints。更新时新 Pod 只有 readiness 通过才会接收流量，旧 Pod 在 preStop 期间 readiness 失败被摘除
- **path 与 CLB 健康检查路径一致**：避免 kube-proxy 与 CLB 摘除时序不一致
- **initialDelaySeconds: 5 / periodSeconds: 5**：快速发现就绪与不健康状态

```bash
kubectl patch deployment APP_NAME -n NAMESPACE --type=strategic -p '{
  "spec": {
    "template": {
      "spec": {
        "containers": [
          {
            "name": "app",
            "readinessProbe": {
              "httpGet": {
                "path": "/healthz",
                "port": 8080
              },
              "initialDelaySeconds": 5,
              "periodSeconds": 5,
              "failureThreshold": 3
            }
          }
        ]
      }
    }
  }
}'
# expected: deployment.apps/APP_NAME patched
```

### 步骤 3：确认 terminationGracePeriodSeconds（需 VPN/IOA）

#### 选择依据

- `terminationGracePeriodSeconds` 必须覆盖 preStop sleep + 应用优雅退出时长。preStop 30s + 应用退出 10s → 设为 60s 留余量
- 超时后 kubelet 发送 SIGKILL，导致进行中的请求被重置

```bash
# 确认当前值
kubectl get deployment APP_NAME -n NAMESPACE -o jsonpath='{.spec.template.spec.terminationGracePeriodSeconds}'
# expected: 60
```

```text
NAME  STATUS  AGE
...
```

### 步骤 4：创建 PodDisruptionBudget（需 VPN/IOA）

#### 选择依据

- **PDB minAvailable: 2**：滚动更新时保证至少 2 个 Pod 可用，防止更新控制器一次驱逐过多 Pod 导致可用容量不足
- **selector 匹配 Deployment label**：PDB 只对匹配的 Pod 生效

```bash
cat > pdb.yaml <<'EOF'
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: APP_NAME-pdb
  namespace: NAMESPACE
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: APP_NAME
EOF

kubectl apply -f pdb.yaml
# expected: poddisruptionbudget.policy/APP_NAME-pdb created
```

### 步骤 5：配置 CLB 健康检查（tccli）

#### 选择依据

- CLB 健康检查摘除速度必须快于 Pod 退出速度。默认 5s 间隔、3 次不健康阈值 = 15s 摘除，配合 preStop 30s 留足时间
- **HealthSwitch: 1** 开启健康检查，**UnHealthNum: 3** 即 3 次失败摘除

```bash
cat > clb-health.json <<'EOF'
{
    "LoadBalancerId": "CLB_ID",
    "ListenerId": "LISTENER_ID",
    "HealthCheck": {
        "HealthSwitch": 1,
        "TimeOut": 2,
        "IntervalTime": 5,
        "HealthNum": 3,
        "UnHealthNum": 3
    }
}
EOF
tccli clb ModifyListener --region <Region> --cli-input-json file://clb-health.json
# expected: RequestId 返回，无 Error
```

### 步骤 6：触发滚动更新（需 VPN/IOA）

```bash
# 更新镜像触发滚动更新
kubectl set image deployment/APP_NAME app=IMAGE:NEW_TAG -n NAMESPACE
# expected: deployment.apps/APP_NAME image updated

# 监控更新进度
kubectl rollout status deployment/APP_NAME -n NAMESPACE --watch
# expected: deployment "APP_NAME" successfully rolled out
```

## 验证

### 控制面（tccli）

```bash
# 确认 CLB 后端全部健康
tccli clb DescribeTargetHealth --region <Region> --LoadBalancerId CLB_ID
# expected: 所有 Targets 的 HealthStatus 为 HealthCheckPassNormal（1）
```

```output
{
  "HealthCheckStatusList": [
    {
      "ListenerId": "lbl-example",
      "Targets": [
        {"IP": "10.0.1.10", "Port": 8080, "HealthStatus": "HealthCheckPassNormal"}
      ]
    }
  ]
}
```

```bash
# 确认集群状态正常
tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]' \
    --filter 'Clusters[0].ClusterStatus'
# expected: "Running"
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

### 数据面（需 VPN/IOA）

```bash
# 确认 PDB 生效
kubectl get pdb APP_NAME-pdb -n NAMESPACE
# expected: ALLOWED DISRUPTIONS >= 1
```

```text
NAME  STATUS  AGE
...
```

```output
NAME           MIN AVAILABLE   ALLOWED DISRUPTIONS   AGE
APP_NAME-pdb   2               1                     5m
```

```bash
# 确认所有 Pod Running 且新镜像
kubectl get pods -n NAMESPACE -l app=APP_NAME
# expected: 所有 Pod Running，IMAGE 列为 NEW_TAG

# 持续访问验证零丢包（更新期间持续 curl）
while true; do curl -s -o /dev/null -w "%{http_code}\n" http://<CLB-IP>/; done
# expected: 持续返回 200，更新期间无 5xx
```

```text
NAME  STATUS  AGE
...
```

## 清理

### 数据面（需 VPN/IOA）

```bash
kubectl delete pdb APP_NAME-pdb -n NAMESPACE
# expected: poddisruptionbudget.policy "APP_NAME-pdb" deleted
```

Deployment 和 CLB 配置通常保留（生产应用）。如为测试环境需清理：

```bash
kubectl delete deployment APP_NAME -n NAMESPACE
# expected: deployment.apps "APP_NAME" deleted

kubectl delete svc APP_NAME -n NAMESPACE
# expected: service "APP_NAME" deleted
```

### 控制面（tccli）

> **计费警告**：CLB 按小时计费，删除 Service 后 TKE 自动释放 CLB（约 1-2 分钟）。手动复用的 CLB 需手动释放。

```bash
# 确认 CLB 已随 Service 删除被释放（等待 1-2 分钟）
tccli clb DescribeLoadBalancers --region <Region>
# expected: 不再返回测试 CLB；如仍存在且为测试用途，手动释放
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 滚动更新期间 502 Bad Gateway | `kubectl logs -n NAMESPACE -l app=APP_NAME --tail=100` 查看退出日志 | preStop sleep 不足，CLB 仍向退出中 Pod 转发新连接 | 增大 preStop sleep 到 30-60s，确保 > CLB 摘除时间（unhealthyThreshold × intervalTime） |
| 更新期间 connection reset by peer | `kubectl describe pod POD_NAME -n NAMESPACE` 查看 Last State | terminationGracePeriodSeconds 不足，Pod 被 SIGKILL | 增大 terminationGracePeriodSeconds 到 preStop + 应用退出时间 + 10s 余量 |
| 滚动更新卡住不动 | `kubectl rollout status deployment/APP_NAME -n NAMESPACE`；`kubectl get pdb APP_NAME-pdb` | PDB minAvailable 过严，可用 Pod 不足无法驱逐 | 临时调低 minAvailable 或 `kubectl delete pdb APP_NAME-pdb` 后重试 |
| 新 Pod 一直不 Ready | `kubectl describe pod POD_NAME -n NAMESPACE` 查看 Events | readinessProbe path/端口错误或应用启动慢 | 核对 readinessProbe path 与端口；增大 initialDelaySeconds 适应应用启动时间 |
| CLB 后端健康检查持续失败 | `tccli clb DescribeTargetHealth --region <Region> --LoadBalancerId CLB_ID` | CLB 健康检查端口与 Service targetPort 不一致，或路径与 readinessProbe 不同 | `tccli clb DescribeListeners --region <Region> --LoadBalancerId CLB_ID` 核对后端端口和健康检查路径 |
| 更新慢/Pod 排水缓慢 | `kubectl get pods -n NAMESPACE -l app=APP_NAME -w` | preStop sleep 过长或 maxUnavailable=0 导致串行更新 | 调整 preStop sleep；设置 `maxSurge: 1` 允许先扩再缩加速更新 |

## 下一步

- [工作负载平滑升级](../../服务部署/工作负载平滑升级/tccli%20操作.md) -- page_id `77969`
- [Nginx 升级最佳实践](../Nginx%20升级最佳实践/tccli%20操作.md) -- page_id `90454`
- [ingress-nginx 应用部署指南](../ingress-nginx%20应用部署指南/tccli%20操作.md) -- page_id `116122`

## 控制台替代

[容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) → 集群 → 工作负载 → Deployment → 更新实例 → 编辑 YAML。
