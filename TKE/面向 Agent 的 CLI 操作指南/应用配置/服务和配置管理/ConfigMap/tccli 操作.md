# ConfigMap 管理（tccli）

> 对照官方：[ConfigMap 管理](https://cloud.tencent.com/document/product/457/31717) · page_id `31717`
## 概述

ConfigMap 是 Kubernetes 中存储非敏感配置数据的对象，以 key-value 形式管理配置。可通过环境变量、命令行参数或挂载卷方式注入容器。

> **kubectl 数据面不可达**：CAM拒绝公网端点(strategyId:240463971)，内网需VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。
## 前置条件

- [环境准备](../../../环境准备.md)：`tccli` 已配置，`kubectl` >= v1.28
- 已获取集群 kubeconfig

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 获取集群凭证 | `tccli tke DescribeClusterKubeconfig` | 是 |
| 创建 ConfigMap（文件） | `kubectl create configmap <name> --from-file=<path>` | 否 |
| 创建 ConfigMap（字面量） | `kubectl create configmap <name> --from-literal=<k>=<v>` | 否 |
| 查看 ConfigMap | `kubectl get configmap` | 是 |
| 查看详情 | `kubectl describe configmap/<name>` | 是 |
| 删除 ConfigMap | `kubectl delete configmap/<name>` | 是 |

## 操作步骤

### 1. 控制面：获取集群凭证

```bash
tccli tke DescribeClusterKubeconfig --region <Region> --ClusterId <ClusterId> \
    --filter "Kubeconfig" > kubeconfig.yaml
export KUBECONFIG=$(pwd)/kubeconfig.yaml
# expected: kubeconfig 写入本地文件
```

```json
{
  "Kubeconfig": "<Kubeconfig>",
  "RequestId": "<RequestId>"
}
```

### 2. 数据面：创建 ConfigMap（需 VPN/IOA）

方式一：从字面量创建：

```bash
kubectl create configmap app-config \
  --from-literal=APP_ENV=production \
  --from-literal=LOG_LEVEL=info
# expected: configmap/app-config created
```

方式二：从文件创建：

```bash
kubectl create configmap nginx-config --from-file=nginx.conf
# expected: configmap/nginx-config created
```

方式三：从 YAML 创建：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: default
data:
  APP_ENV: "production"
  LOG_LEVEL: "info"
  app.properties: |
    server.port=8080
    db.timeout=30
```

```bash
kubectl apply -f configmap.yaml
# expected: configmap/app-config created
```

### 3. 数据面：在 Pod 中使用 ConfigMap

环境变量方式：

```yaml
spec:
  containers:
    - name: app
      image: nginx:alpine
      envFrom:
        - configMapRef:
            name: app-config
```

挂载卷方式：

```yaml
spec:
  containers:
    - name: app
      image: nginx:alpine
      volumeMounts:
        - name: config
          mountPath: /etc/app/config
  volumes:
    - name: config
      configMap:
        name: app-config
```

```bash
kubectl apply -f pod-with-configmap.yaml
# expected: pod/<name> created
```

## 验证

### 控制面（tccli）

```bash
tccli tke DescribeClusterStatus --region <Region> --ClusterIds '["<ClusterId>"]'
# expected: ClusterState "Running"
```

```json
{
  "ClusterStatusSet": "<ClusterStatusSet>",
  "ClusterId": "<ClusterId>",
  "ClusterState": "<ClusterState>",
  "ClusterInstanceState": "<ClusterInstanceState>",
  "ClusterBMonitor": "<ClusterBMonitor>",
  "ClusterInitNodeNum": "<ClusterInitNodeNum>"
}
```

### 数据面（kubectl，需 VPN/IOA）

```bash
kubectl get configmap
# expected: 含 app-config

kubectl describe configmap app-config
# expected: 显示配置数据

kubectl exec <pod> -- env | grep APP_ENV
# expected: APP_ENV=production
```

```text
NAME  STATUS  AGE
...
```

## 清理

```bash
kubectl delete configmap app-config
# expected: configmap "app-config" deleted
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Pod `CreateContainerConfigError` | `kubectl describe pod` | ConfigMap 不存在或挂载路径错误 | `kubectl describe pod` 查看 Events |
| 环境变量未生效 | `kubectl exec <pod> -- env` | ConfigMap 更新不自动同步 | 重启 Pod 或使用 Reloader |

## 下一步

- [Secret 管理](../Secret/tccli%20操作.md) -- page_id `31718`
- [Deployment 管理](../../工作负载管理/Deployment%20管理/tccli%20操作.md) -- page_id `31705`

## 控制台替代

[容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) -> 集群 -> 配置管理 -> ConfigMap -> 新建。
