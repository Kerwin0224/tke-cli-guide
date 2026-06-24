# 容器登录方式（tccli）

> 对照官方：[容器登录方式](https://cloud.tencent.com/document/product/457/118944) · page_id `118944`

## 概述

提供三种容器登录方式：通过 OrcaTerm 一键登录容器、通过 SSH 登录容器（需容器已安装 SSH 服务端）、通过容器所在节点登录容器（SSH 到节点后使用 `docker exec` / `crictl exec`）。OrcaTerm 为腾讯云控制台内建终端，无需额外配置即可登录。

## 前置条件

- [环境准备](../../环境准备.md)：`tccli` 已配置，`kubectl` >= v1.28
- 需要 CAM 权限：`tke:DescribeClusters`、`tke:DescribeClusterKubeconfig`、`tke:DescribeClusterInstances`

```bash
# 检查集群状态
tccli tke DescribeClusters --region <Region> --ClusterIds '["<ClusterId>"]'
# expected: ClusterStatus "Running"

# 获取 kubeconfig
tccli tke DescribeClusterKubeconfig --region <Region> --ClusterId <ClusterId> \
    | jq -r '.Kubeconfig' > kubeconfig.yaml
export KUBECONFIG=$(pwd)/kubeconfig.yaml
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

> **注意**：kubectl 数据面当前因公网端点被组织级 CAM 策略拒绝（strategyId:240463971）而不可达。外网端点被 `tke:clusterExtranetEndpoint=true` 条件拦截，内网端点需通过 IOA/VPN 或同 VPC 内 CVM 访问。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 获取集群凭证 | `tccli tke DescribeClusterKubeconfig --region <Region> --ClusterId <ClusterId>` | 是 |
| 查询集群节点 | `tccli tke DescribeClusterInstances --region <Region> --ClusterId <ClusterId>` | 是 |
| 查看 Pod 列表 | `kubectl get pods -o wide` | 是 |
| 获取 Pod 所在节点 IP | `kubectl get pod <pod> -o jsonpath='{.status.hostIP}'` | 是 |
| 获取容器 ID | `kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[].containerID}'` | 是 |
| 登录容器 | `kubectl exec -it <pod> -- /bin/bash` | 否 |
| SSH 登录节点 | `ssh root@<node-ip>` | 否 |
| Docker 登录容器 | `docker exec -it <container-id> /bin/bash` | 否 |
| containerd 登录容器 | `crictl exec -it <container-id> /bin/bash` | 否 |

## 操作步骤

### 步骤 1：通过 kubectl exec 登录容器（推荐）

最直接的容器登录方式，无需登录节点：

```bash
kubectl exec -it <pod-name> -n <namespace> -- /bin/bash
# expected: 进入容器 shell
```

或指定容器（多容器 Pod）：

```bash
kubectl exec -it <pod-name> -n <namespace> -c <container-name> -- /bin/bash
```

如果容器没有 bash，使用 sh：

```bash
kubectl exec -it <pod-name> -n <namespace> -- /bin/sh
```

### 步骤 2：获取 Pod 信息和所在节点

```bash
kubectl get pods -A -o wide
# expected: 显示所有 Pod 及所在节点
```

```text
NAME  STATUS  AGE
...
```

查看 Pod 所在节点 IP：

```bash
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.status.hostIP}'
# expected: 输出节点内网 IP
```

```text
NAME  STATUS  AGE
...
```

查看容器 ID：

```bash
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.status.containerStatuses[].containerID}'
# expected: 输出 containerd://<id> 或 docker://<id>
```

```text
NAME  STATUS  AGE
...
```

### 步骤 3：通过 SSH 登录容器

前提：容器已安装 SSH 服务端。

获取 Pod IP：

```bash
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.status.podIP}'
# expected: 输出 Pod IP
```

```text
NAME  STATUS  AGE
...
```

在集群内任意节点 SSH 登录：

```bash
ssh root@<pod-ip>
# expected: 进入容器
```

### 步骤 4：通过容器所在节点登录容器

**Step 1** — 获取节点 IP：

```bash
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.status.hostIP}'
```

```text
NAME  STATUS  AGE
...
```

**Step 2** — SSH 登录节点：

```bash
ssh root@<node-ip>
```

**Step 3** — 查看容器列表：

如果节点使用 Docker 运行时：

```bash
docker ps | grep <container-id-prefix>
# expected: 显示匹配的容器
```

如果节点使用 containerd 运行时：

```bash
crictl ps | grep <container-id-prefix>
# expected: 显示匹配的容器
```

**Step 4** — 通过容器运行时登录容器：

Docker 运行时：

```bash
docker exec -it <container-id> /bin/bash
```

containerd 运行时：

```bash
crictl exec -it <container-id> /bin/bash
```

### 步骤 5：通过 OrcaTerm 一键登录容器

控制台专用方式，无 CLI 等效：

1. 登录 [容器服务控制台](https://console.cloud.tencent.com/tke2)
2. 集群详情 → **节点管理** → 单击节点 ID → **Pod 管理**
3. 选择 Pod 实例，操作列单击**登录** → 选择 **OrcaTerm 登录**

### 步骤 6：通过 tccli 获取集群节点信息（控制面）

```bash
tccli tke DescribeClusterInstances --region <Region> --ClusterId <ClusterId> \
  --filter "InstanceSet[].{InstanceId:InstanceId,InstanceState:InstanceState,LanIP:LanIP}"
# expected: 返回节点列表，含 LanIP
```

```json
{
  "TotalCount": "<TotalCount>",
  "InstanceSet": "<InstanceSet>",
  "InstanceId": "<InstanceId>",
  "InstanceRole": "<InstanceRole>",
  "FailedReason": "<FailedReason>",
  "InstanceState": "<InstanceState>"
}
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
kubectl exec <pod-name> -n <namespace> -- hostname
# expected: 输出容器主机名

kubectl exec <pod-name> -n <namespace> -- whoami
# expected: 输出当前用户

kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.status.containerStatuses[].ready}'
# expected: true
```

```text
NAME  STATUS  AGE
...
```

## 清理

无需清理（登录操作不创建资源）。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `Unable to connect to the server` | `kubectl cluster-info` 检查连通性 | 公网端点被 CAM 策略 strategyId:240463971 以 `tke:clusterExtranetEndpoint=true` 条件拒绝 | 通过 IOA/VPN 连接内网，或使用同 VPC CVM 执行 kubectl 命令 |
| `unable to upgrade connection` | `kubectl exec` 失败 | APIServer 连通性问题或 kubeconfig 不正确 | 检查 kubeconfig 配置和网络连通性 |
| `OCI runtime exec failed: exec failed` | `kubectl exec` 失败 | 容器未运行或容器中没有 `/bin/bash` | 改用 `/bin/sh` |

### 登录后操作异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `ssh: connect to host <pod-ip> port 22: Connection refused` | 检查容器是否监听 22 端口 | 容器未安装 SSH 服务端 | 安装 SSH 服务端或改用 kubectl exec |
| `docker: command not found` | 检查节点运行时 | 节点使用 containerd 运行时 | 改用 `crictl` 命令 |
| 无法通过节点登录容器 | 检查节点 SSH 权限 | 无节点 SSH 权限或安全组未放通 22 端口 | 配置节点 SSH 密钥，检查安全组规则 |

## 下一步

- [连接集群](../../集群配置/集群管理/连接集群/tccli%20操作.md)
- [节点概述](../../集群配置/节点管理/节点概述/tccli%20操作.md)

## 控制台替代

[容器服务控制台](https://console.cloud.tencent.com/tke2) → 集群 → 节点管理 → 节点 ID → Pod 管理 → 操作列登录，选择 OrcaTerm 登录。
