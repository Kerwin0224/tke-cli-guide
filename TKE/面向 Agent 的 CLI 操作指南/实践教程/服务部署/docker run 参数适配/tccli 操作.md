# docker run 参数适配（tccli）

> 对照官方：[docker run 参数适配](https://cloud.tencent.com/document/product/457/9883) · page_id `9883`

## 概述

将 `docker run` 命令转换为 Kubernetes YAML 的对照手册，涵盖常用参数映射：端口暴露、环境变量、卷挂载、资源限制、重启策略等。

## 前置条件

- [环境准备](../../../环境准备.md)
- 集群已连接，可用 `kubectl cluster-info`

## 控制台与 CLI 参数映射

| Docker | Kubernetes YAML | 幂等 |
|--------|---------------|------|
| `-p 8080:80` | `containerPort: 80` + Service | 是 |
| `-e KEY=val` | `env: [{name: KEY, value: val}]` | 是 |
| `-v /host:/container` | `hostPath` volume | 是 |
| `--memory 512m` | `resources.limits.memory: 512Mi` | 是 |
| `--restart always` | `restartPolicy: Always` | 是 |
| `--name app` | `metadata.name: app` | 是 |

## 操作步骤

### 常见映射示例

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: nginx:alpine
        ports:
        - containerPort: 80
        env:
        - name: ENV
          value: production
        resources:
          requests:
            memory: 128Mi
            cpu: 100m
          limits:
            memory: 256Mi
            cpu: 200m
        volumeMounts:
        - name: data
          mountPath: /data
      volumes:
      - name: data
        hostPath:
          path: /mnt/data
      restartPolicy: Always
```

```bash
kubectl apply -f myapp.yaml
```

```text
# command executed successfully
```

```output
deployment.apps/myapp created
```

## 验证

```bash
kubectl get deploy myapp
kubectl describe pod -l app=myapp
```

```text
NAME  STATUS  AGE
...
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl` 连接超时/拒绝 | `tccli tke DescribeClusters` 返回集群 Running 但 kubectl 不可达 | CAM 策略 240463971 拒绝公网访问 APIServer 端点 | 通过 VPN/IOA 接入内网后执行 kubectl 命令 |
| hostPath 卷不可用 | `kubectl describe pod <pod>` 查看 Events | 节点路径不存在 | 确认路径存在或改用 PVC |
| 特权容器被拒绝 | `kubectl get pod <pod>` 查看 STATUS | PodSecurityPolicy/PSA 限制 | 移除非必要的 `securityContext.privileged: true` |
| hostNetwork 端口冲突 | `kubectl describe pod <pod>` 查看 Events | Node 端口已被占用 | 换用 ClusterIP Service |

## 清理

```bash
kubectl delete deploy myapp
```

## 下一步

- [应用高可用部署](../应用高可用部署/tccli%20操作.md)
- [设置工作负载的资源限制](../../../应用配置/工作负载管理/设置工作负载的资源限制/tccli%20操作.md)

## 控制台替代

控制台：工作负载 → Deployment → 新建 → 填写容器参数。
