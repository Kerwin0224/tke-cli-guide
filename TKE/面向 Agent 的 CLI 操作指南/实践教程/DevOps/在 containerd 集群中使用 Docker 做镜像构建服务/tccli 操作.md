# 在 containerd 集群中使用 Docker 做镜像构建服务（tccli）

> 对照官方：[在 containerd 集群中使用 Docker 做镜像构建服务](https://cloud.tencent.com/document/product/457/50868) · page_id `50868`

## 概述

自 TKE 1.21 起，新建集群默认使用 containerd 作为容器运行时，不再内置 Docker 守护进程。然而 CI/CD 流水线中常需要 `docker build`、`docker push` 等操作。本文介绍在 containerd 集群中通过 Docker-in-Docker（DinD）模式运行 Docker 构建服务的两种方式：Sidecar 方式和 DaemonSet 方式。

**注意**：本文中 `kubectl` 命令默认不可直接由 Agent 执行，需用户提供已配置 kubeconfig 的 `kubectl` 通道或通过 TKE 控制台 → 集群 → 远程Shell 手动执行。Agent 仅提供 YAML 内容与命令参考。

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 关键参数 | 必填 | 默认值 | 校验规则 | 幂等性 |
|-----------|-----------|---------|------|--------|---------|--------|
| 查询集群列表 | `tccli tke DescribeClusters` | `--ClusterIds`, `--Region` | 否（不填则查全量） | 无 | `ClusterIds` 为 JSON 数组字符串 | 是（只读） |
| 查询集群运行时信息 | `tccli tke DescribeClusters` 结果中 `.Clusters[].ContainerRuntime` 字段 | `--Region`, `--ClusterIds` | 是（需指定集群） | 无 | 集群 ID 格式: `cls-xxxxxxxx` | 是（只读） |
| 查询集群节点列表 | `tccli tke DescribeClusterInstances` | `--ClusterId`, `--Region` | 是 | 无 | 集群 ID 格式: `cls-xxxxxxxx` | 是（只读） |
| 查询集群 Kubeconfig | `tccli tke DescribeClusterKubeconfig` | `--ClusterId`, `--Region` | 是 | 无 | 集群 ID 格式: `cls-xxxxxxxx` | 是（只读） |
| （无直接对应）Pod/DaemonSet 部署 | `kubectl apply -f <yaml>`（数据面） | 无 | — | — | — | — |
| （无直接对应）查询 TCR 命名空间 | `tccli tcr DescribeNamespaces` | `--RegistryId`, `--Region` | 是 | 无 | 实例 ID 格式: `tcr-xxxxxxxx` | 是（只读） |

**说明**：本场景核心操作（部署 Pod/DaemonSet、构建镜像、推送镜像）属于数据面操作，控制面 `tccli` 仅用于查询集群信息和验证。数据面操作通过 `kubectl` 和 `docker` CLI 完成。

## 前置条件

- 已完成[环境准备](../../环境准备.md)
- tccli 已配置凭据和地域

## 操作步骤

### #### 选择依据

| 场景 | 推荐方案 | 理由 |
|------|---------|------|
| 偶发性构建、单次任务 | DinD Sidecar（Pod 级别） | 资源按需分配，无需常驻进程 |
| 频繁构建、多流水线并发 | DaemonSet（节点级别） | 复用 Docker 守护进程，减少启动开销 |
| 多架构镜像构建（amd64/arm64） | 任一方案 + Docker Buildx | Buildx 需额外初始化，但兼容两种部署模式 |
| 安全敏感环境 | DinD Sidecar | 构建容器与 Docker 守护进程生命周期绑定，更易管控 |

### #### 最小配置

#### 第 1 步：确认集群运行时为 containerd

**控制面 — 查询集群信息**：

```bash
tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
```

输出关键字段示例：

```json
{
  "Clusters": [
    {
      "ClusterId": "CLUSTER_ID",
      "ClusterName": "my-cluster",
      "ContainerRuntime": "containerd",
      "ClusterVersion": "1.24",
      "ClusterStatus": "Running"
    }
  ]
}
```

**数据面 — 查看节点运行时**（需 kubectl 通道）：

```bash
kubectl get nodes -o wide
```

输出示例：

```
NAME           STATUS   ROLES    AGE   VERSION          INTERNAL-IP    CONTAINER-RUNTIME
10.0.0.1       Ready    <none>   30d   v1.24.4-tke.1   10.0.0.1       containerd://1.6.9
10.0.0.2       Ready    <none>   30d   v1.24.4-tke.1   10.0.0.2       containerd://1.6.9
```

确认 `CONTAINER-RUNTIME` 列为 `containerd://` 开头。

#### 第 2 步：DinD Sidecar 方式构建镜像

部署一个包含 Docker DinD 和构建容器的 Pod，两者通过 `emptyDir` 共享 Docker socket。

**文件 `dind-pod.yaml`**：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: DEPLOYMENT_NAME
  namespace: NAMESPACE
spec:
  containers:
    # Docker-in-Docker 容器
    - name: dind-daemon
      image: docker:24.0-dind
      securityContext:
        privileged: true
      env:
        - name: DOCKER_TLS_CERTDIR
          value: ""
      volumeMounts:
        - name: docker-socket
          mountPath: /var/run
        - name: docker-storage
          mountPath: /var/lib/docker
    # 构建容器
    - name: docker-builder
      image: docker:24.0-cli
      command: ["sleep", "3600"]
      env:
        - name: DOCKER_HOST
          value: unix:///var/run/docker.sock
      volumeMounts:
        - name: docker-socket
          mountPath: /var/run
        - name: workspace
          mountPath: /workspace
  volumes:
    - name: docker-socket
      emptyDir: {}
    - name: docker-storage
      emptyDir: {}
    - name: workspace
      emptyDir: {}
  restartPolicy: Never
```

**部署 Pod**：

```bash
kubectl apply -f dind-pod.yaml
```

输出示例：

```
pod/DEPLOYMENT_NAME created
```

**进入构建容器执行构建和推送**：

```bash
kubectl exec -it DEPLOYMENT_NAME -n NAMESPACE -c docker-builder -- sh
```

在容器内执行构建：

```bash
# 在容器内执行
cd /workspace
cat > Dockerfile << 'EOF'
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/index.html
EOF
echo "<h1>Hello from DinD</h1>" > index.html

# 构建镜像
docker build -t ccr.ccs.tencentyun.com/NAMESPACE/my-app:v1.0 .

# 登录并推送
docker login ccr.ccs.tencentyun.com --username=USERNAME --password=PASSWORD
docker push ccr.ccs.tencentyun.com/NAMESPACE/my-app:v1.0
```

输出示例（构建）：

```
[+] Building 15.2s (7/7) FINISHED
 => [internal] load build definition from Dockerfile
 => => transferring dockerfile: 102B
 => [internal] load .dockerignore
 => => transferring context: 2B
 => [1/2] FROM docker.io/library/nginx:alpine
 => [2/2] COPY index.html /usr/share/nginx/html/index.html
 => exporting to image
 => => naming to ccr.ccs.tencentyun.com/NAMESPACE/my-app:v1.0
```

输出示例（推送）：

```
The push refers to repository [ccr.ccs.tencentyun.com/NAMESPACE/my-app]
v1.0: digest: sha256:xxxx size: 528
```

### #### 增强配置

#### 第 3 步：DaemonSet 方式在每个节点部署 Docker

如需常驻 Docker 构建服务，可以通过 DaemonSet 在每个节点运行 Docker 守护进程，业务 Pod 通过 `hostPath` 挂载 socket。

**文件 `docker-daemonset.yaml`**：

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: docker-dind
  namespace: NAMESPACE
  labels:
    app: docker-dind
spec:
  selector:
    matchLabels:
      app: docker-dind
  template:
    metadata:
      labels:
        app: docker-dind
    spec:
      containers:
        - name: docker-dind
          image: docker:24.0-dind
          securityContext:
            privileged: true
          env:
            - name: DOCKER_TLS_CERTDIR
              value: ""
          volumeMounts:
            - name: docker-socket
              mountPath: /var/run
            - name: docker-storage
              mountPath: /var/lib/docker
          resources:
            requests:
              cpu: 500m
              memory: 512Mi
            limits:
              cpu: 2000m
              memory: 2Gi
      volumes:
        - name: docker-socket
          hostPath:
            path: /var/run/dind
            type: DirectoryOrCreate
        - name: docker-storage
          hostPath:
            path: /var/lib/docker-dind
            type: DirectoryOrCreate
      tolerations:
        - operator: Exists
```

**部署 DaemonSet**：

```bash
kubectl apply -f docker-daemonset.yaml
```

输出示例：

```
daemonset.apps/docker-dind created
```

验证 DaemonSet 运行状态：

```bash
kubectl get daemonset docker-dind -n NAMESPACE
```

输出示例：

```
NAME          DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
docker-dind   3         3         3       3            3           <none>          30s
```

**业务 Pod 挂载 DaemonSet 共享的 Docker socket（`dind-build-pod.yaml`）**：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: DEPLOYMENT_NAME
  namespace: NAMESPACE
spec:
  containers:
    - name: builder
      image: docker:24.0-cli
      command: ["sleep", "3600"]
      env:
        - name: DOCKER_HOST
          value: unix:///var/run/dind/docker.sock
      volumeMounts:
        - name: docker-socket
          mountPath: /var/run/dind
        - name: workspace
          mountPath: /workspace
  volumes:
    - name: docker-socket
      hostPath:
        path: /var/run/dind
    - name: workspace
      emptyDir: {}
  restartPolicy: Never
```

```bash
kubectl apply -f dind-build-pod.yaml
```

```text
# command executed successfully
```

进入构建容器的方式与第 2 步一致。

#### 第 4 步：使用 Docker Buildx 构建多架构镜像

在 DinD 环境（Sidecar 或 DaemonSet）的构建容器内初始化 Buildx：

```bash
# 进入构建容器
kubectl exec -it DEPLOYMENT_NAME -n NAMESPACE -c docker-builder -- sh

# 创建 buildx builder（使用 docker 驱动）
docker buildx create --name multiarch-builder --use
docker buildx inspect --bootstrap
```

输出示例：

```
Name:          multiarch-builder
Driver:        docker-container
Nodes:
Name:          multiarch-builder0
Endpoint:      unix:///var/run/docker.sock
Status:        running
Platforms:     linux/amd64, linux/arm64, linux/arm/v7, linux/arm/v6
```

**构建多架构镜像并推送**：

```bash
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t ccr.ccs.tencentyun.com/NAMESPACE/my-app:v1.0-multiarch \
  --push \
  /workspace
```

输出示例：

```
[+] Building 45.2s (12/12) FINISHED
 => [linux/amd64 1/2] FROM docker.io/library/nginx:alpine@sha256:...
 => [linux/arm64 1/2] FROM docker.io/library/nginx:alpine@sha256:...
 => exporting to image
 => => pushing layers
 => => pushing manifest for ccr.ccs.tencentyun.com/NAMESPACE/my-app:v1.0-multiarch
```

#### 第 5 步：推送镜像至 TCR（数据面）

**登录 TCR 企业版实例**：

```bash
# 在构建容器内执行
docker login ccr.ccs.tencentyun.com --username=USERNAME --password=PASSWORD
```

输出示例：

```
Login Succeeded
```

**标注并推送镜像**：

```bash
# 打标签
docker tag my-app:v1.0 ccr.ccs.tencentyun.com/NAMESPACE/my-app:v1.0

# 推送
docker push ccr.ccs.tencentyun.com/NAMESPACE/my-app:v1.0
```

输出示例：

```
The push refers to repository [ccr.ccs.tencentyun.com/NAMESPACE/my-app]
v1.0: digest: sha256:abc123... size: 1024
```

**查看 TCR 中的镜像仓库**（控制面）：

```bash
tccli tcr DescribeRepositories --RegistryId INSTANCE_ID --region <Region> --NamespaceName NAMESPACE
```

输出关键字段示例：

```json
{
  "RepositoryList": [
    {
      "Name": "NAMESPACE/my-app",
      "Namespace": "NAMESPACE",
      "Public": false,
      "BriefDescription": ""
    }
  ]
}
```

## 验证

### 控制面 — 集群状态确认

```bash
tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
```

```json
{
  "TotalCount": 0,
  "Clusters": [],
  "ClusterId": "<ClusterId>",
  "ClusterName": "<ClusterName>",
  "ClusterDescription": "<ClusterDescription>",
  "ClusterVersion": "<ClusterVersion>",
  "ClusterOs": "<ClusterOs>",
  "ClusterType": "<ClusterType>"
}
```

验证 `.Clusters[].ContainerRuntime` 为 `containerd`。

### 数据面 — 构建环境验证

进入构建容器，确认 Docker daemon 正常运行：

```bash
kubectl exec -it DEPLOYMENT_NAME -n NAMESPACE -c docker-builder -- docker version
```

输出示例：

```
Client: Docker Engine - Community
 Version:           24.0.7

Server: Docker Engine - Community
 Engine:
  Version:          24.0.7
  API version:      1.43 (minimum version 1.12)
```

查看已构建的镜像列表：

```bash
kubectl exec -it DEPLOYMENT_NAME -n NAMESPACE -c docker-builder -- docker images
```

输出示例：

```
REPOSITORY                                    TAG       IMAGE ID     

```json
{
  "Data": "<Data>",
  "TagCount": 0,
  "TagInfo": [],
  "TagName": "<TagName>",
  "TagId": "<TagId>",
  "ImageId": "<ImageId>",
  "Size": "<Size>",
  "CreationTime": "<CreationTime>"
}
```  CREATED          SIZE
ccr.ccs.tencentyun.com/NAMESPACE/my-app       v1.0      abc123def456   2 minutes ago    10.5MB
```

### 数据面 — TCR 镜像验证

查询 TCR 镜像 tag 列表确认推送成功：

```bash
tccli tcr DescribeImagePersonal --region <Region> --RepoName NAMESPACE/my-app --Tag v1.0
```

或在 TCR 控制台 → 镜像仓库 → 对应命名空间 → 镜像版本页面查看。亦可直接拉取验证：

```bash
docker pull ccr.ccs.tencentyun.com/NAMESPACE/my-app:v1.0
```

## 清理

**注意**：DinD 容器使用特权模式（`privileged: true`），存在安全风险。构建任务完成后请及时清理资源。

```bash
# 删除构建 Pod
kubectl delete pod DEPLOYMENT_NAME -n NAMESPACE

# 如使用 DaemonSet，删除 DaemonSet
kubectl delete daemonset docker-dind -n NAMESPACE

# 如使用业务构建 Pod（增强配置），删除对应 Pod
kubectl delete pod DEPLOYMENT_NAME -n NAMESPACE
```

**清理残留数据**：

- 节点上 DinD 产生的镜像数据存储在 `hostPath`（DaemonSet 模式下的 `/var/lib/docker-dind`），需节点维护时清理。
- TCR 中的测试镜像请手动删除，避免产生存储费用：

```bash
tccli tcr DeleteImage --RegistryId INSTANCE_ID --NamespaceName NAMESPACE --RepositoryName my-app --ImageVersion v1.0
```

## 排障

| 故障现象 | 可能原因 | 排查方法 | 解决方案 |
|---------|---------|---------|---------|
| `docker version` 报错 `Cannot connect to the Docker daemon` | Docker daemon 未启动或 socket 路径不正确 | `kubectl logs DEPLOYMENT_NAME -n NAMESPACE -c dind-daemon` 查看 daemon 日志；确认 `DOCKER_HOST` 环境变量指向正确的 socket | 检查 `DOCKER_HOST` 是否匹配 Sidecar 的 `/var/run/docker.sock` 或 DaemonSet 的 `/var/run/dind/docker.sock`；确认 DinD 容器状态为 Running |
| Pod 创建失败，提示 `privileged mode denied` | Pod Security Policy（PSP）或 Pod Security Admission（PSA）拦截特权容器 | `kubectl describe pod DEPLOYMENT_NAME -n NAMESPACE` 查看 Events；`kubectl get psp` 检查 PSP | 如果集群启用了严格的 PSA（`restricted` 策略），需将命名空间设置为 `baseline` 或 `privileged` 级别：`kubectl label ns NAMESPACE pod-security.kubernetes.io/enforce=privileged` |
| `docker build` 失败，报错 `COPY failed: no source files` | 构建上下文路径不对或文件缺失 | 确认 `WORKDIR` 正确，检查文件是否存在于当前目录 | 进入容器后检查工作目录：`pwd && ls -la`；确认 Dockerfile 及所需文件均在当前目录 |
| 推送 TCR 失败 `unauthorized: authentication required` | TCR 登录凭据过期或命名空间权限不足 | 重新登录：`docker logout ccr.ccs.tencentyun.com && docker login ccr.ccs.tencentyun.com` | 确认 TCR 实例内命名空间已创建且账户具有 push 权限；推荐使用长期访问凭证 |
| 推送 TCR 失败 `dial tcp: lookup ccr.ccs.tencentyun.com` | 容器内 DNS 解析失败 | `kubectl exec -it DEPLOYMENT_NAME -n NAMESPACE -c docker-builder -- nslookup ccr.ccs.tencentyun.com` | 检查 CoreDNS 是否正常；确认节点能访问公网或已配置 TCR 内网访问链路（VPC 内网解析） |
| 节点磁盘空间不足，Pod Evicted | DinD 构建产生大量镜像层和缓存 | `docker system df` 查看磁盘占用；检查 `hostPath` 卷实际大小 | 执行 `docker system prune -a -f` 清理不用的镜像、容器和构建缓存；DaemonSet 模式下为 `docker-storage` 挂载独立的磁盘卷 |
| Buildx 多架构构建报错 `running container xxxxx: ... exec format error` | 内核缺少 QEMU 模拟组件 | 在 DinD 容器中运行 `docker run --rm --privileged multiarch/qemu-user-static --reset -p yes` | 注册 QEMU 二进制翻译支持：`docker pull multiarch/qemu-user-static && docker run --rm --privileged multiarch/qemu-user-static --reset -p yes` |
| `docker buildx build --push` 报 `multiple platforms feature is currently not supported for docker driver` | 默认 docker driver 不支持多平台，需切换到 `docker-container` driver | `docker buildx ls` 查看当前 builder driver 类型 | 重新创建 builder：`docker buildx create --name multiarch-builder --driver docker-container --use && docker buildx inspect --bootstrap` |

## 控制台替代

[TKE 控制台](https://console.cloud.tencent.com/tke2)

## 下一步

- [环境准备](../../环境准备.md)
