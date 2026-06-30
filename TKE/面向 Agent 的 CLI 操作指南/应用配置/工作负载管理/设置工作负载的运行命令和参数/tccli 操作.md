# 设置工作负载的运行命令和参数（tccli）

> 对照官方：[设置工作负载的运行命令和参数](https://cloud.tencent.com/document/product/457/32816) · page_id `32816`

## 概述

通过 `command` 和 `args` 字段覆盖容器镜像默认的 ENTRYPOINT 和 CMD，实现自定义启动命令和运行参数。同时可通过 `env` 注入环境变量。

## 前置条件

- [环境准备](../../../环境准备.md)：`tccli` 已配置，`kubectl` >= v1.28
- 需要 CAM 权限：`tke:DescribeClusters`、`tke:DescribeClusterKubeconfig`

```bash
# 检查集群状态
tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'
# expected: ClusterStatus "Running"

# 获取 kubeconfig
tccli tke DescribeClusterKubeconfig --region <Region> --ClusterId <ClusterId> \
    | jq -r '.Kubeconfig' > kubeconfig.yaml
export KUBECONFIG=$(pwd)/kubeconfig.yaml
```

> **注意**：kubectl 数据面当前因公网端点被组织级 CAM 策略拒绝（strategyId:240463971）而不可达。外网端点被 `tke:clusterExtranetEndpoint=true` 条件拦截，内网端点需通过 IOA/VPN 或同 VPC 内 CVM 访问。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 获取集群凭证 | `tccli tke DescribeClusterKubeconfig --region <Region> --ClusterId <ClusterId>` | 是 |
| 设置运行命令 | `kubectl apply -f pod-with-command.yaml` | 是 |
| 查看容器日志 | `kubectl logs <pod>` | 是 |
| 进入容器 | `kubectl exec -it <pod> -- /bin/sh` | 否 |

## 操作步骤

### 步骤 1：设置 command 和 args

`command` 覆盖镜像 ENTRYPOINT，`args` 覆盖镜像 CMD。YAML 清单：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: command-demo
  namespace: default
spec:
  containers:
    - name: demo
      image: busybox:latest
      command: ["/bin/sh"]
      args: ["-c", "while true; do echo Hello $(date); sleep 10; done"]
```

| 参数 | 说明 | 示例 |
|------|------|------|
| `command` | 覆盖 ENTRYPOINT | `["/bin/sh"]` |
| `args` | 覆盖 CMD | `["-c", "echo hello"]` |
| `env[].value` | 明文环境变量 | `"production"` |
| `env[].valueFrom` | 从 Secret/ConfigMap 引用 | `secretKeyRef` |

```bash
kubectl apply -f pod-command.yaml
# expected: pod/command-demo created
```

**预期输出**：

```text
pod/command-demo created
```

### 步骤 2：验证运行结果

```bash
kubectl logs command-demo
# expected: 输出 Hello 和时间戳
```

**预期输出**（示例）：

```text
Hello Wed Jun 16 10:00:00 UTC 2026
Hello Wed Jun 16 10:00:10 UTC 2026
Hello Wed Jun 16 10:00:20 UTC 2026
```

### 步骤 3：环境变量

通过 `env` 注入环境变量，支持明文和从 Secret/ConfigMap 引用：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: env-demo
  namespace: default
spec:
  containers:
    - name: demo
      image: busybox:latest
      command: ["/bin/sh"]
      args: ["-c", "echo APP_ENV=$APP_ENV; sleep 3600"]
      env:
        - name: APP_ENV
          value: "production"
        - name: DB_HOST
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: host
```

```bash
kubectl apply -f pod-env.yaml
# expected: pod/env-demo created

kubectl logs env-demo
# expected: APP_ENV=production
```

```text
NAME  STATUS  AGE
...
```

### 步骤 4：进入容器验证

```bash
kubectl exec -it command-demo -- /bin/sh
# expected: 进入容器 shell
```

## 验证

### 控制面（tccli）

```bash
tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'
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

### 数据面（kubectl）

```bash
kubectl get pod command-demo
# expected: STATUS Running

kubectl logs command-demo
# expected: 输出命令执行结果

kubectl exec command-demo -- env
# expected: 显示环境变量
```

```text
NAME  STATUS  AGE
...
```

## 清理

```bash
kubectl delete pod command-demo
# expected: pod "command-demo" deleted
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `Unable to connect to the server` | `kubectl cluster-info` 检查连通性 | 公网端点被 CAM 策略 strategyId:240463971 以 `tke:clusterExtranetEndpoint=true` 条件拒绝 | 通过 IOA/VPN 连接内网，或使用同 VPC CVM 执行 kubectl 命令 |
| `exec format error` | `kubectl logs <pod>` 查看错误 | command 与镜像架构不匹配 | 检查镜像平台（amd64/arm64） |

### 资源创建成功但状态异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Pod `CrashLoopBackOff` | `kubectl logs <pod>` 查看日志 | command 执行后立即退出 | 确保前台进程持续运行 |
| 环境变量未生效 | `kubectl exec <pod> -- env` 检查 | Secret 不存在或格式错误 | `kubectl describe pod <pod>` 检查 Events，确认 Secret 存在 |

## 下一步

- [设置工作负载的健康检查](../设置工作负载的健康检查/tccli%20操作.md) — page_id `32815`
- [设置工作负载的资源限制](../设置工作负载的资源限制/tccli%20操作.md) — page_id `32813`

## 控制台替代

[容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) → 集群 → 工作负载 → 新建 → 高级设置 → 运行命令。
