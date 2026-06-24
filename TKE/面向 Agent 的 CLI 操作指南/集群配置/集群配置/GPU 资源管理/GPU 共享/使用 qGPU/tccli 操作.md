# 使用 qGPU

> 对照官方：[使用 qGPU](https://cloud.tencent.com/document/product/457/65734) · page_id `65734`

## 概述

在 TKE 集群中安装 qGPU 调度组件、开启节点池 qGPU 共享，以及为应用分配共享 GPU 资源。qGPU 需 GPU 原生节点才能验证功能。

## 前置条件

- TKE 集群 >= v1.14.x
- 原生节点，支持的 GPU 架构：V100、T4、A100、A10、L40、L40s、Pnv5b、Pnv6
- 支持的驱动版本：450/470/515/525/535/550/570 系列
- kubectl 可达集群以 apply YAML

> **kubectl 可达性说明**：当前共享集群 kubectl 不可达，且共享集群无 GPU 节点。以下 YAML 和命令格式完整，但无法真跑验证。

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 安装 qGPU 组件 | `tccli tke InstallAddon --AddonName qgpu` | 否 |
| 查看 qGPU 组件状态 | `tccli tke DescribeAddon` | 是 |
| 创建 qGPU 节点池 | `tccli tke CreateClusterNodePool` + Label | 否 |
| 创建 qGPU 工作负载 | `kubectl apply -f <yaml>` | 是 |

## 操作步骤

### 安装 qGPU 调度组件

```bash
tccli tke InstallAddon --region <Region> \
    --cli-input-json '{"ClusterId":"<ClusterId>","AddonName":"qgpu","AddonVersion":"<AddonVersion>"}'
# expected: exit 0
```

安装时可通过 `RawValues` 设置调度策略：
- `Spread`：多个 Pod 分散在不同节点、不同显卡（高可用）
- `Binpack`：多个 Pod 优先使用同一节点（提高利用率）

### 部署在集群内的 Kubernetes 对象

| 对象名称 | 类型 | 请求资源 | Namespace |
|---------|------|---------|-----------|
| qgpu-manager | DaemonSet | 每 GPU 节点 Memory:300M, CPU:0.2 | kube-system |
| qgpu-scheduler | Deployment | 单副本 Memory:800M, CPU:1 | kube-system |

### 开启节点池 qGPU 共享

创建原生节点池时，设置 Label：

```bash
# Label 键
tke.cloud.tencent.com/qgpu-schedule-policy

# Label 值（选择隔离策略）
# best-effort    → 争抢模式（默认）
# fixed-share    → 固定配额
# burst-share    → 保证配额加弹性
```

### 为 Pod 分配 qGPU 资源

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: qgpu-demo
spec:
  containers:
  - name: test-container
    image: nvidia/cuda:11.0-base
    resources:
      limits:
        tke.cloud.tencent.com/qgpu-memory: "5"    # 显存（GB），整数，最小 1G
        tke.cloud.tencent.com/qgpu-core: "30"      # 算力（%），5-100，精度 5
      requests:
        tke.cloud.tencent.com/qgpu-memory: "5"
        tke.cloud.tencent.com/qgpu-core: "30"
```

```bash
kubectl apply -f qgpu-pod.yaml
# expected: pod/qgpu-demo created
```

**资源声明规则：**
- requests 和 limits 中 qGPU 值必须一致（按 K8s 规则可省略 requests）
- `qgpu-memory`：显存（GB），整数，最小 1G
- `qgpu-core`：算力（%），5~100，精度 5
- 整卡申请：`qgpu-core: 100 | 200 | ...`（N * 100）
- 单卡最多 16 个 qGPU 设备

## 验证

```bash
# 确认组件运行
kubectl get pods -n kube-system | grep qgpu

# 确认 Pod GPU 资源
kubectl get pod qgpu-demo -o yaml | grep qgpu
```

```text
NAME  STATUS  AGE
...
```

## 清理

```bash
kubectl delete pod qgpu-demo
```

## 排障

| 现象 | 根因 | 修复 |
|------|------|------|
| qGPU Pod 无法调度 | 节点未开启 qGPU 共享 | 确认节点 Label: `kubectl get node -l tke.cloud.tencent.com/qgpu-schedule-policy` |
| 独立集群升级后 qGPU 失效 | Master 升级重置了 qgpu-scheduler 配置 | 卸载重装 qGPU 组件 |
| 算力/显存分配不正确 | 精度或取值超出范围 | core 必须为 5 的倍数（5-100），memory 为整数（>= 1） |

## 下一步

- [qGPU 多卡互联](../qGPU 多卡互联/tccli 操作.md)
- [qGPU 离在线混部说明](../qGPU 离在线混部/qGPU 离在线混部说明/tccli 操作.md)
- [GPU 监控指标获取](../../GPU 监控指标获取/tccli 操作.md)

## 控制台替代

[TKE 控制台 → 集群详情 → 组件管理](https://console.cloud.tencent.com/tke2/cluster)：新建 QGPU 组件。节点池创建时开启 qGPU 共享。工作负载设置 GPU 资源。
