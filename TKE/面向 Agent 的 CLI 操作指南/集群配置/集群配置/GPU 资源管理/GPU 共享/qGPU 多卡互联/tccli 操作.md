# qGPU 多卡互联

> 对照官方：[qGPU 多卡互联](https://cloud.tencent.com/document/product/457/127667) · page_id `127667`

## 概述

qGPU 多卡互联功能解决单张共享 GPU 卡算力或显存不足的问题。在 Pod 声明中设置超过 100 的算力值（`qgpu-core`），系统自动为容器分配多张物理 GPU 卡的切片资源，并保证资源在多卡间均分。适用大模型推理、分布式训练等场景。

## 前置条件

- qGPU 组件 >= v1.1.1
- PoC 版本最大支持 2 张卡互联
- kubectl 可达集群以 apply YAML

> **kubectl 可达性说明**：当前共享集群 kubectl 不可达，且无 GPU 节点。以下 YAML 完整，但无法真跑验证。

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 升级 qGPU 组件 | `tccli tke InstallAddon --AddonName qgpu --AddonVersion <new-version>` | 是 |
| 创建工作负载（多卡） | `kubectl apply -f <yaml>` | 是 |
| 验证 qGPU 模块版本 | 登录节点: `cat /proc/qgpu/version` | 是 |

## 操作步骤

### 工作原理

当 `qgpu-core` 超过 100 时，插件自动均分：

| 填写的总配置 | 实际分配结果 | 说明 |
|---|---|---|
| core=160, memory=60 | 2 张卡，每张 core=80, memory=30 | 均分 160/2=80, 60/2=30 |
| core=121, memory=31 | 2 张卡，每张 core=60, memory=15 | 向下取整：121/2=60.5→60, 31/2=15.5→15 |

### 创建多卡 qGPU Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: qgpu-multi-card-demo
spec:
  containers:
  - name: test-container
    image: nvidia/cuda:11.0-base
    resources:
      limits:
        tke.cloud.tencent.com/qgpu-core: "160"   # 申请 160 算力 → 自动分配 2 张卡
        tke.cloud.tencent.com/qgpu-memory: "60"   # 申请 60GB 显存 → 每张卡 30GB
```

```bash
kubectl apply -f qgpu-multi-card.yaml
# expected: pod/qgpu-multi-card-demo created
```

### 升级存量节点 qGPU 模块

1. 控制台升级 qGPU 组件到最新版本
2. 封锁节点，迁移业务 Pod
3. 登录节点卸载旧模块：

```bash
rmmod qgpu
```

4. 删除节点上 `qgpu-manager` Pod，自动重建并安装新模块
5. 验证版本：

```bash
cat /proc/qgpu/version
# expected: 版本 >= 3.0.5
```

6. 解封节点

## 验证

```bash
kubectl get pod qgpu-multi-card-demo -o yaml | grep qgpu-core
# expected: "160"
```

```text
NAME  STATUS  AGE
...
```

## 清理

```bash
kubectl delete pod qgpu-multi-card-demo
```

## 排障

| 现象 | 根因 | 修复 |
|------|------|------|
| 多卡 Pod 无法调度 | 节点 GPU 卡数不足 | 确保节点至少有 2 张 GPU 卡 |
| 多卡 Pod 实际算力与预期不符 | 向下取整规则 | 调整 `qgpu-core` 为所需卡数的整数倍 |
| 升级后多卡不生效 | 节点 qGPU 内核模块未更新 | 手动 `rmmod qgpu`，重建 qgpu-manager Pod |

## 下一步

- [使用 qGPU](../使用 qGPU/tccli 操作.md)
- [qGPU 离在线混部说明](../qGPU 离在线混部/qGPU 离在线混部说明/tccli 操作.md)

## 控制台替代

[TKE 控制台 → 工作负载](https://console.cloud.tencent.com/tke2/cluster)：新建 Deployment/Job，设置 `qgpu-core` > 100 即可触发多卡互联。
