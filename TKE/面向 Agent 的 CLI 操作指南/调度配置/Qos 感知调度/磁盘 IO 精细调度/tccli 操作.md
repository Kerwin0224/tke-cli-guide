# 磁盘 IO 精细调度（tccli）

> 对照官方：[磁盘 IO 精细调度](https://cloud.tencent.com/document/product/457/79781) · page_id `79781`

## 概述

磁盘 IO 精细调度（Disk IO Fine-Grained Scheduling）是 QoS 感知调度中的一项特性，通过 Crane PodQOS CRD 的 `diskIOQOS` 字段实现 Pod 级磁盘 IO 限制。支持配置 IOPS（每秒输入输出次数）和 BPS（每秒字节吞吐量）上限，避免低优先级业务抢占磁盘 IO 资源，保障高优业务存储性能。

**工作原理**：QoSAgent 通过 cgroup blkio 子系统对匹配 PodQOS selector 的 Pod 施加磁盘 IO 限制。限制维度包括：读/写 IOPS、读/写 BPS（吞吐量）。策略由 craned 下发，qos-agent 在节点上执行。

**tccli 能力范围**：磁盘 IO 精细调度为 CRD/kubectl 配置型特性，tccli 主要负责检查集群状态和 QoSAgent 组件状态。以下 kubectl 命令为数据面参考操作。

## 前置条件

- [环境准备](../../../环境准备.md)
- 集群中已安装 [QoSAgent](../QoSAgent/tccli%20操作.md) 组件（依赖 craned）
- 存储后端为 CBS（云硬盘），节点内核版本 >= 4.19（blkio cgroup v2 特性依赖）
- 演示集群 `cls-xxxxxxxx`（ap-guangzhou，v1.30.0，Running），kubectl 因 CAM 策略限制不可达，需通过 VPN/IOA 内网或数据面集群操作。以下 kubectl 命令为参考格式。

### 控制面检查

```bash
# 确认集群状态正常
tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]'
```

```json
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "cls-xxxxxxxx",
            "ClusterName": "demo-cluster",
            "ClusterType": "MANAGED_CLUSTER",
            "ClusterStatus": "Running",
            "ClusterVersion": "1.30.0",
            "ClusterNodeNum": 3,
            "ClusterOs": "tlinux3.1",
            "VpcId": "vpc-xxxxxxxx",
            "CreatedTime": "2024-06-15T10:30:00Z"
        }
    ],
    "RequestId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
}
```

```bash
# 确认 QoSAgent 组件已安装且运行正常
tccli tke DescribeAddon --region ap-guangzhou \
    --ClusterId cls-xxxxxxxx \
    --AddonName QoSAgent
```

```json
{
    "Addons": [
        {
            "AddonName": "QoSAgent",
            "AddonVersion": "v1.2.0",
            "Status": "Succeeded",
            "Phase": "Running"
        }
    ],
    "RequestId": "b2c3d4e5-f6a7-8901-bcde-f12345678901"
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 查看集群状态 | `tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]'` | 是 |
| 查看 QoSAgent 组件状态 | `tccli tke DescribeAddon --region ap-guangzhou --ClusterId cls-xxxxxxxx --AddonName QoSAgent` | 是 |
| 创建磁盘 IO 限制策略 | `kubectl apply -f podqos-disk.yaml` | 是 |
| 限制 Pod 写 IOPS | PodQOS `diskIOQOS.writeIOPS` | 是 |
| 限制 Pod 读 IOPS | PodQOS `diskIOQOS.readIOPS` | 是 |
| 限制 Pod 写 BPS | PodQOS `diskIOQOS.writeBPS` | 是 |
| 限制 Pod 读 BPS | PodQOS `diskIOQOS.readBPS` | 是 |
| 验证限制效果 | `kubectl exec` 执行 fio 测速对比 | 是 |

## 操作步骤

### 步骤 1：确认控制面状态

```bash
tccli tke DescribeAddon --region ap-guangzhou \
    --ClusterId cls-xxxxxxxx \
    --AddonName QoSAgent
```

```json
{
    "Addons": [
        {
            "AddonName": "QoSAgent",
            "AddonVersion": "v1.2.0",
            "Status": "Succeeded",
            "Phase": "Running"
        }
    ],
    "RequestId": "c3d4e5f6-a7b8-9012-cdef-123456789012"
}
```

### 步骤 2：创建磁盘 IO 限制策略（需 kubectl 可达环境）

`podqos-disk.yaml`：

```yaml
apiVersion: scheduling.crane.io/v1alpha1
kind: PodQOS
metadata:
  name: disk-io-limit
spec:
  resourceQOS:
    diskIOQOS:
      writeIOPS: 1000
      readIOPS: 500
      writeBPS: 104857600
      readBPS: 52428800
  selector:
    matchLabels:
      disk-io: limited
```

```bash
# 需 kubectl 可达环境
kubectl apply -f podqos-disk.yaml
# expected: podqos.scheduling.crane.io/disk-io-limit created
```

### 步骤 3：创建受限制测试 Pod（需 kubectl 可达环境）

`disk-io-pod.yaml`：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: disk-io-test
  labels:
    disk-io: limited
spec:
  containers:
  - name: app
    image: alpine:latest
    command: ["sh", "-c", "apk add fio && fio --name=test --rw=randwrite --size=100M --direct=1 --bs=4k --runtime=30s"]
```

```bash
# 需 kubectl 可达环境
kubectl apply -f disk-io-pod.yaml
# expected: pod/disk-io-test created
```

### diskIOQOS 字段说明

| 字段 | 类型 | 单位 | 说明 |
|------|------|------|------|
| `writeIOPS` | int | 次/秒 | 写 IOPS 上限 |
| `readIOPS` | int | 次/秒 | 读 IOPS 上限 |
| `writeBPS` | int | 字节/秒 | 写吞吐量上限（104857600 = 100MB/s） |
| `readBPS` | int | 字节/秒 | 读吞吐量上限（52428800 = 50MB/s） |

### 工作原理

1. **策略定义**：通过 PodQOS CRD 定义 `diskIOQOS` 字段，指定 IO 限制参数。
2. **Pod 匹配**：Pod 的 label 与 PodQOS `selector.matchLabels` 匹配时，QoSAgent 对该 Pod 施加限制。
3. **cgroup 执行**：QoSAgent 将磁盘 IO 限制写入 Pod 对应 cgroup 的 blkio 子系统（`blkio.throttle.read_bps_device` 等）。
4. **实时生效**：策略变更后 QoSAgent 实时更新 cgroup 配置。

## 验证

### 控制面

```bash
tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]'
```

```json
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "cls-xxxxxxxx",
            "ClusterName": "demo-cluster",
            "ClusterStatus": "Running",
            "ClusterVersion": "1.30.0",
            "ClusterNodeNum": 3
        }
    ],
    "RequestId": "d4e5f6a7-b8c9-0123-def0-234567890123"
}
```

### 数据面（需 kubectl 可达环境）

```bash
# 确认 PodQOS 策略已创建
kubectl get podqos disk-io-limit -o yaml | grep -A5 diskIOQOS
# expected: 包含 writeIOPS、readIOPS、writeBPS、readBPS 字段

# 确认 Pod 已匹配策略
kubectl get pod -l disk-io=limited
# expected: disk-io-test   1/1   Running

# 在 Pod 内执行 fio 测速验证限制生效
kubectl exec disk-io-test -- fio --name=verify --rw=randwrite --size=50M --direct=1 --bs=4k --runtime=10s --output-format=json | jq '.jobs[0].write.iops'
# expected: IOPS 不超过 writeIOPS 限制值（1000）
```

```text
NAME  STATUS  AGE
...
```

## 清理

### 数据面（需 kubectl 可达环境）

```bash
# 删除测试 Pod
kubectl delete pod disk-io-test
# expected: pod "disk-io-test" deleted

# 删除磁盘 IO 限制策略
kubectl delete podqos disk-io-limit
# expected: podqos.scheduling.crane.io "disk-io-limit" deleted
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl apply -f podqos-disk.yaml` 返回 "no matches for kind PodQOS" | `tccli tke DescribeAddon --region ap-guangzhou --ClusterId cls-xxxxxxxx --AddonName QoSAgent` | QoSAgent 未安装，PodQOS CRD 未注册 | 先参见 [QoSAgent](../QoSAgent/tccli%20操作.md) 安装组件 |

### IO 限制不生效

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 策略已创建但 Pod IO 未被限制 | 节点内核 `uname -r` 检查版本 | 内核版本 < 4.19，blkio cgroup v2 特性不支持 | 升级节点内核至 4.19+ |
| 限制值设置过大未观察到限速效果 | 使用 fio 产生足够 IO 压力重测 | 测试负载未达到限制阈值 | 使用 `--size=1G --bs=1M` 等参数提高 IO 压力 |
| 部分节点 IO 限制未生效 | `kubectl get pods -n crane-system -l app=qos-agent -o wide` | 部分节点 qos-agent 未运行 | 排查节点污点和 DaemonSet 调度状态 |

## 下一步

- [内存精细调度](../内存精细调度/tccli%20操作.md) — 通过 PodQOS 实现内存水位控制
- [网络精细调度](../网络精细调度/tccli%20操作.md) — 通过 CNRP CRD 实现网络带宽限制
- [QoSAgent 组件介绍](../组件介绍/QoSAgent/tccli%20操作.md) — 了解 QoSAgent 架构与工作原理

## 控制台替代

无独立控制台界面。磁盘 IO 精细调度通过 CRD 配置，在集群中 [安装 QoSAgent](https://console.cloud.tencent.com/tke2/cluster) 后，通过 kubectl 管理 PodQOS 资源。
