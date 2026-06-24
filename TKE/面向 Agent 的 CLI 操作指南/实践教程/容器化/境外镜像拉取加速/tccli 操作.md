# 境外镜像拉取加速（tccli）

> 对照官方：[境外镜像拉取加速](https://cloud.tencent.com/document/product/457/51237) · page_id `51237`

## 概述

国内拉取 DockerHub、`quay.io` 等境外镜像平台上的容器镜像时，可能因网络问题导致拉取速度慢或失败。TCR 企业版提供主流境外镜像托管平台的加速服务。本页介绍 TKE 集群（Docker 和 Containerd 运行时）如何配置镜像拉取加速。

## 前置条件

- 已开通 [容器镜像服务 TCR](https://console.cloud.tencent.com/tcr) 企业版。
- 集群节点运行时为 Docker 或 Containerd。
- 已 [连接集群](../../../集群配置/集群管理/连接集群/tccli%20操作.md)。

**限制条件**：加速服务仅支持 VPC 内网访问，公网访问暂不支持实际加速功能。

```bash
tccli tke DescribeClusters --region ap-guangzhou --output json \
  --filter "Clusters[?ClusterId=='<ClusterId>'] | [0].{ClusterStatus:ClusterStatus,ClusterId:ClusterId}"
```

```
{
  "ClusterStatus": "Running",
  "ClusterId": "cls-example"
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| DockerHub 镜像加速 | 已默认配置，无需操作 | 是 |
| `quay.io` 加速（Docker 节点） | `docker pull quay.tencentcloudcr.com/<path>` | 是 |
| `quay.io` 加速（Containerd 节点） | 修改 `/etc/containerd/config.toml` + `systemctl restart containerd` | 是 |
| 批量修改存量节点 | Ansible (`ansible-playbook`) | 是 |
| 增量节点自动配置 | 节点启动脚本 | 不适用 |

## 操作步骤

### 集群运行时为 Docker

Docker 不支持 `docker.io` 以外的加速配置，因此需修改镜像地址域名。将 `quay.io` 替换为 `quay.tencentcloudcr.com`：

```bash
docker pull quay.tencentcloudcr.com/k8scsi/csi-resizer:v0.5.0
```

```
v0.5.0: Pulling from k8scsi/csi-resizer
...
Status: Downloaded newer image for quay.tencentcloudcr.com/k8scsi/csi-resizer:v0.5.0
```

**注意**：`docker.io` 公开镜像已默认配置加速，无需额外操作。

### 集群运行时为 Containerd

Containerd 支持任意镜像仓库的加速地址配置，无需修改镜像地址即可自动加速。

#### 增量节点（启动脚本方式）

在创建节点池时通过 `PreStartUserScript` 注入：

```bash
sed -i '/\[plugins\.cri\.registry\.mirrors\]/ a\        [plugins.cri.registry.mirrors."quay.io"]\n          endpoint = ["https://quay.tencentcloudcr.com"]' /etc/containerd/config.toml
```

#### 存量节点（手动修改）

编辑 `/etc/containerd/config.toml`：

```toml
[plugins.cri.registry]
  [plugins.cri.registry.mirrors]
    [plugins.cri.registry.mirrors."quay.io"]
      endpoint = ["https://quay.tencentcloudcr.com"]
    [plugins.cri.registry.mirrors."docker.io"]
      endpoint = ["https://mirror.ccs.tencentyun.com"]
```

重启 Containerd：

```bash
systemctl restart containerd
```

验证加速生效：

```bash
crictl pull quay.io/k8scsi/csi-resizer:v0.5.0
```

```
Image is up to date for quay.io/k8scsi/csi-resizer@sha256:...
```

## 验证

### Data plane

```bash
crictl images | grep quay.io
```

```
quay.io/k8scsi/csi-resizer    v0.5.0     sha256:...
```

或 Docker 节点：

```bash
docker images | grep quay.tencentcloudcr.com
```

```
quay.tencentcloudcr.com/k8scsi/csi-resizer   v0.5.0    sha256:...
```

## 清理

> 本文操作为只读/非持久化操作，无需清理。

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl` 连接超时/拒绝 | `tccli tke DescribeClusters` 返回集群 Running 但 kubectl 不可达 | CAM 策略 240463971 拒绝公网访问 APIServer 端点 | 通过 VPN/IOA 接入内网后执行 kubectl 命令 |
| 修改 config.toml 后不生效 | 节点执行 `crictl pull quay.io/...` 验证是否仍走原地址 | containerd 服务未重启 | 执行 `systemctl restart containerd` 重启服务 |
| Containerd 重启后 Pod 受影响 | `kubectl get pods -o wide --field-selector spec.nodeName=<node>` 查看节点 Pod | 重启导致运行中容器短暂中断 | 在节点维护窗口操作，或逐节点滚动更新 |
| 公网接口返回慢 | 节点执行 `curl -I https://quay.tencentcloudcr.com/v2/` 测试延迟 | 加速服务仅支持 VPC 内网，节点走公网访问 | 确保节点可访问 TCR 内网地址（同 VPC） |
| DockerHub 加速不生效 | `docker pull docker.io/library/nginx` 测试是否走加速地址 | 集群 VPC 内未开通 TCR 企业版 | 确认集群 VPC 已开通 TCR 企业版；默认加速仅覆盖 DockerHub 公开镜像 |

## 下一步

- [镜像分层最佳实践](../镜像分层最佳实践/tccli 操作.md)（page_id `68383`）
- [TCR 企业版快速入门](https://cloud.tencent.com/document/product/1141/39287)

## 控制台替代

控制台 **TCR 控制台 → 镜像加速**页面开通加速服务；对于存量节点需 SSH 登录修改 containerd 配置。
