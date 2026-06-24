# 在 containerd 集群中使用 Docker 做镜像构建服务（tccli）

> 对照官方：[在 containerd 集群中使用 Docker 做镜像构建服务](https://cloud.tencent.com/document/product/457/50868) · page_id `50868`

## 概述

在 containerd 运行时的 TKE 集群中，部分 CI/CD 流水线仍需使用 Docker 构建镜像。本文介绍两种方式实现 Docker 镜像构建：**DinD (Docker-in-Docker) Sidecar** 和 **DaemonSet 部署 Docker**。两种方式均为纯数据面操作，不涉及控制面 API 调用。

## 前置条件

- 已 [连接集群](../../../集群配置/集群管理/连接集群/tccli%20操作.md)，kubectl 可正常访问 APIServer。
- 集群节点运行时为 containerd。
- 已完成 [环境准备](../../../环境准备.md)。

```bash
tccli tke DescribeClusters --region ap-guangzhou --output json \
  --filter "Clusters[?ClusterId=='<ClusterId>'] | [0].{ClusterStatus:ClusterStatus,ClusterVersion:ClusterVersion}"
```

```
{
  "ClusterStatus": "Running",
  "ClusterVersion": "1.28.3"
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 创建 DinD Pod | `kubectl apply -f dind-pod.yaml` | 否（名称冲突） |
| 创建 Docker DaemonSet | `kubectl apply -f docker-ds.yaml` | 是 |
| 查看 Pod 状态 | `kubectl get pods` | 是 |

## 操作步骤

### 方式1：使用 DinD 作为 Pod 的 Sidecar

DinD 容器运行 dockerd，通过 emptyDir 共享 UNIX Socket 给业务容器使用。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: clean-ci
spec:
  containers:
  - name: dind
    image: 'docker:stable-dind'
    command:
    - dockerd
    - --host=unix:///var/run/docker.sock
    - --host=tcp://0.0.0.0:8000
    securityContext:
      privileged: true
    volumeMounts:
    - mountPath: /var/run
      name: cache-dir
  - name: clean-ci
    image: 'docker:stable'
    command: ["/bin/sh"]
    args: ["-c", "docker info >/dev/null 2>&1; while [ $? -ne 0 ] ; do sleep 3; docker info >/dev/null 2>&1; done; docker pull library/busybox:latest; docker save -o busybox-latest.tar library/busybox:latest; docker rmi library/busybox:latest; while true; do sleep 86400; done"]
    volumeMounts:
    - mountPath: /var/run
      name: cache-dir
  volumes:
  - name: cache-dir
    emptyDir: {}
```

```bash
kubectl apply -f dind-pod.yaml
kubectl get pods clean-ci -w
```

```
NAME       READY   STATUS    RESTARTS   AGE
clean-ci   2/2     Running   0          45s
```

验证 Docker 可用：

```bash
kubectl exec clean-ci -c clean-ci -- docker version
```

```
Client:
 Version:           24.0.7
 API version:       1.43
 ...
Server:
 Engine:
  Version:          24.0.7
  ...
```

### 方式2：使用 DaemonSet 在每个 containerd 节点上部署 Docker

在每个节点上以 DaemonSet 形式运行 dockerd，共享 hostPath 给业务 Pod 使用。

**Step 1：部署 DaemonSet**

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: docker-ci
spec:
  selector:
    matchLabels:
      app: docker-ci
  template:
    metadata:
      labels:
        app: docker-ci
    spec:
      containers:
      - name: docker-ci
        image: 'docker:stable-dind'
        command:
        - dockerd
        - --host=unix:///var/run/docker.sock
        - --host=tcp://0.0.0.0:8000
        securityContext:
          privileged: true
        volumeMounts:
        - mountPath: /var/run
          name: host
      volumes:
      - name: host
        hostPath:
          path: /var/run
```

```bash
kubectl apply -f docker-ds.yaml
kubectl get ds docker-ci
```

```
NAME         DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
docker-ci    3         3         3       3            3           <none>          28s
```

**Step 2：业务 Pod 共享 hostPath**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: clean-ci
spec:
  containers:
  - name: clean-ci
    image: 'docker:stable'
    command: ["/bin/sh"]
    args: ["-c", "docker info >/dev/null 2>&1; while [ $? -ne 0 ] ; do sleep 3; docker info >/dev/null 2>&1; done; docker pull library/busybox:latest; docker save -o busybox-latest.tar library/busybox:latest; docker rmi library/busybox:latest; while true; do sleep 86400; done"]
    volumeMounts:
    - mountPath: /var/run
      name: host
  volumes:
  - name: host
    hostPath:
      path: /var/run
```

```bash
kubectl apply -f clean-ci-pod.yaml
kubectl get pods clean-ci
```

```
NAME       READY   STATUS    RESTARTS   AGE
clean-ci   1/1     Running   0          30s
```

## 验证

### Data plane (kubectl)

```bash
kubectl exec clean-ci -- docker pull busybox:latest
kubectl exec clean-ci -- docker images | grep busybox
```

```
busybox    latest    a416a98b71e2   2 weeks ago   4.27MB
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl` 命令报错/无响应（本环境实测） | `tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]' --filter "Clusters[0].ClusterStatus"` | CAM 拒绝公网端点（strategyId:240463971），数据面不可达 | 接入 VPN/IOA 内网后执行 kubectl；控制面 tccli 不受影响 |
| `docker info` 返回 `Cannot connect to the Docker daemon` | `kubectl logs clean-ci -c dind` | dockerd 未启动完成 | 等待 dockerd 就绪；确认 `securityContext.privileged: true` |
| Pod `CrashLoopBackOff` | `kubectl describe pod clean-ci` | 节点禁止特权容器 | 确认 containerd 节点允许特权模式 |
| DaemonSet Pod 未在某些节点运行 | `kubectl get nodes -o wide` + 检查 Taint | 节点 Taint 或资源不足 | 添加 Toleration；确认节点资源充足 |
| 宿主机磁盘占用持续增加 | `kubectl exec clean-ci -- docker system df` | 镜像构建缓存未清理 | 定期 `docker system prune`；DinD 用 emptyDir 自动回收 |
| 多租户安全风险 | `kubectl get pods --all-namespaces` | 方式2 特权 Pod 可 `docker exec` 其他容器 | 生产环境改用 DinD 或 Kaniko |

## 清理

### Data plane (kubectl)

```bash
kubectl delete pod clean-ci
kubectl delete daemonset docker-ci
```

## 下一步

- [在 TKE 上部署 Jenkins](../在 TKE 上部署 Jenkins/tccli 操作.md)（page_id `52330`）
- [使用 Terraform 管理 TKE 集群和节点池](../../Terraform/使用 Terraform 管理 TKE 集群和节点池/tccli 操作.md)（page_id `83830`）

## 控制台替代

控制台 **集群 → 工作负载 → 新建**，分别以 Pod 或 DaemonSet 方式创建上述 YAML。Pod 需勾选"特权级容器"。
