# 使用 TencentDB 的 WordPress

> 对照官方：[使用 TencentDB 的 WordPress](https://cloud.tencent.com/document/product/457/7447) · page_id `7447`
## 概述

在 TKE 集群中部署 WordPress，后端数据库使用腾讯云 TencentDB for MySQL（CDB），通过 Secret 管理数据库密码，PVC 持久化 WordPress 上传文件。

> **注意**：需集群端点可达（公网端点被 CAM 策略 strategyId:240463971 硬拒绝，内网端点创建失败）。在具备 IOA/VPN 或同 VPC CVM 环境下执行。
## 前置条件

### 环境检查

```bash
tccli --version
# expected: tccli version 1.x.x

tccli configure list
# expected: secretId, secretKey, region 均已配置
```

所需 CAM 权限：`tke:DescribeClusterKubeconfig`、`cdb:DescribeDBInstances`

### 资源检查

```bash
tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]'
# expected: ClusterStatus "Running", ClusterType "MANAGED_CLUSTER", ClusterVersion "1.30.0"
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

确认 kubectl 工具可用：

```bash
kubectl version --client --short 2>/dev/null || kubectl version --client
# expected: Client Version >= v1.28
```

确认 TencentDB MySQL 实例已创建且与集群在同一 VPC：

```bash
tccli cdb DescribeDBInstances --region ap-guangzhou --InstanceIds '["<CDB-InstanceId>"]'
# expected: Status 1（运行中）, VpcId 与集群 VPC 一致
```

获取 kubeconfig：

```bash
tccli tke DescribeClusterKubeconfig --region ap-guangzhou --ClusterId cls-xxxxxxxx \
    --filter "Kubeconfig" > kubeconfig.yaml
export KUBECONFIG=$(pwd)/kubeconfig.yaml
# expected: 生成 kubeconfig.yaml
```

```json
{
  "Kubeconfig": "<Kubeconfig>",
  "RequestId": "<RequestId>"
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 获取集群凭证 | `tccli tke DescribeClusterKubeconfig` | 是 |
| 创建数据库密码 Secret | `kubectl create secret generic` | 否 |
| 部署 WordPress + PVC | `kubectl apply -f wordpress-tencentdb.yaml` | 是 |
| 创建 Service | `kubectl apply -f wordpress-svc.yaml` | 是 |
| 查看部署状态 | `kubectl get deployment wordpress` | 是 |
| 查看 PVC 绑定状态 | `kubectl get pvc wordpress-pvc` | 是 |
| 删除应用 | `kubectl delete -f wordpress-tencentdb.yaml` | 是 |

## 操作步骤

### 1. 控制面：获取集群凭证

```bash
tccli tke DescribeClusterKubeconfig --region ap-guangzhou --ClusterId cls-xxxxxxxx \
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

### 2. 数据面：创建数据库密码 Secret（需 VPN/IOA）

```bash
kubectl create secret generic wordpress-db-secret \
  --from-literal=username=<DB_USER> \
  --from-literal=password=<DB_PASSWORD>
# expected: secret/wordpress-db-secret created
```

验证 Secret 已创建：

```bash
kubectl get secret wordpress-db-secret -o yaml
# expected: data.username, data.password 已编码
```

### 3. 数据面：部署 WordPress（需 VPN/IOA）

创建 `wordpress-tencentdb.yaml`：

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: wordpress-pvc
  namespace: default
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
  namespace: default
  labels:
    app: wordpress
spec:
  replicas: 1
  selector:
    matchLabels:
      app: wordpress
  template:
    metadata:
      labels:
        app: wordpress
    spec:
      containers:
        - name: wordpress
          image: wordpress:latest
          ports:
            - containerPort: 80
          env:
            - name: WORDPRESS_DB_HOST
              value: "<CDB_HOST>:3306"
            - name: WORDPRESS_DB_USER
              valueFrom:
                secretKeyRef:
                  name: wordpress-db-secret
                  key: username
            - name: WORDPRESS_DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: wordpress-db-secret
                  key: password
            - name: WORDPRESS_DB_NAME
              value: "wordpress"
          volumeMounts:
            - name: wordpress-data
              mountPath: /var/www/html/wp-content
      volumes:
        - name: wordpress-data
          persistentVolumeClaim:
            claimName: wordpress-pvc
```

创建 `wordpress-svc.yaml`：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: wordpress
  namespace: default
spec:
  type: LoadBalancer
  selector:
    app: wordpress
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

```bash
kubectl apply -f wordpress-tencentdb.yaml
# expected: persistentvolumeclaim/wordpress-pvc created, deployment.apps/wordpress created

kubectl apply -f wordpress-svc.yaml
# expected: service/wordpress created
```

### 4. 数据面：验证部署

```bash
kubectl get deployment wordpress
# expected: READY 1/1

kubectl get pvc wordpress-pvc
# expected: STATUS Bound

kubectl get service wordpress
# expected: EXTERNAL-IP 非 <pending>
```

```text
NAME  STATUS  AGE
...
```

确认 Secret 正确注入：

```bash
kubectl exec deployment/wordpress -- env | grep WORDPRESS_DB
# expected: WORDPRESS_DB_HOST, WORDPRESS_DB_USER, WORDPRESS_DB_NAME 均已注入
```

### 5. 访问 WordPress

浏览器访问 `http://<EXTERNAL-IP>` 完成 WordPress 安装向导，数据库信息已通过 Secret 注入。

## 验证

### 控制面（tccli）

```bash
tccli tke DescribeClusterStatus --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]'
# expected: ClusterState "Running"
```

**预期输出**：

```json
{
    "TotalCount": 1,
    "ClusterStatusSet": [
        {
            "ClusterId": "cls-xxxxxxxx",
            "ClusterState": "Running",
            "ClusterInstanceState": "Available"
        }
    ],
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

### 数据面

```bash
kubectl get deployment wordpress
# expected: READY 1/1

kubectl get pvc wordpress-pvc
# expected: STATUS Bound

kubectl get service wordpress
# expected: EXTERNAL-IP 非 <pending>

kubectl get pods -l app=wordpress
# expected: STATUS Running, READY 1/1
```

```text
NAME  STATUS  AGE
...
```

## 清理

> **注意**：删除 LoadBalancer Service 前确保关联的 CLB 已不再需要，否则 CLB 将持续产生费用。CLB 需在 [CLB 控制台](https://console.cloud.tencent.com/clb) 确认已随 Service 自动释放。

> **注意**：TencentDB MySQL 实例不会随 Kubernetes 资源自动删除，需在 [CDB 控制台](https://console.cloud.tencent.com/cdb) 手动释放（按量计费实例）或等待到期（包年包月实例），否则将持续计费。

### 数据面

```bash
kubectl delete -f wordpress-svc.yaml
# expected: service "wordpress" deleted

kubectl delete -f wordpress-tencentdb.yaml
# expected: persistentvolumeclaim "wordpress-pvc" deleted, deployment.apps "wordpress" deleted

kubectl delete secret wordpress-db-secret
# expected: secret "wordpress-db-secret" deleted
```

### 控制面（tccli）

TencentDB 实例需通过控制台或 CLI 单独删除：

```bash
tccli cdb IsolateDBInstance --region ap-guangzhou --InstanceId <CDB-InstanceId>
# expected: RequestId 返回成功（实例进入隔离状态）
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Pod `Pending` | `kubectl describe pod -l app=wordpress` 查看 Events | 节点资源不足或 PVC 无法绑定 | 扩容节点或检查 StorageClass 可用性 `kubectl get sc` |
| `ImagePullBackOff` | `kubectl describe pod -l app=wordpress` 查看 Events | 镜像拉取失败（公网不可达或仓库鉴权） | 换用 `ccr.ccs.tencentyun.com/library/wordpress:latest`；或配置 imagePullSecrets |
| `CrashLoopBackOff` | `kubectl logs deployment/wordpress` 查看启动日志 | 数据库连接失败（地址/密码错误） | 检查 Secret 值和 CDB 连接信息 |
| Secret 创建失败 | `kubectl describe secret wordpress-db-secret` | Secret 已存在但类型不匹配 | `kubectl delete secret wordpress-db-secret` 后重新创建 |

### 操作成功但异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| PVC `Pending` 不绑定 | `kubectl describe pvc wordpress-pvc` 查看 Events；`kubectl get storageclass` 确认可用 SC | 集群无默认 StorageClass 或存储后端不可用 | 设置默认 StorageClass：`kubectl patch storageclass <sc-name> -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'` |
| WordPress 无法连接 CDB | `kubectl exec deployment/wordpress -- nc -zv <CDB_HOST> 3306` 测试连通性；`kubectl get secret wordpress-db-secret -o jsonpath='{.data.password}' | base64 -d` 验证密码 | CDB 与集群不在同一 VPC，或 CDB 安全组未放通集群 Pod 网段 | 确认 CDB VPC 与集群一致：`tccli cdb DescribeDBInstances --InstanceIds '["<CDB-InstanceId>"]'`；在 CDB 安全组添加集群 Pod CIDR 的 TCP:3306 入站规则 |
| Secret 值未生效 | `kubectl get deployment wordpress -o yaml | grep -A5 secretKeyRef` 确认引用；Pod 重启后生效 | Deployment 创建时 Secret 尚不存在，或 Deployment 未触发滚动更新 | `kubectl rollout restart deployment/wordpress` 触发 Pod 重建以重新读取 Secret |
| PVC 有数据仍被删除 | `kubectl get pvc wordpress-pvc -o yaml` 查看回收策略 | StorageClass 的 `reclaimPolicy: Delete` 导致 PVC 删除时 PV 也删除 | 如需保留数据，将 PV 回收策略改为 Retain：`kubectl patch pv <pv-name> -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'`，再删除 PVC |

## 下一步

- [创建简单的 Nginx 服务](../../../快速入门/入门示例/创建简单的 Nginx 服务/tccli%20操作.md) -- page_id `7851`
- [Secret 管理](../../../应用配置/服务和配置管理/Secret/tccli%20操作.md) -- page_id `31718`

## 控制台替代

[容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) -> 集群 -> 工作负载 -> Deployment -> 新建，WordPress 模板。
