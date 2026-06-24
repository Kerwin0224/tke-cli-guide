# Secret 管理（tccli）

> 对照官方：[Secret 管理](https://cloud.tencent.com/document/product/457/31718) · page_id `31718`
## 概述

Secret 是 Kubernetes 中存储敏感数据（密码、令牌、密钥等）的对象，数据以 Base64 编码存储。支持 Opaque、docker-registry、tls 三种类型。

> **kubectl 数据面不可达**：CAM拒绝公网端点(strategyId:240463971)，内网需VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。
## 前置条件

- [环境准备](../../../环境准备.md)：`tccli` 已配置，`kubectl` >= v1.28
- 已获取集群 kubeconfig

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 获取集群凭证 | `tccli tke DescribeClusterKubeconfig` | 是 |
| 创建 Opaque Secret | `kubectl create secret generic <name>` | 否 |
| 创建 TLS Secret | `kubectl create secret tls <name>` | 否 |
| 创建 docker-registry Secret | `kubectl create secret docker-registry <name>` | 否 |
| 查看 Secret 列表 | `kubectl get secrets` | 是 |
| 删除 Secret | `kubectl delete secret/<name>` | 是 |

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

### 2. 数据面：创建 Opaque Secret（需 VPN/IOA）

```bash
kubectl create secret generic db-secret \
  --from-literal=username=admin \
  --from-literal=password=MyPassw0rd
# expected: secret/db-secret created
```

### 3. 数据面：创建 TLS Secret

```bash
kubectl create secret tls tls-secret \
  --cert=server.crt \
  --key=server.key
# expected: secret/tls-secret created
```

### 4. 数据面：创建 docker-registry Secret

```bash
kubectl create secret docker-registry tcr-secret \
  --docker-server=<RegistryId>.tencentcloudcr.com \
  --docker-username=<TCR_USERNAME> \
  --docker-password=<TCR_TOKEN>
# expected: secret/tcr-secret created
```

### 5. 数据面：在 Pod 中使用 Secret

环境变量方式：

```yaml
spec:
  containers:
    - name: app
      image: nginx:alpine
      env:
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: username
        - name: DB_PASS
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: password
```

挂载卷方式：

```yaml
spec:
  containers:
    - name: app
      image: nginx:alpine
      volumeMounts:
        - name: tls
          mountPath: /etc/tls
  volumes:
    - name: tls
      secret:
        secretName: tls-secret
```

```bash
kubectl apply -f pod-with-secret.yaml
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
kubectl get secrets
# expected: 含 db-secret, tls-secret

kubectl describe secret db-secret
# expected: 显示 Data 字段（值不显示明文）
```

```text
NAME  STATUS  AGE
...
```

## 清理

```bash
kubectl delete secret db-secret tls-secret tcr-secret
# expected: secret deleted
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Pod `CreateContainerConfigError` | `kubectl describe pod` | Secret 不存在或 key 错误 | `kubectl describe pod` 查看 Events |
| `ImagePullBackOff` (TCR) | `kubectl describe pod` | docker-registry Secret 凭证无效 | 重新创建 TCR 令牌并更新 Secret |

## 下一步

- [ConfigMap 管理](../ConfigMap/tccli%20操作.md) -- page_id `31717`
- [使用 TCR 企业版实例内容器镜像创建工作负载](../../工作负载管理/使用%20TCR%20企业版实例内容器镜像创建工作负载/tccli%20操作.md) -- page_id `45624`

## 控制台替代

[容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) -> 集群 -> 配置管理 -> Secret -> 新建。
