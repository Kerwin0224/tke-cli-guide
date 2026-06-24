# 使用 kubecm 管理多集群 kubeconfig（tccli）

> 对照官方：[使用 kubecm 管理多集群 kubeconfig](https://cloud.tencent.com/document/product/457/50659) · page_id `50659`
## 概述

通过 kubecm 工具管理多个 TKE 集群的 kubeconfig，实现集群切换、合并、重命名和备份。

> **kubectl 数据面不可达**：CAM拒绝公网端点(strategyId:240463971)，内网需VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

## 前置条件

- [环境准备](../../../环境准备.md)：`tccli` 已配置，`kubectl` >= v1.28
- 已获取集群 kubeconfig：

```bash
tccli tke DescribeClusterKubeconfig --region <Region> --ClusterId <ClusterId> \
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

- 需要 CAM 权限：`tke:DescribeClusters`、`tke:DescribeClusterKubeconfig`

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 获取集群凭证 | `tccli tke DescribeClusterKubeconfig` | 是 |
| 查看集群状态 | `tccli tke DescribeClusterStatus --region <Region> --ClusterIds '["<ClusterId>"]'` | 是 |
| 相关 kubectl 操作 | 见 Procedure | -- |

## 操作步骤

### 1. 安装 kubecm

```bash
# macOS
brew install kubecm

# Linux
wget https://github.com/sunny0826/kubecm/releases/latest/download/kubecm_Linux_x86_64.tar.gz
tar xzf kubecm_Linux_x86_64.tar.gz
sudo mv kubecm /usr/local/bin/
```

### 2. 控制面：获取各集群 kubeconfig

```bash
# 集群 A
tccli tke DescribeClusterKubeconfig --region <Region> --ClusterId <ClusterId_A> \
    --filter "Kubeconfig" > kubeconfig-a.yaml

# 集群 B
tccli tke DescribeClusterKubeconfig --region <Region> --ClusterId <ClusterId_B> \
    --filter "Kubeconfig" > kubeconfig-b.yaml
```

```json
{
  "Kubeconfig": "<Kubeconfig>",
  "RequestId": "<RequestId>"
}
```

### 3. 合并 kubeconfig

```bash
# 合并多个 kubeconfig 到默认配置
kubecm merge -f kubeconfig-a.yaml -f kubeconfig-b.yaml
# expected: 合并到 ~/.kube/config
```

### 4. kubecm 常用命令

| 命令 | 说明 |
|------|------|
| `kubecm list` | 列出所有 context |
| `kubecm switch` | 交互式切换 context |
| `kubecm add -f <file>` | 添加 kubeconfig 文件 |
| `kubecm delete <name>` | 删除指定 context |
| `kubecm rename -n <old> <new>` | 重命名 context |
| `kubecm alias <name>` | 设置 context 别名 |
| `kubecm merge -f <f1> -f <f2>` | 合并多个 kubeconfig |
| `kubecm backup` | 备份当前 kubeconfig |

### 5. 切换集群

```bash
# 列出所有集群
kubecm list
# expected: 显示所有 context

# 切换到指定集群
kubecm switch
# 交互式选择目标集群
```

### 6. 命名规范建议

```bash
# 重命名 context 为易识别的名称
kubecm rename -n <original> prod-gz-cluster
kubecm rename -n <original> staging-sh-cluster
```

## 验证

```bash
kubecm list
# expected: 所有集群 context 列表

kubectl cluster-info
# expected: 当前集群 APIServer 信息
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubecm merge` 失败 | `tccli tke DescribeClusters --region ap-guangzhou --filter "Clusters[].{Id:ClusterId,Status:ClusterStatus}"` | kubeconfig 格式错误或集群未 Running | 确认 kubeconfig 为有效 YAML；集群 Running 后重新 `DescribeClusterKubeconfig` 拉取 |
| context 切换后 `kubectl` 无法连接 | `tccli tke DescribeClusterEndpoints --ClusterId cls-xxxxxxxx --region ap-guangzhou` | 集群端点不可达或安全组未放行 | 确认端点状态；安全组放行访问机 IP |
| `kubectl get nodes` 报 `Unable to connect to the server` | `tccli tke DescribeClusters --region ap-guangzhou --ClusterId cls-xxxxxxxx --filter "Clusters[].ClusterStatus"` | CAM 拒绝公网端点（strategyId 240463971），数据面不可达 | 接入 VPN/IOA 后在内网执行 kubectl |

## 清理

```bash
kubecm delete <unused-context>
```

## 下一步

- [在 TKE 中自定义 RBAC 授权](../../安全/在%20TKE%20中自定义%20RBAC%20授权/tccli%20操作.md) -- page_id `51683`
- [集群 API Server 网络无法访问排障处理](../../../故障处理/集群%20API%20Server%20网络无法访问排障处理/tccli%20操作.md) -- page_id `80912`

## 控制台替代

[容器服务控制台](https://console.cloud.tencent.com/tke2/cluster)
